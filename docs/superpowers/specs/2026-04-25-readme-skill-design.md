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

**Approval granularity:** The user approves or rejects changes **per category**, not per individual issue. All issues within a category are approved or rejected as a batch. The user can provide feedback to refine a category before approving.

---

## 2. Architecture & Data Flow

**Working directory assumption:** The skill assumes the current working directory is the project root. If invoked from a subdirectory (e.g., `src/`), the agent should search upward for the nearest directory containing a `README.md`, `README.org`, or `.git` folder, and treat that as the project root. If none is found, use the current working directory and proceed normally.

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
4. If both `README.md` and `README.org` exist, use this tie-breaker heuristic to pick the primary README to analyze:
   a. **Line count** — whichever file has more non-empty lines.
   b. If line count is within 10% of each other, compare **number of H2+ headings** — more structured sections wins.
   c. If still tied (identical line count and heading count), prefer `README.md` over `README.org` as the final fallback.
   d. Note the presence of both files to the user.
   e. **Significant difference check:** If one file contains ≥2 major sections (e.g., "Features", "Installation", "API") that the other lacks entirely, prompt the user: "Both README.md and README.org exist. Which should I update?" Otherwise, proceed with the tie-breaker winner.
5. Also check for plain `README` (no extension) or `Readme.md` / `Readme.org` (case variants). If `README` (no extension) is found, treat it as a valid README file. When writing back, preserve the exact original filename (e.g., if the original was `README`, write back to `README`; do not rename to `README.md`). If both a case-variant and standard-cased version exist (e.g., `Readme.md` and `README.md`), prefer the standard-cased `README.md` / `README.org` before applying the line-count heuristic.

### 2.2 Phase 2: Exploration

Systematically explore the codebase using `glob`, `read`, and `bash` tools. Priority order:

1. **Root config files** — `package.json`, `pyproject.toml`, `Cargo.toml`, `pom.xml`, `build.gradle`, `Makefile`, etc.
2. **Entry points** — main application files, CLI entry points, library exports.
3. **Source directories** — representative files from each major directory. A "major directory" is any top-level directory (or immediate subdirectory of `src/`) that contains ≥5 source files or an entry-point marker (e.g., `__init__.py`, `index.js`, `main.py`, `Cargo.toml`).
4. **Tests** — presence and structure of test files.
5. **CI / DevOps** — `.github/workflows/`, `.gitlab-ci.yml`, `Dockerfile`, etc.
6. **Documentation** — `docs/`, `AGENTS.md`, `CLAUDE.md`, changelogs.
7. **Other metadata** — `.gitignore`, `LICENSE`, `CONTRIBUTING.md`.

**Exploration cap:** Max 30 files read in a single invocation to avoid excessive tool calls. Prioritize root configs, entry points, and a representative sample from each major directory.

**Goal:** Build a factual model of what the codebase actually contains, how it works, and how to set it up.

**Factual model schema:** The model is an internal list of claims, each with this structure:
- **Claim:** A factual statement about the codebase (e.g., "Project is a Python CLI tool")
- **Evidence:** The file path or snippet supporting the claim (e.g., `src/main.py`)
- **Type:** One of `feature`, `setup`, `dependency`, `api`, `structure`

Example claims:
```
- Claim: "Project exports a `retro` skill for OpenCode"
  Evidence: "retro/SKILL.md contains skill definition and HTML template"
  Type: feature
- Claim: "Dependencies: none (pure Markdown skill)"
  Evidence: "No package.json, pyproject.toml, or similar found"
  Type: dependency
```

The Analysis phase compares these claims against the README claims.

### 2.3 Phase 3: Analysis

Compare the Discovery claims against the Exploration findings and the hardcoded best-practice checklist from Section 4.

Group discrepancies into these categories. Each category owns a distinct concern — no overlap:

| Category | What to Check | Ownership Boundary |
|----------|---------------|--------------------|
| **Feature Coverage** | Does the README describe all major features? Are new features missing? Are described features still present? | Content-level: what the code *does*. Heuristic: exported functions, CLI commands, `package.json` scripts, HTTP endpoints, or other user-facing capabilities. |
| **Setup / Installation** | Are the install steps correct? Do they match the actual config files (e.g., `npm install` vs `pip install`)? Are missing steps identified? | Content-level: how to *get the project running*. Includes dependency installation, build steps, and environment setup. |
| **Dependencies** | Are dependency lists accurate? Are versions correct? Are optional dependencies noted? | Content-level: what the project *depends on*. Covers runtime and dev dependencies extracted from config files. |
| **Structure / Formatting** | Is the README well-structured? Are sections missing (e.g., no "Getting Started")? Is formatting consistent? | *Meta-level:* presence and order of sections per the Section 4 checklist. Does NOT check the *accuracy* of content within sections (that belongs to Setup, API, etc.). |
| **API / Usage** | Are usage examples correct? Do they match the actual API? Are there missing examples for key functionality? | Content-level: how to *use* the project after setup. Covers code examples, CLI flags, configuration options. |

For each discrepancy, record:
- **Issue:** What is wrong or missing
- **Evidence:** Concrete file paths, code snippets, or directory listings from the codebase
- **Proposed change:** Specific text or section to add, update, or remove

