# Algorithm 3 (Final v3) — Depth-Focused Speculative Actions

*Extension beyond the ICLR 2026 "Speculative Actions" paper's Algorithm 1 (breadth-focused, k-way parallel). This spec is self-contained and incorporates five fixes over v2.*

## What changed from v2

1. **Case A no longer blocks the event loop.** The Actor-before-Speculator race is resolved by state, not by `await_with_timeout`. Either event handler runs HIT/MISS check if the *other* condition is already satisfied; otherwise the loop continues.
2. **`build_watch_set` watches every unresolved Actor future**, not just the one at `confirmed_t`. Adds a new `ActorNextEvent` case that just marks Prediction-level Actors resolved and frees their budget slot.
3. **Dedup-aware abandonment.** When sibling Predictions share an Actor future via per-batch `seen_hq` dedup, `abandon_prediction` takes an `except_future` argument so the winner's future isn't released out from under it.
4. **`cache_C` dropped.** Per-batch `seen_hq` inside `build_predictions` captures the common case without unbounded memory growth.
5. **Error handling on all `future.result()` / `event.result` accesses** via `safe_result`. Actor errors at `confirmed_t` abort; deeper errors abandon the branch; Speculator errors force a conservative MISS.

Design asymmetry (eager at Case S, lazy at Case SN) is preserved intentionally and now documented inline.

---

## 1. Data Structures

```
─── Prediction ──────────────────────────────────────────────
    branch_id:           int
    generation:          int
    abandoned:           bool = False
    â:                   Action
    ŝ_next:              State
    h_next:              APITarget
    q_next:              APIParams
    ā_actor_next:        Future | DEFERRED
    ā_spec_next:         Future | NONE
    actor_next_resolved: bool = False
    spec_next_resolved:  bool = False
    predictions_next:    list[Action] | None = None

─── Node ────────────────────────────────────────────────────
    depth:           int
    generation:      int
    ŝ:               State
    h:               APITarget
    q:               APIParams
    ā_actor:         Future
    actor_resolved:  bool = False
    a_true:          Action | None = None
    ā_spec:          Future
    spec_resolved:   bool = False
    predictions:     list[Prediction] | None = None

─── Global state ────────────────────────────────────────────
    spec_chain:         dict[int → Node]
    confirmed_t:        int
    s:                  dict[int → State]
    current_generation: int = 0
    inflight_actors:    dict[Future → (depth, branch_id_or_None, generation)]

─── Constants ───────────────────────────────────────────────
    T, k, MAX_INFLIGHT_ACTORS
    # No SPEC_TIMEOUT needed — the event loop stays non-blocking.
```

**Note:** No `cache_C` (see "What changed"). Dedup is batch-local inside `build_predictions`.

---

## 2. Helpers

### `safe_result` — error-safe future ingestion

```
function safe_result(fut) → Result | ErrorResult:
    try: return fut.result()
    except Exception as e: return ErrorResult(e)
```

### `launch_actor_call` — register for budget accounting

```
function launch_actor_call(h, q, depth, branch_id, generation) → Future:
    fut ← async_launch(h, q)
    inflight_actors[fut] ← (depth, branch_id, generation)
    return fut
```

### `release_actor_future` — free one budget slot

```
function release_actor_future(fut):
    if fut in inflight_actors:
        del inflight_actors[fut]
```

### `budget_available`

```
function budget_available() → bool:
    return len(inflight_actors) < MAX_INFLIGHT_ACTORS
```

### `build_predictions` — spawn k Predictions from a raw predictions list

