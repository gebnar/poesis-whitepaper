# The Poesis Kernel — A Whitepaper for PSL Programmers

© 2026 Shane Plesner. All rights reserved.
This document may be read here. It may not be copied, modified,
redistributed, or implemented without prior written permission.

> **Audience:** people writing Poesis Scripting Language (PSL) behaviors and modules.  
> **Goal:** describe the *shape* of the Zig kernel — what it does, what guarantees it provides, and what it expects from your code — without descending into implementation minutiae.  
> **Companion documents:** `api_reference.md` for exact API signatures, `behavior_programming_guide.md` for idioms, and §17 below for the PSL language reference.

---

## 1. What the kernel is

The Poesis kernel is the runtime that executes PSL.  It is written in Zig, but from the PSL programmer's point of view it is an *engine* with four jobs:

1. **Load and compile** PSL source into executable bytecode.
2. **Host** behaviors inside modules and modules inside contexts.
3. **Schedule** execution so that many behaviors make progress at once.
4. **Coordinate** communication between behaviors through signals, scratch, territory, and shared snapshots.

A PSL program is not a single thread of control.  It is a collection of independent behaviors that run concurrently inside a module, observe each other's signals, and keep running until the module converges.

---

## 2. Contexts, modules, behaviors, and cycles

These four nouns are the layers of the runtime.

### 2.1 Context

A **context** is the largest unit of runtime life.  It owns:

- a set of modules,
- a shared **territory** (see §4),
- a sandbox root for filesystem access,
- its own lifecycle state.

Contexts are isolated from one another by default.  Isolation is the default, not a wall: a behavior that holds another context's handle can coordinate across the boundary (§4.4).  A behavior can spawn a new context with `membrane.context.new`, and other code can inspect or terminate it through `territory.contexts.*`.  The CLI entry point (`zig build run`) creates one context, loads `behaviors/main.poesis`, and runs it to completion.

**Territory inheritance:** child and sibling modules automatically inherit their spawner's territory.  A module spawned by `membrane.module.run` or `membrane.module.run.async` receives the parent's territory handle, so it can read and write the parent's territory immediately — no explicit configuration needed.

### 2.2 Module

A **module** is a running group of behaviors.  It is the unit of *convergence*: the module runs until it is done, at which point its visible signals become the module's output.  A module can be:

- **top-level** (the module that owns the process),
- **nested** (spawned by `membrane.module.run` from inside a behavior),
- **asynchronous** (spawned by `membrane.module.run.async` and polled later).

Every module has its own signal space, its own behavior set, a cycle counter, and a status: `running`, `converged`, `killed`, `panicked`, or `starved`.

Two spawn-time config keys shape a module's lifetime:

- `config = {max_cycles: N}` — a hard cap on cycles.  Hitting it ends the module `killed`, or `starved` when `produces` declarations are outstanding (§3.1).
- `config = {child: true}` — lifecycle-binds the module to its spawner: `converge` and `kill` on the parent propagate recursively down the module tree.

A context can also be created with a **permission set** that restricts which API namespaces the behaviors inside it can access (§13).  A context without a permission set has full access — this is the default, and the common case.

### 2.3 Behavior

A **behavior** is a single compiled PSL program running inside a module. Behaviors are the atoms of Poesis concurrency: many behaviors may share a module, and the kernel runs them concurrently. A behavior file (`behaviors/foo.poesis`) is compiled once and cached in the global **behavior pool**; each module gets its own frame from that bytecode.

A behavior starts every cycle at instruction zero with no committed locals. Behaviors are the primary unit of computation. All functions (whether `fn` declarations or `membrane.func` registered) operate as descendants of a behavior, and have access to the same territory/context api as their ancestral behavior.

**One-Shot Tasks:** the cycle/signal apparatus is fully optional. A behavior that simply runs its program and ends is perfectly valid and even common. It does work, produces a result (in signals via `converge` or in territory), and is done. Not every behavior needs to be a multi-cycle reactive agent. The drain loop will still run it, commit its signals (if any), and detect convergence naturally (if not forced). If you want a one-shot behavior to leave signals behind, make sure to call `converge` or the dirty signal space will cause it to run another cycle. Usually one-shot behaviors run in a module by themselves, but whatever arrangement suits your needs is fine.

**Long-running Tasks:** a behavior that runs a single loop and never returns for the life of the program (or as long as needed) is also valid. The compiler-inserted **safepoints** (§5.1) ensure that such a behavior never completely monopolizes a worker. As with one-shot behaviors, these usually live alone in a module of their own. Especially because they prevent the module from ever cycling, so behaviors that depend on cycling are not compatible module-mates. Long-running tasks like this should avoid using signals as they will never actually be committed at the cycle boundary (which is never crossed). If you want such a task to always stay running and not share it's worker thread, set its module to high priority (but use with care - high-priority modules can starve the standard queue if there are too many of them, at which point they will start interleaving anyway and you lose the entire benefit).

The scheduler does not care whether a behavior is a one-shot task, a multi-cycle agent, or a continuous loop — it schedules and budgets them the same way.

### 2.4 Cycle

A **cycle** is one full pass through the module's active behaviors.  During a cycle the kernel:

1. Schedules every ready behavior to run until it yields, returns, suspends, or is quarantined.
2. Commits all staged signal writes at the cycle boundary.
3. Decides whether the module has converged, starved, or should run another cycle.

Cycles are the heartbeat of a module.  A behavior that needs to wait for an upstream signal does not block the module; it simply returns early and tries again on the next cycle.

---

## 3. The drain loop

The **drain loop** is the module's execution engine.  Its contract is simple:

> Run all active behaviors in parallel each cycle.  Stop when the module is done.

A module finishes when any of these conditions is true:

1. **Natural convergence** — a full cycle completes with no change to the signal space.  This is the normal, preferred exit.
2. **Explicit convergence** — a behavior calls `converge`, which forces the module to shut down immediately after committing that behavior's staged signals.
3. **Starvation** — a behavior declares `produces` keys and the module ends without them being produced.
4. **Kill** — an external caller (`territory.modules.kill`, `territory.contexts.kill`, or process exit) terminates the module.
5. **Panic / quarantine cascade** — a behavior violates an invariant and is quarantined; the module itself may be marked `panicked` depending on context.

The drain loop is *not* a deterministic step sequencer.  Behaviors run concurrently within a cycle, and their writes are not visible to other behaviors until the cycle boundary.  This means:

- You cannot assume behavior A runs before behavior B.
- A `signal.set` in cycle N is visible as a `signal.get` in cycle N+1.
- When waiting for external data, the guard-and-return pattern is useful for signal coordination, while `yield until` is useful for intra-cycle suspension. Don't `yield until` against a condition whose producer is waiting on your signal — that's a deadlock.

This last point is the central design rule of Poesis: behaviors are cooperative, cycle-driven agents, not sequential routines.

### 3.1 The `produces` contract and starvation

A behavior can declare the signal keys it owes the module:

```poesis
produces output, findings
```

The declaration is a contract enforced at runtime.  Every `signal.set` to a declared key records production on the behavior frame.  When the module ends with declarations outstanding, its status is `starved` instead of `converged`, and the snapshot's `incomplete` key reports what was owed:

```poesis
state = territory.modules.get(handle)
# state.status == "starved"
# state.incomplete == [{behavior: "worker", key: "output", cause: "quiesced"}, ...]
```

`cause` is `quarantined` when the behavior faulted before producing and `quiesced` when it simply never got there.  Production is remembered even if a consumer later frees the signal — the contract is about having produced, not about the value still being present.  `starved` takes priority over `panicked` and `killed`: a module that hits `max_cycles` with declarations outstanding is reported starved, not killed.

---

## 4. Storage layers: locals, scratch, signals, territory

Values in Poesis live in one of four layers.  The rule is:

> **Use the leftmost layer that still expresses the algorithm.**

```
locals  →  scratch  →  signals  →  territory
```

Every step to the right buys reach and charges for it — in visibility you must reason about, in lifetime you must manage, in convergence pressure, or in contention.  Moving right is never free, and moving right *unnecessarily* is the most common category error in Poesis code.

The **territory API** is the surface for inspecting and controlling things outside the current behavior or module, but still within the same context.  Through `territory.*` any behavior can read live product data, inspect module snapshots, and invoke module/context lifecycle operations.  Signals, modules, contexts, and the filesystem are all reachable through territory, but they obey very different read/write rules.

| Layer | Mechanism | Reach | Lifetime | Dirties signals? |
|-------|-----------|-------|----------|------------------|
| **Locals** | `x = 1` | This behavior, this cycle | Cycle | No |
| **Scratch** | `membrane.scratch.*` | This behavior, every cycle | Behavior | No |
| **Signals** | `membrane.signal.*` | Every behavior in the module | Module, until freed | **Yes** |
| **Territory** | `territory.product.*` | Every module in the context, **or cross-context via handle** | Context, until freed | No |

### 4.1 Locals — the default

Ordinary variables.  Because the behavior restarts at instruction zero each cycle, every local is reassigned before it is read in a well-formed program.

**Scope:**

- **Read:** only the owning behavior, during the current cycle.
- **Write:** only the owning behavior, during the current cycle.
- **Lifetime:** one cycle. Locals are discarded at each cycle boundary when the behavior returns.

*Go right when:* the value must outlive the cycle.

### 4.2 Scratch — private memory across cycles

`membrane.scratch.*` is behavior-local, invisible to every other behavior, and it does not dirty the signal space. This is where a behavior keeps its own bookkeeping: a retry count, a cached parse, an async handle it is polling, a partial accumulation.

**Scope:**

- **Read:** only the owning behavior, across all its cycles.
- **Write:** only the owning behavior, across all its cycles.
- **Lifetime:** the behavior's lifetime. Scratch dies when the behavior is removed or quarantined, or when the module converges or is killed.

```poesis
membrane.scratch.set("attempts", 0)
attempts = membrane.scratch.get("attempts")  # returns null if never set
```

**The tell that you should have used scratch:** you wrote a hidden signal that nothing reads.  `hidden = true` hides a signal from module output and snapshots — it does not make it private, and it does not make it free.  A hidden signal still stages, still commits at the boundary, and still dirties the space.  If no other behavior calls `signal.get` on it, you paid the coordination cost for a value nobody coordinates on.

*Go right when:* another behavior must read it.

### 4.3 Signals — coordination inside a module

Signals are the safe data publication mechanism for behaviors. Each behavior has its own namespace, identified by the behavior's name.

```poesis
membrane.signal.set("output", "There is no spoon.")            # writes to self namespace
membrane.signal.set("internal", scratch_state, hidden = true)  # hidden signal
val, err = membrane.signal.get("other", "output")              # reads other namespace
has = membrane.signal.has("other", "output")                   # probes other namespace
membrane.signal.free("other", "output")                        # gone next cycle
```

**Scope:**

- **Write:** a behavior can only write to **its own** namespace (`membrane.self.id()`).  You cannot directly write another behavior's signals.
- **Read:** any behavior in the same module can read any other behavior's namespace via `get` and `has`. Behaviors in other modules can read via territory.modules.
- **Free:** any behavior in the same module can free any key in any namespace.
- **Lifetime:** module lifetime, until freed or overwritten.
- **Visibility boundary:** writes staged during cycle N are committed and become visible at the end of cycle N; reads in cycle N+1 see them.
- **Staged reads:** no behavior can read staged signals — not even the behavior that wrote them.  `membrane.signal.get` and `has` see only the *committed* space from the previous cycle boundary.  A behavior cannot read its own in-flight output in the same cycle.

