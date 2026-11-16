# Leither Structural Questions — Answers from the maintainers

**Provenance:** received 2026-11-13 from the Leither maintainers (3and4), as
`tweet-structural-answers-2026-11-13.md`; preserved verbatim below. Their
baseline: Leither V0.24.74, root repository HEAD `1d34cf41`. It cites files
from their private root repository (`docs/…`, `planning-artifacts/…`,
`frame/…`, `app/…`, `mimei/…`, `pnet/…`) that are not in the public
`3and4/Leither` repository.

**Companion questions:** `LEITHER_STRUCTURAL_QUESTIONS.md` (same directory).

**2026-11-16 correction:** The [follow-up reply, section 5](LEITHER_STRUCTURAL_ANSWERS_2026-11-16.md#5-题-5fromnid-的裁决状态与可行性订正)
supersedes the `fromnid` feasibility details in sections 3 and 7.1 below.
`NodeApiClient.Sid` is available to the caller. The receiving `RunMApp`
RPC has no session argument or transport identity context. A verified
`fromnid` requires a platform change, and the decision remains open.

**What we changed because of it:** the comment peer guard now binds the
platform-injected `nodeappcode` to the expected delegating node, and the
local branch compares against the node-injected `author` key instead of
logging in as the author (TweetBackendApp `owner-only-writes`,
`requirePeerCall`).

---

# 对《Leither Structural Questions (Tweet owner-only writes)》的回答

> **答复对象**：`cfa532/Leither/cfa532-docs/integrations/tweet/` 的结构性问题（2026-11-13）
> **核对基线**：Leither `V0.24.74`，根仓 HEAD `1d34cf41`（工作树含 docs/memory 未提交改动，不影响本文结论）
> **证据口径**：每条结论给出代码/文档位置；凡平台未承诺、或需要对方核对的地方，明确标注。
> **给对方的英文速览**：见 §8（逐题一句话，可直接贴回 memo）。
> **性质**：本文是维护方对**当前代码行为**的判读，不构成对 `lapi` 公开契约的变更；涉及对外承诺的表述，以 `doc/KEY_AUTH_GUIDE.md` 的"承诺面/不在承诺面"为准。

---

## 0. 结论速览

| 题 | 结论 | 承诺状态 |
|---|---|---|
| A1 | 作者 **uid** 来自应用记录 `MMInfo.Author`，同一条记录在各节点一致；但 **author sid 不是全网会话**（会话是节点内存态）。跨节点 RunMApp **会在转发前删掉 `sid`**，目标端不会"接受"一个外节点 sid。 | 未承诺（跨节点同步明确属未承诺范围） |
| A2 | "冻结"是正式落档的裁决：**冻而不废、禁止新增依赖**（含新增调用点）、删除观测驱动（`[BELA-CALL]` 日志），**无期限、无 CI 强制**；`BELoginAsApp` 是替代方向（应用身份 + 节点代劳），但**不继承作者全权**。 | 已裁决并落档 |
| A3 | 四个方法（`BELoginAsAuthor`/`BELoginAsApp`/`BESignPPT`/`BESign`）**都不在 RPC 面**，外部 client 无法经 `/webapi/`、`/ws/`、`/ws3/` 调用。RPC 面只有 `WebApi` 的具体方法。 | 平台结构事实 |
| A4 | `GetVar(authorSid, "userid")` 与用户会话走**同一条实现**（`GetUidFromSession` 读会话里的 `userid`），对 author 会话返回 `authID`；但文档只承诺用户会话，author 会话属"现有行为"。 | 未承诺 / 行为稳定 |
| B1 | 目标端在跑 MApp 前用 `PreLogin` 在**本地会话表**解析 `sid`：空 → guest 放行；不可解析的非空 sid → 直接拒绝；跨节点转发来的请求 `sid` 已被删除 ⇒ 目标端按 guest 跑。**不会**采信外节点 sid。 | 平台结构事实 |
| B2 | `nid` = 调用方填的目标节点（转发前删除）；`sidapp` = 仅 CLI `runapp` 本地设置，容器不注入；`mmroot` = 节点按 Manifest 注入（Manifest 未声明时调用方可自带）；另 `author` 由节点按应用记录覆盖。 | 见逐题 |
| B3 | 没有"节点签名声明"给 MApp 验证。传输层确实以节点 PPT 登录（`NLogin`）认证调用节点，但身份**不进入 Request**；`nodeappcode` 是唯一 app 可见的调用方节点线索，但它是 bearer 码，不是 attestation。 | 无此能力 |
| C1 | 铸造路径 = `getvar?name=nodeappcode&arg0=<aid>`（`frame/Auth.go:getNodeAppCode`），懒创建 1h 会话并写 `nodeid`/`forapp`。`api/Net.md` 的 `Invite`/`Accept`/`AddNode` 是草稿且代码已注释，`NGetGift` 返回 `ErrTBD`；与 app code 无关。 | 未文档化为契约 |
| C2 | 解析只证明"某人知道这串码"；码在**接收节点**铸造，`nodeid` = 铸造时调用方会话的 uid（兄弟节点身份 27 位 key id，普通用户身份也可），TTL ≈1h，`forapp` 需消费方自己校验，服务端不强制。 | 非安全边界 |
| C3 | 仓库内唯一消费者是 JS MApp `LeitherService/ledger.js`（`getFriendByAppCode`）；平台自动路径 = 跨节点 `RunMApp` 先向目标取码再注入 `nodeappcode`。"前端拿码给目标节点"跨节点不成立（目标会话表里没有对方节点铸的码）。`add_tweet.js` 不在本仓，无法确认其原意。 | 需对方核对 |
| D1 | byname 是**遗留/内部路径，明确不在承诺面**；无提升/删除计划；认证数据节点本地、不跨节点同步；注册接口无鉴权无节流。新多用户应用**不应**建在 byname 上。 | 未承诺 |
| D2 | 没有"节点/应用实例绑定会话"的计划性能力。现有近似件：`CertFor=App` 的 backend PPT（节点系统钥签名、非登录型）、`Sign`/`SignPPT` 的 `AidDecl`（应用身份声明 + 拒绝不符）、signacl（应用会话代签一律拒）。 | 未承诺；有开放评估项 |
| E1 | **没有** root-only 版本约束。版本写入按 mid 的 uid 权限判定；作者 uid 在任何持有该应用记录的节点上都是 `DataRight_All`（且 `BELoginAsAuthor` 不需要作者私钥即可拿到该会话）⇒ 提供者节点可以提交分叉版本，root 不会自动知道。 | 无此机制 |
| E2 | 没有"以调用者 uid 执行"的通用能力，也没有"向权限层断言 uid"的参数；权限层只从会话或数据区属主解析 uid。**部分可用**：`MMOpenAppDataUser`（MApp 可见）可开一个以调用者为属主的 per-user 应用数据区——前提是持有调用者本人会话（仅本节点请求有）。 | 无计划；L4 评估项 |

---

## 1. 先订正两处机制（A1/B1 的读法都依赖它）

### 1.1 跨节点 `RunMApp` 的真实身份链

```
A 节点内 MApp（作者会话 sid_A，来自 BELoginAsAuthor）
  └─ lapi.RunMApp(entry, request{nid=B, sid=sid_A, ...})     // 应用间组合
       └─ A.Apps.RunMApp → GetMApp                            app/MFileApps.go:290
            nid != 本节点 → checkRemoteMAppRight(mapParam)    app/MFileApps.go:301
              └─ uid = GetUidFromSession(sid_A) == 目标应用记录 Author → 放行
            delete(mapParam, "nid"); delete(mapParam, "sid")  app/MFileApps.go:306-307
            → createRemoteMApp2(B)                            app/MFileApps.go:309
                 └─ remoteMApp.RunMApp                         app/MApp.go:188
                      ├─ wa.GetVarObj(nodeappcode, aid)        app/MApp.go:203（在 B 上取码）
                      └─ wa.RunMApp(entry, request, args)      app/MApp.go:219
                           └─ B 节点：GetMApp（nid 已删）→ 本地路径
                                PreLogin(sid="") → guest（uid=""）  user/Auth.go:561
```

三个要点：

1. **判权发生在发起节点**，不是目标节点；目标节点不重复判"你是不是作者"（实测与代码见 `planning-artifacts/milestone-2026-09/T1/m12-cross-node-mapp-2026-09-12.md` §1/§4）。
2. **转发前 `sid` 被删掉**（`app/MFileApps.go:306-307`）。所以目标端 MApp 看到的 `Request["sid"]` 是空（除非调用方在请求里另带 `ppt`，由目标端 `ParamAuth` 重新认证并换成本地 sid）。
3. 目标端 MApp 自己调 `BELoginAsAuthor()` 时，拿到的是**目标节点本地会话**，uid = 目标节点本地应用记录的 `Author`（`app/MFileApps.go:1128-1133`）。

⇒ 你们观察到的"同一个作者 uid 在两端成立"，真实机制是：**两端各自从同一条应用记录推导同一个 uid**；不是"一个会话过了网"。这也解释了为什么"一年多在跑"却没有任何跨节点会话同步。

### 1.2 三套"身份"不要混

| 身份 | 载体 | 有效范围 | 证据 |
|---|---|---|---|
| 用户 / 作者 uid | 会话里的 `userid`（或 `appid`） | 由会话所在节点解析 | `frame/ApiNode.go:33-64`、`user/Auth.go:561-608` |
| 应用作者身份 | `BELoginAsAuthor()` → `CreateUserSession(authID)` | **仅本节点会话表** | `app/MFileApps.go:1128-1133`、`frame/Auth.go:664-668` |
| 节点身份（传输层） | 节点 PPT → `NLogin` → 会话 `userid=<调用节点 key id>` | 两边节点之间的 RPC 调用 | `pnet/pnet.go:716-758`、`pnet/service.go:280-282` |
| 节点应用码 | 接收节点上的会话（`nodeid`/`forapp`） | 接收节点 | `frame/Auth.go:975-1010` |

关键：**传输层的节点身份不会进入 MApp 的 `Request`**。MApp 看不到是谁调的。

### 1.3 所以 A1/B1 的准确表述

- 不是"node A 的 author sid 被 node B 接受"；
- 而是"node A 用本地 author 会话通过**发起节点**的跨节点授权闸；node B 以 guest 执行；node B 的 MApp 自己重新取 author 会话"；
- 如果你们真的把 author sid 放在 `sid` 里跨节点传，目标端会**拒绝**（非空不可解析 sid → `2:session id is not safe!`），根本跑不到 MApp。
  **建议你们核对**：在目标端打印 `Request["sid"]` 与 `Request["sidapp"]`，确认实际收到的是空串还是别的。如果是空串，`requireAuthorSession` 比较的"caller sid"就不是这个跨节点 sid。若方便，把跨节点调用的实际 request map（可脱敏）给我们，我们可以直接指出它命中了哪条路径。

**另一种可能（也请一并核对）**：如果你们先在目标节点通过一个 entry 取到该节点的 `BELoginAsAuthor()` sid（即"目标本地 author 会话"），再在后续跨节点请求里把它放在**自定义参数**（不能放 `sid`：`sid` 会在发起节点转发前被删）里带过去，那么目标端 MApp 对自定义 sid 调 `GetVar` 当然能解析、比较也成立。这种情况下"跨节点可用"是真的，但**author 会话已经交给了调用方**：任何能调那个 entry 的人（包括 guest，`RunMApp` 无 sid 形参、分类为 public：`frame/webapi_classification_test.go:263`）都能拿到它并重放。也就是说，peer guard 从"应用实例"退化为"持有该 sid 的任何人"，与你们想建立的信任边界相反。若实际是这种形态，建议把"取 author 会话"的 entry 关掉，改为由目标端自己比较**可验证的调用方声明**（PPT/签名），不要发会话。

---

## 2. A 组：应用作者身份

### A1. 作者身份是否全网有效？

**结论**：分开看——

- **uid 全网一致（有条件）**：应用的作者 uid 记录在应用自己的弥媒记录 `MMInfo.Author` 里（创建时写入上传者 uid：`mimei/MiMei.go:363`；读取时 `app/MFileApps.go:903`；执行时节点用本地记录覆盖请求里的 `author`：`app/param.go:116`）。同一条记录 ⇒ 同一个 uid（appid 本身由 `LAppName2ID(author, name)` 推导，见 `app/param.go:72`；改 author 就是另一个应用）。记录经弥媒同步到其他节点（`pnet/dhtMiMei.go:329-330,417-419`）。
- **author 会话不是全网会话**：会话存在节点内存（`frame/Init.go:48-50`，TTL 3600s、重启即失效；`docs/KEY_AUTH_GUIDE.md` §1.3 公开写明）。跨节点转发还会把 `sid` 删掉（§1.1）。
- **是否契约**：不是。公开契约只承诺"同一密钥可在任意节点登录、userid 由公钥单向推导、全网一致"（`doc/KEY_AUTH_GUIDE.md` §0"正向事实"），并同时限定"**本功能在本地节点可用；跨节点同步属未承诺范围**"。作者身份（app 记录里的 `Author`）**不在这份承诺面内**，也没有签名绑定：它只是弥媒记录的一个字段。
- **一致性不是强保证**：谁能保证各节点的应用记录相同？见 E1——平台不强制 root-only，记录本身可能分叉。所以"author uid 都一样"的准确说法是"**只要记录一致就一样**"，不是平台保证。

**建议你们依赖的是**：`BELoginAsAuthor()` 在本节点返回的 uid，与目标端 `BELoginAsAuthor()` 返回的 uid 相等（因为 appid 由 author+name 推导、记录同源）。这是"同一份数据推导出的同一身份"，不是"跨节点身份认证"。

### A2. 2026-09-03 的 `BELoginAsAuthor` 冻结对基于它的应用意味着什么？

**结论**（逐条）：

**(1) 已依赖它的应用还能不能新增调用点？**
按裁决口径：**不应新增**——"新增调用点"就是新增依赖面。冻结原文是"**对外不再演进、禁止新增依赖；冻而不废，接口继续接线可用，零行为变更**"（`planning-artifacts/milestone-2026-09/T1/f4-server-side-key-holding-analysis.md` §8-6；`docs/KEY_AUTH_MECHANISM_AUDIT.md` §2 F4）。这与冻结目的（收敛到 `BELoginAsApp` + 节点代劳）相反，也会延长观测期、增加将来删除成本。
**但没有强制拦截**：裁决明确"仅代码注释 DEPRECATED + 文档标记，**不做 CI 静态检查**——以调用日志观测替代强制检查"。所以这是**规范约束，不是机制约束**。

**(2) "观测驱动删除"观察什么？有无时间表？**
观察点就是实现里的 `[BELA-CALL]` 日志（`app/MFileApps.go:1132`：每次调用记 `appid` + `authID`，时间戳由日志库带）。裁决原文："**不设硬性删除期限**——节点侧实现处加调用日志……观察确认除已知内部路径外无外部调用后废弃；废弃评审与问题 5 联动的 P2/P3 执行前统一核查"。当前状态：生产内部调用已全部迁移（2026-09-04，`[BELA-CALL]=0`），只剩"MApp 可见冻结 API + 单测 + o5perm 探针"（`docs/KEY_AUTH_MECHANISM_AUDIT.md` §2 F4 的 2026-09-04 收尾段）。**仓库外 MApp 是否在调无法枚举**，所以只能靠日志 + 公告窗口。
`docs/KEY_AUTH_MECHANISM_AUDIT.md` 确实在**主仓（私有）** `docs/` 下，不在 github 公开的 `3and4/Leither` 仓（公开仓只有 `doc/KEY_AUTH_GUIDE.md` 等）。是否公开该审计件属作者裁决；本文引用的裁决内容可直接转述。

**(3) `BELoginAsApp` 是不是预期继任者？它打开的是同一批作者拥有的弥媒对象吗？**
- **是替代方向**：两个方法的 TODO 同文（"换成 ap.appID，实际操作的时候由节点代劳"，`app/MFileApps.go:1129,1137`），裁决也写"替代方向 = 应用身份 + 节点代劳"（f4 §8-6）。`BELoginAsApp` 目前**未冻结**。
- **语义不同**：`BELoginAsApp()` → `CreateAppSession(appID)`，会话里是 `SessionKey_AppID`，`GetUidFromSession` 返回 **appid**（`frame/Auth.go:671-676`、`frame/ApiNode.go:54-63`）。所以通用弥媒判权用的是 appid，不是作者 uid，**不继承作者全权**。
- **但应用数据区仍是作者的**：`BEOpenAppDataApp` 的实现固定读 `MMInfo.Author` 并以其身份打开/创建数据区（`mimei/AppData.go:43-56`），与调用会话是谁无关。所以经 `BEOpenAppDataApp` 拿到的 mmsid 仍是"作者拥有的那批对象"。
- **应用读自己资产**：应用自己的 mid 缺省权限 `DataRight_Default_App = DataRight_App（读|执行）`，对无记录者（guest 位）可取（`api/DataRight.go:37,55`）。F4 A4 已实证应用会话可读 `mm://<appid>/` 资产并完成迁移（`frame/Api.go:1078-1127`）。
- 额外注意：`BESignPPT`/`BESign` 的签名密钥是**节点系统钥**（`AppData` 嵌入的 `SysUser`，`frame/LeitherServer.go:478-486`、`app/MFileApps.go:1141-1174`），不是应用作者钥。`BESignPPT` 强制 `CertFor=App` + `AppID`，而登录侧只接受 `Bind`/`Self`/`Gift`（`user/UserBindID.go:143-172`）⇒ **它签出的不是可登录凭证**。

### A3. 外部 client 能否直接拿到 author 会话？

**结论：不能（就这四个方法而言）。**

- RPC 面是 `AddAllMethods(&WebApi{s})`，挂在 `/webapi/`、`/ws/`、`/ws3/` 三个挂载点，**零 filter**（`frame/Init.go:181`、`pnet/webfunc.go:18,47-77`；注释见 `frame/ApiIpfs.go:19`）。
- `BELoginAsAuthor` / `BELoginAsApp` / `BESignPPT` / `BESign` **没有任何一个**定义在 `*WebApi` 上；它们只存在于进程内 `lapi.BackEndStub` / `api.BackEndStub`（`lapi/backend.go:134-163`、`app/MFileApps.go:1128-1174`、`frame/Api.go:490-510`），由节点在**执行 MApp 时**注入给应用代码。所以它们是"**MApp 可见的 Go 契约**（published Go module 的接口），不是网络端点"。
- 有一个 CI 反射门禁逐方法登记 RPC 面：`frame/webapi_classification_test.go`（方法集 ⇔ 分类表）。表里没有上述四个；`BESessionCreate/Set/Get/Delete/Release` 在列，但**仅系统用户**（`frame/ApiSession.go:36-101` 的 `beSessionACL`）。
- 客户端真正可达的身份/签名家族是：`Login` / `LoginWithPPT` / `NLogin` / `Register` / `SignPPT` / `Sign` / `SignRaw`（`SignRaw` 是 api-only、内部档，见分类表 `frame/webapi_classification_test.go:280-281`）。
- **唯一绕法在应用层**：任何 client 都能 `RunMApp`（无 sid 形参、分类为 public：`frame/webapi_classification_test.go:263`；`user/Auth.go:561-608` 对空 sid 返回 guest）。如果某个已部署 MApp 的入口**把 `BELoginAsAuthor()` 的 sid 作为返回值**交给调用方，那调用方就拿到了。这不是平台面，是应用写法问题——建议你们（以及所有 app）永不把 author sid 返回给外部。

⇒ 你们"故意没从客户端试"是对的，但结论是：**不需要试，它不在网上**。

### A4. `GetVar(authorSid, "userid")` 是契约吗？

**结论**：实现上是同一条路，契约上没有对 author 会话承诺。

- `GetVar(sid,"userid")` 走 `case api.ApiVarUserID`：先 `rejectMMSid(sid)`（数据区会话拒绝），再 `GetUidFromSession(sid)`（`frame/GetVar.go:96-107`）。
- `GetUidFromSession` 依次读 `SessionKey_UserID` → `SessionKey_AppID`（`frame/ApiNode.go:33-64`）。
- `BELoginAsAuthor()` 返回的会话正是 `CreateUserSession(authID)`，即 `SessionKey_UserID=authID`（`frame/Auth.go:664-668`、`app/MFileApps.go:1128-1133`）⇒ 返回 `authID`，稳定。
- 公开文档只承诺用户会话的 `GetVar(sid,"userid")`（`doc/KEY_AUTH_GUIDE.md` §0/§4"身份读取的补充说明"）。对 author 会话，这是**同一实现的自然结果**，不是承诺；且 author 会话本身已冻结。若你们的 peer guard 依赖它，应把它当"现有行为"，并准备在删除 `BELoginAsAuthor` 时一起改。

---

## 3. B 组：跨节点 `RunMApp` 接收方能知道什么

### B1. 目标节点在 dispatch 前认证什么？

**结论**：

- 目标节点执行 `createMApp` 前做 `PreLogin`（`app/MFileApps.go:323`、`user/Auth.go:561-608`）：
  - `sid == ""` → 返回 `uid=""`（guest，不报错）；
  - `sid` 非空 → `GetUidFromSession(sid)` 走**本地** `SessionManager`；不可解析 ⇒ `ErrInvalidSid`，**MApp 不会运行**；
  - `sid` 为空但请求带 `ppt` / `addkey` / `name+pass` → `ParamAuth` 在**目标节点**重新认证，成功后**新建本地会话**并回写 `Request["sid"]`。
- 所以 MApp 看到的 `sid` 只可能是：空（guest）、调用方带来的**目标节点本地有效 sid**、或目标节点刚刚创建的本地 sid。**不会是"外节点的任意字符串"**。
- 跨节点转发来的请求，`sid` 已被发起节点删除（`app/MFileApps.go:306-307`）⇒ 默认 guest。
- 传输层另有节点身份：`DoWorkOnNode` 用节点 PPT `NLogin` 在目标节点建立会话，uid = 调用节点系统 key id（`pnet/pnet.go:716-758`、`pnet/service.go:280-282`）。**这个身份不被注入 Request**。

⇒ 直接回答：**目标节点会校验 `sid`（本地会话表）；但它不把"调用节点是谁"告诉 MApp**。你们担心的"外节点 author 会话被接受"不会发生——同时，你们想要的"caller node 身份"也不存在（见 B2/B3）。

### B2. 接收 MApp 能知道是哪个节点调的吗？

逐键回答（`lapi/backend.go:30-39` 定义的注入键）：

| 键 | 谁写的 | 跨节点 `RunMApp` 里可信吗 |
|---|---|---|
| `nid` | **调用方**填的"目标节点 id"；`GetMApp` 用它决定本地/远程，转发前 `delete`（`app/MFileApps.go:295,306`） | **不可信**；到目标端已被删，且它语义是"目标"不是"来源" |
| `sidapp` | **不是节点注入**。只有 CLI `runapp` 在本地运行前设置（`cryp/cmd/runapp.go:293-296`：`Request[sidapp] = stubApp.Sid`，作者会话） | **不可信**；容器/跨节点路径不注入，值完全来自请求 map |
| `mmroot` | **节点注入**：`injectMMRoot` 从 `Manifest.MMRoot` 覆盖（`app/MApp.go:71-80,112`；Go 路径 `app/xgoApp.go:211`） | **可信**——但仅当 Manifest 声明了 `MMRoot`；若为空，节点不写，调用方自带的 `mmroot` 会保留（`injectMMRoot` 只在非空时写） |
| `author` | **节点注入**：`initMAppParam` 用本地应用记录的 `Author` 覆盖（`app/param.go:116`） | **可信**（前提是本地记录可信，见 E1） |
| `sid` | `PreLogin` 校验/按需重建（见 B1） | 见 B1 |
| `nodeappcode` | **平台自动注入**：`remoteMApp.RunMApp` 在**目标节点**上按 `aid` 取码后写入 `request[Param_NodeAppCode]`（`app/MApp.go:203,209`）。注意：只在请求带 `aid` 时取 | 见 C2：能指认"铸造时那个调用方会话的 uid"，但是 bearer 码，非 attestation |

**结论**：没有"由节点按传输层注入"的 caller 标识。`sidapp` 在容器里根本不存在，别把它当身份。

### B3. 有没有受支持的节点 attestation？

**结论**：对 MApp 没有。

- 传输层有真实签名：`NLogin(ppt)`，PPT 由调用节点系统钥签（`CertFor=Self`），目标端验签后建会话（`pnet/pnet.go:716-758`）。**但它的身份落在 `NodeApiClient.Sid` 上，不进入 `RunMApp` 的 `Request`**。
- app 可见的替代品就是 `nodeappcode`：接收节点在铸造时把 `nodeid`（调用方会话 uid）写进会话，MApp 用 `SessionGet(code,"nodeid")` 读出（`frame/Auth.go:993-1007`；消费例见 `app/ServerApp/LeitherService/ledger.js:328-344`）。问题：**它是 bearer 码**——谁拿到码谁就能出示；铸造点只要求"非空可解析会话"（任何登录用户都能铸，不限于节点），见 C2。
- 没有"节点对（nodeid, entry, timestamp）签名、MApp 可验"的通道；`getvar?name=nodeloginppt` 只给系统用户（`frame/GetVar.go:272-295`），且签的是"给目标节点的登录票"，不是给 MApp 的声明。`BESignPPT` 签的是**本节点**系统钥（MApp 运行在哪个节点就签哪个节点），不能证明调用方节点。
- **可行性**：在 `RunMApp` 入口把传输层会话 uid（节点 key id）作为 `fromnid` 注入 `Request`，技术上不难（入口已有 `NodeApiClient.Sid`/服务端会话），但这属于**新能力**，需要作者裁决；见 §7。

---

## 4. C 组：节点应用码与好友节点

### C1. 谁铸造 node app code？

**结论**：铸造路径是 `GetVar`，不是好友节点 API。

- 入口：`GetVar(sid, "nodeappcode", args[0]=appid)` → `frame/GetVar.go:453-458` → `getNodeAppCode(sid, appid)`（`frame/Auth.go:975-1010`）。
- 行为：先 `GetUidFromSession(sid)` 取 `nodeid`；**空身份拒绝**（看板 #30，`frame/Auth.go:988-991`）；然后按 `CacheKey_NodeAppCode + nodeid + appid` 缓存 1h，懒创建会话并写入 `SessionKey_NodeId=nodeid`、`SessionKey_ForApp=appid`（`frame/Auth.go:993-1007`、`frame/Const.go:23`）。
- 它不是登录会话：会话里**没有** `UserID`/`AppID`，`GetUidFromSession(code)` 会返回 `ErrInvalidSid`；只能拿来做 `SessionGet`。
- **好友节点 API 不是铸造路径**：`api/Net.md` 是"草"稿；`Invite`/`Accept`/`AddNode` 在 `frame/ApiNode.go:77-107` 和 `pnet/pnet.go:904-960` 里**整段注释**；`NGetGift` 返回 `ErrTBD`（`frame/ApiNode.go:68-72`）。唯一活着的是 `NLogin`（节点间登录，与 app code 无关）。
- 文档：开发者指南只在 `GetVar` 键表里列了 `nodeappcode` 一行（`docs/Leither-Developer-Guide.md:1073`），**没有**铸造/信任/生命周期的说明；这就是你们"找不到"的原因。

### C2. 解析出来的码能证明什么？

- **只证明"出示者知道这串码"**。码是接收节点上的一把会话钥匙；`SessionGet(code,"nodeid")` 返回的是**铸造时**调用方会话的 uid，不是出示时的验证结果。传输层不参与码的校验（它只是被 `remoteMApp` 当作普通参数注入 `request["nodeappcode"]`，`app/MApp.go:203-209`）。
- **`nodeid` 不一定是"节点"**：铸造点只要求"非空可解析会话"（`frame/Auth.go:988-991` 的注释写明"兄弟节点登录会话 / 系统用户 / 普通登录用户都不受影响"）。普通用户登录后自己 `getvar?name=nodeappcode` 也能拿到码，其 `nodeid` 就是该用户 uid。所以"node app code"这个名字有误导性。
- **生命周期**：会话 TTL 与节点会话一致（`frame/Init.go:48-50`：`maxLifetime=3600s`，GC 周期 1200s）；缓存条目也按 1h 复用（`frame/Auth.go:995-1007`）。同一 `(nodeid, appid)` 在窗口内复用同一个码；没有显式吊销 API（`RemoveSession` 不在 MApp 面）。
- **`forapp` 绑定**：铸造时写入 `SessionKey_ForApp=appid`；平台**不做**消费侧强制。消费方必须自己 `SessionGet(code,"forapp")` 比对（你们和 `ledger.js:334-341` 都是这么做的）。不比对就等于不校验。
- **结论**：码是"节点范围的应用句柄标题"，不是权限凭证，也不是节点证明。把它当 peer 信任根，等于把"知道字符串"当身份。

### C3. 原本面向客户端的用法是什么？

- **仓库内唯一的消费者**是 JS MApp `app/ServerApp/LeitherService/ledger.js`：
  - 入口注释："这个请求入口是好友节点触发的，使用的是 nodeappcode 鉴权"（`ledger.js:67-68`）；
  - `getFriendByAppCode(nodeAppCode)`：`SessionGet(code,"nodeid")` 取好友 id、`SessionGet(code,"forapp")` 校验 appid（`ledger.js:328-344`）；
  - 用法：跨节点对账/清账请求（`getzledger3`、`clearLedgerReceiver`、`setsuminfo3`）。
- **平台自动路径**：`remoteMApp.RunMApp` 在跨节点调用前，**在目标节点**以调用方节点会话 `GetVar(nodeappcode, aid)` 取码，再把码放进请求（`app/MApp.go:203-209`）。所以"前端传码"的等价物是："调用节点替它的应用取码"。
- **"前端从自己节点取码、拿到目标节点用"** 跨节点**不成立**：码存在**铸造它的那个节点**的会话表里；目标节点 `SessionGet` 找不到对方节点铸的码，只会得到 nil。前端要拿到目标节点能解析的码，必须在**目标节点**有会话并直接向目标节点 `getvar?name=nodeappcode`（当前 guest 会被拒，必须登录目标节点）。这正是旧版（V0.24.18 及以前）guest 也能取码、被看板 #30 收紧的原因。
- `add_tweet.js` 不在 Leither 仓库（本仓无此文件），无法确认其当时的原意；以上是平台侧能支持/不支持的事实。

---

## 5. D 组：会话与身份范围

### D1. `byname`（用户名/密码）登录是什么状态？

**结论**：**遗留/内部/实验路径，明确不在承诺面，无提升/删除计划；新应用不要建在它上面。**

- 文档：`docs/KEY_AUTH_GUIDE.md` §0"不在承诺面"列 byname；§7.4 写明"byname 为遗留路径，不在本期承诺面"，并说明注册接口（`Register` 的 byname/邮箱路径）**无鉴权、无节流**。发布登记里也有同样限制（`planning-artifacts/milestone-2026-09/T0-external-release-process.md` §10 四条已知限制之一）。
- 决策：M10（2026-09-11）选"声明为内部/实验面、不加节流、不作承诺"，**不是**"加证明/加节流"（`planning-artifacts/milestone-2026-09/T1/m-series-decision-cards.md` §M10）。**没有**提升为承诺面的排期，也没有删除排期。
- 实现：`Register` 在节点侧**新建一对密钥**（`frame/Auth.go:54-120`，`NewOwnerByKeyPair(nil)`），name→uid 与口令哈希存在本节点 `SystemDB`；口令自 2026-09-01 起 argon2id（`user/Auth.go:330-347`）。认证数据节点本地、不跨节点同步（`docs/KEY_AUTH_GUIDE.md` §0）⇒ 同一个 byname 账号**只能在其 root 节点登录**。你们"guest 只能在 root 节点登录"的结论正确，且这是设计使然，不是 bug。
- 回答你们的三个子问题：**未排期提升、无变更计划、无删除计划**；多用户应用应使用密钥/PPT 方案，或接受"guest 账号绑定单一节点"的现状。

### D2. 有没有计划中的节点/应用绑定凭证？

**结论**：没有"绑定会话到节点/应用实例"的承诺或计划；现有的是**签名层的应用声明**，不是会话绑定。

- 公开承诺面：`SignPPT` 对**普通用户会话**剥离 `NodeId`/`AppID` 等保留键（`frame/Auth.go:702-726`、`doc/KEY_AUTH_GUIDE.md` §4 表格）；`Sign`/`SignPPT` 的 `req.AidDecl` 是**应用身份声明**——与服务端导出值不一致即拒（`signacl:aid-spoof`，`docs/KEY_AUTH_GUIDE.md` §7.7）。
- 存在但非登录型：`BESignPPT` 会强制 `CertFor=App` + `AppID`（`app/MFileApps.go:1141-1152`），签名钥是**节点系统钥**；登录侧只接受 `Bind`/`Self`/`Gift`，`CertFor=App` 直接判非法（`user/UserBindID.go:143-172`）⇒ 它不能被用来建会话。
- 会话绑定：没有。PPT 明确"不绑定签发/使用节点"，是 bearer（`docs/KEY_AUTH_GUIDE.md` §7.2）；节点会话（`NLogin`）是传输层，不暴露给 MApp；app 会话（`BELoginAsApp`）只带 appid。
- 开放项：MApp "以谁的 sid 运行"（会话/沙箱身份设计）在 `planning-artifacts/direction/lapi-upload-privilege-requirement-2026-11-01.md` §6 被明确列为 **L4 评估卡、不在当前范围**。所以"计划中"的答案：**没有已立项的节点/应用绑定会话凭证**；`AidDecl` 是签名层最接近的方向。

---

## 6. E 组：写授权与版本

### E1. Leither 强制 root-only 版本吗？

**结论**：**不强制。你们的假设正确。**

- 没有"root 节点"概念（`mimei/` 与文档中无此实体；版本指针 `cur`/`last`/`release` 都是**本节点**数据库里的条目：`mimei/Backup.go:590-640`、`mimei/updatetime.go:84-106`）。
- 版本写入按 **uid 的 mid 权限**判定：`MMBackup` 要 `DataRight_Write` 或 `DataRight_Backup`（`mimei/Backup.go:398-425`）；`MFSetCid` 要 `DataRight_Write`（`mimei/MiMei.go:1085`）。权限来源：`readMMRightFromDb` —— **`mminfo.Author == uid` 直接返回 `DataRight_All`**（`mimei/rights.go:12-29`）。
- 任何持有该应用记录的节点，都能通过 `BELoginAsAuthor()` 拿到 uid=作者 的本地会话（**不需要作者私钥**，`app/MFileApps.go:1128-1133`），于是对 mid 有 All 权限，可以打开 `cur`、写、`MMBackup` 出新版本。提供者节点与 root 在代码里**没有区分**。
- 平台能看到的只是"所有提供者中的最大版本号"（`mimei/Backup.go:507` `GetPrivodeMaxVerSeq` 注释自述），这用于清理旧版本，不是一致性校验。
- **计划**：公开材料中没有"只有 root/作者可产生新版本"的机制或排期。你们的 app 侧拒绝 misrouted write 是当前唯一的防线，合理且必要。

### E2. MApp 能否以调用者身份执行（供权限层判权）？

**结论**：没有通用能力，也没有计划；但**有一种可用的"per-user 数据区"原语**，前提是持有调用者本人会话。

- 权限层解析 uid 的方式固定为：会话（`GetUidFromSession`）或数据区属主（`mimei.resolveDataOwner`），**没有"assert uid"参数**（`memory/leither-identity-and-rights-model.md` §0.1：动作看身份，操作看句柄）。
- `MMOpenAppData*` 的 `owner` 参数**不是断言**：`createAppData` 强制 `owner == curUid`，否则返回错误（`mimei/AppData.go:158-186`）。
- 但 `MMOpenAppDataUser(sid, aid, owner, ...)` 在 MApp 可见的 `lapi` 面上（`lapi/stub.go:46-53`）。传入**调用者自己的 sid** + `owner=调用者 uid`，就会在调用者名下创建/打开一个 per-user 应用数据区（`AD_User`，owner=defOwner，`mimei/AppData.go:99-125`），调用者对该数据区有 All 权限（因为是 `Author`）。
  - **本地请求**（`/entry` 或本节点 RPC）：`Request["sid"]` 就是调用者会话，MApp 可以做这件事。
  - **跨节点**：调用者 sid 已被删（§1.1），做不了。
- 所以：想"让 Leither 原生承担 owner-only"，可行的方向是把 per-user 数据区用起来；但"以调用者身份在作者数据区里写"没有原语。另：`frame/ApiSession.go` 的 D-2 v1 只允许系统用户经 RPC 操作会话；MApp 进程内的 `SessionStub` 是另一面（`frame/Api.go:450-460`），不在该 ACL 下——这也说明"MApp 以谁运行"仍是有待设计的边界（L4）。

---

## 7. 对"希望 Leither 提供"五项诉求的回应

1. **跨节点 `RunMApp` 的 caller-node 身份**（transport-set、MApp 可见）。
   - 现状：没有。`nodeappcode` 是唯一的 app 可见线索，但是 bearer 码、可伪造、非节点专用（C2）。
   - 可行性：有。入口 `WebApi.RunMApp`/`runMAppFromApp` 可以拿到调用方在**本节点**的会话（传输层 `NLogin` 身份），把它作为 `fromnid`（27 位系统 key id）注入 `Request` 即可；若要防伪造，由服务端**覆盖写入**（与 `author`/`mmroot` 同款），不读调用方值。
   - 状态：**需要作者裁决**，不是既有承诺；本文不代为承诺版本或时间。
2. **`BELoginAsAuthor` 冻结的正式说明**：见 A2。审计件在私有仓 `docs/KEY_AUTH_MECHANISM_AUDIT.md`；是否公开由作者定。可直接转述的关键句已在 A2 引用。
3. **backend login family 的客户端可达性**：不可达（A3）。若你们希望"MApp 上下文专属变体"，现状就是：它们只在 MApp 的 `lapi` 面。**没有**客户端可达版本，因此不存在"任何人可铸 author 凭证"的漏洞；唯一要防的是应用自己把 sid 交出去。
4. **node app code 的文档**：见 C1/C2。可以补进开发者指南（铸造点、TTL、`nodeid`/`forapp` 语义、"非权限凭证"警告）；但补文档不等于把它升格为信任根。
5. **以调用者 uid 执行（E2）**：没有计划。可选的是 per-user 数据区（见 E2），或等 L4"会话/沙箱身份"评估。

---

## 8. English one-liners（可直接贴回 memo）

- **A1**: The author *uid* is `MMInfo.Author` from the app record and is identical where the record is identical; the author *session* is node-local memory, and cross-node `RunMApp` **deletes `sid` before forwarding** — no foreign session is accepted. It is not a documented contract.
- **A2**: The freeze is formally recorded: frozen-but-not-removed, **no new dependencies/call sites**, removal is observation-driven via the `[BELA-CALL]` log with **no deadline and no CI enforcement**; `BELoginAsApp` is the intended direction (app identity + host-on-behalf) but does **not** inherit author-level rights; app-data regions opened via `BEOpenAppDataApp` remain author-owned.
- **A3**: None of `BELoginAsAuthor`/`BELoginAsApp`/`BESignPPT`/`BESign` is a `WebApi` method; the RPC surface (`/webapi/`, `/ws/`, `/ws3/`, `AddAllMethods(&WebApi{s})`, zero filter) does not expose them. They exist only in the in-process `lapi` backend for MApp code.
- **A4**: `GetVar(authorSid,"userid")` uses the same `GetUidFromSession` → `SessionKey_UserID` path and returns `authID`; promised only for user sessions, "current behavior" for author sessions.
- **B1**: The target validates `sid` against **its own** session store (`PreLogin`); empty → guest, unparsable non-empty → hard reject; forwarded cross-node requests have `sid` removed, so the target runs as guest. Transport-level node identity (`NLogin`) is **not** passed to the MApp.
- **B2**: `nid` is caller-supplied and deleted before forwarding; `sidapp` is CLI-local only (not injected in containers); `mmroot` is injected from the Manifest (only when non-empty); `author` is injected from the app record. None of them names the caller node.
- **B3**: No signed node attestation is available to an MApp. The closest thing is `nodeappcode` (a bearer code minted by the receiving node), which is not evidence of a node.
- **C1**: Minting path is `getvar?name=nodeappcode&arg0=<appid>` → `frame/Auth.go:getNodeAppCode` (lazy 1h session with `nodeid`/`forapp`); the friend-node API in `api/Net.md` is a draft and its `Invite`/`Accept`/`AddNode` are commented out; `NGetGift` is `ErrTBD`.
- **C2**: Resolution proves only knowledge of the string; the code is minted under the caller's session uid (which may be an ordinary user, not a node), lives ≈1h, and `forapp` must be checked by the consumer — the platform does not enforce it.
- **C3**: The only in-repo consumer is the JS MApp `LeitherService/ledger.js`; the platform path is the automatic cross-node `RunMApp` injection. A browser presenting a code it minted on its own node cannot be resolved by the target (wrong session store). `add_tweet.js` is not in this repository, so its original intent cannot be confirmed here.
- **D1**: `byname` is explicitly outside the promise surface (legacy/internal), with no promotion or removal scheduled; its auth data is node-local; new multi-user apps should not build on it.
- **D2**: No node/app-bound session credential is planned. `CertFor=App` PPTs exist (node-key-signed, not accepted for login); `AidDecl` binds signatures to a declared app; the "which sid does a MApp run as" design is an open L4 item.
- **E1**: No root-only enforcement. Version writes are uid-rights based; `MMInfo.Author` gets `DataRight_All` on any node holding the record, and `BELoginAsAuthor` provides that session **without the author's key** — a provider can commit a divergent version. There is no root entity in code.
- **E2**: No general "run as caller" or "assert caller uid" facility, and none planned. `MMOpenAppDataUser` only opens a **per-user data region** when `owner == session uid`; it cannot make the author-owned region per-user, and the caller's sid is not available cross-node.

---

## 9. 证据索引

| 主题 | 位置 |
|---|---|
| RPC 面挂载（三挂载点、零 filter） | `frame/Init.go:181`；`pnet/webfunc.go:18,47-77`；`frame/ApiIpfs.go:19` |
| RPC 面方法分类门禁 | `frame/webapi_classification_test.go`（`BESession*` 102-106；`Login*` 180-181；`NLogin` 240；`Register` 247；`SignPPT`/`SignRaw` 280-281） |
| `WebApi` 结构体 | `frame/Api.go:97-99` |
| `BELoginAsAuthor`/`BELoginAsApp`/`BESignPPT`/`BESign`（lapi 契约 + 实现） | `lapi/backend.go:134-163,210-220`（外部模块 `github.com/3and4/Leither/lapi`）；`app/MFileApps.go:1128-1174`；`frame/Api.go:490-510`；`app/BackEnd.go:60-75` |
| `BELoginAsAuthor` 冻结裁决 | `planning-artifacts/milestone-2026-09/T1/f4-server-side-key-holding-analysis.md` §8-6；`docs/KEY_AUTH_MECHANISM_AUDIT.md` §2 F4；冻结件 `planning-artifacts/milestone-2026-09/T1/f4-belogin-author-freeze-2026-09-02.md` |
| `[BELA-CALL]` 观测点 | `app/MFileApps.go:1132` |
| 作者 uid 来源 | `mimei/MiMei.go:363`；`app/MFileApps.go:903`；`app/param.go:116` |
| 会话 TTL / 内存态 | `frame/Init.go:48-50`；`docs/KEY_AUTH_GUIDE.md` §1.3 |
| `GetUidFromSession` | `frame/ApiNode.go:33-64` |
| `GetVar` 身份读口 | `frame/GetVar.go:96-107` |
| `PreLogin` | `user/Auth.go:561-608` |
| 跨节点 RunMApp（判权、删键、转发、取码） | `app/MFileApps.go:290-310`；`app/MApp.go:188-222`；`frame/ApiApp.go:46-58`；`pnet/pnet.go:843-900` |
| 节点登录 | `pnet/pnet.go:716-758`；`pnet/service.go:280-282` |
| `nodeappcode` 铸造 | `frame/GetVar.go:453-458`；`frame/Auth.go:975-1010`；`frame/Const.go:23`；`frame/node_appcode_gate_test.go` |
| `nodeappcode` 消费例 | `app/ServerApp/LeitherService/ledger.js:67-68,328-344,1044` |
| 好友节点 API 草稿/未实现 | 外部 lapi 模块 `api/Net.md`；`frame/ApiNode.go:77-107`；`pnet/pnet.go:904-960` |
| `mmroot`/`author` 注入 | `app/MApp.go:71-80,112`；`app/xgoApp.go:211`；`app/param.go:116`；`sidapp` 仅 `cryp/cmd/runapp.go:293-296` |
| `BELoginAsApp`/`BEOpenAppDataApp`/权限 | `frame/Auth.go:671-676`；`mimei/AppData.go:43-56,99-125,158-186`；`mimei/rights.go:12-29,143-170`；`api/DataRight.go:37,55` |
| 登录 PPT 类型 | `user/UserBindID.go:143-172`；`cryp/SignInfo.go:34-37` |
| `SignPPT` 保留键 / `AidDecl` | `frame/Auth.go:702-733`；`docs/KEY_AUTH_GUIDE.md` §4/§7.7 |
| 版本写入与权限 | `mimei/Backup.go:398-425,471-537`；`mimei/MiMei.go:1085`；`mimei/updatetime.go:84-106` |
| byname 状态与 M10 裁决 | `docs/KEY_AUTH_GUIDE.md` §0/§7.4；`frame/Auth.go:54-120`；`user/Auth.go:330-347`；`planning-artifacts/milestone-2026-09/T1/m-series-decision-cards.md` §M10；`planning-artifacts/milestone-2026-09/T0-external-release-process.md` §10 |
| L4 会话/沙箱身份开放项 | `planning-artifacts/direction/lapi-upload-privilege-requirement-2026-11-01.md` §6 |
| 跨节点 MApp 取证 | `planning-artifacts/milestone-2026-09/T1/m12-cross-node-mapp-2026-09-12.md` §1/§4.1 |
| 身份口径 | `memory/leither-identity-and-rights-model.md` §0/§0.1 |
