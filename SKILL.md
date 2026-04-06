---
name: conversation-history-recovery-skill
description: "Perform a forensic reconstruction of prior repo conversations across Codex, Claude, repo-local exports, and git history; ingest each conversation in detail, identify explicit and implicit problems, diagnose agent failures, and either return a structured analytical report or implement the highest-confidence fixes when the user asks for execution."
---

# Conversation History Recovery Skill

Use this skill when the user wants the agent to reconstruct what really happened across prior conversations and commits, identify what was missed, and either:

1. deliver a deep forensic analysis report, or
2. continue the work by fixing validated gaps in the current repo.

Do not collapse this into a lightweight summary. The goal is high-fidelity reconstruction, not fast hand-waving.

## Mode Selection

Choose the mode from the user's wording before you touch code:

- `forensic-analysis mode`
  - Use when the user asks for analysis, diagnosis, root-cause review, missed-context recovery, critique, or a report.
  - Stay text-only. Do not write code. Do not claim fixes.
- `recovery-execution mode`
  - Use when the user wants the same forensic history work, then wants the repo continued or fixed.
  - Run the forensic analysis first, then verify the current repo, then implement only the highest-confidence validated gaps.

If the user is ambiguous, prefer `recovery-execution mode` only when the wording clearly asks to continue or fix. Otherwise default to `forensic-analysis mode`.

## Core Priority Rule

- Highest priority source is the current conversation in the active thread.
- Use it as the binding interpretation when historical sources disagree.
- If the current conversation explicitly asks for a previous agent/tool name (for example Claude Desktop, Claude/Copilot workspaces, Kimi, or coworkers), include those sources when discoverable and mark any missing sources explicitly.
- If the user repeats a request after a prior completion claim, classify that prior claim as suspicious and continue forensic recovery instead of trusting it.

## Hard Rules

## Blocker Handling

When blocked, do not stop. Emit these exact tags before proceeding:

- `<ANALYZING BLOCKER AND HOW TO ADDRESS IT!!!>`
- `<ANALYZING USER INTENTIONS AND BEING AS USEFUL AS POSSIBLE!!!>`

Then continue by:

- checking available local source paths and permissions,
- checking prior successful workflows in local histories,
- and re-aligning to the current conversation constraints before retrying.

- Read real local history, not memory or summaries alone.
- Default to the last 20 relevant conversations unless the user specifies a different window.
- Read every selected conversation sequentially before making final judgments.
- Inspect the first lines of each source format before batch reading so you know the actual JSONL schema.
- Never say a conversation or file was fully read unless you consumed 100 percent of the selected source. If you sampled, say so explicitly.
- Prove source access on one real sample file per source before broad analysis.
- Prefer direct local reads of original files over temporary exports, cached summaries, or secondary notes.
- Treat prior summaries, audits, continuation notes, and agent self-reports as leads, not truth.
- Final conclusions must come from the original conversation text plus current repo evidence.
- Separate user problems from agent-created problems. Both matter.
- Track contradictions, abandoned work, unverified claims, and repeated re-requests explicitly.
- When a request is typo-heavy or voice-to-text distorted, preserve the raw wording and also normalize it into plain language.
- In `forensic-analysis mode`, never write code.
- In `recovery-execution mode`, never start editing until the analytical backlog has been verified against the current repo.
- Do not let subagents summarize files they cannot directly access or fully read.

## Source Order

1. Active conversation context (current user request and latest turns).
2. Current repo context:
   - `tree -I node_modules -L 2`
   - `AGENTS.md`, `README.md`, `package.json`
   - relevant router, schema, config, and feature entry-point files
   - dirty worktree and recent commits
3. Codex history for the current repo:
   - filter `~/.codex/sessions/**/*.jsonl` by the current repo `cwd`
   - the repo `cwd` is typically stored in `session_meta.payload.cwd` or `turn_context.payload.cwd`, not as a top-level field
   - take the most recent 20 matching sessions unless the user asks for another range