The signal API:

- `has(behavior, key)` is a lenient probe.  Unknown namespaces return `false` and never harm the prober.
- `get(behavior, key)` is an inquiry returning `(val, err)`.  A miss is `(null, err)` with `err.kind` either `missing_key` (temporal — busy-guard and retry) or `unknown_namespace` (structural — you are wrong about your module).  It never quarantines.
- `territory.modules.get(handle)` gives access to the latest cycle snapshot from a foreign module. This is the exact same data at the same staleness as available locally within the module.
- `free(behavior, key)` is lenient cleanup. Multiple behaviors may free the same signal in one cycle; it is removed at the cycle boundary. Any references to the removed signal remain valid until they are dropped, so use-after-free is not a concern.

Signals are the only layer that drives convergence.  Every `set` is a claim on the module's progress: a new value keeps the module alive another cycle; the same value is idempotent and does not.  A signal is not storage that other behaviors happen to see — it is a coordination event, and the module's termination is computed from it.

- All value types are valid in signals.
- Using booleans, strings like `"done"`, or numbers like `1` as flags is a strong antipattern. Signals should communicate state through data where possible.
- Setting a signal to the **same value it already has** does not dirty the signal space.

**The tell that you should have used scratch:** nothing reads the signal.

*Go right when:* you need to race with other participants to set or clear data.

### 4.4 Territory — the contended arena

Territory is the API surface for state outside the current module.  It covers four faces:

- `territory.product.*` — live, shared key/value storage.
- `territory.modules.*` and `territory.contexts.*` — inspect and control modules and contexts by handle.
- `territory.fs.*` — sandboxed filesystem access (`fs.read`, `fs.write`, `fs.append`).

```poesis
territory.product.set("outputs", "final", result)              # last writer wins
created = territory.product.give("outputs", "final", result)   # first writer wins; false if taken
val, err = territory.product.take("outputs", "final")          # first reader wins
val, err = territory.product.get("outputs", "final")

state = territory.modules.get(handle)       # read-only snapshot, one boundary stale
territory.modules.converge(handle)          # graceful shutdown
territory.modules.kill(handle)              # immediate shutdown
```

**Scope:**

- **Read:** any behavior may get. Only local behaviors may take (an atomic get+free).
- **Write:** any behavior may give (an atomic set only if missing). Only local behaviors may set.
- **Control:** `modules.converge`, `modules.kill`, `modules.free`, and their context equivalents are available to any caller with a handle.
- **Lifetime:** context lifetime, until deleted.

**Territory is where you go when competing *is* the algorithm** — when several producers race for one slot and which one arrives first or last is the answer you want.  For ordinary cross-module reading, a module publishes to its own signal space and foreign readers pull the committed snapshot.

**Discovery.**  Cross-context coordination starts with a handle. Handles are usually acquired by spawners, but can be discovered by anyone. `territory.contexts.list()` returns the whole active set. `territory.contexts.blind_walk(mask)` returns one context handle not in the mask, drawing randomly from the unmasked set — useful for emergent, randomized exploration.

---

### 4.5 Cross-module reading

Signals are scoped to a module, but a behavior that needs another module's output can read its committed signal space via `territory.modules.get(handle)`.  This is a **snapshot** — one boundary stale, read-only, and safe.  The snapshot is not a separate mechanism; it is the signal space of the target module, accessed through territory instead of through the module's own namespace.

The one layer that is genuinely *live* and *contended* is `territory.product.*`.  Everything else freezes at the cycle boundary.

---

## 5. The scheduler: parallel, non-blocking, and intentionally unordered

The kernel schedules behavior execution with a global worker pool.  The PSL programmer should internalize three properties:

1. **Behaviors run in parallel.**  Multiple behaviors in the same module can be on worker threads at the same time.
2. **Execution is non-blocking.**  A behavior that waits for a signal simply returns; it does not hold a worker thread.
3. **Order is not guaranteed.**  The kernel does not promise that behavior A runs before behavior B, either within a cycle or across cycles.

### 5.1 Segments and yields

A behavior runs in **segments**: chunks of execution between yield points.  A segment ends when the behavior:

- returns,
- calls `yield` (cooperative handoff within the same cycle),
- calls `yield until` on a condition that can trigger mid-cycle (such as LLM status or territory state),
- exhausts its reduction budget and hits a **safepoint**,
- suspends on a long-running api call.

**Compiler-inserted safepoints:** the compiler automatically emits `safepoint` instructions at loop headers and after potentially expensive operations.  A safepoint checks the behavior's reduction budget and suspends the behavior if the budget is exhausted.  The budget is configurable via `pragma reductions_budget` (default 1000).  This guarantees that no single behavior can monopolize a thread — the budget forces a yield and reschedule, giving other behaviors a turn.  The safepoint mechanism is invisible to the programmer: you write loops normally, and the compiler ensures they are fair.

**Collection loop safepoints:** the built-in collection iteration methods (`map`, `filter`, `reduce`, `find`, `any`, `all`) desugar into for-loops that include periodic safepoints (every 20 iterations) rather than per-element yields.  The fn body's own safepoints handle API calls (territory, membrane, etc.), so per-element yielding is pure overhead for simple callbacks.  Custom for-loops over collections also receive compiler-inserted safepoints at the loop header, just like any other loop.

**Automatic suspension:** any operation that touches the outside world — LLM queries, file I/O, network calls, `time.sleep`, HTTP requests, process execution, and all other big API calls — get automatic safepoints or callbacks. The behavior is suspended, the thread goes back to the worker pool, and the behavior resumes when the operation completes (typically on the next cycle). A PSL program never blocks a thread. This is enforced by the VM, not by programmer discipline.

**Yields are legal inside function calls.**  The VM keeps a per-behavior frame stack, so a `yield` or `yield until` inside a `fn` suspends the entire call chain and resumes at the same instruction on the next segment — nested calls survive suspension intact.

When a segment ends, the behavior is requeued.  The next cycle boundary commits any signal writes the segment staged.

### 5.2 How work gets to workers

Work is distributed **lock-free across all thread-workers**.  The scheduler claims behaviors for workers by scanning a shared bitmap — no mutex, no queue contention, no serialization.  Workers never stall waiting for each other.

Modules can be configured with `priority: "high"` or `"standard"`.  The scheduler prefers high-priority work, but within a priority level the order is unspecified.  This is intentional: PSL programs must be correct under any interleaving.

Work is delivered in **budgeted slice-batches**: each behavior gets a bounded slice of execution time before it is preempted and the next behavior gets a turn.  This ensures fairness (no single behavior can starve others) and cache locality (adjacent behaviors in the same module share working set).

The programmer-visible guarantee remains simply *parallel, non-blocking, and unordered*.

---

## 6. From PSL source to running behavior

PSL is compiled ahead of execution.  The pipeline has four conceptual stages:

| Stage | What it does |
|-------|--------------|
| **Lexer** | Turns source text into tokens (numbers, strings, identifiers, keywords, operators). |
| **Preprocessor** | Applies `pragma` directives, collects `produces` declarations, and builds compile settings. |
| **Parser** | Builds an abstract syntax tree (AST) from tokens. |
| **Compiler** | Walks the AST and emits bytecode instructions.  Processes escape sequences in string literals and produces final string constants.  Collects all compilation errors across the full source rather than stopping at the first one — you get every problem at once. |

The bytecode is a stack-machine instruction set: constants are pushed, operations pop their operands and push results, and control-flow instructions jump to instruction offsets.  The kernel's virtual machine (VM) executes these instructions inside a behavior frame.

For the PSL programmer this is mostly transparent: you write PSL, the kernel compiles and runs it.  The relevant detail is that compilation errors are caught before the module starts, and bytecode is cached in the behavior pool so that the same behavior file is not recompiled for every module.

The preprocessor accepts three numeric `pragma` directives.  A `pragma` must appear before any code in the file:

| Pragma | Default | Effect |
|--------|---------|--------|
| `pragma number_precision N` | 38 | Decimal digits of precision for the `number` lane (1–10,000). |
| `pragma fn_budget N` | 1000 | Hard cap on total function calls per cycle; exhausting it quarantines (§10). |
| `pragma reductions_budget N` | 1000 | Segment length before a safepoint suspends the behavior (§5.1). |

A function file without an explicit `pragma number_precision` inherits the caller's decimal precision; one that sets its own overrides it.

---

## 7. Persistent, stable data structures

The kernel stores PSL's arrays, dicts, sets, and strings in **persistent** (immutable, structural-sharing) data structures.  This matters to the PSL programmer in three ways:

1. **Copying is cheap.**  Because values share unmodified structure, passing a large array or dict to a function does not copy the whole collection.
2. **Snapshots are safe.**  A module snapshot can hold a committed view of the signal space while the module continues to mutate its staged state.
3. **There are no cycles.**  Values are acyclic by construction, so reference counting is sufficient and the runtime never needs a tracing garbage collector.

The three core structures are:

- **Persistent array** — a 32-way bitmapped vector trie.  Indexing, appending, and setting produce new versions that share most of their nodes with the old version.
- **HAMT (Hash Array Mapped Trie)** — used for dicts and set backing.  Provides persistent `put`, `remove`, and iteration.
- **Ropes** — used for strings.  Small strings are stored inline; larger strings are concatenation trees, which makes interpolation and repeated concatenation efficient.

From PSL these all look like ordinary mutable collections — `arr.append(5)`, `d["key"] = value` — but underneath the kernel produces a new version and releases the old one when it is no longer referenced.

---

## 8. The value and type system

PSL has a small, explicit type system.  Every value in PSL is one of these types:

| Type | What it is | Example |
|------|-----------|---------|
| `number` | Decimal with configurable precision (default 38 digits) | `x = 3.14` |
| `fastnumber` | int64/float64 dual type, normalize-on-write | `n = f42` |
| `string` | Immutable text (UTF-8 codepoints) | `s = "hello"` |
| `bool` | Boolean | `flag = true` |
| `null` | Explicit absence of value | `val = null` |
| `array` | Ordered, zero-indexed collection | `arr = [1, 2, 3]` |
| `dict` | Key-value map (string keys) | `d = {a: 1, b: 2}` |
| `set` | Unordered collection of unique values | `s = set(1, 2, 3)` |
| `fn` | First-class function value with closures | `fn add { require a, b; return a + b }` |
| `error` | Machine-checkable failure with a `kind` and human `message` | `e = error("not_found", "key missing")` |
| `regex` | Compiled regular expression, refcounted | `r = regex("\\d+")` |
| `dynamic` | Dict-like with auto-vivification and null-proxy | `d = dynamic()` or `d = dynamic(dict_val)` |
| `shape` | Structural predicate for dicts — validates, fills defaults, strips extras | `s = shape({x: number, y: number})` |

**The `number` / `fastnumber` wall:** these two numeric types are deliberately separated.  Arithmetic across the wall requires an explicit cast (`n::number`).  Comparisons and membership work across the wall automatically — `f0.1 == 0.1` is true.  This prevents the subtle bugs that arise from mixing binary floats with decimals.

**`null`:** `null` represents the explicit absence of a value.  Accessing a missing key on a `dict` returns `null` or faults depending on the operation.  `null` is truthy-false and is the standard sentinel for "not yet set."