If no README exists, treat all categories as "missing" and generate suggestions from scratch using the codebase model. However, if the codebase model contains no evidence for a particular category (e.g., no dependencies found, no CI config), that category may still have zero issues and should be skipped per the empty-category rule in Phase 4.

### 2.4 Phase 4: Presentation

Present findings to the user **one category at a time**.

**Empty categories:** If a category has zero issues, skip presenting it entirely. Do not say "no issues found." Only present categories with ≥1 issue.

For each non-empty category:
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
> - **Quit** — exit without applying any pending or future changes

**If Feedback:**
1. Record the user's feedback.
2. Refine the category's findings based on the feedback.
3. Re-present the refined category.
4. The user can then Approve, Reject, Skip, or give more Feedback.
5. **Iteration cap:** Maximum 2 feedback rounds per category. After the 2nd round of feedback, force a decision: present the final refined category and ask the user to Approve, Reject, or Skip. Do not allow further feedback.

**If Approve:** Queue all proposed changes in this category for the final Application phase.

**If Reject:** Discard and move to the next category.

**If Skip:** Note the category as skipped and move on. (User can re-invoke `/readme` later to revisit.)

**If Quit:** Immediately discard all pending and future changes, exit the loop, and report: "Exited without applying any changes. README left as-is."

**Unrecognized input:** If the user responds with something other than Approve, Reject, Skip, or Feedback (e.g., "maybe" or a random question), re-prompt with: "I didn't understand that. Please choose: Approve, Reject, Skip, or Feedback." Do not proceed until a valid choice is given.

Continue until all categories are processed.

### 2.6 Phase 6: Application

1. Compile all approved changes from all categories.
2. **Conflict resolution:** If approved changes from different categories conflict, apply this prompt-friendly heuristic in order:
   a. **Content beats structure** — A change that adds or corrects content within a section (e.g., API/Usage adds an example) takes precedence over a change that moves or reorders that section (e.g., Structure/Formatting reorders it).
   b. **Specific beats general** — A change targeting a subsection takes precedence over a change targeting the whole document.
   c. If neither rule resolves the conflict, re-prompt the user with the conflicting changes before writing.
3. If a README exists, apply the changes inline, preserving as much existing wording and structure as possible.
4. If no README exists, create a new one from scratch using the approved content, structured according to best-practice guidelines.
5. Write the result back to the exact original filename (e.g., `README.md`, `README.org`, or `README`). If no README existed, default to `README.md`.
6. Confirm to the user: "README updated with X approved changes from Y categories."

---

## 3. Error Handling & Edge Cases

| Scenario | Behavior |
|----------|----------|
| **No README exists** | Treat as creation mode. Use best-practice template as structural baseline. Present all sections as "needs creation." |
| **README is empty or near-empty** | Same as "no README" — creation mode. |
| **Codebase has no recognizable structure** | If `glob` finds only scattered files with no clear entry points or conventions, present a minimal README suggestion and flag: "Codebase structure unclear — manual review recommended." |
| **User rejects all categories** | Gracefully exit without writing anything: "No changes approved. README left as-is." |
| **User gives feedback that contradicts codebase evidence** | Respect user's feedback but note the discrepancy: "Noted: user prefers X. Codebase evidence suggests Y. Applied user's preference." |
| **Contradictory feedback across rounds** | If the user gives feedback in round 1 (e.g., "don't mention X") and then contradicts it in round 2 (e.g., "why didn't you mention X?"), respect the most recent feedback and flag the inconsistency: "Noted: you previously asked to omit X. Applying your latest request to include X." |
| **Write to README fails** | Report the error and present the full proposed README content in a conversation code block so the user can apply it manually. |
| **README is `.org` format** | Preserve `.org` format. Read existing `README.org` and write updates back to `README.org` using Org-mode syntax. Never suggest converting to `.md`. |
| **Very large README** | If the README exceeds ~5000 lines or ~50KB, summarize it rather than reading in full. Flag to the user: "README is very large; analysis may be incomplete." |
| **Very large codebase** | Cap exploration at 30 files. Prioritize root configs, entry points, and a representative sample from each major directory. |
| **Empty working directory** | If `glob` returns zero files (empty directory), report: "This directory appears to be empty. No README can be generated." Exit gracefully. |
| **Exploration file unreadable** | If `glob` finds a file that is binary, unreadable, or permission-denied, skip it and continue exploration. Do not abort. |
| **Symlinked README** | Follow the symlink and read the target file. Preserve the original filename when writing back. |
| **Read-only README** | If the file system prevents writing, report the error and present the full proposed README content in a conversation code block. |
| **README without extension** | Treat `README` (no extension) as equivalent to `README.md` for reading and writing. |
| **Both `README.md` and `README.org` exist** | Use the tie-breaker heuristic from Section 2.1 (line count, then heading count). Note the presence of both to the user. If content differs significantly, ask which to update before proceeding. |

---

## 4. Best-Practice Guidelines (Reference)

The following checklist is hardcoded into the skill prompt. It is based on the Medium article "README Rules: Structure, Style, and Pro Tips" by Shaun Fulton, but the skill does NOT fetch the article at runtime. These rules are self-contained and deterministic:

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
| **Test H: Conflict resolution** | Verify that when two approved categories propose changes to the same section, the skill resolves the conflict by preferring the narrower-scope change. |
| **Test I: Unreadable exploration file** | Verify the skill skips binary or unreadable files during exploration without aborting. |

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
