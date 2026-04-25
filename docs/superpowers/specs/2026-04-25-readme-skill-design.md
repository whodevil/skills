# `/readme` Skill Design

**Date:** 2026-04-25  
**Topic:** readme-skill  
**Status:** Approved  

---

## 1. Overview

The `/readme` skill is an OpenCode agent skill that deeply analyzes a codebase, compares its findings against the existing README (or creates one if absent), and presents categorized improvement suggestions to the user for approval before applying them.

### 1.1 Purpose

- Ensure READMEs stay accurate, complete, and aligned with the actual codebase.
- Provide a user-guided workflow where the agent does the heavy analysis but the user controls which changes are applied.
- Support both `.md` and `.org` README formats without preference or conversion.

### 1.2 Invocation

```
/readme
```

The skill is triggered by the user typing `/readme` or explicitly asking for README review/updates.

---

## 2. Architecture & Data Flow

The skill follows a six-phase pipeline:

```
User invokes /readme
    |
    v
[Phase 1: Discovery] → Detect & read README.md or README.org
    |
    v
[Phase 2: Exploration] → Deep codebase analysis
    |
    v
[Phase 3: Analysis] → Compare reality vs. README + best practices
    |
    v
[Phase 4: Presentation] → Present each category's findings
    |
    v
[Phase 5: Approval Loop] → User: Approve / Reject / Skip / Feedback
    |
    v
[Phase 6: Application] → Compile approved changes → Write README
```

### 2.1 Phase 1: Discovery

1. Use `glob` to detect `README.md` or `README.org` in the current working directory.
2. If neither exists, record: **No README found — creation mode.**
3. If one exists, `read` it and record its current claims about:
   - Project name and description
   - Directory structure
   - Features / functionality
   - Dependencies
   - Setup / installation instructions
   - Usage / API documentation
   - Contributing / testing info
4. If both `README.md` and `README.org` exist, prefer the one that is larger / more complete, but note the presence of both to the user.

### 2.2 Phase 2: Exploration

Systematically explore the codebase using `glob`, `read`, and `bash` tools. Priority order:

1. **Root config files** — `package.json`, `pyproject.toml`, `Cargo.toml`, `pom.xml`, `build.gradle`, `Makefile`, etc.
2. **Entry points** — main application files, CLI entry points, library exports.
3. **Source directories** — representative files from each major directory (`src/`, `lib/`, `app/`, etc.).
4. **Tests** — presence and structure of test files.
5. **CI / DevOps** — `.github/workflows/`, `.gitlab-ci.yml`, `Dockerfile`, etc.
6. **Documentation** — `docs/`, `AGENTS.md`, `CLAUDE.md`, changelogs.
7. **Other metadata** — `.gitignore`, `LICENSE`, `CONTRIBUTING.md`.

**Exploration cap:** Max 30 files read in a single invocation to avoid excessive tool calls. Prioritize root configs, entry points, and a representative sample from each major directory.

**Goal:** Build a factual model of what the codebase actually contains, how it works, and how to set it up.

### 2.3 Phase 3: Analysis

Compare the Discovery claims against the Exploration findings and the best-practice guidelines from the Medium article.

Group discrepancies into these categories:

| Category | What to Check |
|----------|---------------|
| **Feature Coverage** | Does the README describe all major features? Are new features missing? Are described features still present? |
| **Setup / Installation** | Are the install steps correct? Do they match the actual config files (e.g., `npm install` vs `pip install`)? Are missing steps identified? |
| **Dependencies** | Are dependency lists accurate? Are versions correct? Are optional dependencies noted? |
| **Structure / Formatting** | Is the README well-structured per best practices? Are sections missing (e.g., no "Getting Started")? Is formatting consistent? |
| **API / Usage** | Are usage examples correct? Do they match the actual API? Are there missing examples for key functionality? |

For each discrepancy, record:
- **Issue:** What is wrong or missing
- **Evidence:** Concrete file paths, code snippets, or directory listings from the codebase
- **Proposed change:** Specific text or section to add, update, or remove

If no README exists, treat all categories as "missing" and generate suggestions from scratch using the codebase model.

### 2.4 Phase 4: Presentation

Present findings to the user **one category at a time**.

For each category:
1. State the category name.
2. Summarize how many issues were found.
3. List each issue with:
   - The current README claim (if any)
   - The evidence from the codebase
   - The proposed change
4. Ask the user for a response.

### 2.5 Phase 5: Approval Loop

For each category, prompt the user with:

> **Category: [Feature Coverage / Setup / Dependencies / Structure / API]**
> [Issue 1 description with evidence and proposed change]
> [Issue 2 ...]
>
> What would you like to do?
> - **Approve** — apply all proposed changes in this category
> - **Reject** — discard all proposed changes in this category
> - **Skip** — save for later, move to the next category
> - **Feedback** — tell me how to refine (e.g., "focus on X", "this is wrong because Y")

