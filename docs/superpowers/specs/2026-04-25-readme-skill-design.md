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
Is codebase structure recognizable?
    | No → [Bypass] → Present minimal README suggestion → End
    | Yes
    v
[Phase 3: Analysis] → Compare reality vs. README + best practices
    |
    v
[Phase 4: Presentation] → Present each category's findings
    |
    v
[Phase 5: Approval Loop] → User: Approve / Reject / Skip / Feedback / Quit
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
4. If both `README.md` and `README.org` exist:
   a. Read both files.
   b. **Prompt the user:** "Both README.md and README.org exist. Which would you like to continue with?"
   c. **Merge attempt:** Regardless of which the user chooses, build a merged model of claims by combining the contents of both files. If both files describe the same section differently, flag the discrepancy and use the more detailed version as the default. Do NOT rely on file mtime.
   d. Record the user's chosen primary file for write-back.
5. Also check for plain `README` (no extension) or `Readme.md` / `Readme.org` (case variants). If `README` (no extension) is found, treat it as a valid README file. When writing back, preserve the exact original filename. If both `README` (no extension) and `README.md` / `README.org` exist, prefer `README.md` / `README.org` (the extension-bearing file) and treat `README` as a secondary candidate only if no extension-bearing file exists. If both a case-variant and standard-cased version exist (e.g., `Readme.md` and `README.md`), prefer the standard-cased `README.md` / `README.org`.

### 2.2 Phase 2: Exploration

Systematically explore the codebase using `glob`, `read`, and `bash` tools. Priority order:

1. **Root config files** — `package.json`, `pyproject.toml`, `Cargo.toml`, `pom.xml`, `build.gradle`, `Makefile`, etc.
2. **Entry points** — main application files, CLI entry points, library exports.
3. **Source directories** — representative files from each major directory. A "major directory" is any top-level directory (or immediate subdirectory of `src/`) that contains ≥5 source files or an entry-point marker (e.g., `__init__.py`, `index.js`, `main.py`, `mod.rs`). A "source file" is any file with a code extension (e.g., `.py`, `.js`, `.ts`, `.rs`, `.go`, `.java`, `.c`, `.cpp`, `.h`, `.swift`, `.kt`, `.scala`, `.rb`, `.php`, `.cs`) excluding test files, generated files, lockfiles, and build artifacts.
4. **Tests** — presence and structure of test files.
5. **CI / DevOps** — `.github/workflows/`, `.gitlab-ci.yml`, `Dockerfile`, etc.
6. **Documentation** — `docs/`, `AGENTS.md`, `CLAUDE.md`, changelogs.
7. **Other metadata** — `.gitignore`, `LICENSE`, `CONTRIBUTING.md`.

**Exploration cap:** Max 30 files read in a single invocation. Allocate files per priority level to ensure coverage:
- Root config files: max 5 files
- Entry points: max 5 files
- Source directories: max 15 files (representative sample from each major directory)
- Tests: max 3 files
- CI / DevOps: max 2 files
- Documentation / Metadata: max 2 files combined
**Large exploration files:** If any exploration file exceeds ~2000 lines or ~20KB, read only the first 100 lines (or a representative section) and flag: "File X is very large; read a sample only."
**Context protection:** If the total content gathered exceeds what can be held in the LLM context window, prioritize the most recent/important files and summarize the rest.

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

Group discrepancies into these categories. Each category owns a distinct concern in *analysis* — no overlap during fact-finding. However, during *application*, approved changes from different categories may target the same README section (e.g., Structure/Formatting may reorder a section while API/Usage adds content within it). This is expected and handled by the conflict-resolution rules in Phase 6.

**Deduplication rule:** If the same underlying discrepancy could be flagged by multiple categories (e.g., a missing CLI feature noted in both Feature Coverage and API/Usage), emit it only in the **most specific** category. Specificity hierarchy (most specific first): API/Usage > Setup/Installation > Dependencies > Feature Coverage > Structure/Formatting. If ambiguity remains (e.g., a missing build step that could fit Dependencies or Setup/Installation), prefer the category that appears first in this list.

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

If no README exists, treat all categories as "missing" and generate suggestions from scratch using the codebase model. Generate suggestions for every category that has any discoverable evidence. Only skip a category if the codebase model contains absolutely no evidence for it (e.g., no dependencies found AND no config files AND no package managers). This is the only scenario where an empty category should be skipped in creation mode.

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
4. Ask the user for a response. Valid responses are defined in Phase 5 (Approve, Reject, Skip, Feedback, Quit).

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
5. **Iteration cap:** Maximum 2 feedback rounds per category. One feedback round = one instance of the user providing feedback text. After the 2nd round of feedback, force a decision: present the final refined category and ask the user to Approve, Reject, or Skip. Do not allow further feedback.

**If Approve:** Queue all proposed changes in this category for the final Application phase.

**If Reject:** Discard and move to the next category.

**If Skip:** Note the category as skipped and move on. (User can re-invoke `/readme` later to revisit.)

**If Quit:** Immediately discard all pending and future changes, exit the loop, and report: "Exited without applying any changes. README left as-is."

**Unrecognized input:** If the user responds with something other than Approve, Reject, Skip, Feedback, or Quit (e.g., "maybe" or a random question), re-prompt with: "I didn't understand that. Please choose: Approve, Reject, Skip, Feedback, or Quit." Do not proceed until a valid choice is given.