**`dynamic`:** a dict-like type with auto-vivification: dot-notation access (`d.key`), null-proxy on missing keys, and automatic intermediate dict creation on assignment (`d.y.z = 2` auto-creates `d["y"] = {z: 2}`).  Created with the `dynamic()` built-in: `dynamic()` for an empty dynamic, or `dynamic(dict_val)` for an O(1) re-tag from an existing dict.  Dict-typed values accessed from a dynamic are returned as **lenses** — transparent on read, but writes propagate COW through the lens back to the root dynamic.  Supports iteration, `len`, `in`, and field-set just like a dict.

**`error` as a first-class value:** errors are values, not exceptions.  A fallible function returns a `(val, error)` pair.  The error carries a machine-checkable `kind` string and a human-readable `message`.  Errors are truthy, comparable, and can be stored in signals and territory.  The `or` operator chains fallible calls and handles errors explicitly — there is no hidden control flow.

**`fn` and closures:** first-class function values capture variables from their enclosing scope via `capture`.  A captured variable is deep-copied at closure creation time, so the `fn` is self-contained.  Functions recurse with `self(args)` or by name.

**`shape`:** a structural predicate that describes the expected structure of a dict.  Shapes are values — they pass through signals, territory, and scratch like any other value.  `shape.validate(s, d)` returns a normalized dict (defaults filled, extra fields stripped) or an error.  `require a: {x: number}` validates at the function boundary and quarantines on mismatch.  Shapes have no name, no registry, and no identity — two shapes with identical fields are interchangeable.  Composition functions (`shape.merge`, `shape.extend`, `shape.pick`, etc.) build complex shapes from simple ones.  See §17 for the full reference.

**The type function:** `type(value)` returns the type name as a string: `"number"`, `"string"`, `"dict"`, etc.  Type checking is done with equality, not with keywords.

The type system is intentionally small: no classes, no inheritance, no implicit conversions.  Types are concrete and inspectable.  This makes the runtime predictable and the error messages clear.

**Immutability by default:** PSL values are immutable.  Assignments produce new values; they never mutate in place.  Under the hood, the kernel uses persistent data structures (HAMTs, ropes, persistent arrays) to make this practical — structural sharing means that most operations share most of their nodes with the previous version, so copying is cheap and snapshots are safe.  The PSL programmer gets the benefits of immutability (no hidden mutation, safe concurrent reads, predictable snapshots) without paying the cost of full copies.

**Strings and escape processing:** string literals are double-quoted with `${...}` interpolation for embedding expressions.  Escape sequences (`\\`, `\n`, `\t`, `\"`, `\r`, `\0`, `\u{XXXX}`) are processed at compile time by the compiler.  Unknown escapes pass through unchanged, so regex patterns in strings (`"\\d+"`) work without special treatment.  The `membrane.lib.string.*` library provides both regex-based operations (`find`, `match`, `replace`, `split`) and literal operations (`slice`, `contains`, `index_of`, `repeat`, `reverse`, `char_at`, `upper`, `lower`, `left_pad`, `right_pad`, `count`, `words`, `lines`, `starts_with`, `ends_with`).

PSL is designed to feel familiar while enforcing the Poesis execution model.  It borrows a lot of syntax from Python:

```poesis
# Variables and arithmetic
x = 10
y = x * 2 + 1

# Conditionals and loops
if y > 20 {
    membrane.signal.set("big", true)
}

for i in range(5) {
    membrane.signal.set("item_${i}", i)
}

# First-class functions
fn double { require n; return n * 2 }
result = double(7)
```

The language is intentionally small: no classes, no inheritance, no mutable global state inside a behavior, and no hidden control flow.  The ergonomics come from:

- Interpolated strings (`"value: ${x}"`).
- Dict and array literals with dot and bracket access.
- Named arguments for built-ins and registry functions.
- First-class `fn` values with closures.
- A clear ladder of storage layers (locals → scratch → signals → territory).

The restrictions are also ergonomic in disguise: they force code to be explicit about where state lives and how behaviors coordinate.

---

## 9. Explicit error handling with `or`-chaining

Poesis distinguishes two kinds of failure:

- **Inquiry failures** are expected runtime questions with expected negative answers.  Examples: a missing signal, a failed parse, an LLM timeout, a `take` on an empty key.
- **Faults** are contract violations: division by zero, out-of-bounds indexing, type mismatches, or runtime invariants being broken.

Inquiry failures are **first-class error values**.  A fallible call returns a pair:

```poesis
val, err = membrane.signal.get("producer", "data")
```

The error channel is mandatory to *receive* — you cannot write `val = f()` for a fallible call — but you are not forced to *act* on it.  Discarding the error with `_` is a valid and common response when the caller already knows what to do with the value or doesn't care why it failed:

```poesis
val, _ = membrane.signal.get("producer", "data")
if (val) { process(val) }
```

The compiler enforces that you acknowledge the error was there.  What you do with it is up to you.  The `or` operator chains fallible calls from left to right, returning the first success:

```poesis
val, err = primary() or fallback() or default_value
```

`or` also accepts halt operands (`or return`, `or converge`, `or break`, `or continue`) so that failure can short-circuit cleanly.

Faults, by contrast, are not values.  They quarantine the behavior and record the failure in the module log.  The module may continue with its remaining behaviors, but the faulting behavior is removed immediately.

---

## 10. Quarantine

**Quarantine** is the kernel's response to a fault.  When a behavior:

- divides by zero,
- indexes out of bounds,
- makes an invalid cast,
- exhausts its function-call budget,
- or violates another runtime invariant,

the behavior is stopped immediately, its partial signal writes for that cycle are discarded, and it is excluded from all future cycles.  The module records the reason and continues with the behaviors that remain.

Quarantine is the terminal channel for the fault class only.  A missing signal, a parse failure, or an LLM error is an *inquiry* and is handled through the `(val, err)` mechanism, not quarantine.  This distinction keeps normal control flow explicit and reserves quarantine for "this behavior can no longer be trusted."

---

## 11. Numbers: the explicit `number` / `fastnumber` divide

PSL has two numeric lanes:

- **`number`** — a decimal type with 38 digits of precision by default (configurable via `pragma number_precision`, 1–10,000).  Use this when exactness matters: money, precise ratios, decimal comparisons.
- **`fastnumber`** — an `int64`/`float64` dual type with normalize-on-write.  Use this for counters, indices, loop variables, and performance-sensitive arithmetic.

The two lanes are deliberately separated by a "wall": arithmetic across the wall requires an explicit cast.  This prevents the subtle bugs that arise from mixing binary floats with decimals.

```poesis
# fastnumber literal
n = f42

# explicit cast to decimal
precise = n::number + 0.1
```

Comparisons and membership tests work across the wall automatically.  `f0.1 == 0.1` is true because fastnumber promotes to decimal via shortest representation.

Division behavior also differs by lane:

- `number / number` preserves decimal precision.
- `fastnumber / fastnumber` is always float division; an integral result demotes back to int.
- Division by zero in either lane is a fault and quarantines the behavior.

### 11.1 The pure-Zig decimal engine

Under the hood, `number` is implemented by a pure-Zig decimal engine. This zig-native engine was built using libmpdec for reference, and extensive testing has been conducted to ensure the zig computations remain exact to the libmpdec reference (within the allowed 10k digit range, FNT/NTT has not been implemented in zig yet, and may not ever be...).

**What the PSL programmer gets:**

- **Correct rounding** — four modes (IEEE `half_even` default, `half_up`, `down`, `up`) applied automatically after every arithmetic operation.
- **Precision clamping** — results are clamped to context precision after every operation (add, sub, mul, div), so you never carry more digits than you asked for.
- **Scalable performance** — small numbers use direct u128 paths; large numbers use Karatsuba multiplication (O(n^1.585)) and Newton-Raphson division (O(M(n))), with a fallback Knuth D path for medium divisors.

**What the PSL programmer does not need to worry about:** choosing algorithms, managing intermediate precision, or worrying about when rounding happens.  Every operation produces a correctly-rounded result at context precision, automatically.

---

## 12. Hybrid, lightweight manual memory management

The kernel uses a hybrid memory strategy designed to make PSL's explicit cleanup safe and simple:

- **Slab allocator** — the primary allocator for PSL values, behaviors, modules, and temporary buffers.  Allocation is lock-free on the hot path; slabs are reclaimed by a background timer when empty.  Exposes a standard `std.mem.Allocator` interface.
- **Reference counting** for persistent values (arrays, dicts, sets, strings, closures, decimals).  Values are freed automatically when the last reference is released.  A decimal carries its refcount and lazily-computed hash inline — cloning a `number` is an increment, not a copy.
- **Arena allocation** for context-scoped lifetimes.  A context owns its modules; when the context dies, the arena reclaims most of the memory in bulk.
- **Manual ownership** for transient buffers and error messages.

**What the PSL programmer must do:** free signals, modules, contexts, and territory zones when you are done with them.  Cleanup is easy and simple — frees cascade (freeing a context free its territory and its modules, and the module frees cascade into freeing their signals) and there is no risk of double-free or use-after-free — but the programmer is responsible for doing the work where it matters.  Neglecting cleanup causes leaks; the runtime will not stop you.

**What the PSL programmer does not do:** manage reference counts, choose allocation sizes, or worry about fragmentation.  The kernel handles all of that automatically.

The design has visible consequences:

- Large shared collections are cheap to pass around.
- There is no stop-the-world garbage collector pausing the drain loop.
- Closures capture by deep copy, so a returned `fn` value is self-contained.
- Behavior frames, module snapshots, and territory zones have explicit lifetimes and reader refcounts.

The kernel avoids hidden sharing and hidden mutation.  Values are either owned, borrowed with a documented lifetime, or reference-counted.  This makes the runtime predictable enough to host long-running modules and tight coordination loops.

---

## 13. Security and authentication

PSL provides built-in cryptographic primitives, authentication tokens, HTTP transport, process execution, and permission scoping — all accessible through `membrane.lib.*` and `membrane.*` without leaving the language.

### 13.1 Cryptographic primitives

`membrane.lib.crypto.*` provides symmetric authentication and encryption built on Zig's standard library:

```poesis
hoist membrane.lib.crypto as crypto

# HMAC-SHA256
mac = crypto.hmac("message", "secret")
valid, err = crypto.verify_hmac("message", mac, "secret")

# AES-256-GCM encryption
ciphertext, err = crypto.encrypt("plaintext", "secret_key")
plaintext, err = crypto.decrypt(ciphertext, "secret_key")

# Random bytes (hex-encoded, 1–1024 bytes)
key = crypto.random_bytes(32)
```

- **HMAC:** `hmac(message, key)` returns a hex-encoded MAC.  `verify_hmac` performs constant-time comparison.
- **Encryption:** AES-256-GCM.  The key must be 64 hex characters (32 bytes).  Ciphertext is hex-encoded `nonce || ciphertext || tag`.  Wrong key size or corrupted ciphertext returns an error value.
- **Random bytes:** `random_bytes(n)` returns a hex string of `n` random bytes.

All fallible functions (`verify_hmac`, `encrypt`, `decrypt`) require mandatory error binding.

### 13.2 JWT and API keys

JWT (HS256) and API key management build on the crypto primitives:

```poesis
hoist membrane.lib.jwt as jwt
hoist membrane.lib.apikey as apikey

# JWT: sign and verify
compact, err = jwt.sign(payload_json, "secret")
claims, err = jwt.verify(compact, "secret", issuer = "myapp")

# API keys: mint and verify
token, record, err = apikey.mint(scope = "read", expires_in = 3600)
claims, err = apikey.verify(token, record)
```

