# Algorithm v4 — Unbounded-Depth Speculative Actions

**Goal.** Extend the paper's Algorithm 1 (breadth-focused, single-step) into
a tree-shaped recursion where every speculative state gets its own Actor
and Speculator call. The tree grows as deep as the rate-limit budget
allows, and collapses back to the matching branch whenever the ground
truth is confirmed.

This supersedes v3's "lazy grandchildren" rule. In v4 there is no
distinction between parent-level and child-level speculation — every node
in the tree is uniform. Depth is limited only by `max_inflight_actors`.

---

## 1. Inputs


| Symbol                    | Description                                            |
| ------------------------- | ------------------------------------------------------ |
| `s_0`                     | Initial confirmed state                                |
| `T`                       | Number of steps to run (horizon)                       |
| `actor(s, h, q)`          | Awaitable returning ground-truth action for state `s`  |
| `speculator(s, h, q)`     | Awaitable returning list of *k* candidate actions      |
| `transition(s, a)`        | Pure, cheap: applies action `a` to `s`                 |
| `policy(s)`               | Pure, cheap: returns API target `(h, q)` for state `s` |
| `semantic_match(â, a, d)` | Equality on actions, domain-aware                      |
| `k`                       | Speculator branching factor                            |
| `MAX_INFLIGHT`            | Max concurrent Actor API calls (default **13**)        |


## 2. State (maintained across event-loop iterations)

```
confirmed_t : int                 # highest decided depth
states      : Dict[int → state]
actions     : Dict[int → action]
root        : Optional[SpecNode]  # tree root; parent is None
inflight    : Dict[Task → SpecNode]  # Actor budget tracker
```

### 2.1 `SpecNode`

```
SpecNode {
    depth                : int
    s, h, q              : state + policy inputs
    parent               : SpecNode | None     # None at root
    a_hat_from_parent    : action              # the speculated action that led here
    a_actor              : Task | DEFERRED | None
    a_spec               : Task | None
    actor_resolved       : bool
    spec_resolved        : bool
    a_true               : action | None       # Actor's result
    predictions          : list[action] | None # Speculator's k candidates
    children             : list[SpecNode] | None
    effective_next       : SpecNode | None     # child matching a_true, after prune
    pruned               : bool
    abandoned            : bool
}
```

A node's **children are its predictions** — there is no separate Prediction
object.

## 3. Invariants

- **I1 (lossless):** the trajectory returned equals the strict sequential
baseline `s_0 →^a₀ s_1 →^a₁ … →^a_{T-1} s_T` where each `a_i` is what
`actor(s_i, policy(s_i))` would return.
- **I2 (budget):** `|inflight| ≤ MAX_INFLIGHT + 1`. The `+1` slack
accommodates the root's force-launched Actor on promotion.
- **I3 (root grounded):** `root.a_actor` is either a Task or resolved
(never DEFERRED). Force-launched on promotion if the child being
promoted had been DEFERRED.

## 4. Primitives

### 4.1 Actor launch (bounded)

```
launch_actor_task(node, shared=None):
    if shared is not None:
        return shared                  # sibling dedup; counts only once
    if |inflight| >= MAX_INFLIGHT:
        return DEFERRED
    t = create_task(actor(node.s, node.h, node.q))
    inflight[t] = node
    return t
```

### 4.2 Actor launch (forced, for root promotion)

```
launch_actor_force(node):
    t = create_task(actor(node.s, node.h, node.q))
    inflight[t] = node
    return t           # may temporarily push |inflight| to MAX_INFLIGHT+1
```

### 4.3 Speculator launch (unbounded, but gated)

```
launch_speculator_task(node):
    return create_task(speculator(node.s, node.h, node.q))
```

**Gating rule:** launch a Speculator for a node only if its Actor was
successfully launched (not DEFERRED). This prevents the Speculator tree
from fanning out exponentially past depths where Actors can no longer
follow. DEFERRED-Actor nodes stay "paused" — their Speculator fires only
at promotion.

### 4.4 Node constructors

```
make_root(s, depth):
    n = SpecNode{depth, s, policy(s), parent=None, a_hat_from_parent=None}
    if depth < T:
        n.a_actor = launch_actor_force(n)
        n.a_spec  = launch_speculator_task(n)
    else:
        n.actor_resolved = n.spec_resolved = true    # horizon
    return n

make_child(parent, a_hat, shared_actor):
    s' = transition(parent.s, a_hat)
    n  = SpecNode{depth=parent.depth+1, s=s', policy(s'),
                  parent, a_hat_from_parent=a_hat}
    if n.depth >= T:
        n.actor_resolved = n.spec_resolved = true    # horizon
        return n
    n.a_actor = launch_actor_task(n, shared=shared_actor)
    if n.a_actor is DEFERRED:
        n.a_spec = None                              # gated
    else:
        n.a_spec = launch_speculator_task(n)
    return n
```