**If Feedback:**
1. Record the user's feedback.
2. Refine the category's findings based on the feedback.
3. Re-present the refined category.
4. The user can then Approve, Reject, Skip, or give more Feedback.

**If Approve:** Queue all proposed changes in this category for the final Application phase.

**If Reject:** Discard and move to the next category.

**If Skip:** Note the category as skipped and move on. (User can re-invoke `/readme` later to revisit.)

Continue until all categories are processed.

### 2.6 Phase 6: Application

1. Compile all approved changes from all categories.
2. If a README exists, apply the changes inline, preserving as much existing wording and structure as possible.
3. If no README exists, create a new one from scratch using the approved content, structured according to best-practice guidelines.
4. Write the result to `README.md` or `README.org` (matching the original format, or defaulting to `.md` if none existed).
5. Confirm to the user: "README updated with X approved changes from Y categories."

---

## 3. Error Handling & Edge Cases

| Scenario | Behavior |
|----------|----------|
| **No README exists** | Treat as creation mode. Use best-practice template as structural baseline. Present all sections as "needs creation." |
| **README is empty or near-empty** | Same as "no README" — creation mode. |
| **Codebase has no recognizable structure** | If `glob` finds only scattered files with no clear entry points or conventions, present a minimal README suggestion and flag: "Codebase structure unclear — manual review recommended." |
| **User rejects all categories** | Gracefully exit without writing anything: "No changes approved. README left as-is." |
| **User gives feedback that contradicts codebase evidence** | Respect user's feedback but note the discrepancy: "Noted: user prefers X. Codebase evidence suggests Y. Applied user's preference." |
| **Write to README fails** | Report the error and present the full proposed README content in a conversation code block so the user can apply it manually. |
| **README is `.org` format** | Preserve `.org` format. Read existing `README.org` and write updates back to `README.org` using Org-mode syntax. Never suggest converting to `.md`. |
| **Very large codebase** | Cap exploration at 30 files. Prioritize root configs, entry points, and a representative sample from each major directory. |
| **Both `README.md` and `README.org` exist** | Prefer the larger/more complete one. Note the presence of both to the user and ask which to update if they differ significantly. |

---

## 4. Best-Practice Guidelines (Reference)

Based on the Medium article, a good README should include:

1. **Project Name & One-Liner** — Clear, descriptive title and a single-sentence summary.
2. **Description** — What the project does and why it exists.
3. **Installation** — Step-by-step setup instructions.
4. **Usage** — How to use the project, with examples.
5. **Features** — Key capabilities, ideally in a bulleted list.
6. **API / Configuration** — If applicable, core API or config options.
7. **Contributing** — How to contribute, if open to contributions.
8. **License** — License name and link.
9. **Badges / Status** — Build status, version, etc. (optional but nice).

The skill uses these as a checklist when analyzing Structure/Formatting gaps, but does not enforce them rigidly. It flags missing sections and suggests content based on the actual codebase.

---

## 5. Testing Strategy

Since this is a prompt-based skill (no executable code), testing is manual and scenario-driven:

| Test | Purpose |
|------|---------|
| **Test A: Existing README with gaps** | Verify the skill detects missing features, outdated dependencies, and incorrect setup instructions when the README is stale. |
| **Test B: Missing README** | Verify the skill creates a new README from scratch using codebase analysis and best-practice guidelines. |
| **Test C: `.org` format preservation** | Verify the skill preserves `README.org` format and uses Org-mode syntax for updates. |
| **Test D: User feedback loop** | Verify the approval handler correctly processes Approve, Reject, Skip, and Feedback responses, and refines categories when feedback is given. |
| **Test E: Empty/minimal README** | Verify the skill treats near-empty READMEs as creation scenarios. |
| **Test F: Large codebase** | Verify exploration capping works — the skill reads representative files but doesn't hang on repos with hundreds of files. |
| **Test G: Both `.md` and `.org` present** | Verify the skill handles the presence of both files gracefully. |

---

## 6. File Structure

The skill will be deployed as a single file:

```
readme/
└── SKILL.md
```

Following the OpenCode skill format: YAML frontmatter (`name`, `description`) followed by Markdown instructions.

---

## 7. Success Criteria

- The skill correctly identifies discrepancies between the README and the codebase in at least 3 of the 5 categories.
- The user can approve, reject, skip, or provide feedback on each category.
- The skill preserves `.org` format when updating `README.org`.
- The skill creates a new README if none exists, structured according to best practices.
- The skill caps exploration at ~30 files to avoid excessive tool calls.

---

*End of design spec.*