- **JWT:** HS256 only.  `sign` takes a JSON string payload and returns a compact JWT.  `verify` returns the claims dict on success; optional `issuer` and `audience` checks narrow the verification.
- **API keys:** `mint` returns a one-time token (shown to the client) and a record dict (stored server-side).  Only the hash is stored — the raw token cannot be recovered.  `verify` performs constant-time hash comparison and returns the claims dict on success.

### 13.3 HTTP transactions

`membrane.http.*` provides synchronous HTTP request/response handling.  All HTTP operations use **handles** — fastnumber values that can live anywhere a value can live: signals, territory, scratch, dicts, function arguments.

```poesis
txn = membrane.http.create()            # returns a handle (fastnumber)
membrane.http.set_method(txn, "POST")
membrane.http.set_url(txn, "https://api.example.com/data")
membrane.http.set_header(txn, "Authorization", "Bearer ${token}")
membrane.http.set_body(txn, payload)
txn, err = membrane.http.send(txn)      # suspends until response; handle stays valid

status = membrane.http.status(txn)      # e.g. 200
body = membrane.http.result(txn)        # response body string
membrane.http.free(txn)                 # release resources (optional — auto-freed on timeout)

# Registry visibility
handles = membrane.http.list()          # array of all active handle IDs
```

- **Handles are values.**  A handle is a fastnumber — it passes through signals, territory, and scratch without special casing.  No copy restrictions.
- **Non-consuming send.**  `send` does not consume the handle.  After sending, the handle points at the response — `status` and `result` read from it.
- **Explicit cleanup.**  `http.free(handle)` releases the transaction's memory.  The behavior that created the handle is responsible for freeing it.  No auto-cleanup — predictable, matches the territory pattern.
- **Multiple reads.**  `status` and `result` can be called multiple times on the same handle — they just read the stored response.

HTTP requests are suspended — the behavior yields until the response arrives (typically the next cycle).  The handle is valid before, during, and after `send`.

### 13.4 Process execution

`membrane.lib.process.*` runs external commands in a sandbox:

```poesis
hoist membrane.lib.process as proc

# Synchronous
output, err = proc.run("git", ["log", "--oneline"])
# output = {stdout: "...", stderr: "...", exit_code: 0}

# Asynchronous
handle, err = proc.run_async("long_task", ["--verbose"])
status = proc.status(handle)         # "pending" | "complete" | "failed"
output, err = proc.result(handle)
```

- **Sandbox:** only basename is accepted (no shell paths).  A configurable allowlist and `max_concurrent` limit prevent abuse.
- **Timeout:** default 300 seconds; configurable per call.
- **Async pattern:** `run_async` returns a handle.  Poll with `status`; collect results with `result` once complete.
- **Result shape:** the output dict contains `stdout` (string), `stderr` (string), and `exit_code` (number).

### 13.5 Permissioning

A context can be created with a **permission set** that restricts which API namespaces its behaviors can access:

```poesis
ctx = membrane.context.new(
    ["worker"],
    permissions = {
        membrane: ["signal", "scratch", "self"],
        territory: ["product"]
    }
)
```

The dict maps namespace strings to arrays of allowed capabilities.  A `null` permissions value (the default) grants full access.  Operations outside the allowed set return an error value rather than faulting — the behavior stays alive, but the restricted call fails.

### 13.6 Web server

`membrane.web.*` provides HTTP server primitives as handle-based API.  Connection handles are values — they pass through signals, territory, and scratch freely, just like HTTP handles, module handles, and context handles.

```poesis
server = membrane.web.serve(8080)

conn, err = membrane.web.recv(server)    # suspend until connection arrives
method, _ = membrane.web.method(conn)    # "GET", "POST", etc.
path, _ = membrane.web.path(conn)        # "/api/users"
body, _ = membrane.web.body(conn)        # request body string

_, _ = membrane.web.status(conn, 200)
_, _ = membrane.web.set_header(conn, "Content-Type", "application/json")
_, _ = membrane.web.send(conn, '{"ok": true}')
_, _ = membrane.web.close(conn)
```

All connection operations are **fallible** — they return `(val, err)`.  Discard the error with `_` for noop behavior on expired handles.

#### Server as dispatch

The dispatch is a dumb spawner — it receives a connection, spawns a handler context with the conn handle, and immediately moves on to the next request.  All logic lives in the handler.  The dispatch runs at high priority so it can punch out handlers as fast as requests arrive, even under heavy load:

```poesis
# web:server behavior — the dispatch loop (high priority)
server = membrane.web.serve(8080)

while true {
    conn, err = membrane.web.recv(server)
    if conn == null {
        time.sleep(0.001)  # non-blocking — yields the worker for 1ms
        continue
    }

    # Spawn a handler and move on — fire and forget
    _, _ = membrane.context.new(["web:handler"], {
        conn: conn
    })
}
```

The dispatch does no routing, no file serving, no response writing.  It just hands off the connection and loops.

#### The handler

The handler owns the entire request lifecycle — it can serve static files, route to other behaviors, upgrade to WebSocket, or do anything else the connection needs.  When done, it frees its own context and converges to prevent a pointless cycle restart:

```poesis
# web:handler behavior — handles one connection end-to-end
conn, err = membrane.web.recv(server)
if err != null { continue }

path, _ = membrane.web.path(conn)

if membrane.web.is_websocket(conn) {
    membrane.web.ws_accept(conn)
    while true {
        msg, err = membrane.web.ws_recv(conn)
        if err != null { break }
        membrane.web.ws_send(conn, "echo: ${msg}")
    }
} elif membrane.web.serve_file(conn, path) {
    # Static file served — done
} else {
    # Dynamic route
    result, _ = membrane.func.api.handle_request(path, membrane.web.body(conn))
    _, _ = membrane.web.status(conn, result.status)
    _, _ = membrane.web.send(conn, result.body)
}

_, _ = membrane.web.close(conn)
handle = membrane.context.handle()
territory.contexts.free(handle)   # free the context
converge                          # safeguard: prevent a pointless cycle restart while free cascades
```

#### Routing

Routing is just PSL code — a function that maps paths to behavior names.  No framework, no special syntax:

```poesis
fn route(path) {
    if path == "/health" { return "health_check" }
    if path == "/api/users" { return "api:user_list" }
    if path == "/api/items" { return "api:item_list" }
    return null  # null means "try static files"
}
```

The handler calls `route`, gets a behavior name or `null`, and either spawns the behavior or serves a static file.  This is the simplest routing: a flat function with if/else.  For more complex routing (wildcards, path parameters), use `membrane.lib.string.starts_with` for prefix matching or build a route table:

```poesis
routes = {
    "/health": "health_check",
    "/api/users": "api:user_list",
    "/api/items": "api:item_list",
}

fn route(path) {
    if routes[path] != null { return routes[path] }
    return null
}
```

#### Connection lifecycle

`recv` suspends until a connection arrives.  All connection operations are **fallible** — they return `(val, err)` and return an error for expired or invalid handles.  Discard the error with `_` to get noop behavior:

```poesis
method, _ = membrane.web.method(conn)     # returns "GET" or (null, error)
status, _ = membrane.web.status(conn, 200)  # noop on expired handle
send, _ = membrane.web.send(conn, body)    # noop on expired handle
close, _ = membrane.web.close(conn)        # noop on expired handle
```

This makes cleanup safe — `close` never faults, and you can chain `or` for quick context cleanup:

```poesis
# If anything fails, free the context immediately
path = membrane.web.path(conn) or (
    territory.contexts.free(membrane.context.handle())
    return
)
```

Request data persists on the handle after `send` — you can still read `method`, `path`, `body`, and `headers` for post-response processing.  But you cannot respond again; the connection is done.

Connections auto-expire after a configurable timeout.  If the handler doesn't finish in time, the connection is freed automatically — dropped connections never leak.  The default timeout is 30 seconds; set it per-connection or per-server:

```poesis
# per-connection timeout
membrane.web.request_timeout(conn, 30)  # 30 seconds

# per-server default
membrane.web.set_default_timeout(server, 90)  # 90 seconds
```

#### Static files

Serve files from a directory without writing PSL dispatch:

```poesis
membrane.web.set_root("static")                     # set root directory
served, err = membrane.web.serve_file(conn, "/style.css")  # auto content-type
```

Content types are inferred from file extensions (`.html`, `.css`, `.js`, `.json`, `.png`, etc.).  Directory paths serve `index.html` automatically.

#### Production hardening

```poesis
membrane.web.set_max_concurrent(64)   # connection limit
membrane.web.set_logging(true)         # log requests to stderr
membrane.web.shutdown()                # graceful stop — next recv() fails
```

#### WebSocket support

Upgrade HTTP connections to WebSocket for bidirectional communication:

```poesis
conn, err = membrane.web.recv(server)
if membrane.web.is_websocket(conn) {
    membrane.web.ws_accept(conn)        # complete upgrade handshake

    while true {
        msg, err = membrane.web.ws_recv(conn)
        if err != null { break }
        membrane.web.ws_send(conn, "echo: ${msg}")
    }
}
```

WebSocket APIs: `ws_recv`, `ws_send`, `ws_broadcast`, `ws_subscribe`, `ws_unsubscribe`, `ws_connections`.  Topics enable broadcast patterns — subscribe a connection to a topic, then broadcast to all subscribers:

```poesis
membrane.web.ws_subscribe(conn, "chat")
membrane.web.ws_broadcast("chat", "hello everyone")
handles = membrane.web.ws_connections("chat")
```

---

## 14. Testing (roughly)

The kernel is tested at multiple layers, but the primary verification method is **integration tests** — full end-to-end behavior executions that exercise the runtime the way PSL programs actually use it.  Unit tests cover individual operations (values, lexer, parser, decimal arithmetic), and component tests cover internal machinery (signal store, scheduler, territory locking), but these exist to support the integration layer, not replace it.  PSL behaviors are emergent — the interesting behavior arises from the interaction of signals, cycles, convergence, and territory — so meaningful verification is integration-shaped more often than unit-shaped. The project treats correctness of design as primary: if the design is wrong, no amount of unit tests will catch it.

---

## 15. Design philosophy: why Poesis doesn't promise global state

### The pattern

Many programming paradigms make a global promise early — one address space, one event ordering, one consistent state, one type hierarchy.  The promise buys real ergonomics: you stop reasoning about who wrote a value, when, or from where, because the paradigm has already decided the answer is "it doesn't matter, there's only one of it."

The problem isn't that these promises are false.  It's that they're expensive to keep true, and the cost doesn't show up where the promise was made — it shows up wherever the underlying hardware or network actually is what it is: multiple cores, multiple machines, multiple independent failure domains.  Cache coherence protocols, memory fences, two-phase commit, consensus algorithms — these exist because something promised a single global truth, and someone has to do continuous work to make that promise keep looking true under conditions where it isn't structurally free.

That work is a **constraint**.  Not a moral failing of the paradigm — a real, load-bearing cost that the paradigm's users mostly don't see, because it's been pushed down into the runtime or the hardware.  You pay for it in latency, in the size of the "here's how memory ordering actually works" chapter you have to learn, in the entire discipline of distributed-systems engineering whose job is to keep the promise honest.

### What "relaxing the constraint" means

Poesis's storage ladder — locals, scratch, signals, territory — doesn't refuse to give you global anything.  It just refuses to give it to you **by default, silently, at every layer**.  Instead, each layer states plainly what it actually guarantees:

