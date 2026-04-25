# `/readme` Skill Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create the `/readme` OpenCode skill that analyzes a codebase, compares it against the README, and presents categorized improvement suggestions for user approval before applying them.

**Architecture:** Single `SKILL.md` file following the OpenCode skill format (YAML frontmatter + Markdown instructions). The skill implements a six-phase pipeline: Discovery, Exploration, Analysis, Presentation, Approval Loop, Application.

**Tech Stack:** Markdown, OpenCode skill format (YAML frontmatter)

**Spec reference:** `docs/superpowers/specs/2026-04-25-readme-skill-design.md`

---

## Chunk 1: File Structure & YAML Frontmatter

**Files:**
- Create: `readme/SKILL.md`

- [ ] **Step 1: Create directory and file with YAML frontmatter**

```yaml
---
name: readme
description: Use when the user wants to review, update, or create a README for the current project. Analyzes the codebase, compares against the existing README (or best practices if missing), and presents categorized suggestions for approval.
---
```

- [ ] **Step 2: Write the Overview section**

Include:
- What the skill does (6-phase pipeline summary)
- When to use (user types `/readme` or asks for README review)
- Approval granularity note (per-category, not per-issue)

- [ ] **Step 3: Commit**

```bash
git add readme/
git commit -m "feat: scaffold /readme skill with frontmatter and overview"
```

---

## Chunk 2: Discovery Phase

**Files:**
- Modify: `readme/SKILL.md`

- [ ] **Step 1: Write Discovery phase instructions**

Cover:
1. **Working directory root detection:** If invoked from a subdirectory, search upward for nearest directory containing `README.md`, `README.org`, or `.git` folder. Treat that as project root. If none found, use current working directory and proceed normally.
2. `glob` for README files (`README.md`, `README.org`, case variants, no extension)
3. If neither exists → creation mode
4. If one exists → read and record claims (name, description, structure, features, dependencies, setup, usage, contributing, testing)
5. If both `README.md` and `README.org` exist → prompt user to choose, read both, merge claims (use more detailed version for conflicts)
6. If `README` (no extension) coexists with `README.md` or `README.org` → prefer the extension-bearing file without prompting
7. Tie-breaker rules for case variants (prefer standard-cased over `Readme.md`)
8. Record chosen primary file for write-back

- [ ] **Step 2: Commit**

```bash
git add readme/SKILL.md
git commit -m "feat: add Discovery phase to /readme skill"
```

---

## Chunk 3: Exploration Phase

**Files:**
- Modify: `readme/SKILL.md`

- [ ] **Step 1: Write Exploration phase instructions**

Cover:
1. Priority order (root configs → entry points → source dirs → tests → CI → docs → metadata)
2. "Source file" definition (code extensions, exclude tests/generated/lockfiles)
3. "Major directory" heuristic (≥5 source files or entry-point marker)
4. Exploration cap with per-priority allocation (max 30 total):
   - Root configs: max 5
   - Entry points: max 5
   - Source dirs: max 13 (spec says 15, but capped at 13 to respect 30-file total limit)
   - Tests: max 3
   - CI/DevOps: max 2
   - Docs/Metadata: max 2 combined
5. Large file handling (>2000 lines or >20KB → read first 100 lines or a representative section)
6. Context protection: if total content exceeds LLM context window, prioritize most recent/important files and summarize the rest
7. Unreadable file handling (skip binaries/permission-denied)
8. Factual model schema (Claim, Evidence, Type)
9. **Structure check:** After exploration, assess whether the codebase has recognizable structure (clear entry points, conventions, major directories). If not, bypass the category system entirely → present a minimal README suggestion and flag "Codebase structure unclear — manual review recommended." Do not proceed to Analysis/Presentation/Approval.

- [ ] **Step 2: Commit**

```bash
git add readme/SKILL.md
git commit -m "feat: add Exploration phase to /readme skill"
```

---

## Chunk 4: Analysis Phase

**Files:**
- Modify: `readme/SKILL.md`

- [ ] **Step 1: Write Analysis phase instructions**

