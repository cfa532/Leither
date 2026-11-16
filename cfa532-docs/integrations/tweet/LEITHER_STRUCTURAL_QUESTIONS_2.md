# Leither Structural Questions II — follow-up from the build

**Canonical home:** `cfa532/Leither/cfa532-docs/integrations/tweet/`
**Date:** 2026-11-14
**Status:** The [maintainers replied on 2026-11-16](LEITHER_STRUCTURAL_ANSWERS_2026-11-16.md)
with findings from the V0.24.74 source. The `fromnid` decision remains open.
**Predecessor:** `LEITHER_STRUCTURAL_QUESTIONS.md` (2026-11-13) with the
maintainers' reply in `LEITHER_STRUCTURAL_ANSWERS_2026-11-13.md` — read
first; this memo only covers what building on those answers surfaced next.
**Baseline:** Leither V0.24.74 as before; TweetBackendApp `twbe` 1729 on the
debug network.
**Nature:** as before, these are questions about platform behavior our
correctness now relies on, or decisions that shape the remaining roadmap.
Evidence: each says where we observed it.

---

## Context: what was built on top of the answers

Since the reply, the Tweet backend completed its owner-only-writes port for
comments, likes, bookmarks, retweet counts and the follower side (structure in
`DATA_AND_SYNC_CONTRACT.md`, "Caller Identity" section). Every cross-party
write is now performed by the other party's node through peer entries guarded
by the platform-injected `nodeappcode`, bound to the acting user's root —
exactly the pattern the first reply made possible. Submissions of tweets and
comments are idempotent through a caller-generated `requestid`. These six
questions came out of that work.

## Maintainers' answers (2026-11-16)

| Question | Answer |
|---|---|
| 1. Deterministic MiMei IDs | The namespace is `(uid, appid, ext, mark, tp)`. Strict DHT verification enforces the derivation. It is outside the published API contract. Re-creation returns the existing ID and leaves content validation to the app. |
| 2. Local absence | For File MiMei, `OpenMM ver is empty` indicates a missing local version alias when metadata is present. Local metadata, version records and read permission have separate checks. A failed network lookup is inconclusive. |
| 3. Sync timing | The case-sensitive `SourcePeer` option takes a Leither node ID and skips provider discovery for the requested MiMei. `vers` is parsed but unused. The bounded MApp call can return after five seconds while the worker continues. |
| 4. Running app version | `GetVar("ver")` reports the node version. Local aliases can advance after an invocation loads its code. A build ID embedded in the app and returned by `health` identifies the code that handled the request. Serving nodes converge by pulling updates, with no guaranteed delay. |
| 5. `fromnid` | The maintainers found no recorded ruling. The target RPC receives neither a session argument nor request context for the transport identity. Exposing a verified caller identity requires a platform change. |
| 6. Author-login freeze | External Tweet calls are part of the observation used for removal decisions. The freeze has no removal deadline or CI enforcement. The available app-identity APIs cover some existing objects and app-level data, with gaps in per-object creation and first publication. |

## Questions

### 1. Is mark→mid derivation a stable contract, and how is it namespaced?

Our idempotent submission feature rests on the property documented only in a
source comment (`file_store.go`: "A deterministic mark resolves an existing
identity"): a client-generated `requestid` is passed as the `MMCreate` mark,
and a retry resolves the **same Mimei id**. Verified live twice (same
`requestid`, same mid, on `twbe` 1729).

- Is that derivation a promised contract, or incidental behavior that could
  change between versions?
- Is the derived id namespaced by object kind and application — or only by
  app id? A tweet and a comment share `ext=tweet-file-v1` under one app id,
  so a `requestid` used for both could collide. We defensively derive the
  quote half's id as `<requestid>-quote` on the client, but we would rather
  know the actual namespacing rule.

### 2. What is the canonical "this node holds no copy" signal?

A node with no copy of a user object answers `MMOpen(mid, last)` with
`OpenMM ver is empty` — observed live when a freshly registered account was
first referenced on a peer node. We now treat *any* read failure as "absent →
`MiMeiSync` once → re-read" (the `userRootNode` path in TweetBackendApp), which
conflates real errors with absence.

- Is `OpenMM ver is empty` the intended sentinel for "no local copy"?
- Are the failure modes distinguishable — no copy here yet, no such object
  anywhere, no rights to read — so a MApp can branch instead of guessing?

### 3. `MiMeiSync` timing for freshly published objects

The same fresh-account case showed the first `MiMeiSync` on the peer node
failing and a later one succeeding, minutes apart, with the publishing node
announced as provider all along. It reads as provider-discovery lag.

- Is there a recommended wait/retry pattern for syncing a just-published
  mid on another node?
- Can `MiMeiSync`'s param map carry a source hint (the publishing node's
  address) to skip discovery, the way cross-node `RunMApp` addresses `nid`
  directly?

