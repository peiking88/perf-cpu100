# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

This is a **Claude Code skill** (`perf-cpu100`) — a markdown-based troubleshooting guide, not a buildable/testable codebase. It teaches the model how to diagnose Linux CPU/Load issues using the strace → perf → ftrace workflow.

There is no build system, linter, or unit test runner here. "Correctness" means the skill produces accurate, well-structured troubleshooting guidance when triggered.

## Structure

- **`SKILL.md`** — the skill itself. Frontmatter (`name`, `description`) controls triggering; the body is the full SOP (decision tree, three-tool workflow, D-state番外, common pitfalls, verification steps, case study).
- **`references/`** — deep-dive material loaded on demand: `tma-metrics.md`（TMA 四分类判读与度量陷阱）、`optimization-playbook.md`（按瓶颈分类的修复手册）、`code-examples-{memory,compute,frontend-thread}.md`（反例→正例代码对，来源 perf-book/perf-ninja）、`split-lock.md`（整机 CPI 突增：split lock/总线锁检测、平台差异、预防编码规范）.
- **`evals/evals.json`** — evaluation cases. Each entry: `id`, `name`, `user prompt`, `expected_output` (behavioral rubric, not exact text), `files` (attachments, currently unused).
- **`.skill-forge/state.json`** — skill-forge bookkeeping (tool call count, compaction flag). Handled by the skill-forge harness; don't edit manually.
- **`.claude/skills/skill_registry.json`** — local skill registry. Managed by the Claude Code skill system.

## How to Modify

**Editing the skill content:** edit `SKILL.md` body directly. Keep the SOP order (strace → perf → ftrace) and the decision tree intact — they're load-bearing, not decorative.

**Adding an eval:** append to `evals/evals.json` array with the next `id`. `expected_output` should describe the _behavior_ the response must exhibit (tools called, branches covered, order of operations), not a verbatim string.

**Triggering keywords** are in `SKILL.md` frontmatter `description`. Tune these if the skill fires too eagerly or not at all.

## Scope (What This Skill Covers vs. Doesn't)

**In scope:** CPU high, load spike, slow responses, distinguishing user/kernel overhead, futex lock contention, I/O slowness, D-state (kill -9 immune) processes, reading strace/perf/flamegraph/pidstat output.

**Out of scope:** memory leaks/OOM, DB/Redis slow queries and locks, network packet loss, capacity planning. Don't expand the skill into these — they need separate skills.

## Eval Cases Currently Cover

1. `cpu-spike-unnamed` — generic CPU 95% spike, verifies full SOP order + verification step
2. `d-state-stuck` — D-state process, verifies 5-step D-state path + kernel stack reading
3. `container-perf-blocked` — container perf permissions, verifies host-side PID workaround