```
function build_predictions(â_list, parent_ŝ, parent_depth, generation)
                                                  → list[Prediction]:
    seen_hq ← {}          # local dedup: (h, q) → Future
    predictions ← []

    for i in 0..len(â_list)-1:
        â_i ← â_list[i]
        ŝ_next ← f(parent_ŝ, â_i)
        (ĥ_next, q̂_next) ← π(ŝ_next)

        if parent_depth + 1 >= T:
            predictions.append(Prediction(
                branch_id=i, generation=generation,
                â=â_i, ŝ_next=ŝ_next,
                h_next=ĥ_next, q_next=q̂_next,
                ā_actor_next=DEFERRED,
                ā_spec_next=NONE,
                spec_next_resolved=True,
                predictions_next=None))
            continue

        # Per-batch dedup of Actor calls
        key ← (ĥ_next, q̂_next)
        if key in seen_hq:
            ā_actor_next ← seen_hq[key]      # SHARED across siblings
        elif budget_available():
            ā_actor_next ← launch_actor_call(
                ĥ_next, q̂_next, parent_depth+1, i, generation)
            seen_hq[key] ← ā_actor_next
        else:
            ā_actor_next ← DEFERRED

        # Speculator is cheap, always launched
        ā_spec_next ← async_launch(ĝ, ŝ_next, (ĥ_next, q̂_next))

        predictions.append(Prediction(
            branch_id=i, generation=generation,
            â=â_i, ŝ_next=ŝ_next,
            h_next=ĥ_next, q_next=q̂_next,
            ā_actor_next=ā_actor_next,
            ā_spec_next=ā_spec_next,
            spec_next_resolved=False,
            predictions_next=None))

    return predictions
```

### `abandon_prediction` — dedup-aware release

```
function abandon_prediction(pred, except_future=None):
    pred.abandoned ← True
    if (pred.ā_actor_next is not DEFERRED
        and not pred.actor_next_resolved
        and pred.ā_actor_next is not except_future):
        release_actor_future(pred.ā_actor_next)
```

### `abandon_node` — called on MISS, frees whole node

```
function abandon_node(node):
    if not node.actor_resolved:
        release_actor_future(node.ā_actor)
    if node.predictions is not None:
        for pred in node.predictions:
            abandon_prediction(pred)    # no shared-future risk here
```

### `build_watch_set` — watch EVERY unresolved future

```
function build_watch_set() → set[TaggedFuture]:
    watched ← {}
    for depth, node in spec_chain.items():
        # Node-level Actor (ground-truth candidate at this depth)
        if not node.actor_resolved:
            watched.add(tag(node.ā_actor, ActorEvent,
                           depth=depth, generation=node.generation))
        # Node-level Speculator
        if not node.spec_resolved:
            watched.add(tag(node.ā_spec, SpecEvent,
                           depth=depth, generation=node.generation))
        # Prediction-level futures
        if node.predictions is not None:
            for pred in node.predictions:
                if pred.abandoned: continue
                if (pred.ā_actor_next is not DEFERRED
                    and not pred.actor_next_resolved):
                    watched.add(tag(pred.ā_actor_next, ActorNextEvent,
                                   depth=depth,
                                   branch_id=pred.branch_id,
                                   generation=pred.generation))
                if (pred.ā_spec_next is not NONE
                    and not pred.spec_next_resolved):
                    watched.add(tag(pred.ā_spec_next, SpecNextEvent,
                                   depth=depth,
                                   branch_id=pred.branch_id,
                                   generation=pred.generation))
    return watched
```

### `semantic_match(â, a, domain)` — pluggable equivalence check

```
# Chess:       â == a                           (UCI strings equal)
# E-commerce:  same api_name + key_params
# HotpotQA:    normalize(â) == normalize(a)
# OS tuning:   |â - a| / a < tolerance
```

---

## 3. Main Algorithm

### Initialization

```
confirmed_t        ← 0
s[0]               ← s_0
spec_chain         ← {}
current_generation ← 0
inflight_actors    ← {}

(h_0, q_0) ← π(s_0)
ā_actor_0  ← launch_actor_call(h_0, q_0, 0, None, 0)
ā_spec_0   ← async_launch(ĝ, s_0, (h_0, q_0))

spec_chain[0] ← Node(
    depth=0, generation=0, ŝ=s_0, h=h_0, q=q_0,
    ā_actor=ā_actor_0, actor_resolved=False, a_true=None,
    ā_spec=ā_spec_0,   spec_resolved=False,  predictions=None)
```

### Event loop

```
while confirmed_t < T:
    watched ← build_watch_set()
    assert watched is not empty

    event ← first_completed(watched)

    # Generation guard — stale events from pre-MISS eras
    if event.generation != current_generation: continue

    # Abandonment guard — for all Prediction-level events
    if event is SpecNextEvent or event is ActorNextEvent:
        node ← spec_chain.get(event.depth)
        if node is None or node.predictions is None: continue
        pred ← node.predictions[event.branch_id]
        if pred.abandoned: continue
```

### Case A — Node-level Actor at confirmed_t or deeper