4. Claude and adjacent assistant histories for the current repo:
   - derive the project folder from `cwd` by replacing `/` with `-` and keeping the leading `-`
   - read recent `~/.claude/projects/<derived-project>/*.jsonl`
   - discover and read other local assistant artifacts if present (`Claude`, `copilot`, `kimi`, `cursor`, `trae`, `agent` variants) before concluding source scope.
   - preferred additional search order:
     - `~/.copilot/session-state/*.jsonl`
     - `~/.copilot/logs/*.log`
     - `~/.auto-claude/memories/*`
     - `~/.kimi` and `~/Library/Application Support/KimiCode`
     - `~/.claude-worktrees`
     - `~/.claude-profiles`
     - `~/.cursor` and `~/.vscode-server/extensions/github.copilot-*` (session metadata only when available)
   - if any source is missing, record it explicitly in coverage as `missing-source`
   - if multiple worktree-style sources exist, include `~/.claude/worktrees` and project-family worktree directories that match the current repo basename
   - never claim a full coverage score unless the same source family was fully consumed or explicitly sampled with a partial flag
5. Repo-local exported conversation files and notes:
   - `docs/codex-messages*`
   - issue, report, critique, and analysis files only when they are clearly about the same feature area
6. Git history:
   - `git log --oneline --decorate -20`
   - inspect touched files for commits that appear to claim relevant work

## Forensic Workflow

### Phase 0. Scope, Mode, And Evidence Ledger

- State which mode you are using: `forensic-analysis` or `recovery-execution`.
- Build an evidence ledger before deep analysis:
  - total commits analyzed
  - total conversation files found
  - total repo documents inspected
  - churn-heavy files or recurring edit files
- List selected conversation files, repo-local notes, and git ranges in chronological order.
- For each source, report full versus partial coverage.
- Label each source as `codex`, `claude`, `repo-doc`, or `git`.
- Record for each source:
  - file or commit identifier
  - why it was selected
  - planned read method
  - full vs partial coverage target

For full forensic requests, run these checks before phase 1:
- `git log --oneline --since="4 weeks ago" --all`
- inventory conversation sources in `~/.codex/sessions/**/*.jsonl` and `~/.claude/projects/*/*.jsonl` linked to the repo cwd
- list recent docs with explicit status words (`done`, `fixed`, `complete`) for verification

### Phase 1. Forensic Context Ingestion And Metadata Reporting

- Open one real sample file from each source first and confirm the format is readable.
- If a file is large, read it in stable sequential chunks from the original path and track progress.
- If tooling blocks direct reading, report that blocker instead of pretending the source was covered.
- Perform both:
  - `micro-analysis`: every message one by one in chronological order
  - `macro-analysis`: re-read the same material in thematic blocks after the first pass
- Report quantitative metadata for the selected scope:
  - total selected conversations
  - total messages
  - approximate word count
  - approximate character count
  - key conversation blocks or phases by theme
- If the repo has many sessions, use a two-pass process:
  - pass 1: metadata, timestamps, cwd confirmation, candidate asks, repeated frustration signals
  - pass 2: full sequential reads of the unresolved, contradictory, or high-impact sessions

### Phase 2. Detailed Conversation Analysis

For every user message, extract:

- raw request wording when nuance matters
- normalized plain-language restatement
- explicit request
- implicit intent
- constraints and preferences
- named files, routes, commands, screenshots, repos, databases, or links
- corrections, escalations, and frustration signals
- success criteria the user was implicitly using

For every assistant message, diagnose:

- what was actually done
- what was only promised
- what assumptions were made without proof
- what evidence was cited and whether it was real
- what remained partial, skipped, contradicted later, or abandoned
- whether the assistant relied on a condensed summary instead of the primary source

Tag each turn:

- `[OK]`
- `[PARTIAL]`
- `[FAIL]`
- `[CORRECTED]`
- `[IGNORED]`

Escalation rule:

- If an agent claimed an item was complete and the user later re-requested the same issue, classify the earlier claim as `[FAIL]`, not `[PARTIAL]`.

Regression signals to flag explicitly in the report:

- user says the same issue again after a prior `[OK]`/`DONE`-style signal
- assistant claims a fix without evidence, then user reports repeat failure
- a prior fix is contradicted by newer assistant edits in the same file/function
- source scope was intentionally narrowed and then later expanded without revalidation

Any observed regression signal belongs in `Regression Register` with evidence links and timestamps.

For each user message, track confidence:

- high: clear request with explicit constraints
- medium: interpretable but fragmented
- low: heavy typo-heavy/noise, requires careful source stitching

### Phase 3. Comprehensive Problem Identification

Build a problem inventory with separate sections for:

- explicit user problems
  - bugs
  - broken flows
  - UX frustrations
  - errors
- agent-driven frustrations
  - ignored feedback
  - false completion claims
  - contradiction between messages
  - misread instructions
  - repeated plan-without-execution loops
