# Leither Structural Questions (Tweet owner-only writes)

**Canonical home:** `cfa532/Leither/cfa532-docs/integrations/tweet/`
**Date:** 2026-11-13
**Status:** questions posed to the Leither developers; answers pending. Where
a fact below is already documented, the document is cited; the questions are
about the parts that are not.
**Audience:** Leither maintainers (3and4/Leither), and anyone extending the
Tweet backend's caller-identity model.
**Related:** `DATA_AND_SYNC_CONTRACT.md` (same directory),
`../../reference/LEITHER_AND_MIMEI.md` §8 (identity, sessions, sign-in probes),
`TweetBackendApp` branch `owner-only-writes`, `Tweet-iOS/docs/OWNER_ONLY_WRITES_MEMO.md`.

---

## Why we are asking

The Tweet backend now identifies its users with Leither sessions: a client
signs in on the node (`LoginWithPPT` for node owners, `byname` for guests),
passes the sid as `usersid`, and the MApp resolves it with
`GetVar(sid, "userid")` against the account's recorded `authUid`. Writes that
span two accounts on two nodes — a comment on another author's tweet, from
2026-11-13 — are performed by the parent author's node at the request of the
writer's node: the writer's node verifies the writer, then makes a cross-node
`RunMApp` call under the application-author identity, which the target admits
only after checking that the caller's sid resolves to the same uid as its own
`BELoginAsAuthor` session (`requireAuthorSession`,
TweetBackendApp `session_entries.go:120`).

Every layer of that design rests on a structural fact about Leither. Some we
verified at runtime, some we could only read out of source comments and
changelogs, and one — the freeze on `BELoginAsAuthor` — we found only while
drafting this memo. We would like authoritative statements, because the
remaining steps of the refactor (likes, bookmarks, retweets, followers,
messages) will multiply the reliance on these facts.

## A. The application-author identity

**A1. Is the author identity valid network-wide?**
A `BELoginAsAuthor` session obtained on node A is accepted by node B when A
makes a cross-node `RunMApp` call — this has worked in production between our
nodes for over a year, and our new comment flow depends on it. `KEY_AUTH_GUIDE`
promises cross-node validity for *key* identities (same key logs in on any
node, `doc/KEY_AUTH_GUIDE.md:23-24`), but says nothing about the
application-author identity, whose key we have never seen: the node derives it
from the published app. Is the author session's network-wide acceptance a
contract, and is the author uid the same on every node?

**A2. What does the 2026-09-03 freeze on `BELoginAsAuthor` mean for an app
built on it?**
`lapi/backend.go:130-138` and `:218-221` mark `BELoginAsAuthor` 冻结（冻而不废）:
no external evolution, 禁止新增依赖, removal to be observation-driven per
`docs/KEY_AUTH_MECHANISM_AUDIT.md §2 F4` — a document that is not in the public
repository. Our entire write path has used `BELoginAsAuthor` since 2024
(`Install.md:940-944`), and the new peer-call guard adds another dependency in
spirit. Concretely: (1) may an application that already depends on it keep
adding call sites, or is that 新增依赖? (2) what does "observation-driven
removal" observe — is there a deprecation timeline? (3) is `BELoginAsApp`
(`lapi/backend.go:141-146`, currently unfrozen) the intended successor for an
app acting in its own identity, and does it open the same Mimei objects the
author identity owns?

**A3. Can an external client obtain an author session directly?**
`BELoginAsAuthor` sits on the published `BackEndStub`
(`lapi/backend.go:196`), alongside `BELoginAsApp`, `BESignPPT` and `BESign`
(`lapi/backend.go:214-217`). If a browser or mobile client can invoke it over
`/webapi/` or `/ws/`, then anyone can mint the very credential our peer guard
treats as "an app instance on a node", and the same holds for anything else
gated on author sessions. Is the backend API surface reachable from outside,
and if so, which of these methods are intended to be client-reachable?

**A4. Is `GetVar(authorSid, "userid")` contractual?**
The guide puts `GetVar(sid, "userid")` in the promised surface for user
sessions (`doc/KEY_AUTH_GUIDE.md:16`). We verified on gen8 (V0.24.73, 2026-11-13)
that it also returns a stable uid for an author session — our peer guard
compares two such values. Incidental or promised?

## B. Cross-node RunMApp: what the receiver can know

**B1. What does the target node authenticate before dispatch?**
Our calls carry the caller's author sid as the `sid` parameter. All we know
from our own source is a note that `RunMApp` "rejects BEOpenAppDataNode and
MMOpen data sessions" (TweetBackendApp `caps.go:123-124`). Does the target
validate `sid` against its own session store before the MApp runs, or does the
MApp see whatever string the caller placed in the parameter map? If the check
exists, is a *foreign* node's author session accepted by design (see A1)?

**B2. Can the receiving MApp learn which node called it?**
`lapi/backend.go:29-42` defines request keys the node injects:
`Request_NodeID` ("nid"), `Request_SidApp` ("sidapp", commented 应用sid, 作者身份)
and `Request_MMRoot` (commented 由节点注入). None is documented beyond that
comment. For a request arriving over a cross-node `RunMApp`: which of these
does the *receiving* node inject, and are they set by the transport (so a MApp
may trust them) or taken from the caller's parameter map (so it may not)? We
found no way for a MApp to name its caller; if one exists we would rather use
it than infer trust from session uids.

**B3. Is there a supported node attestation?**
Our guard currently proves "an instance of this application holding the author
identity called", not "node X called". If a node could sign a short statement
(its node id, the entry, a timestamp) that the receiving MApp can verify — or
if the transport attested the caller — peer-call trust could name nodes. Does
anything in this direction exist or is it planned (see also C)?

## C. Node app codes and friend nodes

