# butterfly 🦋

Reactive signal graph for [Irij](https://irij.online) — signals, cached
derivations, watchers, batching and time travel, written in Irij on top of
the effect system. Inspired by
[replikativ/spindel](https://github.com/replikativ/spindel).

```irij
use butterfly.core :open

fn demo ::: Reactive Console
  =>
  with reactive
    count   := signal 1
    doubled := compute (-> (deref count) * 2)
    watch doubled (o n -> println ("doubled: " ++ to-str o ++ " → " ++ to-str n))
    swap count (x -> x + 4)      ;; prints: doubled: 2 → 10
    deref doubled                ;; ⇒ 10
```

## Install

```toml
[seeds]
butterfly = "0.1"
```

```irij
use butterfly.core :open
use butterfly.combinators :open     ;; optional
```

Everything runs inside `with reactive` — the handler owns the graph.

## API

### Signals

| fn | spec | |
|---|---|---|
| `signal` | `_ Sig ::: Reactive` | create with an initial value |
| `deref` | `Sig _ ::: Reactive` | read; forces dirty computes, records dependencies inside a compute thunk |
| `peek` | `Sig _ ::: Reactive` | read without tracking or forcing (computes give their cached value) |
| `reset` | `Sig _ _ ::: Reactive` | write; returns the new value |
| `swap` | `Sig (Fn):eff _ ::: Reactive eff` | apply a function to the current value |
| `sig-version` | `Sig Int ::: Reactive` | writes seen (signals) / recomputes done (computes) |
| `reset-graph` | `() ::: Reactive` | drop everything — test isolation |

### Computes

`compute :: (Fn):eff Sig ::: Reactive eff` — a cached derived value. The thunk
re-runs when a dependency changes; dependencies are recorded automatically by
every `deref` inside it and re-recorded on each run, so branch switches drop
stale edges:

```irij
total := compute (-> (deref price) * (deref qty))
```

Unwatched computes are lazy (recompute on the next `deref`); watched ones are
re-forced eagerly at write time so their watchers fire. Propagation is
glitch-free: in a diamond `a → (b, c) → d`, one write recomputes each node
exactly once. Computes are read-only — `reset` on one is an error.

### Watchers

| fn | spec | |
|---|---|---|
| `watch` | `Sig (Fn):eff Watch ::: Reactive eff` | run `f old new` on every write; works on signals and computes |
| `unwatch` | `Watch () ::: Reactive` | remove; idempotent |

Watchers fire synchronously in registration order. Watching a compute makes it
eager. Don't write a signal from its own watcher.

### Batching

`batch :: (Fn):eff _ ::: Reactive eff` — defer notifications; one wave at the
outermost exit. Writes commit immediately, so reads inside see fresh values.
Per signal the wave carries the **first** old and **last** new value; watched
computes fire once with their pre-batch value. Nested batches are transparent.
If the thunk throws, writes made before the error still notify, then the error
propagates.

### Blocking reads

| fn | spec | |
|---|---|---|
| `listen` | `Sig _ ::: Reactive Time` | block until the next write, return the fresh value |
| `listen-or` | `Int Sig _ _ ::: Reactive Time` | same, with a deadline and fallback |

A loop of `listen` calls is the subscriber pattern for fibers. Fork inside
`with reactive` so the fiber inherits the handler (forks belong in plain
helper fns — `spawn` inside a `with` body isn't a supported shape):

```irij
fn print-changes ::: Reactive Time Console
  => s
  v := listen s
  println ("changed: " ++ to-str v)
  print-changes s                  ;; TCO'd — loops forever

fn main-flow ::: Reactive Time Console
  => s
  scope sc
    sc.fork (-> print-changes s)
    swap s (x -> x + 1)
    swap s (x -> x + 1)
```

v0.1 transport is a 5 ms poll on the version counter. Real fiber parking is
post-0.1.

### Combinators (`butterfly.combinators`)

| fn | spec | |
|---|---|---|
| `map-signal` | `(Fn):eff Sig Sig ::: Reactive eff` | derived signal; sugar over `compute` |
| `merge-signals` | `#[Sig] Sig ::: Reactive` | follows the latest write of any source |
| `debounce` | `Int Sig Sig ::: Reactive Time` | emit after `ms` of write silence |
| `throttle` | `Int Sig Sig ::: Reactive Time` | leading-edge rate limit |

### Time travel

| fn | spec | |
|---|---|---|
| `snapshot` | `Map ::: Reactive` | the whole graph as a value (persistent map, O(1)) |
| `restore` | `Map () ::: Reactive` | rewind; fires no watchers |

Handles created after a snapshot dangle once you restore — their ids get
reused. Snapshots hold lambdas, so they're in-memory only. Don't restore
mid-batch.

## Design

One `pub effect Reactive`, one handler owning the entire graph as handler
state (a single persistent map). Two rules shape everything:

- **Handler clauses are pure state transitions** plus a single `resume`. They
  never invoke a stored lambda — that keeps them tier-a and clear of the
  one-shot resume rule.
- **Lambdas are invoked caller-side.** Write ops return a notification bundle;
  the wrapper functions run the watcher callbacks and compute thunks. Effect
  transparency lives exactly where lambdas enter the library
  (`#[(Fn):eff]`, `(Fn):eff`), so row variables stay bound and the rest of the
  chain carries concrete rows.

Every public function carries a spec annotation — they're runtime contracts,
not documentation.

### v0.1 limits

- One global graph (handler state is a singleton), single-threaded mutation.
- Watcher and compute effects are checked where they're registered; the later
  dispatch runs them under an ambient frame.
- `listen` polls rather than parks.

## Deferred

Stream layer (`emit`/`done`/`error` + operator handlers), forkable
copy-on-write contexts, incremental delta algebra, JVM-interop concurrent
store with real parking, distribution.

## Development

```
irij test        # run tests/  (54 tests)
```
