---
name: aoci-cognition-maintenance
description: "Maintain an AOCI cognition layer in any repository — bootstrap AOCI into a project, write or update index entries (FRAS v2), handle managed-scope changes, or recover when aoci_maintain / verify / check / guide reports drift. Pairs with gitnexus: check impact before editing, update cognition after. Use when a repo has or should have an .aoci/ directory."
---

# AOCI 认知层维护

AOCI 存**语义与踩坑事实**，gitnexus 存**调用关系图**：前者管"改之后意味着什么"，后者管"改之前会坏谁"。

## 〇、先确认能不能用（60 秒，缺哪补哪）

| 步 | 命令 | 缺了怎么办 |
|---|---|---|
| ① 二进制 | `aoci --version` | 从官方 release 取签名包，按 `docs/install.md` 的 basic / recommended / full 三档验证之一，**并报告走了哪档**。不用来源不明的二进制 |
| ② 项目契约 | `ls .aoci/` | `aoci --repo <项目绝对路径> init --agent <host>` |
| ③ Baseline | `aoci scan` | **别跳过这步**：没有 Baseline 时 `guide` 直接 `stage=blocked`；`scan` 后变 `authoring_required`，并告诉你 index 文件数与预计 Maintain 轮数 |
| ④ 下一步 | `aoci index agent guide --agent <host>` | 按它给的 `next_action` 走，**别自己猜**。首次授权阶段典型值：`call_no_argument_aoci_maintain_for_current_machine_batch` |

`init` 实际会写四样东西（都是项目级，不污染全局）：`.aoci/` 骨架与 `config.json`、`AGENTS.md` 的 aoci 管理块、
host 的 MCP 配置（如 `claude` → `.mcp.json`）、以及把该 MCP 文件加进 `.gitignore`（不进 Git、不进 Managed Scope）。
`init` **不调模型、不编造条目**——它只搭骨架。

**`init --agent` 一条命令打通 MCP 接入**——写的是**项目级**配置（不是全局），所以多项目天然并存：

```
aoci --repo /abs/path/to/repo init --agent claude|codex|cursor|opencode|qoder|all
# 可选：--hooks（Claude Code 写前钩子 / Codex 上下文压缩）
#       --locale zh-CN   --scope-profile production|full|custom
```

（`--agent` 的可选值以 `aoci init --help` 为准，release 间可能增减。）

**host 不在上面那个列表里**（自研 agent，或 MCP 配置不落在项目级文件的宿主）→ 手工加一条 **stdio** MCP：

```
command : <aoci 绝对路径>
args    : ["--repo", "<项目绝对路径>", "mcp"]
```

两个坑：路径用**正斜杠**（MSYS / Git Bash 会把 `X:` + 反斜杠 变成 `X://` 导致进程起不来）；`--repo` 必须指**当前项目根**，否则工具照常返回但改的是别的仓库——**静默出错，比报错更难发现**。

**冲突时谁说了算**（不要跳过这条）：

```
项目内 AGENTS.md（init 生成）  >  aoci index agent guide（运行时 live）
                              >  本 skill（方法层）
```

本 skill 讲**怎么做**，不覆盖项目自己的契约；两者冲突以项目为准。

## 一、When to Use / When NOT

**用**：新建或维护认知条目（FRAS v2、S 字段）；`aoci_maintain` 报 candidates 或漂移；`verify` / `check` /
`index agent guide` 任一处非 aligned；要改作用域（`.aoci/config.json`）；要建立"认知可靠"收据。

**两条入口先分清**：MCP `aoci_maintain` 是**唯一**签发候选的地方（返回 `candidate_id` /
`source_sha256` / `code_plan.batch_id`）；CLI `aoci index update` 只把漂移**分类**（changed / new /
pending_curation / deleted…），**不签发**任何身份。CLI 没有 `aoci maintain` / `aoci overview`（报 `unknown command`）。

**不要调 `aoci_maintain`**：

| 情形 | 原因 / 可判定信号 |
|---|---|
| 纯只读问答、分析、版本核验 | 没有代码变更，候选全是噪声 |
| 受管对象**还在改动中** | 拿到一批马上又失效的候选。判据：`git status --porcelain` 非空、还有未 apply 的 staged 批次、或 `check` 处于 blocked |
| 已 `aligned=true` 且 `refresh_status=refresh_not_required` | 无漂移，重复调用只刷新 ledger |

唯一例外 `aoci_maintain(intent=cognition_optimization)`——aligned 状态下**唯一**允许复核已存在条目的入口，
可配 `object_refs: ["code:<仓库相对路径>"]` 限定范围；用完 `optimization.state=complete` 且 `replaced` +1。

## 二、完整链路