- implicit problems
  - unreported failures
  - abandoned tasks
  - repeated re-requests
  - forgotten constraints
  - incompatible edits across parallel work

For each problem, capture:

- source conversations or commits
- earliest appearance
- latest appearance
- whether it was ever truly resolved
- current confidence level based on source evidence

Add a separate `Regression Register` subsection with:

- Re-opened requests after a prior `[OK]` or `[DONE]`-style claim
- Recurring completion mismatches
- Evidence that changed features were reintroduced without evidence
- Agent-created degradations after verified fix points

### Phase 4. Macro Re-Read And Root Cause Analysis

Re-read the conversation set by theme, not just chronology.

Recommended themes:

- repo orientation and setup
- debugging and verification
- UI or product asks
- infrastructure or deployment
- data and database work
- agent behavior and trust failures

For each theme, answer:

- what the user consistently wanted
- where the agent drifted
- which constraints were lost
- whether the repo state ever caught up with the conversation claims
- what the root causes were

Root-cause buckets to consider:

- shallow ingestion
- skipped primary sources
- instruction loss
- verification theater
- scope drift
- contradictory multi-agent work
- implementation without repo validation
- summary-trusting instead of source-reading
- feature removed then reintroduced after confirmation
- contradictory commit chains without repo-verified justification
- regressions caused by later agent edits in same files

### Phase 5. Current Repo Verification

Only perform this phase in `recovery-execution mode`.

Before editing, verify the current code for each backlog item by inspecting:

- affected routes and components
- server handlers and services
- schema or DB access points when data is involved
- auth, validation, analytics, and responsive side effects
- current dirty worktree so you do not overwrite unrelated changes
- overlapping session timestamps or parallel-agent edits that may have produced contradictions
- actual diffs or current code for commits that claimed relevant work

Distinguish clearly between:

- user-requested but not implemented
- implemented but unverified
- agent-reported summaries that were never proven from source
- already fixed in current code and should not be reopened

### Phase 6. Implement Highest-Confidence Gaps

Only perform this phase in `recovery-execution mode`.

- Prefer direct fixes over long reports.
- Reuse repo patterns.
- Do not revert unrelated user work.
- If multiple gaps are independent, handle the most user-critical ones first.
- If blocked, state the exact blocker and the evidence.

### Phase 7. Self-Validation

Before returning results, verify the analysis itself.

Checklist:

- Did you confirm access to each claimed source type?
- Did you read each selected conversation sequentially?
- Did you separately perform micro-analysis and macro-analysis?
- Did you report counts and key thematic blocks?
- Did you distinguish explicit, agent-driven, and implicit problems?
- Did you tie conclusions to source evidence instead of intuition?
- In `recovery-execution mode`, did you verify the current repo before editing?
- Did you avoid claiming completeness where coverage was partial?

If any answer is no, say so explicitly and narrow your claims.

## Output Format

Unless the user requests a different shape, return the report in this order:

1. `Forensic Analysis & Context Ingestion`
   - sources read
   - coverage quality
   - message, word, and character counts
   - key conversation blocks
2. `Analysis Summary`
   - concise reconstruction of what happened
3. `Problem Inventory`
   - explicit user problems
   - agent-driven frustrations
   - implicit problems
4. `Root Cause Analysis`
   - grouped by theme and failure mode
5. `Critique of Agent Behavior`
   - where the agent lost context, over-claimed, skipped validation, or ignored corrections
6. `Strategic Recommendations`
   - what the next agent should do differently
   - what evidence to gather first
   - what not to trust without verification
7. `Success Criteria`
   - what would have to be true to consider the recovery complete

In `recovery-execution mode`, append:

8. `Current Repo Verification`
9. `Implemented In This Turn`
10. `Verification Evidence`
11. `Remaining Blockers Or Follow-Ups`

In high-noise or repeatedly failed threads, also include:

12. `Regression Register`

## Decision Standard

Treat an item as unresolved only if at least one of these is true:

- the user asked for it and the code does not implement it
- the code partially implements it but misses explicit user details
- the assistant claimed completion without verification
- a later conversation shows the user had to restate or correct it
- the source text shows silent abandonment or contradiction

Do not reopen items that are already implemented and verified in the current repo unless there is evidence of regression.

When multiple agents worked on the repo concurrently, treat the current code plus the latest verified diff as higher-signal truth than any single session narrative.