### 4.5 Expansion (called once, when a node's Speculator resolves)

```
expand_children(node):
    if node.children is not None or node.predictions is None: return
    if node.abandoned: return
    seen : dict[(h,q) → Task] = {}
    kids = []
    for â in node.predictions:
        try: s' = transition(node.s, â)
        except: continue                   # illegal speculated action
        h', q' = policy(s')
        key    = (h', hashable(q'))
        shared = seen.get(key)
        child  = make_child(node, â, shared)
        if isinstance(child.a_actor, Task) and key not in seen:
            seen[key] = child.a_actor
        kids.append(child)
    node.children = kids
```

### 4.6 Abandonment

```
abandon_subtree(node):
    for cur in DFS(node):
        if cur.abandoned: continue
        cur.abandoned = true
        if isinstance(cur.a_actor, Task) and not cur.actor_resolved:
            release_actor(cur.a_actor)    # free budget slot
        # Do NOT cancel Tasks: a_actor may be shared with a surviving
        # sibling via dedup. Cancellation happens at final cleanup.
```

### 4.7 Pruning (called whenever a node has BOTH actor and spec resolved)

```
prune_on_resolution(node):
    if node.pruned or node.abandoned: return
    if not (node.actor_resolved and node.spec_resolved): return
    if node.children is None: return              # predictions were empty
    match = find_match(node, node.a_true)
    node.effective_next = match
    node.pruned         = true
    for c in node.children:
        if c is not match: abandon_subtree(c)
```

Pruning runs at every depth, not only at root. Speculative nodes with
a resolved Actor that doesn't match any speculated child immediately
free the whole child subtree's budget — useful work stops on wrong
branches as soon as we learn they're wrong.

## 5. Event loop

```
root = make_root(s_0, 0)
while confirmed_t < T and root is not None:
    watched = build_watch_set()               # DFS live tree, collect all
                                              # unresolved Actor/Spec Tasks
    if not watched: raise  # stalled
    done, _ = await wait(watched, FIRST_COMPLETED)

    for e in watched:
        if e.task not in done: continue
        if e.node.abandoned:   continue

        if e.type == 'actor':
            r = safe_result(e.task)
            release_actor(e.task)
            if error(r):
                if e.node is root: raise     # unrecoverable
                abandon_subtree(e.node); continue
            e.node.actor_resolved = true
            e.node.a_true         = r
            prune_on_resolution(e.node)

        elif e.type == 'spec':
            r = safe_result(e.task)
            e.node.spec_resolved = true
            e.node.predictions   = [] if error(r) else list(r)
            expand_children(e.node)
            prune_on_resolution(e.node)

    if root and root.actor_resolved and root.a_true is None:
        await handle_miss_synthetic()         # root Actor errored
    elif root:
        try_cascade()
```

### 5.1 Cascade

```
try_cascade():
    cascaded = 0
    while confirmed_t < T and root is not None:
        if not (root.actor_resolved and root.spec_resolved): return

        prune_on_resolution(root)
        a_true = root.a_true
        match  = root.effective_next

        if match is not None:
            # HIT
            stats.hits++  if cascaded == 0  else  stats.cascaded_hits++
            cascaded++
            states[confirmed_t+1] = transition(states[confirmed_t], a_true)
            actions[confirmed_t]  = a_true
            new_root = match
            new_root.parent = None
            root            = new_root
            confirmed_t++
            if confirmed_t < T and root.a_actor is DEFERRED:
                root.a_actor = launch_actor_force(root)
                if root.a_spec is None:
                    root.a_spec = launch_speculator_task(root)
        else:
            # MISS
            stats.misses++
            states[confirmed_t+1] = transition(states[confirmed_t], a_true)
            actions[confirmed_t]  = a_true
            abandon_subtree(root)
            confirmed_t++
            root = make_root(states[confirmed_t], confirmed_t) if confirmed_t < T else None
            return
```

**Note on `cascaded_hits`:** counts HITs that advance `confirmed_t` a
second-or-later time within one `try_cascade` invocation. This requires
the deeper node's Actor to have finished before the event that triggered
the cascade. Under deterministic latencies where deeper Actors always
launch later than their parents, real cascades only happen when
ground-truth Actor latency varies (e.g., network jitter puts a deeper
Actor ahead of its parent). The primary speedup comes from **depth
pipelining**: the next step's Actor is already in flight when the current
step confirms, so we save α per step even without a multi-hit cascade.