Continue until all categories are processed.

### 2.6 Phase 6: Application

1. Compile all approved changes from all categories.
2. **Conflict resolution:** If approved changes from different categories conflict, apply this prompt-friendly heuristic in order:
   a. **Content beats structure** — A change that adds or corrects content within a section (e.g., API/Usage adds an example) takes precedence over a change that moves or reorders that section (e.g., Structure/Formatting reorders it).
   b. **Specific beats general** — A change targeting a subsection takes precedence over a change targeting the whole document.
   c. If neither rule resolves the conflict, re-prompt the user with the conflicting changes and offer these options:
      - **Choose Change A** — apply the first conflicting change
      - **Choose Change B** — apply the second conflicting change
      - **Reject Both** — discard both conflicting changes
      - **Abort** — exit without writing any changes
      If the user provides unrecognized input, re-prompt: "Please choose: Change A, Change B, Reject Both, or Abort." This mid-application re-prompt is a separate forced decision and is NOT subject to the 2-round feedback cap from Phase 5.
3. If a README exists (or both exist), apply the changes inline to the user's chosen primary file, preserving as much existing wording and structure as possible. If both files existed, the merged model from Discovery ensures content from both is considered.
4. If no README exists, create a new one from scratch using the approved content, structured according to best-practice guidelines.
5. Write the result back to the user's chosen primary filename. If no README existed, default to `README.md`.
6. If both `README.md` and `README.org` existed, confirm: "README updated with X approved changes from Y categories. [Primary file] was updated; [other file] was left as-is."
7. If only one README existed, confirm: "README updated with X approved changes from Y categories."

---

## 3. Error Handling & Edge Cases

| Scenario | Behavior |
|----------|----------|
| **No README exists** | Treat as creation mode. Use best-practice template as structural baseline. Present all sections as "needs creation." |
| **README is empty or near-empty** | Same as "no README" — creation mode. |
| **Codebase has no recognizable structure** | If `glob` finds only scattered files with no clear entry points or conventions, bypass the category system entirely. Present a minimal README suggestion directly and flag: "Codebase structure unclear — manual review recommended." Do not attempt the per-category approval loop.
| **User rejects all categories** | Gracefully exit without writing anything: "No changes approved. README left as-is." |
| **User gives feedback that contradicts codebase evidence** | Respect user's feedback but note the discrepancy in the conversation transcript: "Noted: user prefers X. Codebase evidence suggests Y. Applied user's preference." Do NOT append notes to the README itself. |
| **Contradictory feedback across rounds** | If the user gives feedback in round 1 (e.g., "don't mention X") and then contradicts it in round 2 (e.g., "why didn't you mention X?"), respect the most recent feedback and flag the inconsistency in the conversation transcript: "Noted: you previously asked to omit X. Applying your latest request to include X." Do NOT append notes to the README itself. |
| **Write to README fails** | Report the error and present the full proposed README content in a conversation code block so the user can apply it manually. |
| **README is `.org` format** | Preserve `.org` format. Read existing `README.org` and write updates back to `README.org` using Org-mode syntax. Never suggest converting to `.md`. |
| **Very large README** | If the README exceeds ~5000 lines or ~50KB, summarize it rather than reading in full. Flag to the user: "README is very large; analysis may be incomplete." |
| **Binary or unreadable README** | If `README.md`/`README.org` is binary, unreadable, or permission-denied, report: "README file found but cannot be read. Please check permissions or encoding." Treat as missing README (creation mode) but warn the user not to overwrite without investigating. |
| **Very large codebase** | Cap exploration at 30 files. Prioritize root configs, entry points, and a representative sample from each major directory. |
| **Empty working directory** | If `glob` returns zero files (empty directory), report: "This directory appears to be empty. No README can be generated." Exit gracefully. |
| **Exploration file unreadable** | If `glob` finds a file that is binary, unreadable, or permission-denied, skip it and continue exploration. Do not abort. |
| **Symlinked README** | Follow the symlink and read the target file only if the target is within the working directory. If the symlink points outside the working directory, do NOT follow it; treat as missing README. When writing back, if the symlink was not followed, create a new `README.md` (or `README.org`) alongside the symlink rather than overwriting the symlink itself. |
| **Read-only README** | If the file system prevents writing, report the error and present the full proposed README content in a conversation code block. |
| **README without extension** | Treat `README` (no extension) as equivalent to `README.md` for reading and writing. |
| **Both `README.md` and `README.org` exist** | Read both files. Prompt the user to choose which to continue with. Build a merged model of claims from both files, flagging discrepancies. Write updates back to the user's chosen primary file only. |
| **README references assets** | Preserve all image paths, relative links, and asset references in the existing README during updates. Do not modify asset paths unless the user explicitly approves a change that affects them. |
| **Partial analysis** | If the 30-file exploration cap is reached, flag to the user: "Analysis capped at 30 files — some areas of the codebase may not have been fully explored." |
| **Loose source files in root** | If source files exist in the project root but not within a "major directory," read a representative sample of them (up to 3 files) as part of the exploration. |

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
| **Test G: Both `.md` and `.org` present** | Verify the skill prompts the user to choose a primary file, merges claims from both, and writes back to the chosen file only. |
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