```
改之前   ① gitnexus impact / trace              改这个会坏谁
         ② 改代码
改之后   ③ aoci_maintain → aoci_update_entry     更新 F/R/A/S（MCP）
         ④ aoci verify + check + guide           证明对齐
         ⑤ aoci_overview + attestation           认知可靠收据
```

**分工判据**：只有 ① 没 ③④ → 改坏调用方且下一个人重复踩坑；只有 ③④ 没 ① → 认知漂亮但已弄断 d=1。
`S` 字段是两条链的接缝：gitnexus 说"谁依赖它"，S 说"依赖它的那个坑在哪"。

**每次改动对账单**：
- [ ] 改前跑过 `impact`，看过 d=1 清单
- [ ] 代码已到**最终稳定状态**（`git status --porcelain` 为空、无未 apply 的 staged 批次）
- [ ] `aoci_maintain` → `remaining=0`
- [ ] `check` 输出「✓ 可提交」（Entries 漂移 / 待策展 / 字典 / 格式 / 草稿 五净），且 `guide` 为
      `stage=aligned complete=true next_action=none`

## 三、改之前：先用 gitnexus 摸清影响面

**前置两步**：① `gitnexus analyze` 建索引——没建过四个工具全返回空；② `list_repos`（或 `group.yaml`）拿
`repo` 名。**四个工具都必填 `repo`**（跨仓用 `@<groupName>`）；`impact` 另必填 `direction`，
`query` 另必填 `search_query`（**不是 `query`**）。

| 问题 | 工具 | 关键参数 |
|---|---|---|
| 改它会坏谁？ | `impact` | `repo`、`direction: "upstream"`、`target`、`maxDepth: 3` |
| 这符号长什么样、谁调它、它调谁 | `context` | `repo`、`name`、`file_path`（同名消歧）、`kind` |
| A 怎么走到 B？ | `trace` | `repo`、`from`、`to`、`from_file` / `to_file` |
| 这块代码在哪个流程里？ | `query` | `repo`、**`search_query`**、`task_context`、`goal` |

**读结果顺序**：d=1（WILL BREAK，直接调用方）→ d=2（LIKELY AFFECTED）→ d=3（MAY NEED TESTING）。
d=1 里每一个都要在改完后确认还活着——它们同时是条目 `R:` 要写的内容。

**`trace` vs `impact`**：`impact` 给扇出（谁依赖我），`trace` 给链（A→B 经过哪几跳，带 `file:line`
与每跳 edge type + confidence）。调试"这条路怎么通的"时，一次 trace ≈ 3–8 次手搓 context。

**这一步的产物不是结论，是候选写入项**：d=1 里凡属"作者不知道的调用关系"，都应在 ③ 落到 `R:` 或 `S:`。

## 四、改之后：AOCI 闭环

主路径走 MCP——**只有它能签发候选身份**；CLI 只做校验和单条落盘。

```
① aoci_maintain({})        领候选。记下 code_plan.batch_id；
                           每条候选的 candidate_id / path / source_sha256 原样保留，一个字别手打
② 整批作者化后一次提交：
   aoci_update_entry({
     code_batch_id: "<code_plan.batch_id>",   // 不是 authoring_batch.batch_identity，两者不是一回事
     entries: [{ candidate_id, path, source_sha256, new_entry }, ...]
   })
③ remaining 非零 → 立刻再调 aoci_maintain，直到 remaining=0，中间不问用户
④ 证明对齐（CLI）
```

```bash
aoci verify && aoci check
aoci index agent guide --agent <host>   # --agent 必填；期望 stage=aligned complete=true next_action=none
```

**CLI 两条路径**（都不签发身份，`candidate_id` / `source_sha256` 仍只能来自 `aoci_maintain`）：
- 单条直写 `aoci update-entry --path <相对路径> --entry '<整行条目>' --source-sha256 <hash>`，先加
  `--preview` 看红绿 diff；它没有 `--candidate-id` / `--code-batch-id`（那是 MCP 字段）
- 批量走 staged 协议 `aoci index entries check` → `diff` → `apply`（`recover` 收尾）

`aoci_maintain` 只签发候选，**不创作** F/R/A/S、不压缩、不降 C——语义永远由作者写。

**判干净的硬标准**：`check` 五净（Entries 漂移 / 待策展 / 字典 / 格式 / 草稿）。`verify` 的告警也要读，
尤其"E 规模档位错配"——不阻断，但会一直挂着。

**两个前置例外**：新仓 / 索引不完整必须先用 `guide` 走完整流程；`guide` 一直 `evidence_required`
说明要先做 overview + attestation，不是继续 `aoci_maintain`。

## 五、条目怎么写（FRAS v2）