- A local is yours, this cycle, no one else can see it.
- Scratch is yours, across cycles, no one else can see it.
- A signal is yours to write, visible to your module, one cycle stale, at a defined commit boundary.
- Territory's `product` store is genuinely shared and genuinely live — the one place two writers are allowed to race, because sometimes racing is the actual answer you want.

None of these layers promise a single global anything.  What they promise is smaller and cheaper: a well-defined boundary, a well-defined staleness, a well-defined set of readers and writers.  That's the relaxation — you don't get "everyone agrees on everything, always," you get "here is exactly who agrees on what, and when."

### Why the good result doesn't become harder to get

The natural worry is that giving up the global promise means giving up the thing the promise was for — a shared clock, shared state, consistent ordering.  In practice it doesn't, because those things were never actually free under the old paradigms either; the paradigm just hid the bill.  Poesis hands you the bill up front, and it turns out to be small, because the primitives you need to pay it were already sitting in the ladder for other reasons.

**A global clock** is a module that writes a tick signal; other modules read the snapshot.  No territory contention, no consensus protocol — a single writer, several readers, exactly the case the signal layer already exists for.  It's a dozen lines of ordinary PSL, not a workaround bolted onto a system that "doesn't have" a clock.

**Global state** is the identical move: a well-known module owns it, publishes it as signals, everyone else reads the snapshot.  If it needs to be genuinely mutable from many places with no single owner, that's the signal that you've left "shared state with one writer" and entered "several writers competing for one slot" — and *that's* what `territory.product.*` is for.  Most things people reach for a shared mutable store to do don't actually need contention, they need publication, and publication is cheaper.

**Hard real-time timing** — where you can't tolerate one-cycle staleness — doesn't route through a simulated global clock.  It routes through the scheduler's priority queue: mark the module high-priority and let the bitmap scheduler guarantee it gets claimed ahead of standard work, every cycle.  This is more honest than a "global timeline" abstraction, because it asks the actual mechanism that can make a timing guarantee rather than asking a fictional universal clock to make a promise no distributed system can actually keep without cost. It doesn't guarantee you run — the OS and hardware can still get in the way. But it's the closest guarantee you can get without OS priority, regardless of language or platform.

### The cost that matters more

There's a second cost to a totalizing frame that's easy to miss, because it's not paid in latency or memory-model chapters — it's paid in which questions you ever get around to asking.

When a language hands you shared mutable memory for free, "just mutate the shared thing" is the path of least resistance, and most designs take it — not because it's the best answer, but because nothing ever forced the alternative into view.  The totalizing frame doesn't just make the naive implementation cheap.  It makes the naive implementation *invisible as a choice at all*.  Nobody stops to ask "does this really need to be synchronous," "does this really need to be one structure," "does this really need to update at the same rate as everything touching it" — because the global promise answers all of those questions before they're asked, silently, on your behalf, in whichever direction requires the least new code this week.

That's a more expensive thing to lose than a few cycles of cache coherence traffic.  It's the moment where a better architecture could have occurred to you, quietly removed from the search space.

### Where the constraint pays for itself

Poesis's ladder doesn't hand you that free answer.  Every time you reach past locals and scratch, you have to say which of the remaining layers you actually mean — and that friction, small as it is, is exactly the moment the better question gets asked instead of skipped.

Take a physics simulation's collision detection.  The reflexive design, in a language with shared mutable state, is a single global spatial index — an octree or BVH, rebuilt in lockstep with the simulation tick, mutated in place by whatever's currently moving.  It's the totalizing-frame answer: one structure, one truth, always current, because current is free.

In Poesis, that answer isn't free, so the question underneath it actually gets asked: does the index need to be that current?  Decomposed the way the ladder pushes you to decompose it, it doesn't.  Narrow-phase collision is naturally local — each entity checks its own movement against its own neighbors, at whatever tick rate its own distance and level of detail warrant.  Broad-phase — the part that's genuinely global, because *something* has to know who's near whom — doesn't need to be synchronous either.  It becomes a reader: a behavior that gathers world state at its own cadence and publishes a spatial snapshot.  Every consumer reads that snapshot instead of touching a shared structure at all.

That isn't a workaround for lacking free shared memory.  It's a better architecture than the naive version — decoupled update rates, no synchronization point, no contention — and it's one that most engines never reach for, because the free global answer never made them ask whether the coupling was load-bearing in the first place.

The same shape shows up in coordination.  Consider an online transaction: debit a customer, credit a vendor, adjust inventory.  In a paradigm with shared mutable state, the reflexive answer is a lock, a transaction coordinator, or two-phase commit — a totalizing mechanism that makes "all three happen or none do" a global invariant enforced by the runtime.  In Poesis, the same guarantee emerges from the signal layer without any of that machinery.  Three behaviors in one module — one per operation — each check the other two's signal outputs:

```poesis
# debit.poesis
if not membrane.signal.has("debit", "status") {
    # first cycle: nobody has produced yet, do the work
    # ... do the debit ...
    membrane.signal.set("status", "committed")
} elif (membrane.signal.get("credit", "status") in ["error", "rollback"]
    or membrane.signal.get("inventory", "status") in ["error", "rollback"])
    and membrane.signal.get("debit", "status") == "committed" {
    # sibling failed after I committed — roll back
    # ... do the rollback ...
    membrane.signal.set("status", "rollback")
}
```

Each behavior independently commits on its first cycle — nobody has failed yet, there is nothing to wait for.  On the second cycle, it checks whether any sibling has rolled back.  If so, it rolls back too.  Natural quiescence — signal stability — happens when either all three have committed or all three have rolled back — which, given the design here, will happen at cycle 2 if everybody committed, or cycle 3 if anyone rolled back.  No coordinator, no lock, the drain loop's convergence detection is the transaction boundary, and the storage ladder's semantics are the atomicity guarantee.  The friction of expressing coordination through signals instead of shared state forced the question "does this need a lock, or does it need a protocol?" — and the answer was a protocol that's simpler than either alternative.

### The actual claim

Poesis doesn't eliminate the need for a clock, for shared state, for consistent ordering when a problem genuinely requires them.  What it eliminates is the runtime's obligation to *pretend* those things exist everywhere, all the time, whether or not the problem in front of you needs them — and the cost that pretense carries when it's wrong for your case.

The relaxed constraint isn't "you can't have global things."  It's "global things cost what they actually cost, visibly, at the place you decided you need one — instead of being charged uniformly, invisibly, everywhere, whether you needed them or not."  And often, when the cost becomes visible, the better architecture — decoupled update rates, no synchronization point, no contention — turns out to be cheaper than the global version would have been.

---

## 16. Universally namespaced visibility: the other inversion

§15 argued that Poesis doesn't refuse you global anything — it refuses to give it to you silently, by default, at every layer.  That argument was framed around *write* cost: reaching for territory, or promoting a value to a signal, has a visible price precisely where shared mutable state would normally be free.  But there is a second inversion sitting next to it, on the *read* side, that is easy to miss because it runs in the opposite direction from what every mainstream language trains you to expect.

### The usual assumption

In most languages, visibility is scarce and declared.  A variable is local unless you go out of your way to make it otherwise — `global`, `static`, `pub`, an exported symbol, a shared struct threaded through a dozen constructors.  Every promotion to wider visibility is a decision you make once, and from that point on it is a fact about your program that everyone else has to know and respect.  The cost of this model shows up as *namespace pollution*: a global is a single name in a single flat space, so every other global has to avoid colliding with it, and the number of things you can safely make visible is bounded by how many names you're willing to coordinate across a team, a module, a codebase.  Concurrency compounds this — a goroutine, a Rust task, a thread — is born opaque.  If two units of concurrent work need to see each other's progress, someone has to build that: a channel, a shared `Arc<Mutex<_>>`, a mailbox address, decided in advance, at the point where the units are spawned.  Late-bound curiosity — "actually, I'd like to see what unit 12 is doing" from inside unit 47, decided after both are already running — is exactly what these mechanisms make hard, because visibility was never the default; it was a wire someone had to run.

### What Poesis does instead

Poesis inverts this.  Nothing has to be declared global, because in the read direction, almost everything already is.  Any behavior holding a handle can read any module's entire committed signal space via `territory.modules.get(handle)` — at the same one-boundary staleness a sibling behavior gets from `membrane.signal.get`.  This is not bounded by module membership, and it is not bounded by context.  §4.4 is explicit that `get` and `give` are the two territory operations that cross context boundaries by design — reading a foreign context's product store, or a foreign module's snapshot, needs nothing more than the handle.  Context isolation (§2.1) is "the default, not a wall," and this is where that phrase is load-bearing: the wall in Poesis was never around *observation*.  It is around *mutation* — `set` and `take` are local-only, force and consume are the two things that "stay home" (§4.4), because those are the two operations that can actually hurt a foreign owner.  Reading can't corrupt anything, so reading isn't gated by anything structural at all — only by whether you can get a handle, and handles are cheap: `module.run.async` hands you one directly, and `territory.contexts.list()` or `blind_walk` hands you ones you never spawned.

The only things that are genuinely private in the entire system are locals (gone at the cycle boundary) scratch (never leaves the behavior that owns it, full stop) and hidden signals (the special rare case when a module wants to keep a secret).  Everything past that — every non-hidden signal, and by extension every module's complete public state — is legible from anywhere, to anyone holding a handle, the instant it commits.

### Why this doesn't collapse into the same mess

The obvious objection is that "everything is global" is exactly the failure mode namespace discipline exists to prevent.  It doesn't collapse here, because Poesis never merges the names.  A signal is never just `output` — it is `worker_12:output`, addressed by the behavior that owns it, inside a module, inside a context.  Two behaviors can both write a signal called `output` and never once collide, because visibility was never mediated by a shared name in the first place — it was mediated by a shared *view* into namespaces that stay separate all the way down.  This is the precise inversion: other languages make global cheap to declare and expensive to keep collision-free, because the currency of global-ness is a name in one flat space.  Poesis makes visibility unconditional and keeps it collision-free for free, because the currency of identity was never a name you had to coordinate — it was a handle plus an address (`behavior:key`, `module`, `context`) that was unique from the moment it was created.

That is also why the write-side wall (§4.4) is the *only* wall the system needs.  A flat-global design has to defend against collision on every axis, because reads and writes share the same namespace and the same access path.  Poesis only has to defend the write side — the one place two writers can actually stomp the same slot — because the read side was never a shared resource to begin with, only a shared lens onto resources that were namespaced from birth.

### The consequence

This is what makes patterns like fanning out a hundred async workers and letting any of them peek at any other's progress look almost too easy.  In a language where visibility has to be wired up per-relationship, ad hoc cross-talk between concurrent units is exactly the kind of thing you have to plan for at spawn time, because nothing is watchable unless someone built the pipe first.  In Poesis, there is no pipe to build.  The pipe is the default, the same one every behavior already has to its siblings, extended without modification to every module and every context reachable by handle.  What you design, instead, is what to keep out of it — which is exactly what hidden signals and scratch are for: not ways of making something visible, but the two deliberate ways of choosing not to be.

---

## 17. PSL language reference

> This section specifies the language itself — types, operators, control flow, functions, and idioms.  For the runtime execution model (signals, convergence, quarantine, storage layers), see §§3–5, 9–10 above.

### Types

