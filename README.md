# butterfly 🦋

Reactive signal graph for [Irij](https://irij.online), inspired by
[replikativ/spindel](https://github.com/replikativ/spindel).

**Status: pre-0.1.** The library is being built as a series of small,
individually reviewed pull requests. The API below lands incrementally.

## Planned surface

| Concept | API |
|---|---|
| Signals | `signal` · `deref` · `peek` · `swap` · `reset` |
| Watchers | `watch` · `unwatch` |
| Derived values | `compute` (auto-tracked, cached, glitch-free) |
| Batching | `batch` |
| Blocking reads | `listen` · `listen-or` |
| Combinators | `map-signal` · `merge` · `debounce` · `throttle` |
| Time travel | `snapshot` · `restore` |

## Design

- One `pub effect Reactive` + one handler owning the whole graph as handler state.
- Handler clauses are pure state transitions; watcher/compute lambdas are
  invoked caller-side by wrapper functions, so effect rows stay checkable and
  the one-shot resume rule is never violated.
- Every public function carries a spec annotation.
- v0.1 discipline: one global graph, single-threaded mutation.

## Loop subscribers

`listen` blocks until the next write and returns the fresh value — a
loop of listens is the subscription pattern for fibers. Fork inside
`with reactive` (`scope`/`par`; note `spawn` is fire-and-forget) so
the fiber shares the graph:

```irij
fn print-changes ::: Reactive Time Console
  => s
  v := listen s
  println ("changed: " ++ to-str v)
  print-changes s          ;; TCO'd — loops forever

fn main-flow ::: Reactive Time Console
  => s
  scope sc
    sc.fork (-> print-changes s)
    swap s (x -> x + 1)
    swap s (x -> x + 1)
```

`listen-or ms s fallback` is the deadline variant.

## Development

```
irij test        # run tests/
```
