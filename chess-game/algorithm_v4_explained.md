# Algorithm v4 — Unbounded-Depth Speculative Actions, Explained

Companion narrative for [algorithm_v4.md](algorithm_v4.md) and
[speculative_workflow/depth_algorithm.py](speculative_workflow/depth_algorithm.py).
If you've read [algorithm_v3_explained.md](algorithm_v3_explained.md),
skim §2 below for the key v3→v4 shift, then jump to §4.

---

## 1. The core idea

Every speculative state is uniform. When a Speculator returns *k*
candidate actions at any node — root, child, grandchild, great-grandchild
— we immediately:

1. Apply `transition` and `policy` to get each candidate's next state.
2. Launch an Actor call for each next state (budget permitting).
3. Launch a Speculator call for each next state too.

Those Speculators will fire eventually, and when they do, we recurse: their
returned candidates spawn the next level, and so on. The tree grows until
one of:

- The Actor budget `max_inflight_actors` is saturated. Further Actor launches
  are marked `DEFERRED`. Speculator launches are also gated — we don't fire a
  Speculator when its Actor is DEFERRED (otherwise the Speculator tree would
  explode exponentially even when Actors can't follow).
- A MISS abandons the whole tree. Fresh speculation starts at the new
  `confirmed_t`.
- The horizon `T` is reached at a leaf.

There is no notion of "grandchildren" — all descendants are just `SpecNode`s
with `parent`/`children` pointers. The tree recurses naturally.

---

## 2. Shift from v3

v3 explicitly baked in a **two-level** structure:

- **Node** = speculation at confirmed_t. Always exactly one.
- **Prediction** = the k candidates inside that Node.
- Prediction.a_actor_next and Prediction.a_spec_next — eagerly launched
  at depth +1.
- But no `Prediction.a_actor_grand` or `Prediction.a_spec_grand` —
  grandchildren existed only as an action-list stored on the Prediction,
  not as real API calls. Lazy.

v4 removes this asymmetry:

- **One type, `SpecNode`.** Its children are its predictions.
- Recursion is uniform. Depth +2, +3, +4, … all fire the same way.
- The only brake is `max_inflight_actors` (default **13**, user-settable
  via CLI or config).

### Why was v3 designed that way?

v3's lazy-grandchildren rule was a simple budget guardrail: with eager
recursion and unlimited budget, an oracle-accurate Speculator would
build `1 + k + k² + k³ + …` Actor calls. For k=3 and a deep tree that's
thousands. v3 sidestepped this by only being eager at depth +1 (cost
≤ k Actors per confirmed step).

v4 keeps the eager recursion — but bounds it explicitly. With
`max_inflight_actors = 13` and k=3, the tree can reach depth
`log_3(13) ≈ 2.3`, so typically 2 or 3 full levels of Actor fan-out
before budget saturates. After that, further expansions produce
DEFERRED-Actor leaves that remain until either abandonment frees budget
or promotion turns them into the new root.

---

## 3. The SpecNode tree

A node represents a (possibly speculated) state:

```
SpecNode
├── depth                   step index (root = confirmed_t)
├── s, h, q                 state + policy outputs
├── parent, children        tree pointers
├── a_hat_from_parent       action that led here (None at root)
├── a_actor                 Task | DEFERRED | None
├── a_spec                  Task | None
├── actor_resolved          has Actor returned?
├── spec_resolved           has Speculator returned?
├── a_true                  Actor's result
├── predictions             Speculator's k candidate actions
├── effective_next          child matching a_true, after prune
├── pruned                  already ran match-based sibling prune?
└── abandoned               this subtree was invalidated
```

Visually, immediately after the root Speculator has fanned out three
levels with *k = 3*:

```
root (depth=0, confirmed)
├── A1 (â=X)                    ← depth 1 (3 nodes total)
│   ├── B1 (â=M)                ← depth 2 (9 nodes total)
│   │   ├── C1 (â=...)          ← depth 3 — Actor DEFERRED if budget full
│   │   ├── C2 (â=...)
│   │   └── C3 (â=...)
│   ├── B2 (â=N)
│   │   └── ...
│   └── B3 (â=O)
├── A2 (â=Y)
│   └── ...
└── A3 (â=Z)
    └── ...
```

With `max_inflight_actors = 13` and k = 3: levels 0+1+2 hold `1 + 3 + 9 =
13` Actor Tasks, saturating the budget. Level 3 children (27 slots) get
DEFERRED Actors and no Speculators — they're leaves until promotion.

---

## 4. Event loop — what the algorithm actually does

One blocking point: `asyncio.wait(return_when=FIRST_COMPLETED)` on every
live Actor and Speculator Task in the whole tree. When a Task completes:

### `spec` event — Speculator returned
1. Set `spec_resolved = True`, stash `predictions = [â₁, â₂, …, â_k]`.
2. Call `expand_children(node)`:
   - For each â_i: `s' = transition(node.s, â_i)`, then `make_child`
     which launches Actor+Speculator (subject to budget + gating).
   - Sibling dedup: if two children land on the same `(h', q')`, they
     share one Actor Task (tracked locally in `seen` dict).
3. Call `prune_on_resolution(node)` — may be a no-op if Actor hasn't
   resolved yet.

### `actor` event — Actor returned
1. Set `actor_resolved = True`, stash `a_true = result`.
2. If this is root and result is an error: raise (unrecoverable).
3. If this is a speculative node and result is an error: abandon that
   subtree.
4. Call `prune_on_resolution(node)`:
   - If both `actor_resolved` and `spec_resolved` are now True,
     `find_match(node.children, a_true)` picks the child whose
     `a_hat_from_parent` matches, sets `node.effective_next = match`,
     and abandons every other child.
   - This runs at all depths — a speculative node whose Actor result
     doesn't match any of its speculated children frees the whole
     wrong-subtree budget immediately. Useful work stops fast on
     branches we've already learned are wrong.

### After processing events — `try_cascade`
If the root has both Actor and Speculator resolved:
- If `root.effective_next` exists (HIT): advance `confirmed_t`, promote
  the winning child to the new root. If the new root already has Actor
  resolved AND matches its own effective_next, keep cascading in-loop
  (costs no extra event-loop cycle).
- If no match (MISS): abandon the whole tree, advance `confirmed_t`,
  create a fresh root.

`cascaded_hits` counts the 2nd+ HIT within a single `try_cascade` call.
In practice, for deterministic latencies where a deeper Actor always
launches later than its parent, cascades only chain if Actor latency
varies (real LLMs do jitter). Even without chained cascades, you get
**depth pipelining**: when R0's Actor resolves and we promote to R1,
R1's Actor has been running since R0's Speculator returned (much earlier
than R0's Actor did), so we save roughly `α − β` per step compared to
sequential.

---

## 5. Budget & promotion — the tricky bits

### 5.1 Gating Speculator launches

If a child's Actor is DEFERRED, we skip the child's Speculator too. Why:
- The Speculator's output is only useful if it gets matched against a
  real Actor result — and that requires running the Actor eventually.
- If budget is saturated, the Actor can't run yet. So firing the
  Speculator now just produces grandchildren with DEFERRED Actors,
  which themselves need Speculators firing, and so on.
- Without this gate, the Speculator tree grows exponentially even when
  the Actor tree is bounded, burning API budget on Speculator calls
  whose results won't be matched.

At promotion, both Actor and Speculator get launched together (the
Speculator was deferred along with the Actor).

### 5.2 Promotion of DEFERRED Actor

When a HIT promotes a child to root:
- Sibling subtrees were abandoned in `prune_on_resolution` — this
  released their Actor budget slots.
- If the winning child had a real Actor Task: reuse it. `inflight`
  already tracks it.
- If the winning child had DEFERRED Actor: `launch_actor_force` creates
  a fresh Task, adds it to `inflight`. The invariant
  `|inflight| ≤ MAX_INFLIGHT + 1` allows a one-slot overshoot for this
  transient case (siblings' abandons may not have fully propagated).

### 5.3 Sibling dedup

When two of a parent's children speculate the same next-state `(h', q')`
(duplicate predictions from the Speculator), they share one Actor Task.
This:
- Saves an Actor API call.
- Complicates abandonment: we must NOT cancel a shared Task when the
  loser sibling is abandoned, because the winner still needs it.

The implementation handles this by never calling `task.cancel()` in
`abandon_subtree` — only `release_actor` (remove from `inflight`). Orphan
Tasks are cancelled at the very end of the run as cleanup.

---

## 6. Reading the run output

A run writes four JSON files under `<out>/<run_id>/`:

### `stepsinfo.json` — per-step analysis (parallel to breadth version)
```json
[
  {"step": 0, "action": "[e2e4]", "kind": "hit",
   "predictions": ["[e2e4]", "[d2d4]", "[g1f3]"],
   "player_id": 0,
   "fen_before": "rnbqkbnr/...", "fen_after": "rnbqkbnr/...",
   "time_to_confirm_seconds": 4.18,
   "inflight_actors_at_confirm": 7},
  ...
]
```

Mirrors the fields that `Speculative_Chess.py` writes in its
`stepsinfo.json`: move, predictions, hit/miss kind, per-step timing.
Extended with depth-algorithm specifics (`inflight_actors_at_confirm`,
`kind="cascaded_hit"`).

### `result.json` — run summary
moves, final FEN, wall-clock seconds, run config, stats.

### `stats.json`, `run_config.json` — flat per-run fields
Convenient for cross-run aggregation without parsing nested structure.

### Key stats to look at

| Stat                         | Healthy range                                                                 |
|------------------------------|--------------------------------------------------------------------------------|
| `max_inflight_actors_observed` | Close to `max_inflight_actors` — budget is being used                        |
| `max_tree_depth`             | ≥ 2 — we're speculating past immediate children                                |
| `max_tree_nodes`             | Close to `1 + k + k² + …` up to budget                                         |
| `hits / (hits+misses)`       | Speculator accuracy                                                            |
| `actor_promoted_launch`      | Small — high values mean budget was too tight and DEFERRED promotions dominated |
| `actor_dedup_reuse`          | Domain-dependent — high means Speculator emits duplicate predictions           |
| `cascaded_hits`              | Any positive value confirms multi-step cascades fired; 0 is still fine         |

---

## 7. CLI

```
# Use config defaults (max_inflight_actors from config.game.max_inflight_actors)
uv run python speculative_workflow/depth_speculative_chess.py \
    --stop-after 50

# Override budget at the CLI
uv run python speculative_workflow/depth_speculative_chess.py \
    --stop-after 50 --max-inflight-actors 20
```

Priority: `--max-inflight-actors` CLI flag > `config.game.max_inflight_actors`
> default 13.

---

## 8. Tradeoffs vs. v3

**Pros of v4:**
- Simpler code: one node type, uniform recursion.
- Scales further: if Speculator accuracy is high, the tree reaches
  max_inflight levels deep, exploiting every available budget slot.
- User-tunable: increase `max_inflight_actors` on higher-rate-limit
  providers; decrease under tight quotas without code changes.

**Cons / things to watch:**
- Dead speculative work: when Speculator is only moderately accurate,
  most of the k² / k³ Actor calls below depth 1 will be abandoned on the
  first MISS at `confirmed_t`. Budget spent on wrong branches is real.
- Speculator API cost: even with the Actor gate, every live SpecNode
  with a running Actor also has a running Speculator. For chess where
  both use the same model, this doubles the effective API-call rate.
  Consider providers where Speculator is a smaller/cheaper model.
- No explicit depth cap: budget-limited depth is less predictable than
  v3's D=1 hard cap. If you care about upper-bounding wall-clock per
  step, tune `max_inflight_actors` downward.

---

## 9. Files

- [algorithm_v4.md](algorithm_v4.md) — formal pseudocode spec
- [speculative_workflow/depth_algorithm.py](speculative_workflow/depth_algorithm.py) — generic implementation + 7-scenario harness
- [speculative_workflow/depth_speculative_chess.py](speculative_workflow/depth_speculative_chess.py) — chess adapter with `--max-inflight-actors`
- [config.yml](config.yml) — `game.max_inflight_actors: 13`
- [algorithm_v3.md](algorithm_v3.md), [algorithm_v3_explained.md](algorithm_v3_explained.md) — previous version (lazy grandchildren), kept for reference