| Type | Literal | Notes |
|------|---------|-------|
| `number` | `42`, `3.14` | Decimal with configurable precision (default 38 digits). Coerced to bool in `if`/`while` conditions. Finite only — NaN/Infinity are rejected (quarantine). |
| `fastnumber` | `f42`, `f3.14` | Ergonomic numeric lane, internally int64/float64 with normalize-on-write. Finite only, same as `number`. See §11. |
| `string` | `"hello"`, `"x: ${y + 1}"` | Double-quoted, `${...}` interpolation.  Supports escape sequences (see "String escapes" below). |
| `bool` | `true`, `false` | Boolean value |
| `null` | `null` | Null value |
| `array` | `[1, 2, 3]` | Zero-indexed. Mutable via index assignment. |
| `dict` | `{a: 1, b: 2}` | String keys. `d.a` and `d["a"]` both work. |
| `set` | `set(1, 2, 3)` | Immutable elements only. Union `\|`, intersection `&`. |
| `fn` | `fn { require x; return x * 2 }` | First-class function value. See Functions below. |
| `error` | `error("missing_key", "no such key")` | First-class failure value: `kind` (machine-checkable) + `message` (human). See §9. |
| `regex` | `regex("\\d+")` | Compiled regular expression, refcounted. |
| `dynamic` | `dynamic()`, `dynamic(dict_val)` | Dict-like with auto-vivification, null-proxy, and lensing. See below. |

### String escapes

PSL string literals support backslash escape sequences.  Escape processing happens at compile time — the compiler transforms the raw source text into the final string value before the VM sees it.

| Escape | Result |
|--------|--------|
| `\\` | literal backslash |
| `\n` | newline |
| `\t` | tab |
| `\"` | literal double quote |
| `\r` | carriage return |
| `\0` | null byte (for C interop) |
| `\u{XXXX}` | Unicode codepoint (hex digits, encoded as UTF-8) |

Unknown escapes (like `\d` or `\w`) are preserved as-is — both characters pass through unchanged.  This means regex patterns in strings (`"\\d+"`) continue to work without special treatment.

```poesis
# Newlines and tabs in strings
msg = "line1\nline2"
tabbed = "col1\tcol2"

# Embedded quotes
quoted = 'say \"hello\"'

# Unicode
accent = "caf\u{00E9}"   # café

# Double backslash
path = "C:\\Users\\doc"   # C:\Users\doc
```

The `string.lines` function (see "String and collection methods" below) splits on the newline characters produced by `\n`, so `"line1\nline2".lines()` returns `["line1", "line2"]`.

### Fastnumbers

`fastnumber` is the ergonomic numeric lane: one user-visible type with automatic internal int64/float64 handling.  `number` remains the exactness lane (38-digit decimal).

- **Duality:** internally `int64` when integral, `float64` when not.  Integral values are int, period — `f3.0` and `f3` are the same value (normalize-on-write).
- **No negative zero:** `-0.0` flattens to `f0` by construction.
- **No NaN/Infinity:** same rule as `number`.  An operation producing a non-finite result faults → quarantine.
- **Overflow:** int arithmetic overflowing int64 promotes silently to float (one-way, lossy past 2⁵³; exact big integers are what `number` is for).
- **Division is always float** (`f6 / f2` → 3.0 → demotes to int `f3`).  Division by zero errors → quarantine.
- **The wall:** `fastnumber` and `number` are different types.  **Arithmetic across the wall requires an explicit cast** (`f0.1::number + 0.2`), else TypeMismatch → quarantine.  **Comparisons never require same-type** — fastnumber promotes to decimal via shortest-repr, so `f0.1 == 0.1` is true.  Membership (`in`) inherits comparison: `f1 in [1, 2]` works.
- **Same-lane operators:** `++`, `--`, and unary `-` preserve the operand's lane — `f1++` is fastnumber arithmetic, `1++` is number arithmetic.
- **Casts** (the `::` operator): `n::fastnumber` rounds to nearest binary64; `f::number` crosses via shortest-repr (round-trips exactly); `f::string` gives the shortest representation.
- **Literals:** `f` immediately followed by a digit: `f3`, `f0.1`, `f1e3`.
- **Indexing:** integral fastnumber values are int, so `arr[f1]` works.

### Variables and assignment

```poesis
x = 1                    # declaration + assignment
a, b = 1, 2              # multi-declare (RHS must be multiple expressions)
_ = discard()            # underscore is the discard identifier
a, _ = f()               # discard error channel in multi-assign

# Compound assignment
x += 1                   # x = x + 1
x -= 1                   # x = x - 1
x *= 2                   # x = x * 2
x /= 2                   # x = x / 2
x %= 3                   # x = x % 3
x++                      # same-lane increment
x--                      # same-lane decrement
```

### Operators

```poesis
# Arithmetic
+  -  *  /  %

# Comparison
==  !=  <  >  <=  >=

# Logical
and  or  not           # and/or short-circuit: RHS evaluates only if LHS doesn't decide

# Membership
in  not in           # "a" in dict, 3 in arr, "he" in "hello"

# Cast (postfix, binds tight — same lane as . and [])
x::number            # strict: unconvertible quarantines; unknown type name is a compile error

# Type test
x == null            # true only for null values

# Ternary (right-associative)
cond ? a : b
```

**Operator precedence** (high to low): `()` `.` `[]` `::` `++` `--` → `*` `/` `%` → `+` `-` → `in` `not in` → `<` `>` `<=` `>=` `==` `!=` → `not` → `and` → `or` → `? :` → `=` `+=` `-=` `*=` `/=` `%=`

In pair position (`val, err = f() or g()`), `or` is error-coalescing, not boolean — see §9.

### Type Casts

The `::` operator performs explicit type conversion.  Invalid casts return an error value (not quarantine) — the caller must bind the error channel.

```poesis
x::number            # string/bool/fastnumber → number
x::string            # any type → string representation
x::fastnumber        # number/string → fastnumber (rounds to binary64)
x::bool              # string/number → bool ("true" → true, 0 → false)
x::array             # string (JSON) → array
x::dict              # string (JSON) → dict
x::set               # array → set (dedup)
x::dynamic           # dict → dynamic (re-tag)
x::dict              # dynamic → dict (re-tag)
x::null              # any type → null (explicit nullification)
x::regex             # string → compiled regex
```

**Valid cast matrix:**

| From → To | string | number | fastnumber | bool | array | dict | set | dynamic | regex | null |
|-----------|--------|--------|------------|------|-------|------|-----|---------|-------|------|
| **string** | — | ✓ | ✓ | ✓ | ✓¹ | ✓¹ | — | — | ✓ | ✓ |
| **number** | ✓ | — | ✓ | ✓ | — | — | — | — | — | ✓ |
| **fastnumber** | ✓ | ✓ | — | ✓ | — | — | — | — | — | ✓ |
| **bool** | ✓ | ✓ | ✓ | — | — | — | — | — | — | ✓ |
| **array** | ✓ | — | — | — | — | — | ✓ | — | — | ✓ |
| **dict** | ✓ | — | — | — | — | — | — | ✓ | — | ✓ |
| **set** | ✓ | — | — | — | — | — | — | — | — | ✓ |
| **dynamic** | ✓ | — | — | — | — | ✓ | — | — | — | ✓ |
| **regex** | ✓ | — | — | — | — | — | — | — | — | ✓ |
| **null** | ✓ | — | — | — | — | — | — | — | — | — |
| **shape** | — | — | — | — | — | ✓ | — | — | — | — |

¹ JSON syntax parsing — escape sequences are processed first, so `\\\"` in source becomes `\"` in the string before JSON parsing.

**Notes:**
- Cross-wall casts (`number` ↔ `fastnumber`) require explicit `::` — implicit coercion is a TypeMismatch error.
- `::string` for `dict`/`array` produces recursive string representations (JSON-like).
- `::string` for `set` uses set literal notation: `"set(1, 2, 3)"`.
- `::dynamic` and `::dict` are O(1) re-tags (no copy).
- `::shape` constructs a shape from a dict by inferring descriptors from value types (infallible). Nested dicts become nested shapes; arrays become `shape.array(T)`.
- `::null` is always valid — explicit nullification of any value.
- Invalid casts (e.g., `fn::number`) return an error value: `error("type_mismatch", "cannot cast fn to number")`.

### Shapes

A **shape** is a structural predicate that describes the expected structure of a dict.  Shapes are values — they pass through signals, territory, and scratch like any other value.  Two shapes with identical fields are interchangeable; there is no name, no registry, and no identity.

#### Construction

```poesis
Point = shape({x: number, y: number})
```

`shape(spec)` takes a dict mapping field names to type descriptors and returns a shape value.  An empty spec is a fault.  Type descriptors are primitive type names (`number`, `string`, `bool`, etc.), nested shapes, or parameterized descriptors (`shape.array(T)`, `shape.optional(T)`, etc.).

Inline dict sugar in `require` annotations desugars at compile time:

```poesis
require a: {x: number, y: number}
# desugars to:
require a: shape({x: number, y: number})
```

Named shapes from enclosing scope also desugar:

```poesis
Point = shape({x: number, y: number})
fn dist {
    require a: Point, b: Point
    # desugars to:
    # require a: shape({x: number, y: number}), b: shape({x: number, y: number})
}
```

#### Validation

```poesis
value, err = shape.validate(Point, d)
```

Returns `(normalized, err)`. On success, `normalized` is `d` with defaults filled and extra fields stripped.  On failure, `err` is a structured dict with `kind: "shape_mismatch"` or `kind: "not_a_shape"`.  This is a fallible inquiry, not a fault.

```poesis
p, err = d::Point
```

Equivalent to `shape.validate(Point, d)`.  Fallible; requires error binding.

#### Boundary validation

```poesis
fn dist {
    require a: {x: number, y: number}, b: {x: number, y: number}
    return sqrt((a.x - b.x)**2 + (a.y - b.y)**2)
}
```

`require` validates the argument against the shape on entry, fills defaults, strips extra fields, and rebinds the parameter.  A mismatch is a **fault** — the behavior is quarantined.

#### Composition

| Operation | Effect |
|---|---|
| `shape.merge(a, b)` | Union of fields. Same field with different descriptor is an error. |
| `shape.extend(a, b)` | Like merge, but `b`'s descriptors override `a`'s on conflict. |
| `shape.pick(s, fields)` | Keep only the listed fields. |
| `shape.omit(s, fields)` | Drop the listed fields. |
| `shape.partial(s)` | Every field becomes optional. Defaults preserved. |
| `shape.required(s)` | Inverse of partial. Defaults removed. |
| `shape.strict(s)` | Reject dicts with fields not in the shape. |
| `shape.union(a, b, ...)` | Untagged union. Passes if it satisfies any branch. |
| `shape.intersect(a, b, ...)` | Must satisfy all. Semantically equivalent to `merge`. |

#### Introspection

```poesis
shape.fields(Point)         # {x: number = 0, y: number = 0}
shape.field(Point, "x")     # number = 0
shape.has_field(Point, "z") # false
shape.field_names(Point)    # ["x", "y"]
shape.describe(Point)       # "shape({x: number = 0, y: number = 0})"
```

`shape.fields` returns descriptors with defaults included — the inverse of `d::shape`.

### Control flow

#### Conditional

```poesis
if x > 10 {
    # ...
} elif x > 5 {
    # ...
} else {
    # ...
}
```

#### While

```poesis
while x < 100 {
    if x == 50 { break }
    if x == 25 { continue }
    x = x + 1
}
```

#### For (range)

```poesis
for i in range(10) {       # 0..9
    # i is scoped to the loop body
}

for i in range(1, 5) {     # 1..4
    # ...
}

for i in range(10, 0, -2) { # 10, 8, 6, 4, 2
    # ...
}
```

#### For (array iteration)