## 6. Budget accounting under promotion

When `match` is promoted to `root`:

- Sibling subtrees are abandoned in `prune_on_resolution`, releasing
their Actor slots.
- If `match.a_actor` is a Task (not DEFERRED), it remains inflight with
the same Task identity; no re-registration needed.
- If `match.a_actor` is DEFERRED, `launch_actor_force` adds a new Task
to `inflight`. Since abandonment freed slots, the budget check is
usually slack; the `+1` in I2 covers the rare case where abandonment
hasn't fully completed before promotion (e.g., deeply nested subtree).

## 7. Error handling


| Error site                                  | Algorithm response                                                                  |
| ------------------------------------------- | ----------------------------------------------------------------------------------- |
| Actor error at root                         | Unrecoverable — raise                                                               |
| Actor error at speculative node             | `abandon_subtree(node)` — that branch is dead                                       |
| Speculator error (any node)                 | `predictions = []` → guaranteed MISS if that node reaches root                      |
| Root Actor returns `None` (synthetic error) | `handle_miss_synthetic`: re-issue the ground-truth Actor and treat result as a MISS |


## 8. Stats


| Field                          | Meaning                                                              |
| ------------------------------ | -------------------------------------------------------------------- |
| `hits`                         | First HIT in a `try_cascade` call                                    |
| `cascaded_hits`                | Subsequent HITs in the same `try_cascade` call                       |
| `misses`                       | MISSes at `confirmed_t`                                              |
| `actor_launches`               | Real Actor API calls made (excludes DEFERRED and shared)             |
| `actor_dedup_reuse`            | Times a sibling shared another sibling's Actor Task                  |
| `actor_deferred`               | Times `make_child` marked Actor DEFERRED for budget                  |
| `actor_promoted_launch`        | Times a root was force-launched (initial root + DEFERRED promotions) |
| `speculator_launches`          | Speculator API calls made                                            |
| `max_inflight_actors_observed` | Peak `                                                               |
| `max_tree_nodes`               | Peak live (non-abandoned) node count in the tree                     |
| `max_tree_depth`               | Peak `node.depth` reached                                            |
| `max_cascade_run_length`       | Longest `cascaded` within a single `try_cascade`                     |


## 9. Differences from v3


| v3 (lazy grandchildren)                              | v4 (this)                                              |
| ---------------------------------------------------- | ------------------------------------------------------ |
| `Node` + `Prediction` (two types)                    | Unified `SpecNode`                                     |
| Eager at depth+1, lazy at depth+2                    | Eager at every depth (budget permitting)               |
| Tree is always exactly one `Node` wide at each depth | Tree is a full branching structure                     |
| `max_inflight_actors` defaults to `k+1`              | `max_inflight_actors` defaults to **13**, user-tunable |
| Speculator fired for every created node              | Speculator gated: only if Actor wasn't DEFERRED        |
| `spec_chain` dict keyed by depth                     | `root` + `parent`/`children` pointers                  |
| `cache_C` (dropped in v3 review)                     | Same: no cross-batch cache; per-node `(h, q)` dedup    |


## 10. Output artifacts (chess adapter)

Per-run directory `<out>/<run_id>/`:

- `log.txt` — per-move ACTOR/SPEC log lines
- `result.json` — moves, final FEN, wall time, run config, stats
- `stepsinfo.json` — per-step list: `step`, `action`, `kind`
(`hit` / `cascaded_hit` / `miss`), `predictions`, `player_id`,
`fen_before`, `fen_after`, `time_to_confirm_seconds`,
`inflight_actors_at_confirm`
- `stats.json` — stats dict alone (flat, for aggregation)
- `run_config.json` — run config alone (for aggregation)

This mirrors the breadth-focused `stepsinfo.json` structure so the same
analysis tooling works on both.

## 11. How to run

From the `chess-game/` directory:

```
uv run python speculative_workflow/depth_speculative_chess.py \
    --config config.yml \
    --stop-after 10 \
    --max-inflight-actors 13 \
    --trajectories-dir ./trajectories/test_trajectories
```

All flags are optional and fall back to `config.yml`:


| Flag                    | Fallback                          |
| ----------------------- | --------------------------------- |
| `--config`              | `config.yml`                      |
| `--stop-after`          | `config.game.stop_after`          |
| `--max-inflight-actors` | `config.game.max_inflight_actors` |
| `--trajectories-dir`    | `config.paths.trajectories`       |