Cover:
1. Compare Discovery claims vs Exploration findings vs best-practice checklist
2. Five categories with ownership boundaries:
   - Feature Coverage (what code does)
   - Setup/Installation (how to get running)
   - Dependencies (what project depends on)
   - Structure/Formatting (meta-level: section presence/order)
   - API/Usage (how to use after setup)
3. Deduplication rule with specificity hierarchy: API/Usage > Setup/Installation > Dependencies > Feature Coverage > Structure/Formatting
4. For each discrepancy: Issue, Evidence, Proposed change
5. Creation mode behavior (generate suggestions for every category with evidence; only skip categories with absolutely no evidence)

- [ ] **Step 2: Commit**

```bash
git add readme/SKILL.md
git commit -m "feat: add Analysis phase to /readme skill"
```

---

## Chunk 5: Presentation & Approval Loop

**Files:**
- Modify: `readme/SKILL.md`

- [ ] **Step 1: Write Presentation phase**

Cover:
1. Present one category at a time
2. Empty categories: skip entirely (no "no issues found")
3. Per category: name, issue count, list each issue (current claim, evidence, proposed change)
4. Ask for response

- [ ] **Step 2: Write Approval Loop**

Cover:
1. Prompt format with all options: Approve, Reject, Skip, Feedback, Quit
2. **Approve:** queue all changes in this category for Application
3. **Reject:** discard and move to next category
4. **Skip:** note category as skipped, move on (user can re-invoke `/readme` later)
5. **Feedback:** record, refine, re-present
6. Iteration cap: max 2 feedback rounds per category (1 round = 1 user feedback text)
7. Forced decision after 2nd round
8. Unrecognized input: re-prompt with valid options
9. **Quit:** immediately discard all pending/future changes, report: "Exited without applying any changes. README left as-is."

- [ ] **Step 3: Commit**

```bash
git add readme/SKILL.md
git commit -m "feat: add Presentation and Approval Loop to /readme skill"
```

---

## Chunk 6: Application Phase

**Files:**
- Modify: `readme/SKILL.md`

- [ ] **Step 1: Write Application phase**

Cover:
1. Compile approved changes
2. Conflict resolution heuristic:
   a. Content beats structure
   b. Specific beats general
   c. If unresolved → re-prompt with conflicting changes. Options:
      - Choose Change A — apply first conflicting change
      - Choose Change B — apply second conflicting change
      - Reject Both — discard both
      - Abort — exit without writing any changes
      If unrecognized input, re-prompt with same options. This mid-application re-prompt is NOT subject to the 2-round feedback cap.
3. Apply inline to chosen primary file (preserve wording/structure)
4. If no README → create from scratch
5. Write back to exact original filename (or `README.md` if none existed)
6. Confirmation messages:
   - Single README existed: "README updated with X approved changes from Y categories."
   - Both existed: "README updated with X approved changes from Y categories. [Primary file] was updated; [other file] was left as-is."

- [ ] **Step 2: Commit**

```bash
git add readme/SKILL.md
git commit -m "feat: add Application phase to /readme skill"
```

---

## Chunk 7: Error Handling & Edge Cases

**Files:**
- Modify: `readme/SKILL.md`

- [ ] **Step 1: Write Error Handling section**