格式：`文件名[标签]: F:职责 | R:关系 | A:锚点 | S:不可推论的事实`

**文件名写裸名，不带目录前缀**——AOCI 按段目录（`===<段路径>===`）自动拼路径；带前缀会告警「含目录前缀，已归一为裸文件名」。

### 标签 = A + B + C + E

| 位 | 取值 | 判据 |
|---|---|---|
| **A 层** | C 共享基础 / E 入口边界 / A 应用编排 / D 领域逻辑 / K 算法 / M 中间件 / P 持久化 / I 集成适配 / R 运行基础 / L 库 / F 声明配置 / O 运维交付 / T 测试验证 / S 文档规范 / X 开发工具 / Z 其他 | 按对象**真实角色**选 |
| **B 模块** | G 跨域通用 / U 用户交互 / B 核心业务 / D 数据状态 / I 身份权限 / N 网络协议 / M 消息事件 / S 安全隐私 / C 配置策略 / O 可观测性 / R 可靠性恢复 / P 性能资源 / W 流程调度 / A 分析智能 / V 构建发布 / Q 质量保障 | 同上 |
| **C 重要度** | 9 最高 → 1 最低 | 核心链路 9，工具/守卫 8，一次性脚本 6，文档 3–7 |
| **E 规模** | L >400 行 / M 200–400 / S 100–200 / T <100 | 按 `grep -c ''` **实际行数**；不一致 → verify 报「E 规模档位错配」 |

**A 层最容易写歪**：不是"越通用越好"，同一对象常被误判层位（把领域逻辑判成"跨域通用"）——
**取 `aoci_header` 看字典，不要凭记忆**。

### 字段配额（超了直接拒）

F ≤ 160 字；R ≤ 360 字 / 8 项；A ≤ 400 字 / 6 项；**S 按 C 档位**：C9–8 ≤ 600 字（token ≤ 200）、
C7–4 ≤ 200（token ≤ 80）、C3–1 ≤ 50。S 的 token 按 UTF-8 字节 / 3 计（一个中日韩汉字 ≈ 1 token）。
超了报 `entry_field_budget_exceeded`，要求**压缩语义重写**，不是机械截断。

### S 字段准入（最容易写歪的一栏）

| ✅ 该写 | ❌ 不该写 |
|---|---|
| 真实踩过的坑，带日期与实测数字 | 从 F/R/A 能推出来的东西 |
| 违反直觉的约束（"上游不支持按分数排序"、"水位的 `>=` 是刻意的重取边界"） | 泛泛的"要注意质量" |
| 双真源 / 废弃副本（"真值在 X，本文件已废弃"） | 复述代码注释 |
| 平台 / 环境限制（"`--check` 只能在对端机跑"） | 已过期且无防坑价值的历史 |

**对 C7+ 对象写 `S:-` 之前**，必须先在真正决定行为的地方真找过一圈（实现、拦截器、SQL、配置、模板）。
**没看就写 `S:-` 与编造同等错误。**

## 六、Scope 变更（必走人工审批 + 一个致命时序陷阱）

```bash
aoci scope status        # 1. 先看 index / observe / exclude 三项计数，别记数字，现查
# 2. 改 .aoci/config.json（exclude_dirs 或 managed_scope.rules）
aoci scope activate      # 3. 生成变更单：preview 落在 .aoci/scope-change/，且它会打印出下面两步
# 4. 下面两步【必须由真人执行】——命令照抄 activate 的输出，别自拼 preview 路径
aoci scope approve --preview-file <...> --actor <id> --out-file <...>
aoci scope apply   --preview-file <...> --approval-file <...>
```

**两种审批产物别混**：`--approval-file` 收 `managed-scope-change-approval/v2`（常规）；
`--authorization-file` 收 `aoci-policy-bound-approval/v1`（项目**启用了 policy-bound 审批**时才用）。
两者不是同一份产物，传错校验失败。走哪种看 `.aoci/config.json` 的 policy 配置，不要猜。

### ⚠️ 时序陷阱

**`scope activate` 生成 preview 之后、到 `scope apply` 完成之前：不能修改任何文件、不能 git commit。**
replay 校验把 Git HEAD 算进指纹，中间提交一次 commit → HEAD 变了 → `apply` 报
`managed_scope_replay_mismatch`，整条变更单作废重来。

### 三条语法坑

| 坑 | 正确做法 |
|---|---|
| **`exclude_dirs` 是「任意层级同名目录」匹配，不是路径前缀** | 写 `pipeline` 会把所有层级的 `pipeline/` 一起排掉（实测误伤几十个核心文件）。只排顶层 → 用 `managed_scope.rules` 的 glob：`pipeline/**` |
| **`rule_id` 必须四段式** | 形如 `<项目>-<用途>-exclude`；两段式报 `managed_scope_rule_metadata_invalid`，且需要 `created_by` 字段 |
| **`order` 数字越大优先级越高** | 不是越小越优先；写反了规则会被另一条盖掉 |

