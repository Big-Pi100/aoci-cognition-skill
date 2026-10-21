# AOCI 认知层维护 —— Agent Skill

给任意 coding agent 用的技能：教它怎么在**任何仓库**里维护 [AOCI](https://github.com/aoci-spec/aoci-code) 认知层
—— 在项目里自举、写索引条目、改受管作用域、从漂移里恢复。

一个文件，除 `aoci` 本身外无依赖。

> English: [README.md](README.md)

## 为什么需要它

AOCI 官方文档定义的是**契约**：索引是什么、工具接受什么、attestation 怎么算。
它不告诉 agent **怎么干活**：

- `gitnexus` 和 AOCI 各自的活是哪一半？
- `S` 字段到底该写什么——哪些看着合理、实际会污染索引？
- 哪些错误是"重试一下"，哪些是"整批作废重来"？
- 怎么区分"还没对齐"和"已对齐、别再来 maintain"？

这份 skill 回答这些。它是**方法层**，不是契约层。

## 内容

| § | 内容 |
|---|---|
| 〇 | **自举** —— 三项检查决定能不能开工，以及本 skill 与项目规则冲突时谁说了算 |
| 一 | 何时该用 / 何时调 `aoci_maintain` 是错的（给可判定信号，不给空泛建议）|
| 二 | 完整链路：`gitnexus impact` → 改代码 → `aoci_maintain` → `verify/check/guide` → attestation |
| 三 | 改前读影响面 —— 精确到参数，含两个"静默返回空"的坑 |
| 四 | AOCI 收口：MCP 路径 vs CLI 路径，以及为什么只有一条能签发身份 |
| 五 | 条目写法：FRAS v2、标签选择、字段配额、`S` 字段准入 |
| 六 | 受管作用域变更 —— 审批流程 + 会让整批作废的时序陷阱 |
| 七 | `overview` + attestation，以及 challenge ordinal 的正确数法 |
| 八 | 校验实际会抛的四个错误码与对应对策 |
| 九 | 工具链与排障，以及上游权威文档入口 |

## 安装

`SKILL.md` 是标准 Agent Skill 格式（YAML frontmatter + Markdown）。三种方式任选：

**1. host 有 skills 目录** —— 放进去：

```
~/.claude/skills/aoci-cognition-maintenance/SKILL.md     # Claude Code
~/.workbuddy/skills/aoci-cognition-maintenance/SKILL.md  # WorkBuddy
```

其他支持 Agent Skills 的 host 用各自的目录，查该 host 文档。

**2. host 没有 skill 机制** —— 把 `SKILL.md` 正文贴进 agent 读的项目规则
（`AGENTS.md`、`.cursor/rules`、system prompt 都行）。内容自包含。

**3. 让 agent 自己去取** —— 对能取 URL 的 agent：

```
读 https://raw.githubusercontent.com/Big-Pi100/aoci-cognition-skill/main/SKILL.md
并按它对当前仓库执行。
```

## 前置

skill 自己会检查这些并给出补法，但你该知道要付出什么：

| 需要 | 为什么 |
|---|---|
| `aoci` 在 `PATH` 上 | 所有操作都经过它；`aoci --version` 必须能跑 |
| 一个你愿意加 `.aoci/` 的仓库 | 认知层住在那儿（纯文本，进 git 版本管理）|
| 支持 MCP 的 host | AOCI 通过 stdio MCP 与 agent 通信 |
| 可选：`gitnexus` | 只用于"改前查影响"那一半（§三）；只用 AOCI 那一半不需要 |

自举四步（skill 会带 agent 走完）：

```bash
aoci --version                                   # 1. 二进制在不在
aoci --repo /absolute/path/to/repo init --agent <host>   # 2. 契约 + 项目级 MCP 配置
aoci --repo /absolute/path/to/repo scan          # 3. 建 Baseline
aoci --repo /absolute/path/to/repo index agent guide --agent <host>   # 4. 下一步做什么
```

`init` 会写项目契约（`AGENTS.md`）和该 host 的**项目级** MCP 配置，所以多个项目可以各有一份 AOCI 实例，
不用抢一个全局设置。跳过 `scan` → `guide` 会报 `stage=blocked`，这是最常被漏掉的一步。

> 不在该列表里的 host 要手工配 stdio MCP —— skill 给了确切形状，以及两个会让它"静默失败"的坑。

## 它不做什么

- **不装 AOCI。** 用官方[安装指南](https://github.com/aoci-spec/aoci-code/blob/main/docs/install.md)（签名包，三档验证）。
- **不覆盖你项目的规则。** `AGENTS.md` 或 live `guide` 与本 skill 冲突时以它们为准——skill §〇 明写了这条，
  agent 不用猜。
- **不抄上游的配置语法。** 逐 host 的接入写在 AOCI 自己的
  [agent 集成文档](https://github.com/aoci-spec/aoci-code/blob/main/docs/agent-integrations.md)，随 release 更新；
  抄到这儿只会腐烂。

## 验证

`SKILL.md` 里每个 `aoci` 命令与 flag 都对真实二进制的 `--help` 核过（15 处引用 / 12 个不同命令），
**不是从另一份文档抄的**。这点很重要：本 skill 的第一版就继承了笔记里一个错误的命令名，
是跑二进制才发现的。

随时复验：

```bash
grep -oE "aoci [a-z-]+( [a-z-]+)?" SKILL.md | sort -u
```

验证基准：`aoci` 0.1.0-rc21。

## 相关

- [aoci-code](https://github.com/aoci-spec/aoci-code) —— 工具本体，安装与集成文档
- [GitNexus](https://github.com/abhigyanpatwari/GitNexus) —— 代码图谱，用于"改前查影响"那一半；本 skill 只指路，不教它

## 许可

MIT，见 [LICENSE](LICENSE)。这是原创方法论文本，不是 AOCI-CODE 源码的副本；AOCI-CODE 本体按其自身的
FSL-1.1-MIT 授权分发。
