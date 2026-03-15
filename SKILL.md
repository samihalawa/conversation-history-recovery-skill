---
name: conversation-history-recovery-skill
description: "Audit the real prior conversations for the current repo across Codex, Claude, repo-local exports, and recent git history; identify missed user requirements, partial implementations, and unresolved work; then implement the highest-confidence fixes immediately. Use when the user asks what was promised but not finished, wants lost context recovered, or wants prior agent work critically continued instead of summarized."
---

# Conversation History Recovery Skill

Use this skill when the user wants the agent to reconstruct what really happened across prior conversations and commits, then close the gaps in the current repo instead of stopping at a report.

## Hard Rules

- Read real local history, not memory or summaries alone.
- Default to the last 20 relevant conversations unless the user specifies a different window.
- Read every message in each selected conversation sequentially. Do not skip to summaries.
- Inspect the first few lines of each source format before batch reading so you know the actual JSONL schema.
- Never say a conversation or file was "fully read" unless you consumed 100 percent of the selected source. If you sampled, say so explicitly.
- Prove source access on one real sample file per source before launching broad analysis or subagents.
- Prefer direct local reads of the real source files over temporary tool-output files or sandbox-exported summaries.
- Focus on user feature requirements, constraints, corrections, and partial work.
- Ignore deployment chatter, agent self-critique, and tool noise unless they directly affect missing functionality.
- Verify whether each claimed feature is already implemented before calling it missing.
- Extraction scripts may help index or shortlist sources, but final judgments must still come from direct sequential reads of the original conversation files.
- Do not let subagents summarize files they cannot directly access or fully read.
- Fix validated gaps immediately when safe; do not stop at critique.

## Source Order

1. Current repo context:
   - `tree -I node_modules -L 2`
   - `AGENTS.md`, `README.md`, `package.json`
   - relevant router, schema, config, and feature entry-point files
   - dirty worktree and recent commits
2. Codex history for the current repo:
   - filter `~/.codex/sessions/**/*.jsonl` by the current repo `cwd`
   - the repo `cwd` is typically stored in `session_meta.payload.cwd` or `turn_context.payload.cwd`, not as a top-level field
   - take the most recent 20 matching sessions unless the user asks for another range
3. Claude history for the current repo:
   - derive the project folder from `cwd` by replacing `/` with `-` and keeping the leading `-`
   - read recent `~/.claude/projects/<derived-project>/*.jsonl`
4. Repo-local exported conversation files and notes:
   - `docs/codex-messages*`
   - issue, report, critique, and analysis files only when they are clearly about the same feature area
5. Git history:
   - `git log --oneline --decorate -20`
   - inspect touched files for commits that appear to claim relevant work

## Workflow

### 1. Build The Evidence Ledger

- List the selected conversation files, commit range, and repo-local notes you will use.
- Keep the ledger chronological and source-labeled: `codex`, `claude`, `repo-doc`, `git`.
- Record coverage for each source: selected file, read method, and whether coverage was full or partial.

### 1.5. Validate Access And Chunking Early

- Open one small real file from each source first and confirm the format is readable.
- If a file is too large, read it in stable sequential chunks from the original path and track progress.
- Do not switch to a different derived file or temporary export unless the original source is unavailable, and note that downgrade explicitly.
- If tooling blocks direct reading, report that blocker instead of pretending the source was covered.
- If the repo has a large session count, use a two-pass approach:
  - pass 1: lightweight session metadata, cwd confirmation, timestamps, and candidate user asks
  - pass 2: full sequential reads for the sessions that appear unresolved, contradictory, or high impact

### 2. Analyze Conversations One By One

For every user message extract:

- explicit request
- implicit intent
- constraints and preferences
- important examples, files, links, screenshots, or DB details
- corrections or frustration signals
- the exact wording when a typo-heavy request could change meaning
- a normalized plain-language restatement when the original message is typo-heavy or voice-to-text distorted

For every assistant message diagnose:

- what was actually done
- what was assumed without proof
- what remained partial, skipped, or contradicted later
- whether verification was real or only claimed
- whether the assistant relied on a condensed extractor instead of the primary source

Tag each turn:

- `[OK]`
- `[PARTIAL]`
- `[FAIL]`
- `[CORRECTED]`
- `[IGNORED]`

Escalation rule:

- If an agent claimed an item was complete and the user later re-requested the same issue, classify that earlier claim as `[FAIL]`, not `[PARTIAL]`.

### 3. Build The Unresolved Backlog

- Deduplicate repeated asks across all sources.
- Group repeated asks by functional area such as route, component, service, table, or workflow, not by wording alone.
- Separate:
  - missing feature
  - partial implementation
  - regression risk
  - unverified claim
  - forgotten constraint
- Map each backlog item to real files, routes, services, tables, env vars, or commit(s).
- Distinguish clearly between:
  - user-requested but not implemented
  - implemented but unverified
  - agent-reported summaries that were never proven from source

### 4. Verify The Current Repo State

Before editing, inspect the actual code and runtime entry points that control each backlog item.

At minimum verify:

- affected routes and components
- server handlers and services
- schema or DB access points when data is involved
- auth, validation, analytics, and mobile or responsive side effects
- current dirty worktree so you do not overwrite someone else's work
- overlapping session timestamps or parallel-agent edits that may have produced contradictory implementations
- actual diffs or current code for commits that claim relevant work, because commit messages alone are not proof

### 5. Implement The Highest-Confidence Gaps Now

- Prefer direct fixes over long reports.
- Reuse the repo's existing patterns.
- Do not revert unrelated user changes.
- If multiple gaps are independent, handle the most user-critical ones first.
- If blocked, state the exact blocker and the evidence.

### 6. Verify After Changes

Use the fastest real verification that fits the change:

- targeted tests or typecheck
- grep and route checks
- runtime logs
- browser or visual validation when UI changed
- DB reads when data behavior changed

### 7. Return A Compact Recovery Report

Unless the user asks for a different format, return:

1. timeline of what the user asked and what changed
2. unresolved or partially addressed items still relevant now
3. what you implemented in this turn
4. verification evidence
5. remaining blockers or follow-ups

## Decision Standard

Treat an item as unresolved only if at least one of these is true:

- the user asked for it and the code does not implement it
- the code partially implements it but misses explicit user details
- the assistant claimed completion without verification
- a later conversation shows the user had to restate or correct it

Do not reopen items that are already implemented and verified in the current repo unless there is evidence of regression.

When multiple agents worked on the repo concurrently, treat the current code plus the latest verified diff as higher-signal truth than any single session narrative.