```
    if event is ActorEvent(depth=d):
        node ← spec_chain.get(d)
        if node is None: continue

        result ← safe_result(event.fut)
        if result is ErrorResult:
            if d == confirmed_t:
                raise EpisodeAborted(result.err)  # ground truth missing
            else:
                # Deeper Actor errored: free slot, mark resolved w/ None.
                # On promotion, if this branch wins, a cascade MISS fires.
                node.actor_resolved ← True
                node.a_true ← None
                release_actor_future(node.ā_actor)
                continue

        node.actor_resolved ← True
        node.a_true ← result
        release_actor_future(node.ā_actor)

        # Only drive HIT/MISS if Speculator predictions are ready.
        # Otherwise, the SpecEvent handler will trigger it.
        if d == confirmed_t and node.spec_resolved:
            resolve_hit_or_miss(node, result)
        continue
```

### Case S — Node-level Speculator

```
    if event is SpecEvent(depth=d):
        node ← spec_chain.get(d)
        if node is None: continue

        result ← safe_result(event.fut)
        if result is ErrorResult:
            # Conservative MISS — no matching prediction possible
            node.spec_resolved ← True
            node.predictions ← []
        else:
            node.spec_resolved ← True
            node.predictions ← build_predictions(
                result, node.ŝ, d, node.generation)

        # If Actor at this depth already landed and was waiting
        # on predictions, trigger HIT/MISS now.
        if d == confirmed_t and node.actor_resolved:
            resolve_hit_or_miss(node, node.a_true)
        continue
```

### Case AN — Prediction-level Actor resolved (at depth d+1 for some branch)

```
    if event is ActorNextEvent(depth=d, branch_id=i):
        # Guards already passed above.
        node ← spec_chain[d]
        pred ← node.predictions[i]

        result ← safe_result(event.fut)
        if result is ErrorResult:
            # Deeper Actor failed; abandon this branch. If it was
            # the would-be winner, promote_branch detects the
            # missing result and cascade falls through to MISS.
            abandon_prediction(pred)
            continue

        pred.actor_next_resolved ← True
        release_actor_future(pred.ā_actor_next)
        # Result is cached in the future object; consumed on promotion
        # or discarded if this branch is abandoned.
        continue
```

### Case SN — Prediction-level Speculator resolved (raw k preds for d+2)

```
    if event is SpecNextEvent(depth=d, branch_id=i):
        node ← spec_chain[d]
        pred ← node.predictions[i]

        result ← safe_result(event.fut)
        if result is ErrorResult:
            # Can't speculate beyond this branch; mark resolved
            # with empty. On promotion, promote_branch will see no
            # grandchildren to launch — pipeline resumes at depth d+2
            # freshly when the promotion finally happens.
            pred.spec_next_resolved ← True
            pred.predictions_next ← []
            continue

        pred.spec_next_resolved ← True
        pred.predictions_next ← result

        # DESIGN NOTE — LAZY GRANDCHILDREN:
        # We store predictions_next but do NOT pre-launch Actor or
        # Speculator calls for depth d+2 here. Rationale:
        #
        #   * Eager grandchildren would launch up to k*k Actor calls
        #     across sibling Predictions. Only one sibling wins per
        #     level, so k-1 of each sibling's grandchildren are waste.
        #     Budget pressure grows exponentially in depth (k^D).
        #
        #   * Lazy grandchildren launch only AFTER promotion (see
        #     promote_branch). This costs some pipeline depth (d+2
        #     Actor starts at promote time, not creation time), but
        #     keeps max inflight Actors O(k + 1) per chain.
        #
        # For k=3 and typical speedup ratios in the paper, lazy is the
        # right tradeoff. If profiling shows consistent idle Actor
        # slots, consider adding a speculation-depth budget D_max > 1.
        continue
```

### `resolve_hit_or_miss` — called exactly once per confirmed depth

```
function resolve_hit_or_miss(node, a_d):
    matching_pred ← None
    for pred in node.predictions:
        if semantic_match(pred.â, a_d, domain):
            matching_pred ← pred
            break
    if matching_pred is not None:
        handle_hit(node, matching_pred, a_d)
    else:
        handle_miss(a_d)
```

### `handle_hit` — commit, promote, cascade

