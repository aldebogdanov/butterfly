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
| Blocking reads | `listen` · `listen-timeout` |
| Combinators | `map-signal` · `merge` · `debounce` · `throttle` |
| Time travel | `snapshot` · `restore` |

## Design

- One `pub effect Reactive` + one handler owning the whole graph as handler state.
- Handler clauses are pure state transitions; watcher/compute lambdas are
  invoked caller-side by wrapper functions, so effect rows stay checkable and
  the one-shot resume rule is never violated.
- Every public function carries a spec annotation.
- v0.1 discipline: one global graph, single-threaded mutation.

## Development

```
irij test        # run tests/
```
