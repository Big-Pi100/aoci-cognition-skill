# AOCI Cognition Maintenance — an Agent Skill

A drop-in [Agent Skill](https://docs.claude.com/en/docs/claude-code/skills) that teaches any coding agent how to
**maintain an [AOCI](https://github.com/aoci-spec/aoci-code) cognition layer** in any repository: bootstrap it in a
project, write index entries, change managed scope, and recover when the index drifts.

One file. No dependencies beyond `aoci` itself.

---

## Why this exists

AOCI's own documentation defines the **contract**: what the index is, what the tools accept, how attestation works.
It does not tell an agent **how to work**:

- Which half of the job belongs to `gitnexus` and which half to AOCI?
- What actually belongs in the `S` field — and what looks reasonable but pollutes the index?
- Which mistakes cost you a full re-do instead of a retry?
- How do you tell "not aligned yet" from "aligned, stop calling maintain"?

This skill answers those. It is the **method layer**, not the contract layer.

## What's inside

| § | Content |
|---|---|
| 〇 | **Bootstrap** — three checks that decide whether you can work at all, plus the authority order when the skill conflicts with project rules |
| 一 | When to use / when calling `aoci_maintain` is actively wrong (with decidable signals, not vague advice) |
| 二 | The full loop: `gitnexus impact` → change code → `aoci_maintain` → `verify/check/guide` → attestation |
| 三 | Reading impact before editing — parameters, plus **why an empty result is not a safe result** (`epistemic` / `causes` / `risk: UNKNOWN`) |
| 四 | The AOCI close-out, MCP path vs CLI paths, and why only one of them can issue identities |
| 五 | Writing entries: FRAS v2, tag selection, field budgets, and the `S` admission test |
| 六 | Managed scope changes — the approval flow and the timing trap that voids a whole change set |
| 七 | `overview` + attestation, and how to number challenge ordinals correctly |
| 八 | Four error codes that validation actually raises, and the fix for each |
| 九 | Toolchain and troubleshooting, plus pointers to the authoritative upstream docs |

## Install

`SKILL.md` is plain text in the standard Agent Skill format (YAML frontmatter + Markdown). Three ways, pick one:

**1. Hosts with a skills directory** — drop the folder in:

```
~/.claude/skills/aoci-cognition-maintenance/SKILL.md     # Claude Code
~/.workbuddy/skills/aoci-cognition-maintenance/SKILL.md  # WorkBuddy
```

Other hosts that support Agent Skills use their own directory — check that host's docs.

**2. Hosts without a skill mechanism** — paste the body of `SKILL.md` into your agent's project rules
(`AGENTS.md`, `.cursor/rules`, system prompt, whatever the host reads). The content is self-contained.

**3. Point the agent at this repo** — for agents that can fetch URLs:

```
Read https://raw.githubusercontent.com/Big-Pi100/aoci-cognition-skill/main/SKILL.md
and follow it for this repository.
```

If the agent can't reach GitHub (restricted egress, corporate proxy), use option 1 or 2 — the file is the whole
skill, nothing is fetched at runtime.

## Prerequisites

The skill checks these itself and tells the agent what to do about each — but so you know what you're signing up for:

| Requirement | Why |
|---|---|
| `aoci` binary on `PATH` | Everything runs through it. `aoci --version` must work |
| A repository where you're happy to add `.aoci/` | That's where the cognition layer lives (plain text, versioned in git) |
| An MCP-capable host | AOCI talks to the agent over stdio MCP |
| Optional: `gitnexus` | Only needed for the "check impact before editing" half (§三). The AOCI half works without it |

Bootstrap is four commands (the skill walks the agent through all of them):

```bash
aoci --version                                   # 1. binary present?
aoci --repo /absolute/path/to/repo init --agent <host>   # 2. contract + project-level MCP config
aoci --repo /absolute/path/to/repo scan          # 3. establish the Baseline
aoci --repo /absolute/path/to/repo index agent guide --agent <host>   # 4. what to do next
```

`init` writes the project contract (`AGENTS.md`) and the host's **project-level** MCP config, so several projects can
each have their own AOCI instance without fighting over a global setting. Skip `scan` and `guide` reports
`stage=blocked` — that's the step people miss.

> Hosts outside that list configure a stdio MCP entry manually — the skill gives the exact shape and the two
> mistakes that make it fail quietly.

## What this does NOT do

- **Doesn't install AOCI.** Use the official [installation guide](https://github.com/aoci-spec/aoci-code/blob/main/docs/install.md)
  (signed release packages, three verification levels).
- **Doesn't override your project's rules.** If `AGENTS.md` or the live `guide` output disagrees with this skill,
  they win. The skill says so explicitly in §〇 so an agent doesn't have to guess.
- **Doesn't copy upstream config syntax.** Host-specific setup lives in AOCI's own
  [agent integration docs](https://github.com/aoci-spec/aoci-code/blob/main/docs/agent-integrations.md), which stay
  current as releases change. Duplicating them here would only rot.

## Field notes

This skill has been run against real repositories, and one round of field feedback produced three reported
"gotchas". Two held up; **one did not**, and the correction is worth keeping:

| Reported | Verdict after reproducing |
|---|---|
| `guide` returns exit 0 on failure | ❌ **Wrong root cause.** `aoci` returns 3. The report measured `cmd \| head; echo $?` — that is `head`'s status, always 0. The trap is real but belongs to the caller, so the skill now warns about the pipe, not about `aoci` |
| MCP config is project-level, so projects coexist | ✅ True for Claude Code / Codex / Cursor / OpenCode / Qoder. **False for hosts that keep one global MCP file** — those can't run several AOCI entries at once, and server-prefixed tool names stop matching this skill's bare tool references |
| Stale index returns an empty answer, not an error | ✅ Worse than that. `impact` reports `risk: UNKNOWN` (never `LOW`) on an empty upstream, plus `epistemic: lower-bound` and machine-readable `causes` naming what was dropped |

Keeping the disproved one documented is deliberate: the instinct "a tool's exit code is untrustworthy" would have
spread into every script that reads this skill.

## Verification

Every `aoci` command and flag in `SKILL.md` was checked against the real binary's `--help` output
(15 command references, 12 distinct commands) — not copied from another document. That matters: the first draft of
this skill inherited a wrong command name from its source notes, and it took running the binary to catch it.

Re-check anytime:

```bash
# list every CLI invocation in the skill, then compare to --help
grep -oE "aoci [a-z-]+( [a-z-]+)?" SKILL.md | sort -u
```

Verified against `aoci` 0.1.0-rc21.

## Related

- [aoci-code](https://github.com/aoci-spec/aoci-code) — the tool itself, install and integration docs
- [GitNexus](https://github.com/abhigyanpatwari/GitNexus) — code graph used for the impact half; the skill points at
  it but doesn't teach it

## License

MIT — see [LICENSE](LICENSE). This is original methodology text, not a copy of AOCI-CODE's source; AOCI-CODE itself
is distributed under its own FSL-1.1-MIT license.