```
function handle_hit(node, matching_pred, a_d):
    # Dedup-aware abandon: don't release the winner's own future
    except_fut ← matching_pred.ā_actor_next
    for pred in node.predictions:
        if pred is not matching_pred:
            abandon_prediction(pred, except_future=except_fut)

    s[confirmed_t + 1] ← f(s[confirmed_t], a_d)
    del spec_chain[confirmed_t]
    confirmed_t ← confirmed_t + 1

    if confirmed_t < T:
        promote_branch(matching_pred, confirmed_t)
    cascade_loop()
```

### `cascade_loop` — chase already-ready next HITs

```
function cascade_loop():
    while confirmed_t < T and confirmed_t in spec_chain:
        next_node ← spec_chain[confirmed_t]

        # With Issue 2's fix, actor_resolved is authoritative —
        # no .done() polling needed.
        if not next_node.actor_resolved: break
        if not next_node.spec_resolved:  break

        a_next ← next_node.a_true
        if a_next is None:
            # Deeper Actor had errored; treat as MISS at this depth
            handle_miss_synthetic()
            break

        match ← None
        for pred in next_node.predictions:
            if semantic_match(pred.â, a_next, domain):
                match ← pred
                break

        if match is not None:
            except_fut ← match.ā_actor_next
            for pred in next_node.predictions:
                if pred is not match:
                    abandon_prediction(pred, except_future=except_fut)
            s[confirmed_t + 1] ← f(s[confirmed_t], a_next)
            del spec_chain[confirmed_t]
            confirmed_t ← confirmed_t + 1
            if confirmed_t < T:
                promote_branch(match, confirmed_t)
            # loop continues: maybe next depth is also ready
        else:
            handle_miss(a_next)
            break
```

### `handle_miss` — restart pipeline

```
function handle_miss(a_ground_truth):
    s[confirmed_t + 1] ← f(s[confirmed_t], a_ground_truth)

    # One increment invalidates every stale event via the generation guard
    current_generation ← current_generation + 1

    # Free budget slots; futures keep running server-side but we ignore them
    for depth, node in spec_chain.items():
        abandon_node(node)

    spec_chain ← {}
    confirmed_t ← confirmed_t + 1

    if confirmed_t < T:
        (h_new, q_new) ← π(s[confirmed_t])
        ā_actor_new ← launch_actor_call(h_new, q_new, confirmed_t,
                                        None, current_generation)
        ā_spec_new ← async_launch(ĝ, s[confirmed_t], (h_new, q_new))
        spec_chain[confirmed_t] ← Node(
            depth=confirmed_t, generation=current_generation,
            ŝ=s[confirmed_t], h=h_new, q=q_new,
            ā_actor=ā_actor_new, actor_resolved=False, a_true=None,
            ā_spec=ā_spec_new,   spec_resolved=False,  predictions=None)
```

`handle_miss_synthetic()` is a variant used by cascade when a deeper Actor errored — same logic but `a_ground_truth` is already committed (was `a_true` from the errored Actor). In practice, re-issue the ground-truth Actor at this depth before committing. Implementation: `a_true ← await launch_actor_call(node.h, node.q, confirmed_t, None, current_generation).result()` then proceed as `handle_miss(a_true)`.

### `promote_branch` — move winner's Prediction into a new Node

```
function promote_branch(pred, depth):
    # Launch the Actor if it was budget-deferred
    if pred.ā_actor_next is DEFERRED:
        ā_actor ← launch_actor_call(pred.h_next, pred.q_next,
                                    depth, None, current_generation)
    else:
        ā_actor ← pred.ā_actor_next
        # Reclassify: it's now mandatory ground truth at `depth`
        if ā_actor in inflight_actors:
            inflight_actors[ā_actor] ← (depth, None, current_generation)

    if pred.spec_next_resolved:
        # Grandchildren are ready — build lazily NOW (the lazy→eager point)
        new_node ← Node(
            depth=depth, generation=current_generation,
            ŝ=pred.ŝ_next, h=pred.h_next, q=pred.q_next,
            ā_actor=ā_actor,         actor_resolved=pred.actor_next_resolved,
            a_true=None,
            ā_spec=pred.ā_spec_next, spec_resolved=True,
            predictions=build_predictions(
                pred.predictions_next, pred.ŝ_next,
                depth, current_generation))
        # Note: if actor_next_resolved was True, we know a_true can be
        # harvested from the future lazily when resolve_hit_or_miss
        # needs it. Simpler: harvest here.
        if pred.actor_next_resolved:
            new_node.a_true ← safe_result(ā_actor)
            if new_node.a_true is ErrorResult:
                new_node.a_true ← None  # cascade-loop handles this
    else:
        # Speculator still running; it'll fire SpecEvent naturally
        new_node ← Node(
            depth=depth, generation=current_generation,
            ŝ=pred.ŝ_next, h=pred.h_next, q=pred.q_next,
            ā_actor=ā_actor,         actor_resolved=pred.actor_next_resolved,
            a_true=None,
            ā_spec=pred.ā_spec_next, spec_resolved=False,  predictions=None)
        if pred.actor_next_resolved:
            r ← safe_result(ā_actor)
            new_node.a_true ← r if r is not ErrorResult else None

    spec_chain[depth] ← new_node
```