```poesis
for elem in arr {          # array elements
    # ...
}

for i, v in arr {          # index-value pairs (i is fastnumber)
    # ...
}
```

The two-variable form yields `(index, value)` pairs.  The index is a `fastnumber` (`f0`, `f1`, ...).  This is equivalent to calling `.enumerate()` but built into the language.

#### For (dict iteration)

```poesis
for k in dict {            # dict keys (order not guaranteed)
    # ...
}

for k, v in dict {         # key-value pairs (order not guaranteed)
    # ...
}
```

#### For (set iteration)

```poesis
for elem in set {          # set elements (order not guaranteed)
    # ...
}
```

The two-variable form is not supported for sets — sets have no natural key/index pairing.  `for k, v in set` faults (TypeMismatch → quarantine).

#### For (string iteration)

```poesis
for ch in str {            # single-character strings (UTF-8 codepoint-wise)
    # ...
}
```

#### Match

```poesis
match x {
    0: {
        label = "zero"
    }
    42: {
        label = "forty-two"
    }
    _: {
        label = "other"     # wildcard, required, must be the LAST arm
    }
}
```

Match arms are evaluated in order.  The first matching pattern executes its block.  The wildcard `_` is required as a fallback and must be the last arm — a non-wildcard arm after `_` is a compile error.  Patterns can be strings, numbers, booleans, `null`, or identifiers.

#### Yield

```poesis
yield                    # give up the worker; halt the cycle while waiting; resume when picked up by another woker

yield until territory.product.has("test", "ready")   # give up the worker; halt the cycle while waiting; re-check on each resume and yield again if false; proceed when true
```

- `yield` suspends the frame **intra-cycle**, freeing up the worker to process other behaviors. The frame is immediately put back into the work queue to be picked up whenever another worker gets around to it.
- `yield until cond` works like yield, but re-evaluates `cond` on each resume repeats until true.
- **Never `yield until` a condition whose producer is waiting on your signal**. This is a deadlock.

### Functions

#### Built-ins

```poesis
len(value)              # array/string/dict/set length
type(value)             # "number" | "fastnumber" | "string" | "bool" | "null" | "array" | "dict" | "set" | "fn" | "error" | "dynamic"
error(kind, message)    # construct an error value (see §9)
range(stop)             # lazy iterator: 0, 1, ..., stop-1
range(start, stop)      # lazy iterator: start, ..., stop-1
range(start, stop, step)   # lazy iterator: start, start+step, ...
dynamic()              # create empty dynamic
dynamic(dict_val)      # O(1) re-tag from dict to dynamic
```

Built-in functions support named arguments — as do `membrane.lib.*` functions and `membrane`/`territory` API calls:

```poesis
range(stop = 10, step = 2)   # [0, 2, 4, 6, 8]
range(start = 5, stop = 10)  # [5, 6, 7, 8, 9]
```

#### One Function Model

`fn` values and file-based registry functions are the same kind of thing: **`fn` is the inline form; function files are the file form.  Both produce fn values.**  `membrane.func.*` is an address namespace over fn values:

- A function file registers its fn value at its path address.
- Referencing a `membrane.func.*` address in expression position yields the fn value — assignable to a variable, passable as an argument.
- Assigning a fn value to a `membrane.func.*` address at runtime registers/replaces that entry.

#### User-defined Functions (file form)

Functions are defined by creating `.poesis` files in the `functions/` directory (or subdirectories).  The file path determines the namespace and name:

```
functions/math/add.poesis        →  membrane.func.math.add(a, b)
functions/string/reverse.poesis  →  membrane.func.string.reverse(s)
```

A function file begins with a `require` statement followed by the function body:

*Inline variant*

```poesis
require anyvar, a: number, b: number = 1, c = (3.14 * 0.001)

return a + b
```

*Braced variant*

```poesis
require {
    anyvar,
    a: number,
    b: number = 1,
    c = (3.14 * 0.001)
}
```

- `require` declares parameters with optional type annotations and optional defaults.
- Parameters with defaults are optional; parameters without defaults are required.
- Named arguments are supported at call sites: `membrane.func.math.add(a = 1, b = 2)`.
- Functions execute inline with the caller's membrane and territory.

**Default values:** when a parameter has a default, omitting the argument evaluates the default at each call site.  This means `add(5)` with `b = 10` as default produces `15`, not `null`.  The omission is distinct from an explicit `null` — the caller can tell whether an argument was provided or defaulted.

Any code before the `require` block in a registry function will error.  Comments may be freely placed before `require`.

#### First-Class Functions (fn values)

In addition to file-based registry functions, Poesis supports first-class function values defined inline with the `fn` keyword:

```poesis
# Named declaration at behavior top level
fn add {
    require a: number, b: number
    return a + b
}
result = add(1, 2)

# Anonymous expression
fn apply {
    require f: fn, arr: array
    result = []
    for item in arr {
        result.append(f(item))
    }
    return result
}
doubled = apply(fn { require x: number; return x * 2 }, [1, 2, 3])

# Store in collections
ops = {
    add: add,
    sub: fn { require a, b; return a - b }
}
result = ops.add(5, 3)

# Return from function
fn make_multiplier {
    require factor: number
    return fn {
        require x: number
        capture factor
        return x * factor
    }
}
mult5 = make_multiplier(5)
result = mult5(10)   # 50
```

**Closure semantics:**

- `fn` values are self-contained code values.
- Hoist aliases from the enclosing document are resolved at compile time.
- `capture` binds variables from the enclosing scope by deep copy at creation time.
- `capture` must appear after `require` and before other code.
- `fn` blocks cannot declare new `hoist` aliases.

**Calling:** function values use the same call syntax as registry functions: `myfn(1, 2)`.  Multi-return is supported.

**Recursion:** use `self(args)` to call the innermost enclosing function.  Named functions can also recurse by name.

**Type annotation:** `fn` is a valid type for `require`: `require f: fn`.

**Equality:** function values compare by source hash and closure contents.

**Serialization:** `fn` values round-trip through strings via `::string` and `membrane.lib.fn.from_string`.  Captured values are inlined in a `with { ... }` suffix.

**Recursion limit:** a per-behavior `fn_budget` (default 1000, configurable via `pragma fn_budget`) is a hard cap on total function calls per cycle, nested calls included.  Exhausting the budget quarantines the behavior.

#### Declared Return Packs (`returns`)

A function may declare its return pack after `require` and before `capture`:

```poesis
fn divide {
    require a: number, b: number
    returns (number, error)
    if b == 0 { return null, error("division_by_zero", "b is zero") }
    return a / b, null
}
```

- The pack is a flat list of type tags: `returns (number, error)`.
- Declared packs are enforced at compile time: every explicit `return` must match the declared count, and a function with a declared pack gets no implicit trailing return — it must return explicitly.
- A function whose last tag is `error` is **fallible**: callers must bind the error channel (`val, err = divide(10, 2)` — mandatory binding applies), and the call participates in `or`-coalescing.
- `return f()` with fallible `f` forwards the whole `(val, err)` pair.

### Hoist

`hoist` creates a compile-time alias for a dotted API path.  Expanded at compile — not visible to other behaviors.

```poesis
hoist membrane.signal as sig
hoist membrane.lib.string as strlib

sig.set("output", "some real data")
parts = strlib.split("a,b,c", ",")
```

**Rules:**

- `as` is mandatory.
- Alias must not conflict with built-ins or keywords.
- Behavior-local only.
- Assigning to a hoist alias is a compile error — **except** assigning a fn value to a `membrane.func.*` address, which registers it (see One Function Model).
- In behaviors, `hoist` must come before other code.
- In functions, `require` is first, then `hoist`.
- Any code other than `require` before a `hoist` will cause a compiler error.

### Collection literals

#### Array

```poesis
arr = [1, 2, 3]
arr[0] = 10              # index assignment
first = arr[0]
```

#### Dict

```poesis
d = {a: 1, b: 2}
d.a = 3                  # field assignment
d["c"] = 4               # index assignment
val = d.a                # or d["a"]
```

#### Set

```poesis
s = set(1, 2, 3)
union = s1 | s2          # set union
inter = s1 & s2          # set intersection
```

### String and collection methods

```poesis
# String methods — regex-based
start, end = membrane.lib.string.find("hello 123", "\\d+")       # [7, 9] or [-1, -1]
parts = membrane.lib.string.match("hello 123", "(\\w+) (\\d+)")  # ["hello", "123"]
result, count = membrane.lib.string.replace("a,b,c", ",", ";")   # ("a;b;c", 2)
arr = membrane.lib.string.split("a,b,c", ",")                    # ["a", "b", "c"]

# String methods — literal operations (no regex)
sub = membrane.lib.string.slice("hello", 1, 4)     # "ell" — byte-offset substring, clamps out of bounds
upper = membrane.lib.string.upper("hello")          # "HELLO" — ASCII uppercase
lower = membrane.lib.string.lower("HELLO")          # "hello" — ASCII lowercase
has = membrane.lib.string.contains("hello", "ell") # true — literal substring search
idx = membrane.lib.string.index_of("hello", "ell") # 2 — byte offset, or -1
repeated = membrane.lib.string.repeat("ab", 3)      # "ababab" — clamped to 0..10000
rev = membrane.lib.string.reverse("abc")            # "cba" — UTF-8 codepoint-aware
ch = membrane.lib.string.char_at("hello", 1)        # "e" — single-char string at codepoint index

# String methods — aliases and counting
sw = membrane.lib.string.starts_with("hello", "he")  # true — alias for has_prefix
ew = membrane.lib.string.ends_with("hello", "lo")    # true — alias for has_suffix
cnt = membrane.lib.string.count("banana", "a")       # 3 — non-overlapping literal count

# String methods — padding
lp = membrane.lib.string.left_pad("hi", 5, "*")     # "***hi" — left-pad to width 5
rp = membrane.lib.string.right_pad("hi", 5, "-")    # "hi---" — right-pad to width 5

# String methods — splitting
words = membrane.lib.string.words("  hello  world  ") # ["hello", "world"] — split on whitespace
lines = membrane.lib.string.lines("a\nb\nc")        # ["a", "b", "c"] — split on newlines
```

### Codecs

```poesis
# Hex encoding
encoded = membrane.lib.encode.hex_encode("hello")   # "68656c6c6f"
decoded, err = membrane.lib.encode.hex_decode(encoded)

# CSV (RFC 4180)
csv_str, err = membrane.lib.csv.encode([{name: "Alice", age: 30}])
rows, err = membrane.lib.csv.decode("name,age\nAlice,30")

# YAML (JSON-compatible subset)
val, err = membrane.lib.yaml.decode("name: Alice\nage: 30")

# Columnar (Arrow IPC)
bytes, err = membrane.lib.columnar.encode([{name: "Alice", age: 30}])
rows, err = membrane.lib.columnar.decode(bytes)
```

- **Hex:** lowercase hex encoding/decoding.  `hex_decode` returns an error for invalid input.
- **CSV:** first row is headers; subsequent rows become dicts.  Numbers are parsed as `number` type.  `csv_encode` sorts keys for deterministic output.
- **YAML:** JSON-compatible subset only (mappings, sequences, quoted strings, scalars).  Full YAML features (anchors, complex keys, multi-document) are not supported.
- **Columnar:** Arrow IPC format for columnar data exchange.  Supports `string`, `number`, `fastnumber`, `bool`, and `null`.  Useful for bulk data transfer between behaviors or external tools.