Cover all edge cases from the spec:
- No README / empty README → creation mode
- Codebase has no recognizable structure → bypass category system, present minimal suggestion
- User rejects all categories → "No changes approved. README left as-is."
- User feedback contradicts evidence → respect but note in transcript only: "Noted: user prefers X. Codebase evidence suggests Y. Applied user's preference." Do NOT append notes to README itself
- Contradictory feedback across rounds → respect latest, flag inconsistency in transcript only. Do NOT append notes to README itself
- Write fails → present full README in code block
- Very large README (>5000 lines / >50KB) → summarize. Flag: "README is very large; analysis may be incomplete."
- Binary/unreadable README → "README file found but cannot be read. Please check permissions or encoding." Treat as missing but warn not to overwrite without investigating
- Empty working directory → "This directory appears to be empty. No README can be generated." Exit gracefully.
- Exploration file unreadable (binary, permission-denied) → skip, continue exploration
- Symlinked README → follow only if target within working directory. If symlink points outside working directory, do NOT follow it; treat as missing README. When writing back, if symlink was not followed, create new `README.md`/`README.org` alongside symlink rather than overwriting symlink itself
- Read-only README → report error, present content
- README without extension → preserve exact filename
- README is `.org` format (only one exists) → preserve Org-mode syntax, never suggest `.md` conversion
- Both `.md` and `.org` exist → prompt user, merge, write to chosen file
- Git repo detected → create backup before writing (e.g., `README.md.bak.<timestamp>`)
- README references assets → preserve paths
- Partial analysis (30-file cap reached) → flag: "Analysis capped at 30 files — some areas of the codebase may not have been fully explored."
- Loose source files in root → read up to 3

- [ ] **Step 2: Commit**

```bash
git add readme/SKILL.md
git commit -m "feat: add Error Handling and Edge Cases to /readme skill"
```

---

## Chunk 8: Best-Practice Guidelines & Final Review

**Files:**
- Modify: `readme/SKILL.md`
- Modify: `README.md` (update to list the new skill)

- [ ] **Step 1: Write Best-Practice Guidelines section**

Hardcoded checklist (self-contained, no external fetching):
1. Project Name & One-Liner
2. Description
3. Installation
4. Usage
5. Features
6. API / Configuration
7. Contributing
8. License
9. Badges / Status (optional)

Note: used as checklist for Structure/Formatting analysis, not rigidly enforced.

- [ ] **Step 2: Update root README.md to list the new skill**

Add to the "Included Skills" table:
```
| **readme** | [`readme/`](readme/) | README review and maintenance. When invoked (e.g., `/readme`), the agent analyzes the codebase, compares it against the existing README (or best practices if missing), and presents categorized improvement suggestions for user approval before applying them. |
```

- [ ] **Step 3: Final review of SKILL.md**

Verify:
- YAML frontmatter is valid
- All six phases are covered completely
- Error handling is comprehensive
- Instructions are clear and actionable for an LLM agent
- No TODOs or placeholders remain
- All spec Section 7 Success Criteria are met (discrepancies in ≥3 categories, per-category approval, .org preservation, creation mode, 30-file cap)

- [ ] **Step 4: Commit**

```bash
git add readme/SKILL.md README.md
git commit -m "feat: add best-practice guidelines and update root README"
```

---

## Testing Strategy (Manual Validation)

Since this is a prompt-based skill, testing is manual:

**Test A:** Run `/readme` against this repo (has existing `README.md` with gaps). Verify it detects the missing `readme` skill in the table, suggests updates, and handles approval loop.

**Test B:** Run `/readme` in a repo with no README. Verify it creates a new one from scratch.

**Test C:** Create a test repo with `README.org`. Verify the skill preserves `.org` format.

**Test D:** Create a test repo with both `README.md` and `README.org`. Verify it prompts user to choose and merges concepts.

**Test E:** Verify the feedback loop works (provide feedback, see refinement, hit cap after 2 rounds). Verify Quit works mid-loop.

**Test F (Spec E):** Empty/minimal README. Verify skill treats near-empty README as creation scenario.

**Test G (Spec F):** Large codebase. Verify exploration capping works and skill doesn't hang.

**Test H (Spec H):** Conflict resolution. Verify when two approved categories propose changes to same section, skill resolves by preferring narrower-scope change.

**Test I (Spec I):** Unreadable exploration file. Verify skill skips binary/unreadable files without aborting.

---

## Plan Review Loop

After completing each chunk:
1. Read the plan-document-reviewer prompt from `skills/writing-plans/plan-document-reviewer-prompt.md`
2. Dispatch reviewer with chunk content and spec path
3. Fix any issues found
4. Re-dispatch if needed (max 5 iterations)
5. Proceed to next chunk

---

*End of implementation plan.*