---

## 4. Correctness Invariants

These should hold after every event-loop iteration (good targets for `assert`s in implementation):

1. `len(inflight_actors) <= MAX_INFLIGHT_ACTORS + (1 for the mandatory confirmed_t Actor if it wasn't launched speculatively)`
2. For every `Node` in `spec_chain` with `predictions` set, exactly one of `actor_resolved` / waiting-for-ActorEvent is true per Prediction.
3. Every `Future` in `inflight_actors` is held by at most one `Node.ā_actor` OR at least one non-abandoned `Prediction.ā_actor_next`. (Sharing across abandoned Predictions is fine.)
4. On MISS, every pre-MISS future either (a) is no longer in `inflight_actors`, or (b) fires with a generation that gets filtered.
5. The committed trajectory `s[0..confirmed_t]` is identical to the non-speculative baseline's trajectory on the same actions (lossless property, paper Assumption 1–2).

---

## 5. Complexity / Budget

- **Max inflight Actors per episode:** `k + 1` — one mandatory Actor at `confirmed_t`, up to `k` speculative Actors at `confirmed_t + 1`. Lazy grandchildren mean depth `≥ confirmed_t + 2` holds zero inflight.
- **Max inflight Speculators:** unbounded by design (Speculators are cheap and don't count against `MAX_INFLIGHT_ACTORS`). In practice bound is ~`k * (chain depth)` since each surviving Node has one, plus `k` within the current frontier.
- **Wasted calls per MISS:** up to `k + 1` Actor calls (one at `confirmed_t`, `k` at `confirmed_t + 1`). Does not grow with chain length because of lazy grandchildren.
- **Memory:** `O(k * chain_depth)` Predictions held simultaneously. No unbounded `cache_C`.

---

## 6. Implementation Notes for Python asyncio

- `first_completed` ≡ `asyncio.wait(..., return_when=FIRST_COMPLETED)`.
- Tagged futures: wrap with a small dataclass `class Tagged: fut, event_type, depth, branch_id, generation`.
- `async_launch` ≡ `asyncio.create_task(...)` for Speculator; `asyncio.create_task(http_client.post(...))` for Actor.
- **Invariant check:** grep the handler bodies for `await` — there should be none. All blocking should happen only at `first_completed(watched)`.
- **Never call `fut.result()` on an unresolved Future** — `safe_result` assumes the future completed (returned by `first_completed`) or polls `fut.done()` first.
- Prefer a single event-loop thread; the `inflight_actors` dict needs no lock because it's mutated only in handler code.

---

## 7. Verification Plan

1. **Desk-check**: trace the algorithm on (a) all-HITs T steps; (b) MISS at t=2 with deeper Predictions mid-flight; (c) budget exhaustion at t=1 forcing DEFERRED; (d) dedup-sibling HIT (two Predictions map to same `(h, q)`).
2. **Simulation harness**: stub Actor (`asyncio.sleep(α) + fixed action`) and Speculator (`asyncio.sleep(β) + k deterministic predictions`); verify resulting `s[0..T]` equals sequential baseline.
3. **Budget assertion**: `assert len(inflight_actors) <= k + 1` after every event.
4. **A/B on chess**: run the existing breadth impl (`speculative_workflow/Speculative_Chess.py`) and this depth impl on the same seeded trajectory from `chess-game/trajectories/sample_trajectories/`. Depth should dominate on wall-clock for long HIT-runs; should be within epsilon on frequent-MISS runs.