**C1. Who mints a node app code?**
`friendByAppCode` (TweetBackendApp `auth.go:184`) resolves a presented
`nodeappcode` through the *receiving* node's session store:
`SessionGet(code, "nodeid")` and `SessionGet(code, "forapp")`. Nothing in the
public repository says how such a session comes to exist. Candidates we found:
the friend-node API `Invite`/`Accept`/`AddNode`/`NLogin`/`NGetGift`
(`api/Net.md:17-31`, a draft marked 草), and `GetVar("nodeappcode")`
(`Install.md:377-378`, V0.19.60 "节点应用码 getvar nodeappcode"; validated two
versions later at `Install.md:363-365`). Which is the minting path, and is it
documented anywhere we missed?

**C2. What does a resolved code prove?**
After resolution, is the statement "node N presented this code" verified by
the transport, or does resolution only prove that someone knows the code
string? What are the code's lifetime and its `forapp` binding (we do check the
latter)?

**C3. What was the intended client-facing use?**
The legacy JavaScript entry `add_tweet.js` described its authorization as
"Traditional: Frontend calls with nodeappcode validation" — a *frontend*
presenting a code. Was the intended flow that a browser client obtains a code
from its own node and presents it to the target node? If so, how did the
target node's session store come to hold that code?

## D. Session and identity scope

**D1. What is the status of `byname` (username/password) login?**
`KEY_AUTH_GUIDE.md:26-27` lists byname login under 不在承诺面（内部/实验，勿依赖）,
while the promised surface covers only key/PPT login. Our guest accounts — and
LifeAlbum's household users before us — register and sign in with
`Register(name, pass, "")` / `Login(name, pass)`, verified working on V0.24.47
and V0.24.73 (`../../reference/LEITHER_AND_MIMEI.md` §8.2). Since the same
guide promises that authentication data is node-local and never synchronized,
our guests can never sign in anywhere but their root node, which our new
comment design already accommodates. But: is byname scheduled for promotion,
change, or removal? Should new multi-user apps build on it?

**D2. Is a node- or app-bound credential planned?**
The guide strips `NodeId`, `AppID` and friends from user-minted SignPPT
payloads (`doc/KEY_AUTH_GUIDE.md:184`), and `SignPPT` since F4b takes a
structured request with `req.AidDecl` — an application identity declaration.
Both suggest the platform is moving toward bindings we cannot forge
client-side. Is there a planned credential that binds a session (or a signed
statement) to a specific node or app instance, and would it be visible to a
MApp through `GetVar`?

## E. Write authority and versioning

**E1. Does Leither enforce root-only versioning?**
Our backend refuses a write that reaches any node other than the object's
root, on the assumption that Leither itself will not stop a provider node
from committing a divergent version of a MiMei it merely holds a copy of (a
silent fork the root never learns about). Is that assumption right — is there
today, or planned, any enforcement that only the root (or the object's author)
may produce new versions?

**E2. Could a MApp act under the caller's identity for rights purposes?**
MiMei rights are per-uid, but everything our app writes is written under the
author identity, so Leither-level rights cannot distinguish our users — the
reason this whole refactor is application-level. Is there a planned way for a
MApp operation to execute with the *caller's* uid rights (or to assert a
caller uid to the rights layer), so owner-only writes could be enforced by
Leither instead of by each MApp?

## What we would like Leither to provide

In priority order, behind the questions above:

1. **A caller-node identity on cross-node `RunMApp`** — transport-set and
   visible to the receiving MApp (an injected request key, or an attestation
   the MApp can verify). This single fact would let peer-call trust name
   nodes, and the follower/like/bookmark/message halves of our refactor are
   waiting on exactly that boundary.
2. **A statement on the `BELoginAsAuthor` freeze** — successor, migration
   expectation, and whether the referenced audit (`KEY_AUTH_MECHANISM_AUDIT.md
   §2 F4`) can be made public.
3. **A statement on client reachability of the backend login family**
   (`BELoginAsAuthor`/`BELoginAsApp`/`BESignPPT`), and if they are reachable,
   MApp-context-only variants.
4. **Documentation of the node app code**: minting path, trust statement,
   lifetime, `forapp` binding.
5. Optionally, **execution under a caller uid** (E2), which would let the
   platform carry the owner-only rule natively.

## Evidence appendix

| Question | Verified at runtime | Source/comment evidence only |
|---|---|---|
| A1 | Cross-node author-sid calls work in production ≥1 year (gen8 ↔ other nodes, twbe) | No doc states author-identity scope |
| A2 | — | Freeze notice `lapi/backend.go:130-138, 218-221`; changelog `Install.md:940-944`; audit doc absent from public repo |
| A3 | Not probed (we deliberately did not try from a client) | `lapi/backend.go:196, 214-217` |
| A4 | gen8 V0.24.73, 2026-11-13: guard passes for own author sid, refuses foreign/user sids (`twbe` 1721 live probes) | Promised only for user sids, `doc/KEY_AUTH_GUIDE.md:16` |
| B1/B2/B3 | — | `caps.go:123-124`; `lapi/backend.go:29-42` |
| C1/C2/C3 | — | `auth.go:184`; `api/Net.md:17-31` (draft); `Install.md:363-365, 377-378`; legacy `add_tweet.js` header |
| D1 | byname Register/Login/SetUserInfo verified V0.24.47 + V0.24.73 (§8.2 of the technical foundation) | Promise scope `doc/KEY_AUTH_GUIDE.md:12-27` |
| D2 | — | `doc/KEY_AUTH_GUIDE.md:184` |
| E1 | Never attempted a provider-node fork (we refuse misrouted writes app-side) | Routing rationale in TweetBackendApp `routing.go` |
| E2 | — | Rights model in `../../reference/LEITHER_AND_MIMEI.md` §8.1 |