### 4. Can a MApp learn its own running version?

During a rollout we watched a serving node answer `health` with an older app
version than the one just published (observed: `engagementDelegation` absent
on a node minutes after the publishing node shipped it, present later — no
redeploy in between). Our clients probe capabilities per node and fall back to
the old wire form, so this is survivable, but version skew across serving
nodes is currently invisible to us.

- Is there a `GetVar` (or another way) for a MApp to report the exact app
  version/build it is running, so `health` replies can carry it and clients
  can detect skew explicitly?
- Is there a documented expectation for how quickly serving nodes converge
  on a newly published app version?

### 5. `fromnid` — has the author ruled?

The first reply's §7.1 judged a transport-set caller-node identity on
cross-node `RunMApp` feasible, pending an author ruling. It remains the most
valuable platform change for us: it would replace the bearer-grade
`nodeappcode` binding (residuals: a code leaked from the expected node replays
≈1 h; the check must trust that node) with an attested caller identity, before
we port messages to the same pattern.

- Has the ruling been made? Is there anything we can do to help evaluate it
  (our node now logs every peer request's sid/nodeappcode/author shape — the
  evidence your §1.3 asked for, available on request)?

### 6. `BELoginAsAuthor` freeze — guidance for an app built on it

The Tweet app's entire storage layer writes under author sessions — every
create, backup, publish and peer delegation, not a few call sites. The freeze
is observation-driven via `[BELA-CALL]`, and the first reply notes
`BELoginAsApp` does not inherit author-level rights.

- Does an application with this usage profile count as a "known internal
  path" for the removal decision, or should we be planning a migration?
- If migration is expected: what is the sanctioned path for an app whose
  objects are all author-owned, given `BELoginAsApp` sessions do not carry
  author rights over them?

## What we would like, in priority order

1. A statement on mark→mid stability and namespacing (item 1) — a small doc
   paragraph would do; our idempotency feature is built on it.
2. Distinguishable absence signals (item 2) or confirmation of the sentinel.
3. The `fromnid` ruling (item 5).
4. A self-version capability for MApps (item 4).
5. Sync guidance or a source-hint param (item 3).
6. Freeze guidance for author-session-based apps (item 6).

## Evidence appendix

| Question | Observed | Where |
|---|---|---|
| 1 | Same `requestid` twice → same mid, one object, live | twbe 1729, gen8; probe transcripts in `Tweet-iOS/docs/OWNER_ONLY_WRITES_MEMO.md` §5 |
| 2 | `MMOpen(<new account>, last): OpenMM ver is empty` on a peer node | gen8→pcadmin-node, webguest1 registration test, 2026-11-14; fixed app-side in TweetBackendApp `userRootNode` |
| 3 | First `MiMeiSync` failed, later one succeeded, provider announced throughout | same test; provider IPs queried via `get_provider_ips` before and after |
| 4 | Serving node briefly answered `health` without `engagementDelegation` after the flag shipped | gen8 (publisher) vs pcadmin's serving node, 2026-11-14 rollout of twbe 1727 |
| 5 | Feasibility judgment + pending ruling | maintainers' reply §7.1 |
| 6 | App-wide author-session usage; freeze notice | `lapi/backend.go:130-138`; TweetBackendApp passim |