```poesis
# Array methods
arr.append(5)           # append to element
total = arr.sum()       # sum of numeric elements
lo = arr.min()          # minimum element
hi = arr.max()          # maximum element
deduped = arr.unique()  # deduplicated copy
pairs = arr.enumerate() # [[0, a], [1, b], ...]
found = arr.contains(x) # membership check
sorted, err = arr.sort()                        # stable sort, natural order
sorted, err = arr.sort(desc = true)             # descending
sorted, err = arr.sort(fn(a, b) { ... })        # comparator function
sorted, err = arr.sort(key = myfn)              # key function

# Dict methods
keys = d.keys()         # ["a", "b", ...]
vals = d.values()       # [1, 2, ...]
merged = d1.merge(d2)   # shallow merge
empty = d.is_empty()    # true if no entries
```

### Patterns

#### Producer / Consumer

```poesis
# producer.poesis
membrane.signal.set("data", processed)

# consumer.poesis
if membrane.signal.has("producer", "data") {
    data, err = membrane.signal.get("producer", "data")  # err is null — the has-guard ran first
    membrane.signal.free("producer", "data")
    # ... process ...
}
```

#### Busy Guard (Pipeline Stage)

```poesis
# Wait for upstream, produce for downstream
if not membrane.signal.has("upstream", "stage2") {
    return  # not ready yet, try next cycle
}
input, err = membrane.signal.get("upstream", "stage2")
result = process(input)
membrane.signal.set("stage3", result)
```

#### Idempotency Guard

```poesis
self_name = membrane.self.id()
if membrane.signal.has(self_name, "output") {
    return  # already done
}
# ... compute ...
membrane.signal.set("output", result)
```

#### Module Run

```poesis
result, err = membrane.module.run(["worker"], {worker: {input: "seeded"}})
# result: dict of "behavior:key" → value for every visible committed signal
# err: null on success, error string otherwise
```

#### Cross-Module Kill

```poesis
# forceful exit of own module
converge

# graceful stop of another module
territory.modules.converge(handle)

# instant kill of another module
territory.modules.kill(handle)

# free a module (kill if running, then clear signals and unregister)
territory.modules.free(handle)

# same for contexts: territory.contexts.converge / .kill / .free
```

#### Scheduled Behaviors

```poesis
# One-shot timer: inject signal after 5 seconds
membrane.timer.schedule("delayed_task", 5)

# Repeating timer: inject signal every 30 seconds
membrane.timer.every("heartbeat", 30)

# Cancel a pending timer
membrane.timer.cancel("heartbeat")
```

Timers inject a signal `{key: "fired"}` into the behavior's namespace after the specified delay.  `timer.every` repeats at the given interval until cancelled.  Timers are per-behavior — they fire in the context of the behavior that created them and cannot target other behaviors.

```poesis
# In a behavior — wait for the timer signal
if membrane.signal.has(self_name, "fired") {
    membrane.signal.free(self_name, "fired")
    # ... do recurring work ...
}
```

#### Async Work

```poesis
handle = membrane.llm.query.async(prompt, 4096)

# In a later cycle, check status and return if still pending
status = membrane.llm.status(handle)
if status == "pending" {
    return  # exit behavior for this cycle only; check again next cycle
}
response, err = membrane.llm.result(handle)
```

Note: `membrane.llm.result(handle)` returns `(null, error("not_ready", ...))` for a pending handle — poll again.  An unknown or reaped handle returns `(null, error("unknown_handle", ...))` — it will never resolve.

#### PubSub

PubSub is a global, topic-based message bus.  It is useful when behaviors in different modules (or different contexts) need to communicate without holding handles to each other — fire-and-forget publishing, or waiting for a message to arrive on a named topic.

```poesis
# Publisher — fire-and-forget
membrane.pub("events", {type: "tick", value: 42})

# Subscriber — suspends until a message arrives
msg, err = membrane.pub.await("events")
if err != null { return }
process(msg)

# Non-blocking receive — returns null if no message is queued
msg = membrane.pub.try_receive("events")
if (msg) { process(msg) }
```

**How it works:** `membrane.pub(topic, message)` publishes immediately and returns.  `membrane.pub.await(topic)` suspends the behavior until a message arrives on that topic — if one is already queued, it returns immediately.  `membrane.pub.try_receive(topic)` is non-blocking: it returns the oldest queued message or `null`.

**Queue depth:** each topic holds up to 100 messages.  Messages published when the queue is full are silently dropped.  Messages are values, not references — ownership transfers to the consumer.

**Scope:** pubsub is global across all modules and contexts in the process.  There is no isolation: any behavior can publish to any topic, and any behavior can subscribe.  If you need scoped coordination, use signals (intra-module) or territory (intra-context).

**Use signals for most coordination.** PubSub exists for the cases where you genuinely need decoupled, cross-module, cross-context communication without handle management — event buses, notification channels, loosely-coupled plugins.  For anything where both sides are in the same module, signals are simpler and participate in convergence.

#### HTTP Requests

```poesis
# Build and send an HTTP request (all via handles)
txn = membrane.http.create()            # handle — a fastnumber value
membrane.http.set_method(txn, "POST")
membrane.http.set_url(txn, "https://api.example.com/data")
membrane.http.set_header(txn, "Content-Type", "application/json")
membrane.http.set_body(txn, '{"key": "value"}')
txn, err = membrane.http.send(txn)      # suspends; handle stays valid

if err == null {
    status = membrane.http.status(txn)   # 200
    body = membrane.http.result(txn)     # response body
}
membrane.http.free(txn)                 # optional cleanup
```

HTTP handles are values — they pass through signals, territory, and scratch freely.  `send` suspends the behavior until the response arrives (typically the next cycle).  The handle remains valid after send; `status` and `result` can be called multiple times.  For fire-and-forget or polling patterns, combine with `module.run.async` or scratch-based state machines.

## 18. Author's Notes on the development process

Just some excerpts from a few conversations about the project's history...

"fwiw... that code you read? it's all ai-generated. Obviously it took a tremendous amount of human effort to achieve that kind of code quality. But it was achieved in record time because ai is very good at translating spec into code, and million-token context windows are a godsend for debugging and analysis...
What ai cannot do... Is teach the programmer the instinctive understanding of computer memory the way one summer 30 years ago a 10-year-old picked up C++ for dummies and learned to code up doubly-linked lists by hand back in the 90s. Memory and pointers are fucking everything."

"I haven't used memory-managed languages since that summer in the 90s. This project is my first foray back into the low-level stuff. And the AI is better than you'd think. But it still needs careful direction. I say "how do we build a lock-free scheduler" and it gives me a collection of known patterns, and I peruse each one and go "yeah... the bitmap pattern looks like what we need here... Implement that." And then the next conversation is "I want to free RSS after a spike... how do we do that?" And the ai says "Well in the bitmap pattern you really can't... you just accept the RSS waste. If you want to free things you need a mutex." And I say back "No, there's got to be a way to do it without mucking up the hot path... Can we double-buffer the bitmap?" And the ai says "that's not enough, it still won't work." And I push for why, and dig deeper, over and over and over until we build a 7-stage cold-path bitmap reduction algorithm that ignores ghost bits and has deeply-commented red-herring race conditions that are unreachable in principle..."

"And along the way, at each point where we think we've sorted the problem, I step back, and have that same convenient million-token context window create a work proposal document that captures all the important points of the discussion, including reasoning, discarded paths, whatever is relevant. And we build it into a plan of work. Before ever touching the code.
And then we do it again, and get another proposal on a different subject. And when we have built a collection of them, we plan out an implementation wave and plot all the work into an alignment doc full of checklists that reference back to the proposal library.
And at the end of it... I just sit back and tell the ai to "continue" over and over until all the checkboxes are ticked off. By that point, it has no leeway for guesswork.
And of course it fucks things up constantly. So we build test suites and bench harnesses. Every discovery turns into a regression test. We plan out every forward-thinking feature-verification test we can. We add leak-detection to the entire test suite, because we don't trust the code, regardless of who wrote it.
And then I take advantage of that million token context window all over again, in a new thread. I ask for documentation updates based on the git diffs. And then I review every doc change by hand. Because the invariants that were established back in the exploratory conversations might have drifted despite all the alignment effort. And if they have, the documentation is where that drift will become immediately apparent. And then when I find the drift, I don't fix the doc - I go back to the drawing board, and we plan out the changes again, and we do as many rounds of fixing as it takes to make the implementation align with the design constraints. And after it's all done, we update the docs to match the commit history again...
And it just goes round and round. The implementation waves, the fix waves, the documentation always catching up because it's the best verification tool on offer. I can read the code. I've read lots of the code. I still do read lots of code. I miss things when I read code, because code is complicated. When I read a single sentence where the AI documented a feature that was implemented misaligned with the intent, I notice that immediately.
And basically the goal is to reach at least some sort of stable state after each wave. This is all unreleased software still, so there's no CI/CD pipeline to satisfy. There's no customers depending on the latest fixes. There's no customers vulnerable to the latest bugs. But if it ever sees use, there will be. So I treat my implementation waves as releases anyway. I try to deliver on the commitments I set up, and validate correctness in every way available to me."

"Proposals die at the end of the wave, when we hit stable. And by die I mean we literally delete them, keeping only the git history. They remain retrievable. But if they were lost completely, they would not be missed. The current alignment doc is just edited/replaced constantly, so it is also captured in git, but the capture reflects the current state of work to some degree or other, and will be a messy in-progress thing, not a clean polished spec.
And proposals are ai-written, and if I'm honest, I don't always even read them before we move onto implementation. usually where I review is the active work checklist that gets pumped into the current alignment doc. But this means every once in a while the spec is wrong. I could spend the extra hours to pre-review every work order in exacting detail, but my experience has been that just plowing forward with what looks like a decent spec from the checklist and discovering the wrong-ness in-flight is significantly less time investment overall. It's not like a bad spec ruins the wave. The implementation is never perfect on the first go no matter how much I plan. Coders, whether ai or human, do things that make sense in the moment, but don't match the wider vision. If I were writing the code myself I'd drift from the spec probably more than the ai does... Not because of ill intent, but because coding is writing, and wandering, and researching, and mulling, and it gets in your head if you're doing it well.
So no. The proposals are not monuments. They are footprints in the sand, washed away almost as quickly as they arrive."

"Scaling the process is like scaling any other dev process. Ultimately, design-by-committee can only carry you so far. What you really get from the committee is the required invariants. The design must live up to those invariants. But the design is not in the committee's head. It forms in the mind of whoever the invariants get passed down to. Maybe they pass through a whole chain of people. But at some point, somewhere in the chain, someone has to actually wrap those invariants in an idea.  And then the idea is what's getting propagated. The invariants go with it, but the idea is what's getting implemented. The invariants are what it's measured by. And if you have multiple people each forming their own idea about how to meet the invariants, then you end up with parallel projects, not one project being shared across a team. Leadership in this sense isn't about authority over others. It's about fostering alignment to a single idea. And whether that alignment is between an agent and a single human, or spread across a whole team of people and ai, or just people, the end is the same: Your spec will never be perfect. Because the idea is not transferable. It is only translatable. And translation is lossy. The implementation will never be perfect, because implementation is translation. And translation is lossy. The documentation will never be perfect. Because documentation is translation. And translation is lossy. So you stop aiming for perfection, embrace the limits of the situation and start asking a different question... How do we minimize the waste? Everything we do carries some measure of waste. But how we choose to spend our time, and where we pay those costs is *everything*."

© 2026 Shane Plesner. All rights reserved.
