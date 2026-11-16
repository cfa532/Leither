# Leither Structural Questions II — Answers from the maintainers

**Provenance:** Received 2026-11-16 as `tweet-structural-answers-2-2026-11-16.md`.
The attachment matches the [published reply](http://vzhan.cn/mm/xIuiX4OVNLmXmdGgV2SfxdnPdaW/)
and is preserved below. The maintainers reviewed Leither V0.24.74 at private root
revision `a3ce96e8`. Their code citations refer to that repository. The reply
records a source review and cites earlier Tweet observations.

**Companion questions:** [Leither Structural Questions II](LEITHER_STRUCTURAL_QUESTIONS_2.md).
Section 5 corrects the RPC feasibility note in the
[2026-11-13 reply](LEITHER_STRUCTURAL_ANSWERS_2026-11-13.md).

---

# 对《Leither Structural Questions II》的回答

> **答复对象**：`cfa532/Leither/cfa532-docs/integrations/tweet/` 的六题（2026-11-14）
> **核对基线**：Leither `V0.24.74`，根仓 HEAD `a3ce96e8`（2026-11-16；工作树含 docs/memory 未提交改动，不影响本文结论）
> **证据口径**：以当前代码判读为主，逐条给出文件/行号；引用“现场”处标注为你方 memo 的观测。本轮没有做新的线上实测。
> **性质**：维护方对**当前代码行为**的判读，不是 `lapi` 公开契约的变更；涉及对外承诺的表述，以 `docs/KEY_AUTH_GUIDE.md` 的“承诺面/不在承诺面”为准。凡平台未承诺、或需要作者裁决的地方，本文明确标注。
> **给对方的英文速览**：见 §7，可直接贴回 memo。

---

## 0. 结论速览

| 题 | 结论 | 性质 |
|---|---|---|
| 1 | mark→mid 是**确定性派生**，且被接收方在同步校验路径上**重算并拒绝不符**；命名空间是 `(uid, appid, ext, mark, tp)`，**没有独立的“对象种类”字段**——tweet 与 comment 若共用 appid/ext/tp 且 requestid 相同，会撞同一 mid。你们的 `<requestid>-quote` 后缀有效；更干净的是让 ext 按种类分开。 | 协议内稳定，但**未写进公开承诺面** |
| 2 | `OpenMM ver is empty` 不是设计出来的哨兵，是文件型对象在“**元数据在本地、请求的版本别名不在本地版本表**”时返回的**无类型错误串**。本地无元数据是 `No MiMei mid[...]`（30000）；权限不在 `MMOpen` 检查；对象是否“在网络上存在”只能用 `MiMeiShow` 探测。 | 现有行为；可分支，但需组合探测 |
| 3 | `MiMeiSync` 的 param 已经支持 `SourcePeer`（**节点 id**，不是地址），会直接向该节点取版本信息、跳过 provider 发现。`vers` 参数当前**被解析但未使用**。MApp 侧 `MiMeiSync` 有 **5s 等待预算**，`mapp call bounded` 表示“还在跑”，不等于失败。无推送、无收敛 SLA。 | 现成能力；无承诺时限 |
| 4 | **没有** `GetVar` 可以返回“这个 MApp 正在运行的精确版本”。`ver` 是节点软件版本；`appinfo` 只给 `release`/`last` 别名。`Request["ver"]` 是调用方传的**别名**。最可靠的“运行版本”信号是应用源码里自带的 build 常量。 | 无此能力；建议应用侧自带 |
| 5 | **作者尚未裁决**（仓库、看板、记忆索引均无记录）。另外，上一份 §7.1 的可行性描述需要订正：`RunMApp` 的 RPC 入口**不带 sid/上下文**，目标端拿不到传输层会话；实现不是“读一下 `NodeApiClient.Sid`”那么简单。 | 待裁决；需平台改动 |
| 6 | 你们的使用画像**不算**“known internal path”（那是指仓库内平台自己的调用点）；外部 MApp 的调用正是冻结观测要看的对象。冻结**无期限、无 CI 强制**。但当前**没有完整替代件**：`BELoginAsApp` 的会话只有 `SessionKey_AppID`，而 `MMCreate`/`resolveDataOwner`（备份/发布/引用）只认 `SessionKey_UserID` 或 mmsid 属主；`BEOpenAppDataApp` 只覆盖“应用数据区”一个 DB mid。 | 无期限冻结；迁移目标不完整 |

---

## 1. 题 1：mark→mid 的派生规则、命名空间与稳定性

### 1.1 实际算法

派生只有一处：

```go
// mimei/mmpub/pub.go:31-41
func CreateMID(uid, appid, ext, mark string, tp byte) (string, error) {
    if len(appid)+len(uid) == 0 {
        return "", fmt.Errorf("appid and uid is null")
    }
    if mark == api.MM_Auto {
        panic("CreateMID api.MM_Auto")
    }
    return pub.CreateID([]byte(uid+"\x06"+appid+"\x06"+ext+"\x06"+mark), tp), nil
}
```

`CreateID` 走的是**遗留**算法：`SHA-256(输入)` → `Sha256Trunc20`（取前 20 字节）→ 加类型字节 `tp` 编码成 27 字符 id（`pub/IDManger.go:116-120`）。`tp` 只影响 id 的类型头，不进入哈希输入。

创建路径（`mimei/MiMei.go:306,333`）调用 `CreateMID(uid, appid, ext, mark, dataType)`；其中 `uid` 是**执行创建的那个会话的 uid**（`MMCreate` 只从会话读 `SessionKey_UserID`，`mimei/MiMei.go:196-218`），`appid` 是应用 id，`ext` 是弥媒类别/格式串，`mark` 是你们传的 `requestid`。

### 1.2 命名空间：只有四段 + 类型头

| 维度 | 是否参与 | 说明 |
|---|---|---|
| `uid` | ✅ | 创建会话的 uid（作者/属主） |
| `appid` | ✅ | 应用 id；同一 ext/mark 在不同应用下不撞 |
| `ext` | ✅ | 自由字符串；这是**唯一接近“种类”的维度** |
| `mark` | ✅ | 你们的 `requestid` |
| `tp` | ✅（只进 id 类型头） | 文件/数据库等存储类型；同一四元组、不同 tp 会得到不同 id 字符串 |
| 独立“对象种类”字段 | ❌ | 不存在 |

所以：

- **同一个用户、同一个 appid、同一个 ext、同一个 mark、同一个 tp** ⇒ 同一个 mid。这正是幂等的基础。
- tweet 与 comment 若都写 `ext=tweet-file-v1`、同一 appid、同一 requestid ⇒ **同一个 mid**，第二次 `MMCreate` 会把第一次的对象原样返回（见 §1.3），不会报“种类不符”。你们在客户端把 quote 半边的 mark 派生为 `<requestid>-quote`，因此避开了撞车。
- 更干净的做法是**按种类拆 ext**（例如 `tweet-file-v1` / `comment-file-v1`），而不是只靠 mark 后缀；ext 本来就是“类别/格式”维度，拆开后命名空间语义更清楚，也不依赖客户端对后缀规则的记忆。
- 如果你们希望同一 requestid 在“重试”和“不同种类”之间都安全，建议把命名规则写成：`mark = <kind>:<requestid>`（或 `ext` 带 kind），并保证 `<kind>:<requestid>` 全局唯一。

### 1.3 幂等是怎么发生的

`MMCreate` 先算 mid，再 `CreateMiMei`；若该 mid 已存在，`CreateMiMei` 返回 `ErrFmtMiMeiExist`（`mimei/MiMei.go:349`），`MMCreate` **把这个错误吞掉并返回已有 mid**（`mimei/MiMei.go:224-229`）：

```go
mid, _, err = mc.CreateMiMei(uid, appid, ext, mark, dataType, right, false)
if err != nil {
    if !mmpub.ErrFmtMiMeiExist.Is(err) {
        return "", err
    }
    err = nil
}
```

这意味着：

- 同 `requestid` 重试 ⇒ 同 mid，**不会重复建对象**。你们现场两次同 requestid 同 mid，与代码一致。
- 但它**不校验内容**：如果第一次创建成功、数据只写了一半，第二次会拿到同一个 mid，而不是“重新写一遍”。应用侧仍需在“拿到已有 mid”后检查/补齐数据，不能把 `MMCreate` 当成事务性的“create-or-replace”。
- 幂等的“同 mid”只在**同一 uid** 下成立。如果重试被路由到另一个主体（guest、别的用户、系统用户）的会话上，会派生出不同 mid。你们把跨方写绑定到“行为用户的 root 节点”，只要重试仍由同一 uid 的会话执行，就成立。
- 跨节点重试即使本地还没有该 mid，也会在本地按同一 uid/四元组算出同一 mid 并写入；最终仍是同一个逻辑 mid，但可能出现“同一 mid 的两份本地副本”，合并语义另算。

### 1.4 稳定性：它是协议级不变量，不是随时可改的实现细节

三层证据：

1. **接收方会重算**。`/mimei/` DHT 记录的验签链明确写着“mid 绑定作者：`CreateMID(Author, AppType, Ext, Mark, tp)` 重算必须等于 key 中的 mid”（`pnet/coreapi/verify.go:1-19`）。实际代码在 `pnet/coreapi/verify.go:173-179` 重算并拒绝不符。默认 `strict`（`pnet/coreapi/verify.go:42-60`），只有显式设 `LEITHER_DHT_RECORD_VERIFY=observe` 才灰度放行。
2. **刻意不认新算法**。`CreateMIDByKeyType` 存在（按 owner 的 btVer 切到 V2 RIPEMD-160），但生产无调用方；验签注释明确“不认 `CreateMIDByKeyType`：info 包签名不覆盖 mid ⇒ 同一份作者签名可同时挂到旧/新两个派生 key 上（别名 mid 复制口）”（`pnet/coreapi/verify.go:173-176`）。也就是说，mid 派生被钉在**遗留算法**上，换算法会造成旧节点拒绝新对象。
3. **文档缺口**。公开开发者指南只列了 `MMCreate` 的签名（`docs/Leither-Developer-Guide.md:441`），没有写派生规则；唯一接近的表述在内部测试规范里：“`MMCreate` 对相同 `(owner, ext, mark)` 返回相同 mid”（`docs/API_INTEGRATION_TESTING.md:38,83`，注意它漏了 `appid` 和 `tp`）。

**结论**：它不是公开承诺面里的契约，但在当前协议的同步校验路径上被强制，属于**协议级不变量**。对你们而言，按“稳定”依赖是合理的；但要按“未承诺”做防御——不要把 mid 当成内容哈希，也不要在跨大版本升级时假设派生规则永远不变。

**平台侧建议**：这段规则值得补进开发者指南（派生输入、命名空间、`MMCreate` 幂等但**不校验内容**、跨 uid 不幂等）。这是文档缺口，不是行为变更。

---

## 2. 题 2：`OpenMM ver is empty` 到底意味着什么

### 2.1 准确的触发条件

以你们的情况（`ext=tweet-file-v1`，文件型 `MM_File`）为例，`MMOpen(mid, "last")` 的路径是：

```
WebApi.MMOpen (frame/Api.go:902)
  → GetUidFromSession(sid)
  → MiMeiCenter.MMOpen (mimei/MiMei.go:813)
    → ConverSpecialVer(db, mid, "last") (mimei/MiMei.go:898)
       → GetVerByName: HGet("mmversion"+mid, "last") (mimei/mmpub/MMVers.go:75-82)
          字段不存在 → HGet 返回 nil, nil（mimei/mmdb/redis/t_hash.go:175-186）
          ⇒ newVer = ""，不报错
    → MM_File 分支 → openByParam2 (mimei/MFile.go:243)
       → InitMMInfo(uid, mid) (mimei/MFile.go:266)
          本地无元数据 → ErrFmtNoMiMei（30000，“No MiMei mid[...]”）
       → initSessionInfo(mid) (mimei/MFile.go:290)
          oVer == "" → fmt.Errorf("OpenMM ver is empty") (mimei/MiMei.go:929)
```

所以对**文件型**对象：

- `No MiMei mid[...]`（30000）= **本地连元数据都没有**（可能是从未同步，也可能是 mid 根本不存在）。
- `OpenMM ver is empty` = **本地元数据在，但请求的这个版本别名（`last`）在本地版本表里没有对应项**。它等价于“这个节点没有该版本的本地副本（或没有这个别名）”，而不是“对象不存在”。
- **权限不在这条路径上检查**。`InitMMInfo` 只是算出 `curRight`，不拒绝；`MMOpen` 不会因权限被拒（是否成功仍取决于本地元数据/文件是否存在）。真正的读权限检查在读数据时（`getFileDataerAndCheck`，`mimei/MFile.go:166-185`，调用点 `:411`、`:501`）。所以“打开成功”不等于“有读权限”。

两个容易踩的坑：

1. **`ver=""` 不是“cur”**。`ConverSpecialVer` 对空串不短路，`GetVerByName(mid, "")` 会因空字段被 DB 层拒绝（`mimei/mmdb/redis/t_hash.go:14-23` 的 `len(field)==0` → `ErrHashFieldSize`），错误是 `mimei(...) no ver()`。探测时请显式传 `last` / `release` / 具体版本号。
2. **不要用 `cur` 做“存在性探测”**。文件型 `MMOpen(..., "cur")` 会带 `O_CREATE`（`mimei/MiMei.go:905-910`），在本地没有数据时可能**创建一个空文件**，把“缺失”伪装成“存在但为空”。探测请用只读别名 `last`/`release`。

### 2.2 哪些状态可以区分

| 想知道 | 探测 | 可区分的结果 |
|---|---|---|
| 本地有没有元数据 | `GetVar("mminfo", mid)`（`frame/GetVar.go:309-318` → `ReadMiMeiInfoWithCheckRight`） | `30000 No MiMei` = 元数据不在本地；`1101 Permission denied` = 元数据在、但你的权限为 0；成功 = 元数据在且有权限 |
| 本地有没有版本数据 | `GetVar("mmversions", mid)`（`frame/GetVar.go:340-345` → `GetAllVerKeys`） | 返回字段列表；**注意过滤 `cur`**——它是更新时间字段（`mimei/updatetime.go:84-106`），不是版本。过滤后为空 = 本地没有版本记录 |
| 请求的别名在不在 | `GetVar("mmvernamed", mid, "last")` | 返回具体版本号；空/错误 = 没有该别名 |
| 你对某 mid 的权限位 | `MMGetRight(sid, mid, "self")`（`frame/ApiMiMeiDA.go:28-50`） | 0 = 无权限；非 0 可看具体位 |
| 网络上有没有这个对象 | `MiMeiShow(sid, dhts, mid)`（`frame/ApiIpfs.go:647-651` → `pnet/coreapi/coreapi.go:630-676`） | 有：返回签名记录+版本；无：`get mimei(...) info fail ...`。这是**尽力而为**的 DHT 查询，网络不通时不能当作“确定不存在” |

**直接回答**：

- `OpenMM ver is empty` **是**“本地没有该版本副本”的事实信号（文件型、且元数据已在本地），但**不是设计出来的哨兵**：它是裸 `fmt.Errorf`，没有错误码，靠字符串匹配；数据库型对象的行为还不一样——`openMDB` 把 `oVer==""` 当作 `last` 继续打开，只打一条 warn（`mimei/MDb.go:51-55`）。
- “本地无副本 / 全局不存在 / 无读权限”三者**在 `MMOpen` 一个调用里不可区分**；用 §2.2 的组合可以区分**本地状态**，但“全局不存在”只能靠 `MiMeiShow` 的网络查询，且不可完全证伪。
- 权限问题不会让 `MMOpen` 失败，会让后续 `MFGetData`/`MFReaddir` 失败。所以你们“任何读失败都当 absent → 同步一次”会把“无权限”也当成“缺失”，这是当前混淆的根源。

### 2.3 推荐分支

```text
1) mminfo(mid)
   - 30000 → 本地无元数据：
       MiMeiShow(mid)
         - 找到 → 对象在网络上存在；MiMeiSync(mid, SourcePeer=publisher) 后重试
         - 没找到/网络失败 → 不要创建；按“未知/不存在”处理
   - 1101  → 元数据在本地，但你无权限：走授权/换会话，不要靠同步“猜”
   - 成功  → 元数据+权限都有：
       mmversions(mid)（过滤 cur）
         - 空 → 本地无版本数据：同步
         - 非空 → 选 last → release → 最大数字版本，逐个 MMOpen
                 若 last 报 “OpenMM ver is empty”，先试 release/数字版本，
                 不要直接断言“对象不存在”
2) MMOpen 成功后，真正的读失败再按错误类型分：
   - Permission denied(1101) → 权限问题
   - not exist / bad mac 等 → 数据块缺失或损坏，走同步/修复
```

### 2.4 平台侧可选改进（不影响你们现在分支）

给“本地无该版本”一个**带错误码的类型化错误**（例如 `ErrNoLocalVer`），并把“元数据不存在”“无权限”“无本地版本”三者的错误码固定下来，MApp 就不必字符串匹配。这是小改动，但属于新契约面，需要作者点头；本文不代为承诺。

---

## 3. 题 3：`MiMeiSync` 的源提示、时序与重试

### 3.1 源提示已经存在：`SourcePeer`

`MiMeiSync` 的 `param` 支持以下键（`pnet/netpub/cmd.go:16-22`，**大小写敏感**）：

| 键 | 值 | 作用 |
|---|---|---|
| `SourcePeer` | **节点 id**（27 位 LID） | 直接向该节点取版本/数据，`reqType=FromPeer`，跳过 DHT provider 发现（`pnet/coreapi/coreapi.go:715,772-776`；`pnet/dhtMiMei.go:360-390`） |
| `WaitAllData` | 任意值 | 等引用数据也同步完再返回 |
| `reclevel` | 整数 | 引用递归层数（默认 2） |
| `WithoutIpfs` | 任意值 | 不同步 IPFS 数据 |
| `info` | 任意值 | 打开消息通道，回传进度 |
| `forcesync` | 任意值 | 跳过 MAC 校验强制提交（危险，只用于修复） |
| `vers` | 版本串 | **当前被解析但未使用**（`pnet/coreapi/coreapi.go:720-724` 只赋值/打日志；`DoSyncTask` 不接收它）。不要依赖它钉版本 |

所以答案是：**可以**，用 `param["SourcePeer"] = "<发布节点 id>"`，效果等价于 CLI 的 `Leither mimei sync --mid <mid> -s <node-id>`（`pnet/cmd/mimei/net.go:722`）。注意它是**节点 id**，不是 IP/地址；`DoWorkOnNode` 仍要做节点寻址，发布节点离线时照样失败。

### 3.2 为什么“provider 一直在，第一次却失败”

三个独立原因，现场表现都像“过一会儿就好了”：

1. **provider 记录传播滞后**。不带 `SourcePeer` 时，`MiMeiSync` 先做 `MiMeiShow` / `GetProvsByMid` 发现；DHT 里已有 provider 记录，不代表 `GetProvsByMid` 立刻返回（`pnet/dhtMiMei.go:392-475`）。
2. **“网络版本比本地新、但找不到其他 provider”会显式报错**，不再静默成功：`ErrSyncNoProvider` / `no Provider`（`pnet/dhtMiMei.go:1284-1300`；这是 2026-09-01 静默成功问题的修复）。
3. **MApp 侧的 5s 等待预算**。`LApi.MiMeiSync` 用 `api.MAppInLockCallBudget = 5s`（`frame/LApi.go:135-140`、`api/mappcall_budget.go:31`）；超时返回固定文案 `mapp call bounded: node is still working for this app`，但**任务仍在 worker 侧继续**（`pnet/dhtMiMei.go` 的 `waitSyncRet` 只放弃等待，不取消任务）。所以“第一次报错、几分钟后成功”很可能只是等待预算到点，数据其实已经在后台落地。

### 3.3 平台自己的退避节奏

“无 provider”的自动重排是**有界**的：`1m / 5m / 15m / 30m / 1h / 2h`，最多 6 次；显式 `MiMeiSync` 会清零重来（`pnet/dhtMiMeiRetry.go:16-28`）。这可以当作平台对“provider 记录传播”的经验预期：**早期分钟级，长尾小时级**。

### 3.4 推荐模式

1. 发布后，在需要立刻可用的服务节点上**显式同步**：`MiMeiSync(sid, "", mid, {"SourcePeer": publisherNodeID, "WaitAllData": "true"})`。有 `SourcePeer` 时不需要等 provider 发现。
2. 把错误分三档：
   - `mapp call bounded`：**不是失败**；1–2 秒后重试，或直接等后台完成后再读。
   - `ErrSyncNoProvider` / `no Provider` / `no peer can use`：按 1m、5m、15m… 退避重试，不要紧凑轮询。
   - 其它错误（bad mac、not found、权限）：按对应语义处理，不要盲目重试。
3. **不要用 `MiMeiSync` 的返回值当唯一验收**。同步成功只表示“这次拿到了”，最终以 `MMOpen(mid,"last")` / `mmversions` / `mmvernamed` 的读数为准。
4. 如果要“跳过发现”的语义，`SourcePeer` 已经提供了；如果要**钉版本**，当前没有——`vers` 不生效。

### 3.5 文档缺口

`SourcePeer` / `WaitAllData` / `reclevel` / `WithoutIpfs` 这些键在开发者指南里没有成表；`vers` 未生效也没有标注。建议补进 `MiMeiSync` 的 API 说明（这可以只做文档，不改行为）。

---

## 4. 题 4：MApp 能否知道自己的运行版本

### 4.1 没有“运行版本”的 GetVar

逐个排除：

| 调用 | 返回 | 是不是“运行版本” |
|---|---|---|
| `GetVar(sid, "ver")` | 本节点 Leither 软件版本 `pub.ApplicationVer`；带 `arg0` 可查别的节点（`frame/GetVar.go:522-536`） | ❌ 是节点版本，不是应用版本 |
| `GetVar(sid, "appinfo", aid)` | `api.App{ID, Name, Author, Release, Last}`（`app/MFileApps.go:641-655`、`:900-915`） | ❌ 只有 `release`/`last` 两个**别名**，不是解析后的运行版本 |
| `GetVar(sid, "appversions", aid)` | 该应用的所有版本键 | ❌ 是清单，不是“我在跑哪个” |
| `GetVar(sid, "manifest", aid, ver)` | 指定版本的 manifest | ❌ 需要你已知 `ver` |
| `GetVar(sid, "mmvernamed", aid, "last")` | 本地 `last` 解析出的具体版本号 | ⚠️ 近似：能告诉你“本节点现在的 last 是什么”，但不保证就是本次代码加载的那个（见下） |
| `Request["ver"]` | 调用方传的版本**别名**（`app/param.go:118`；文档也写作“应用版本”，`docs/mapp-api-guide.md:27`） | ❌ 调用方控制，可能是空/`last`/`release` |

`Request["ver"]` 在 `initMAppParam` 里只读不删，MApp 确实能看到；但它是**别名**，不是解析后的具体版本号。

### 4.2 为什么“别名”不等于“运行版本”

- `createMApp` 用 `Request["ver"]` 去 `GetManifest`（`app/MFileApps.go:463-489`）；`GetManifest` 的缓存戳会把 `last`/`release` 解析成具体版本号（`manifestVerStamp`），所以**同一节点上别名轮转后缓存会失效、新版本会生效**——这是 #86 的修复，不需要重启。
- 但 MApp 在 `GetVar("mmvernamed", ...)` 拿到的“当前 last”与它本次加载的 manifest 之间有一个小竞态：加载之后、查询之前如果 `last` 又动了，两者会不一致。
- 如果调用方传的是空/`cur`，`mmvernamed` 不能直接解析（空字段会被拒；`cur` 不是版本别名）——所以“别名解析”只在 `last`/`release`/数字版本时可用。

### 4.3 推荐做法

**最可靠的是应用源码里自带 build 常量**，例如：

```go
const Build = "twbe-1729+<commit>"
```

在 `health` 里原样返回 `Build`。理由：

- 它**跟着代码走**：哪个节点加载了哪份源码，就返回哪个常量；不存在“别名解析”竞态，也不依赖调用方传什么。
- 客户端比较各节点 `health.Build` 就能直接发现 skew；你们现有的“按能力探测 + 回退旧 wire form”可以保留，`Build` 只是把 skew 从“不可见”变成“可见”。
- 如果还想同时报告“请求的别名”和“本节点解析出的具体版本”，再加两个字段：`Request["ver"]` 和 `GetVar("mmvernamed", aid, ver)`（仅在 `ver` 是 `last`/`release`/数字时）。

注意：Go MApp 的编译产物缓存按 `(appid, 原始版本别名)` 建键，并用 manifest 内容指纹识别“内容变了”重建（`app/goAppCache.go` 的 `goAppCacheEntry.digest` 注释）。所以**只要本地 `last` 更新了，下一次请求就会用新代码**，不需要重启；问题只剩“本地 `last` 什么时候更新”。

### 4.4 收敛预期：无 SLA，pull-based

- 没有“应用更新推送”。`createMApp` 读的是**本地** manifest（`app/MFileApps.go:463` → `readManifest` → `OpenMFile`，`mimei/MiMei.go:1404`）；本地没有新版本，就不会自己去网络拉。
- 服务节点要看到新版本，必须有东西触发同步：显式 `MiMeiSync`、`OnUpdate` 的 DHT 记录传播、或后台引用链同步。平台没有“应用版本订阅”。
- 因此**没有文档化的收敛时间**；可能几分钟（provider 记录传播/后台同步），也可能一直不收敛（没人触发）。你们观察到的“几分钟后出现”与这个模型一致，但不是保证。
- 发布流程里可靠的做法：发布后对每个服务节点执行 `MiMeiSync(appMid, SourcePeer=publisher)`，再用 `mmvernamed` / `health.Build` 验收（`docs/RELEASE_AND_DEPLOY.md:58` 对官网 mid 就是这种显式 sync 模式）。

---

## 5. 题 5：`fromnid` 的裁决状态与可行性订正

### 5.1 裁决状态：未裁决

截至 2026-11-16：

- 仓库内 `fromnid` 只出现在上一份回答 `planning-artifacts/tweet-structural-answers-2026-11-13.md` §7.1，没有任何实现/裁决记录。
- 看板无对应卡片（`fromnid` / `调用节点` 检索无命中；`身份` 命中的卡片均不是此项）。
- 记忆索引与 `planning-artifacts/direction/` 也没有裁决。

所以准确回答是：**作者尚未裁决**；上一份 §7.1 是“可行性判断”，不是决定。如果作者已在别处口头裁决，请以裁决为准。

### 5.2 上一份 §7.1 的实现细节需要订正

上一份写“入口已有 `NodeApiClient.Sid`/服务端会话，把它作为 `fromnid` 注入即可”。这句话**在调用侧成立，在目标侧不成立**：

- 跨节点调用链是 `remoteMApp.RunMApp` → `wa.RunMApp(entry, request, args)`（`app/MApp.go:188-222`）。`wa` 是 `*api.NodeApiClient`，它确实有 `Sid`（调用方在目标节点上的节点会话）。
- 但 `RunMApp` 的 RPC 方法签名是 `RunMApp(entry string, mapRequest map[string]string, args []any, stropts ...string)`（`frame/ApiApp.go:46-58`），**没有 sid 形参，也没有每请求上下文**；客户端走 hprose over ws（`api/Login.go:48-75`），目标端 `AddAllMethods` 零 filter 挂载（`pnet/webfunc.go:17-26`）。目标端的 `WebApi.RunMApp` 拿不到“这个请求是哪个传输层会话发的”。
- 跨节点转发前 `sid` 会被删掉（`app/MFileApps.go:301-308`），所以也不能指望 `Request["sid"]`。
- `remoteMApp` 只把 `wa.Sid` 用在 `GetVarObj(nodeappcode, aid)` 上（`app/MApp.go:203-209`），没有把它传给 `RunMApp`。

因此，要得到“传输层设置、MApp 可见”的 `fromnid`，至少要在 RPC 入口加一个**携带调用方会话的形参/新方法**，或者做请求签名；不是“读一下现成字段”。这会影响可行性评估的工作量。

### 5.3 三个可选形态

| 形态 | 做法 | 能证明什么 | 残余风险 |
|---|---|---|---|
| A. 现有 `nodeappcode`（不改） | 目标节点铸造、调用方回传；MApp `SessionGet(code,"nodeid"/"forapp")`（`frame/Auth.go:975-1010`） | “出示者知道这串码”，且码是目标节点在该调用方会话下铸的 | bearer；泄漏后 ≈1h 内可重放；任何登录用户都能铸（`frame/Auth.go:988-991` 只要求非空会话） |
| B. 新增 `RunMAppWithSid`/形参，服务端覆盖写 `fromnid` | 调用方把 `wa.Sid`（目标节点上的节点会话）作为参数传；目标 `GetUidFromSession` 解析，校验会话类型=节点（可新增会话标记），然后**覆盖写** `Request["fromnid"]` | 目标端认证过的会话身份（节点 id）；比“任意人可铸的码”来源更强 | 仍是 bearer：sid 本身可重放（≈1h）；不是签名 |
| C. 签名请求（强证明） | 调用节点用节点钥对 `(entry, request 摘要, 时间戳/随机数)` 签名；目标验签后注入 `fromnid` | 真正的节点 attestation；不可重放（配 nonce/时间窗） | 改动最大；需要新格式/兼容期 |

如果作者只想要“比 `nodeappcode` 更干净的来源”，B 是成本最低的；如果目标是“替换 bearer 信任根”，只有 C 够。上一份 §7.1 说的“防伪造就服务端覆盖写入”只在 B 的意义上成立——覆盖写入防的是**应用伪造请求字段**，不防**会话泄漏重放**。

### 5.4 你们能提供的证据

有用，但不是决定性的（决定性的部分在平台代码/裁决）：

- 一份脱敏的跨节点请求样本：`sid` 是否为空、`nodeappcode` 是否存在、`SessionGet(code,"nodeid")`/`"forapp"` 的值、`author`、`aid`、`entry`、时间。
- 你们是否在任何客户端/日志/URL 里暴露过 `nodeappcode`（泄漏面）。
- 你们需要的是“调用**节点**身份”还是“调用**用户**身份”——B 只能给会话 uid（可能是节点，也可能是普通用户），C 可以签节点钥。
- 你们的 peer guard 现在用 `nodeappcode` 做的判断（`forapp` 比对、`nodeid` 白名单等），以及如果换成 `fromnid` 希望保留哪些。

### 5.5 建议

- 在裁决前**不要把 messages 也迁到 `nodeappcode` 模式**；现有 peer guard 先保持，并把 `forapp` 校验、TTL 认知写进代码注释。
- 如果只是为了“少一个 bearer 码”，B 的收益有限；如果目标是“attested caller”，应直接评估 C，别在 B 上花两次工。
- 这一步需要作者拍板，本文不代为承诺版本或时间。

---

## 6. 题 6：`BELoginAsAuthor` 冻结下的迁移指引

### 6.1 冻结的准确口径

- 代码标记：`lapi/backend.go:131-140` 与 `app/MFileApps.go:1124-1137`，`Deprecated: 2026-09-03 冻结（冻而不废）`；`app/MFileApps.go:1132` 每次调用落 `[BELA-CALL] appid=... authID=...`。
- 裁决原文：对外不再演进、**禁止新增依赖**；接口继续接线可用、零行为变更；删除走观测驱动，**不设硬性删除期限**；只做注释+文档标记，**不做 CI 静态检查**（`planning-artifacts/milestone-2026-09/T1/f4-server-side-key-holding-analysis.md:374-381`；`docs/KEY_AUTH_MECHANISM_AUDIT.md:138-153`）。
- `BELoginAsAuthor()` 的实现只是 `CreateUserSession(authID)`（`app/MFileApps.go:1128-1133`），**不需要作者私钥**。这是它能在任意持有应用记录的节点上工作的原因，也是冻结要收敛的理由：它把“作者身份”变成了节点可自取的会话。
- 冻结的是“应用可自取作者会话”这条授权捷径，**不是删除“作者身份”概念**；数据区 owner=作者、上传/备份/发布权模型、服务端 `CreateUserSession` 都不动。

### 6.2 “known internal path”是什么意思

裁决里的“已知内部路径”指的是**仓库内平台自己拥有的调用点**（当时枚举的 `rpcBEReadFile`、宿主 `getBEReadFile`、apitest 等；见同段 §8-6）。你们的 Tweet 后端是**外部 MApp**，不在这个清单里；它的调用正是 `[BELA-CALL]` 观测要捕捉的“外部调用”。

所以：

- **不算**“known internal path”。
- 但这也不等于“你们被针对”：删除条件是“确认除已知内部路径外无外部调用后废弃”，外部调用越多，删除越会被推迟；没有期限，也没有 CI 强制。
- 现实结论：对你们这种“整个存储层都建在它上面”的应用，冻结在可预见的未来是**长期可用**的；但它是 frozen API，不会再有新能力，不能当作永久契约。

### 6.3 替代件逐项核对（关键）

很多人会把“替代方向 = 应用身份 + 节点代劳”理解成“把 `BELoginAsAuthor()` 换成 `BELoginAsApp()` 就行”。**当前代码不是这样**：

| 能力 | `BELoginAsAuthor`（冻结） | `BELoginAsApp` | `BEOpenAppDataApp`（mmsid） | 作者给 appid 授权 |
|---|---|---|---|---|
| 新建任意 mid | ✅（uid=作者） | ❌ `MMCreate` 只读 `SessionKey_UserID`（`mimei/MiMei.go:196-218`），App 会话只有 `SessionKey_AppID`，直接 `ErrInvalidSid` | ❌ 只创建/打开“应用数据区”一个 DB mid，不是任意 mid | — |
| 打开/读写某 mid | ✅ | ⚠️ `MMOpen` 走 `GetUidFromSession`（有 AppID 回退）可开；但打开后的读写要 appid 在 ACL 里有权限 | ❌ 仅数据区 | ✅ 打开后可读写 |
| `MMBackup` / `MMRelease` / `MMAddRef` | ✅ | ❌ 直接传 appid sid 不行：这些走 `resolveDataOwner`（`mimei/MiMei.go:1161-1187`），只认 `SessionKey_UserID` 或 **mmsid 的 `UId`**；app 会话两者都没有 | ✅ 数据区 mmsid 的 `UId`=作者，备份/Release/引用都按作者权限走 | ⚠️ 需先用 appid `MMOpen` 拿到 mmsid，再用 mmsid 调；直接传 appid sid 仍失败 |
| 发布新 mid（`MiMeiPublish`） | ✅（作者钥；或节点钥 + 记录上的 Publish 权） | ❌ appid 无密钥；`getPublishKeyPair` 回退节点钥后，`canPublish` 对**新记录**要求 `uid == Author`（`pnet/coreapi/coreapi.go:157-199,256-268`） | ❌ mmsid 不是登录身份，`GetUidFromSession` 会拒绝（5031） | ❌ 仅“旧记录已存在、调用身份与节点在该记录上都有 Publish 权”时可代发；首次发布仍要作者钥（在发布节点上） |
| 应用级 KV/配置 | — | N/A（用 `BEOpenAppDataApp`，与会话无关） | ✅（一个应用一个 DB 数据区，作者所有） | — |

关键点：

1. **`BELoginAsApp` 不能替代存储层**。它的会话只设 `SessionKey_AppID`（`frame/Auth.go:671-676`），而 `MMCreate` 和 `resolveDataOwner` 都不看 AppID。新建、备份、发布、引用这些“写操作”会直接失败，不是权限不够，是解析不到属主。`GetUidFromSession` 有 AppID 回退，所以 `MMOpen`/`GetVar` 这类“身份读口”能工作——但这只覆盖读/打开，不覆盖写。
2. **`BEOpenAppDataApp` 是当前覆盖存储面最完整的“节点代劳”通道**。它不看调用会话，直接按应用记录里的 `Author` 打开/创建 `AD_App` 数据区（`mimei/AppData.go:43-56`），返回的 mmsid  `UId`=作者；`MMBackup`/`MMRelease`/`MMAddRef` 在这个 mmsid 上都按作者权限工作（`resolveDataOwner` 第二步）。代价是它只给你**一个应用级 DB 数据区**，不是“每条推文一个文件 mid”的模型；而且要接受它的合并语义（hash 按 field 合并、list 整键 last-writer-wins、秒级可见性滞后，见 `MAPP_DEVELOPMENT_GUIDE.md` §4）。per-user 变体 `BEOpenAppDataUser` 要求 `owner == 当前会话 uid`（`mimei/AppData.go:158-186`），跨节点请求的 sid 已被删，所以不能拿来做跨方写。
3. **作者给 appid 授权**（`MMSetRight`，作者有 Admin/Super 才能设；`mimei/rights.go:236-252,306-308`）可以让 appid 在**已存在的 mid** 上有读/写/备份权；配合 `MMOpen` 拿到的 mmsid，`MMBackup`/`MMRelease` 可以工作。但它**不能创建新 mid**（`MMCreate` 仍失败），也不能解决首次发布（`canPublish` 要求 Author）。所以它是“维护存量”的方案，不是“新建”的方案。
4. **`BESignPPT`/`BESign` 不是替代**：它们用节点系统钥签名，`CertFor=App`，不是登录凭证（`app/MFileApps.go:1141-1174`；登录侧只接受 `Bind`/`Self`/`Gift`，`user/UserBindID.go:143-172`）。

### 6.4 所以：有没有 sanctioned migration path？

**对“每条对象一个 author-owned file mid”的画像：当前没有完整的 sanctioned path。** 能做的和不能做的分开：

- 能：存量 mid 的读/写/备份/引用（授权 + mmsid）；应用级 KV/配置（`BEOpenAppDataApp`）；如果旧记录已存在、且调用身份与节点在该记录上都有 Publish 权，后续发布可由节点钥代发。
- 不能：**新建** author-owned 的 per-object mid；**首次发布**新 mid（除非发布节点持有作者钥）。
- 这不是你们应用写法能补的，是平台缺少“以应用/作者名义、由节点代劳创建并发布”的原语。

### 6.5 建议

1. **不要为了“去冻结”而做半迁移**。把 `BELoginAsAuthor` 的调用收拢到一个接口/适配层（storage identity），保持调用点数量不再增加；这样将来若有替代原语，替换点集中。
2. 新增的、**应用级**的状态（配置、计数、索引）优先用 `BEOpenAppDataApp`，不要再扩大 author-session 的使用面。
3. per-object 的推文/评论 mid 继续走现有路径；在平台补上“节点代劳创建/发布”之前，不要改成 `BELoginAsApp`——那会在 `MMCreate`/`MMBackup` 上直接坏掉。
4. 把“`BELoginAsAuthor` 的替代件不完整”作为**冻结/删除决策的输入**反馈给作者：如果将来要删除，至少需要补上 create-as-author/app、backup、release、first-publish 四件事，否则这类应用会被搁浅。这不是要求排期，是要求决策时知情。
5. 如果作者愿意评估最小补丁：让 `MMCreate`/`resolveDataOwner` 接受 `SessionKey_AppID`（或新增 node-on-behalf 入口），并给 appid 一个发布委托（节点钥 + 记录上的 Publish 权）。这两件都不小，且发布侧仍要解决“首次发布无记录”的签名问题；不要低估。

---

## 7. English summary（可直接贴回 memo）

**1. mark→mid.** The derivation is deterministic: `sha256(uid \x06 appid \x06 ext \x06 mark)` (legacy trunc-20 id, type byte `tp` in the header). It is re-computed by every receiving node on the `/mimei/` DHT verification path and mismatches are rejected in `strict` mode (`pnet/coreapi/verify.go:173-179`), so it is a **protocol invariant**, not a documented public promise. The namespace is exactly `(uid, appid, ext, mark, tp)`; there is **no separate object-kind field**, so a tweet and a comment sharing appid/ext/tp and the same requestid will collide. Your `<requestid>-quote` suffix avoids it; giving each kind its own `ext` is cleaner. `MMCreate` returns the existing mid on a collision but does **not** verify content, and idempotency only holds for the same acting uid.

**2. Absence signal.** `OpenMM ver is empty` is not a designed sentinel; it is an untyped error returned for **file-type** objects when the metadata is local but the requested version alias is missing from the local version map. Metadata absence is `No MiMei mid[...]` (code 30000); rights are **not** checked at open and must be read with `MMGetRight`; a global existence check requires `MiMeiShow`. So local states are distinguishable with `mminfo` + `mmversions` (filter out `cur`, which is an update-time field), but `MMOpen` alone is not enough. Do not probe with `ver="cur"` (it can create an empty file) or `ver=""` (it yields a different `no ver()` error).

**3. MiMeiSync source hint.** Yes: `param["SourcePeer"] = "<node-id>"` (case-sensitive) makes `MiMeiSync` go directly to that node (`FromPeer`) and skip provider discovery. It is a node id, not an address. The older `vers` param is parsed but currently ignored. The MApp-side `MiMeiSync` has a 5s wait budget; `mapp call bounded` means the worker is still running, not that the data is absent. There is no push and no documented convergence time; the platform's own no-provider backoff is 1m/5m/15m/30m/1h/2h. For deterministic rollout, sync each serving node explicitly after publish.

**4. MApp version.** No GetVar returns the running app version. `GetVar("ver")` is the node software version; `appinfo` returns only the `release`/`last` aliases; `Request["ver"]` is the caller-supplied alias. `mmvernamed` can resolve an alias on the local node, but the only exact “running code” signal is a build constant embedded in the app source and returned in `health`. Convergence is pull-based: a serving node advances only when something syncs the app mid; there is no update push and no SLA.

**5. fromnid.** No ruling is recorded (repo, board, memory index) as of 2026-11-16. The earlier feasibility note also needs a correction: the `RunMApp` RPC signature carries no sid/context (`frame/ApiApp.go:46-58`), and `sid` is deleted before forwarding (`app/MFileApps.go:301-308`), so the target cannot see the transport session. Options: (a) a new entry/param carrying the caller's node session, server-verified and overwritten as `fromnid` (still bearer, ~1h); (b) a signed caller declaration (true attestation); or (c) keep `nodeappcode` with `forapp` checks. This needs your decision.

**6. BELoginAsAuthor freeze.** Your usage profile is **not** a “known internal path” — that term means in-repo platform call sites; external MApp calls are exactly what the `[BELA-CALL]` observation is meant to see. The freeze has no deadline and no CI enforcement. But there is **no complete migration target today**: `BELoginAsApp`’s session carries only `SessionKey_AppID`, while `MMCreate` (`mimei/MiMei.go:196-218`) and `resolveDataOwner` (`mimei/MiMei.go:1161-1187`, used by backup/release/ref) read only `SessionKey_UserID` or an mmsid owner. Existing author-owned mids can be maintained via author-granted appid rights + an mmsid; new per-object mids cannot be created under app identity, and first publish still requires the author key. `BEOpenAppDataApp` covers only the app-level DB data region. Treat the frozen API as long-term for this app, contain the dependency behind one interface, and feed the missing node-on-behalf create/backup/publish primitive into the removal decision.

---

## 8. 证据索引

| 主题 | 位置 |
|---|---|
| `CreateMID` 派生 | `mimei/mmpub/pub.go:31-41`；`pub/IDManger.go:116-120`；调用点 `mimei/MiMei.go:306,333` |
| 验签链 mid 重算 / 默认 strict | `pnet/coreapi/verify.go:1-19,42-60,173-179` |
| `MMCreate` 幂等（吞 `ErrFmtMiMeiExist`） | `mimei/MiMei.go:196-240,349` |
| 创建串行化键 | `mimei/MiMei.go:251-275` |
| 内部测试规范的幂等表述 | `docs/API_INTEGRATION_TESTING.md:38,83` |
| `OpenMM ver is empty` | `mimei/MiMei.go:898,929`；`mimei/MFile.go:243,266,290` |
| `GetVerByName` / 空字段错误 | `mimei/mmpub/MMVers.go:75-82`；`mimei/mmdb/redis/t_hash.go:14-23,175-186` |
| 数据库型对空版本的宽容 | `mimei/MDb.go:51-55` |
| 本地元数据 / 30000 | `mimei/cache.go:25-52`；`mimei/mmpub/const.go:166` |
| 权限检查点 | `mimei/rights.go:190-202`；`mimei/MFile.go:166-185,411,501` |
| `mminfo` / `mmversions` / `mmvernamed` | `frame/GetVar.go:309-345` |
| `MMGetRight` | `frame/ApiMiMeiDA.go:28-50` |
| `MiMeiShow` 网络查询 | `pnet/coreapi/coreapi.go:630-676` |
| `MiMeiSync` 参数 | `pnet/netpub/cmd.go:16-22`；`pnet/coreapi/coreapi.go:695-780` |
| `SourcePeer` 直连 | `pnet/coreapi/coreapi.go:715,772-776`；`pnet/dhtMiMei.go:349-390` |
| MApp 侧 5s 预算 | `frame/LApi.go:135-140`；`api/mappcall_budget.go:31` |
| 无 provider 错误 / 退避 | `pnet/dhtMiMei.go:1284-1300`；`pnet/dhtMiMeiRetry.go:16-28` |
| `ver` GetVar = 节点版本 | `frame/GetVar.go:522-536` |
| `appinfo` 只有 release/last | `app/MFileApps.go:641-655,900-915`；`lapi/lapi2.go:86-93` |
| `Request["ver"]` | `app/param.go:118`；`docs/mapp-api-guide.md:27` |
| manifest 本地读取 | `app/MFileApps.go:463-489`；`mimei/MiMei.go:1404` |
| Go 缓存 digest | `app/goAppCache.go` 的 `goAppCacheEntry.digest` 注释 |
| `RunMApp` 无 sid | `frame/ApiApp.go:46-58`；`api/Login.go:48-75`；`pnet/webfunc.go:17-26` |
| 跨节点 `sid` 删除 / nodeappcode | `app/MFileApps.go:301-308`；`app/MApp.go:203-209` |
| 冻结标记 / `[BELA-CALL]` | `lapi/backend.go:131-140`；`app/MFileApps.go:1124-1137` |
| 冻结裁决原文 | `planning-artifacts/milestone-2026-09/T1/f4-server-side-key-holding-analysis.md:374-381`；`docs/KEY_AUTH_MECHANISM_AUDIT.md:138-153` |
| `CreateAppSession` 只设 AppID | `frame/Auth.go:664-676` |
| `MMCreate` 只读 UserID | `mimei/MiMei.go:196-218` |
| `resolveDataOwner` 只认 UserID/mmsid | `mimei/MiMei.go:1161-1187` |
| `BEOpenAppDataApp` 按作者打开 | `mimei/AppData.go:43-56` |
| `MMSetRight` 需 Admin + 27 位成员 | `mimei/rights.go:236-252,306-308` |
| 发布取钥 / canPublish | `pnet/coreapi/coreapi.go:157-199,256-268` |
| `BESignPPT`/`BESign` 节点钥 | `app/MFileApps.go:1141-1174`；`user/UserBindID.go:143-172` |
| appdata 合并语义 | `MAPP_DEVELOPMENT_GUIDE.md` §4 |

---

## 9. 需要作者裁决 / 需要平台决定的事项

以下不是本文能代为承诺的：

1. **`fromnid` 是否做、做 A/B/C 哪一档**（§5.3）。这是新能力，涉及跨节点身份与重放面。
2. **是否补“节点代劳创建/发布”原语**（§6.4）。如果未来要删除 `BELoginAsAuthor`，这是前置；否则“每条一个 author-owned mid”的应用没有替代路径。
3. **是否把“本地无该版本”做成类型化错误**（§2.4）。小改动，但属于新契约面。
4. **是否把 mark→mid 派生与 `MMCreate` 幂等语义写进公开开发者指南**（§1.4）。纯文档。

在这些决定之前，本文的建议是：第 1 题按“协议内稳定”依赖、第 2 题按组合探测分支、第 3 题用 `SourcePeer` + 退避、第 4 题用 build 常量、第 5 题等裁决、第 6 题收拢依赖不做半迁移。

---

## 10. 给对方的复现/验收建议

- 幂等：同 `requestid` 连发两次 `MMCreate`，确认返回同 mid；再用同 requestid 发**不同内容**，确认仍返回旧 mid（说明它不校验内容，应用侧需自己防）。
- 缺失分支：对“新账号首次被对端引用”的场景，分别记录 `mminfo`、`mmversions`、`MMOpen("last")` 的返回；确认是 30000 还是 `OpenMM ver is empty`。
- 同步：发布后立即 `MiMeiSync(mid, {"SourcePeer": publisher})`，记录返回值；若拿到 `mapp call bounded`，等 2s 再读 `mmversions`，确认后台是否已落地。
- 版本：在 `health` 里加 `Build` 常量；发布后用 `mmvernamed` 与各节点 `Build` 对比，观察收敛时间。
- 冻结：在测试节点 `grep '\[BELA-CALL\]' logs/Leither.log`，确认你们自己的调用画像（appid/authID/频率），这也是给作者评估删除风险的直接材料。

---

*本文为维护方对当前代码的判读，不是对 `lapi` 公开契约的变更；引用代码行号以 HEAD `a3ce96e8` 为准。*
