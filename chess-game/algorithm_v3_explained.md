# Algorithm 3 (v3) — Depth-Focused Speculative Actions, Explained

This document is a walkthrough of what the algorithm actually does, paired
with [speculative_workflow/depth_algorithm.py](speculative_workflow/depth_algorithm.py).
The formal pseudocode lives in [algorithm_v3.md](algorithm_v3.md); this file
is the narrative explanation with a worked example.

---

## 1. TL;DR — Answer to the headline question

**Yes.** When the Speculator at depth *d* returns with *k* candidate actions
`[â₁, â₂, …, â_k]`, the algorithm immediately launches **both** an Actor
call *and* a Speculator call for each of the *k* speculated next states at
depth *d+1*.

Concretely, for each candidate `â_i`:
1. Compute the speculated next state `ŝ_next = transition(ŝ, â_i)` (pure, cheap).
2. Compute its policy inputs `(h_next, q_next) = policy(ŝ_next)` (pure, cheap).
3. Launch an Actor API call on `(ŝ_next, h_next, q_next)` — this produces the
   candidate's *ground-truth next action* if we later confirm this branch.
4. Launch a Speculator API call on the same `(ŝ_next, h_next, q_next)` — this
   produces *that branch's own k grand-predictions*.

See [depth_algorithm.py:164-200](speculative_workflow/depth_algorithm.py#L164-L200).
`a_actor_next` is created by `launch_actor(...)` on line 185; `a_spec_next` is
created by `asyncio.create_task(speculator(...))` on line 192.

What we do **not** launch eagerly: the depth *d+2* API calls. That is the
lazy-grandchildren rule — §6 below explains why.

---

## 2. Why "depth" speculation at all?

The original Speculative Actions paper (Algorithm 1) is **breadth**-focused.
At confirmed depth *t* it runs the single ground-truth Actor in parallel
with the Speculator; the Speculator returns *k* candidate actions; if any of
them matches the Actor's output (HIT), we have the action for step *t* and
move on. If none matches (MISS), we fall back to the Actor and lose nothing
compared to a sequential baseline.

Algorithm 1 never pipelines past one step. Even on a HIT, step *t+1*
starts with a fresh Actor+Speculator pair and waits for them.

Algorithm 3 (this algorithm) **chains speculation**: on HIT at *t*, the
depth *t+1* work is already partly in flight (the speculative Actor for the
matching branch, and its Speculator) — we can immediately check if depth
*t+1* also HITs, then *t+2*, and so on. A run of consecutive HITs becomes a
**cascade**, where *m* steps resolve in roughly `α + m·δ` wall time instead
of `m·α` (α = Actor latency, δ = per-step event dispatch overhead ≪ α).

---

## 3. The state model — what lives where

At any moment, the algorithm maintains:

| Name               | Type                             | Meaning                                                               |
|--------------------|----------------------------------|-----------------------------------------------------------------------|
| `confirmed_t`      | int                              | Highest depth whose action is decided.                                |
| `s: Dict[t → state]` | confirmed states                | `s[0..confirmed_t]` are ground-truth.                                 |
| `actions: Dict[t → action]` | confirmed actions        | `actions[0..confirmed_t-1]`.                                          |
| `spec_chain`       | `Dict[depth → Node]`             | The speculative pipeline. See below.                                  |
| `inflight_actors`  | `Dict[asyncio.Task → (d, branch, gen)]` | Budget tracker for concurrent Actor API calls.                    |
| `current_generation` | int                            | Bumped on every MISS; stale futures/events are discarded.             |

### 3.1 `spec_chain` — the structure that makes "depth" work

By design, `spec_chain` usually contains **exactly one Node** at a time — the
Node at `confirmed_t`. The *breadth* of speculation (the *k* candidates) does
**not** live as separate Nodes. Instead, each Node holds a list of `Prediction`
objects, one per candidate. The shape is:

```
Node (depth=t, confirmed; has 1 Actor + 1 Speculator in flight)
 └── predictions: List[Prediction]       # filled when Node's Speculator returns
       ├── Prediction 0 (â₁)  [a_actor_next, a_spec_next, predictions_next?]
       ├── Prediction 1 (â₂)  [a_actor_next, a_spec_next, predictions_next?]
       └── Prediction 2 (â₃)  [a_actor_next, a_spec_next, predictions_next?]
```

- `Node.a_actor` = ground-truth Actor at depth *t*.
- `Node.a_spec` = Speculator at depth *t* (produces `â₁…â_k`).
- Each `Prediction.a_actor_next` = Actor at depth *t+1* **for that branch's**
  speculated next state.
- Each `Prediction.a_spec_next` = Speculator at depth *t+1* for that branch.
- Each `Prediction.predictions_next` = the *k* actions that Speculator
  returns (but we do NOT create Prediction objects or launch API calls for
  them — that's lazy-grandchildren, §6).

**This is why `max_spec_chain_size` in the stats is almost always 1.**
The *depth* of speculation is visible in `Node.predictions[i].a_actor_next`
and `Node.predictions[i].a_spec_next` being in-flight concurrently with
`Node.a_actor`. If *k=3*, that's **up to 4 Actor calls in flight
simultaneously** — one ground-truth at *t*, plus three speculative at *t+1*,
one per branch. `max_inflight_actors_observed ≈ k+1` is the real depth
pipelining signal.

---

## 4. The event loop — one blocking point

All concurrency funnels through a single `asyncio.wait(..., FIRST_COMPLETED)`
at [depth_algorithm.py:403-405](speculative_workflow/depth_algorithm.py#L403-L405).
Handler bodies never `await` on another API call — the only place we yield
control is back to that `wait`. Four event types can fire:

| Event        | Source                              | Handler action                                                                                                   |
|--------------|-------------------------------------|------------------------------------------------------------------------------------------------------------------|
| `actor`      | Node-level Actor (at `confirmed_t`) | Record `a_true`; if Speculator also resolved, resolve HIT/MISS.                                                  |
| `spec`       | Node-level Speculator               | Run `build_predictions` — launches depth *+1* Actors and Speculators. If Actor already resolved, resolve HIT/MISS. |
| `actor_next` | Prediction-level Actor (depth *+1*) | Mark `actor_next_resolved`; release budget slot. Keep the future's `.result()` around for promotion.             |
| `spec_next`  | Prediction-level Speculator         | Store the returned *k* actions into `pred.predictions_next`. **Do not** launch their children.                   |

`build_watch_set()` at [depth_algorithm.py:217-238](speculative_workflow/depth_algorithm.py#L217-L238)
collects every unresolved future — Node-level AND Prediction-level — into
the watch list. This is what lets a depth *+1* Actor resolution wake up the
event loop *before* a HIT at `confirmed_t` even fires. When the HIT
eventually fires, the depth *+1* Actor's result is already sitting in its
`Task.result()` and the cascade loop uses it for free.

---

## 5. HIT, MISS, and the cascade — worked example

Suppose *k = 3*, *T = 5*. Speculator is an oracle (always predicts the
truth first in its list).

### t = 0

```
launch: a_actor_0, a_spec_0      # both fire at t=0
```

At `t₀ ≈ 0` both API calls start. The Speculator is cheaper (β < α), so it
likely returns first.

### Speculator₀ returns first

`spec` event fires. `build_predictions` runs immediately:

```
pred[0]: â_hat = 2, ŝ_next = 2,  launch a_actor_next[0], a_spec_next[0]
pred[1]: â_hat = 1, ŝ_next = 1,  launch a_actor_next[1], a_spec_next[1]
pred[2]: â_hat = 3, ŝ_next = 3,  launch a_actor_next[2], a_spec_next[2]
```

Now **four Actor API calls are in flight**: `a_actor_0` (depth 0) and three
`a_actor_next[i]` (all at depth 1). Plus three new Speculator calls.

### Actor₀ returns

`actor` event at depth=0. Node.actor_resolved=True, a_true=2. Since
Speculator₀ already resolved (predictions exist), handler runs:

```
match = find_match(predictions, 2)   # → pred[0] since â_hat=2
handle_hit(node, pred[0], 2)
```

`handle_hit`:
1. Mark HIT. Abandon pred[1] and pred[2] (release their a_actor_next and
   a_spec_next budget slots; cancel is implicit — we just stop watching
   them and drop references).
2. `s[1] = transition(s[0], 2)`. `actions[0] = 2`. Delete `spec_chain[0]`.
3. `confirmed_t = 1`.
4. `promote_branch(pred[0], 1)` — pred[0] becomes the new `Node` at depth 1,
   reusing its already-in-flight `a_actor_next[0]` as the Node's `a_actor`,
   and its `a_spec_next[0]` as the Node's `a_spec`. **No new API calls.**
5. `cascade_loop()`.

### Inside `cascade_loop` — the depth payoff

`cascade_loop` at [depth_algorithm.py:279-311](speculative_workflow/depth_algorithm.py#L279-L311)
repeatedly checks: is the Node at `confirmed_t` fully resolved (both Actor
AND Speculator AND its predictions have been built)?

- If the depth *+1* Actor we pre-launched finished while we were waiting on
  Actor₀, its result is already sitting in `a_actor_next[0].result()`. When
  `promote_branch` runs, it checks `pred.actor_next_resolved` (set by the
  `actor_next` event handler earlier) and sets the new Node's
  `actor_resolved = True` with `a_true = r` on the spot.
- Same for the depth *+1* Speculator — if `pred.spec_next_resolved`, the new
  Node gets its predictions built right there from `pred.predictions_next`.

If both conditions hold, `cascade_loop` finds the match and consumes another
depth **without yielding to the event loop**. `stats.cascaded_hits` and
`stats.max_cascade_run_length` increment.

If either isn't resolved yet, `cascade_loop` breaks and we return to the
main `asyncio.wait` — which is still watching those futures. When they fire,
we naturally fall through the HIT path again.

### What MISS looks like

If `find_match` returns `None`: `handle_miss(a_truth)`.
1. `stats.misses += 1`. `current_generation += 1`.
2. `abandon_node` on every Node in `spec_chain` — release all budget slots
   for in-flight speculative Actors whose work is now invalidated.
3. `spec_chain.clear()`. Advance `confirmed_t`. Launch a **fresh**
   `a_actor` and `a_spec` at the new `confirmed_t`. Back to sequential.

The generation bump is the key — any event fired by an abandoned task will
arrive at the main loop, fail the `event.generation == current_generation`
check on [line 415](speculative_workflow/depth_algorithm.py#L415), and get
silently dropped. No stale state leaks across a MISS.

---

## 6. The eager-vs-lazy asymmetry

| Depth relative to confirmed | Actor launched? | Speculator launched? |
|-----------------------------|------------------|-----------------------|
| `t` (confirmed)             | Yes (ground-truth) | Yes                  |
| `t + 1` (Predictions of Node) | **Yes — one per candidate** | **Yes — one per candidate** |
| `t + 2` (grandchildren of a Prediction) | **No** — deferred until promotion | **No** — deferred until promotion |

Why the asymmetry?

Eager *+1* is a small, bounded cost: up to *k* Actor calls per level. Depth
pipelining requires them to be in flight so a HIT at *t* can consume *t+1*
without round-tripping.

Eager *+2* would fan out as *k × k = k²* Actor calls per level; eager *+3*
is *k³*; etc. At most one sibling at each level survives, so most of these
calls get abandoned. Budget pressure grows as `k^D` where D is the eager
depth, quickly swamping the rate limit and crowding out actually-useful
work.

Lazy grandchildren keep max inflight Actors bounded by *k + 1* per chain.
When `promote_branch` fires, the newly-confirmed Node has `predictions_next`
already sitting in its old Prediction — `build_predictions` rebuilds them as
fresh Prediction objects and launches *k* new Actors+Speculators at that
point. So cascades still work, they just cost one dispatch round per hop
instead of being fully pre-computed.

If profiling showed consistent idle Actor budget, you could add an
`eager_grandchildren=True` flag that pre-launches depth *+2* on `spec_next`
resolution — not currently wired up, but the data model already supports
it via `pred.predictions_next`.

---

## 7. What makes the implementation safe

Five subtle things can go wrong if you're not careful. v3 handles each:

### 7.1 Actor-before-Speculator race

The ground-truth Actor at `confirmed_t` can finish *before* its Speculator.
If so, we can't match yet — we don't know the predictions.

Fix: both the `actor` and `spec` handlers check "is the *other* side already
resolved?" and only then resolve HIT/MISS. Whichever lands second triggers.
Neither handler blocks on the other. See
[depth_algorithm.py:426-476](speculative_workflow/depth_algorithm.py#L426-L476).

### 7.2 Watching all Actors at all depths

`build_watch_set` adds **every** unresolved future — Node-level AND
Prediction-level — to `asyncio.wait`. A depth *+1* Actor resolution wakes
the loop immediately, so its budget slot is released in a timely way and
the cascade can see its result without polling.

### 7.3 Dedup-aware abandon

`build_predictions` deduplicates: if two siblings share the same
`(h_next, q_next)`, they share one Actor future (`seen` dict at
[line 166](speculative_workflow/depth_algorithm.py#L166)). On HIT, we abandon
every non-matching sibling — but if the winning sibling shares its future
with a losing one, naively calling `release_actor` on the loser would free
the budget slot the winner still needs. `abandon_prediction` takes an
`except_future` argument and skips the shared Task. See
[lines 202-208](speculative_workflow/depth_algorithm.py#L202-L208).

### 7.4 Generation guard

A MISS cancels a lot of pending work, but `asyncio` tasks can't be
instantaneously un-fired — their results may still flow into the event
loop. The `current_generation` counter, stamped on every Node and
Prediction at creation time and checked on every event, drops stale events
silently. See [line 415](speculative_workflow/depth_algorithm.py#L415).

### 7.5 Actor errors

API calls fail (rate limits, 500s). `safe_result` at
[line 240](speculative_workflow/depth_algorithm.py#L240) wraps every
`.result()` access.
- **Ground-truth Actor errors at `confirmed_t`:** unrecoverable, raise.
- **Speculative Actor errors at `> confirmed_t`:** that Prediction is
  abandoned; its branch is dead but the overall run continues.
- **Speculator errors:** treated as an empty prediction list → guaranteed
  MISS, falling back to sequential for this step.
- **Cascade promotes a Prediction whose Actor errored:**
  `handle_miss_synthetic` re-issues the ground-truth Actor at
  `confirmed_t` and continues.

---

## 8. Reading the run stats

From the chess adapter's `result.json`:

```json
"stats": {
  "hits": 7,
  "cascaded_hits": 1,
  "misses": 2,
  "actor_launches": 30,
  "actor_deferred": 0,
  "actor_dedup_reuse": 0,
  "speculator_launches": ...,
  "max_inflight_actors_observed": 4,
  "max_spec_chain_size": 1,
  "max_cascade_run_length": 1
}
```

How to read this (assuming *k=3*):

- **`max_inflight_actors_observed = 4 = k+1`**: depth pipelining is
  active — 1 ground-truth + 3 speculative Actors ran concurrently.
- **`max_spec_chain_size = 1`**: expected. Breadth lives inside a Node's
  predictions, not as separate Nodes (§3.1). Not a bug.
- **`max_cascade_run_length = 1`**: one HIT at `confirmed_t` chained into
  one additional HIT at `confirmed_t+1` without returning to the main
  `wait`. With real LLM latencies (α ≫ β doesn't always hold for GPT-5),
  cascade depth > 2 requires the depth *+1* Actor to finish *before* the
  depth *t* Actor — not guaranteed.
- **`hits + cascaded_hits = 8`, `misses = 2`**: 8 of 10 confirmed steps
  benefited from speculation.
- **`actor_launches = 30`**: sanity check: `1 (init) + 7 HITs × 3
  new speculatives + 2 MISSes × 4 relaunches (1 fresh Actor after MISS + 3
  speculative siblings once its Speculator returns) = 30`. ✓

---

## 9. Where to go next

- **Eager grandchildren (opt-in):** §6 sketches when this would help.
  Requires tracking `Prediction.a_actor_grand` futures and extending
  `build_predictions` to run recursively up to a configurable `D_max`.
- **Benchmarking vs. Algorithm 1 (breadth):** run the existing
  `Speculative_Chess.py` on a seeded trajectory, then
  `depth_speculative_chess.py` on the same seeds, and compare wall time.
  Lossless property means trajectories should be identical; only wall
  time differs.
- **Other domains:** `depth_algorithm.py` is domain-agnostic. An e-commerce
  or HotpotQA adapter mirrors `depth_speculative_chess.py` — provide
  `actor`, `speculator`, `transition`, `policy`, `semantic_match` and call
  `run_depth_speculative`.

---

## File map

- [speculative_workflow/depth_algorithm.py](speculative_workflow/depth_algorithm.py) — generic algorithm + 6-scenario harness
- [speculative_workflow/depth_speculative_chess.py](speculative_workflow/depth_speculative_chess.py) — chess adapter
- [algorithm_v3.md](algorithm_v3.md) — formal pseudocode spec
- This document — narrative walkthrough