## 七、overview + attestation（建立"认知可靠"收据）

只有 MCP `aoci_overview`（**CLI 无 `overview` 命令**）；按 `chunk_tokens` 分块，**默认 7000 / 范围
4000–24000**（配错直接失败、不 clamp），装不下先返回第 1 块。

- `continuation_required=true` → 带 **`cursor=<next_cursor 的值>`** 继续（请求参数叫 `cursor`，
  `next_cursor` 只是响应字段名），直到 `completed=true`
- 中途**不得**用 `aoci_search` / `aoci_get_entries` / `scope` 补齐，也不得询问用户
- 正文只含开始 Marker + 原始 `aoci.txt` 字节 + 结束 Marker；截断/缺块/重复/乱序 → 停链
- 长效优化：`check_only=true` 只回紧凑 checkpoint，`probe=true` 出两道凭记忆作答的探针，不必重传全量

### 提交 receipt + attestation

`host_delivery_confirmation` 三字段：`body_sha256` / `body_bytes` / `end_marker_observed`。它与
`model_cognition_attestation`（`model-cognition-attestation/v1`）可**一次提交**，也可**分两次**——服务端记得
已到的一半。Attestation 必须原样回绑本次 Challenge 发布的 `index_sha256` / `entry_sequence_sha256` /
`entry_count`，旧身份一律失败。`partial` / `fail` 时**不得**声称完整系统认知，但不阻碍继续做 source-bound 工程。
通过后产出 `cognition_level=4` / `cognition_governed` / `model_full_cognition_reliable=true`。

### Challenge ordinal 怎么数（数错后面全错）

**ordinal = 正式 Entry 序列的 1-based 位置。**

- **不计数**：Header、注释行、空行、`===路径===` 段头、`AOCI Cognition Asset` 行、
  Section / Overview / Chunk Marker、Receipt、Metadata
- **只数**真正的条目行（`文件名[标签]: F:...`）
- Chunk Receipt 与 Challenge 答案**消费同一序列**——跨块序号连续，不按块重置

### 判定档位

| 结果 | 条件 |
|---|---|
| `pass` | ≥80% ordinal 完全正确，且对象身份至多失手一处 |
| `partial` | 未达上述，但信封身份正确 |
| `fail` | 信封身份不符，或全错 |

## 八、四类真实错误（都是校验实测拦下的）

| 错误码 | 现象 | 怎么改 |
|---|---|---|
| `code_candidate_id_mismatch` | candidate_id 抄错一位（`4e3` vs `4a2`） | 从 `aoci_maintain` 响应**原样复制**，别手打 |
| `impact_object_identity_invalid` | 条目里写了目录前缀（`pipeline/__init__.py`） | 改裸文件名（`__init__.py`） |
| `entry_field_budget_exceeded` | S 222 tokens > C9 档位 200 | **压缩语义**重写，不要截断 |
| E 规模档位错配（verify 告警） | 106 行标成 T | 改 S；或 `aoci_maintain(intent=cognition_optimization)` 走复核 |

共同点：**只允许改指定字段，其余候选逐字节不变**。整条重写会被拒。

## 九、工具链与排障

| 项 | 要点 |
|---|---|
| 二进制 | 命令名 `aoci`。`which aoci` 为空 → 全流程起不来。**版本以 `aoci --version` 为准**，别信任何文档写的版本号 |
| 环境诊断 | `aoci doctor` 查四组（仓库与核心资产 / Agent 集成 / AI 增强层 / 平台能力），默认不联网，`--net` 才探端点 |
| MCP 接入 | 见 §〇：优先 `init --agent`（自动写项目级）；不在列表的 host 手配 stdio（**正斜杠** + `--repo` 指项目根） |
| 本地测 MCP 握手 | 必须保持 stdin 打开：`{ printf '...'; sleep 10; } \| aoci mcp`，否则进程立刻 EOF 退出 |
| 不是漂移 | 二进制文件被跳过（`code_skipped: binary`）；条目里刻意保留的事故记录（某日丢可执行位之类）是**防重复踩坑的证据**，不要当过期信息删掉 |

**官方权威文档**（版本随 release 变，按你手上的 tag 取）：

```
docs/install.md              安装与签名验证三档
docs/agent-integrations.md   逐 host 的 MCP 接入写法
docs/getting-started.md      首次建索引
docs/troubleshooting.md      排障
```
