# Reflect Skill Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a self-contained opencode skill (`reflect/SKILL.md`) that analyzes the current session and generates a WIRED-styled HTML retrospective report, then opens it in the default browser.

**Architecture:** Single-file technique skill with embedded analysis instructions and complete WIRED HTML template. The skill instructs the agent to scan conversation context, categorize effort, detect friction points, fill the template, write to `/tmp`, and open the browser.

**Tech Stack:** Markdown (SKILL.md), HTML/CSS (embedded template), standard OS commands (`open`/`xdg-open`, `date`).

---

## Chunk 1: Skill Foundation

### Task 1: Create Skill Directory and Frontmatter

**Files:**
- Create: `/Users/whodevil/src/skills/reflect/SKILL.md`

- [ ] **Step 1: Create directory structure**

```bash
mkdir -p /Users/whodevil/src/skills/reflect
```

- [ ] **Step 2: Write YAML frontmatter and overview**

Create `/Users/whodevil/src/skills/reflect/SKILL.md` with the frontmatter and overview section:

```markdown
---
name: reflect
description: Use when the user wants to review the current session, analyze time spent, identify repo improvements, or generate a retrospective report.
---

# Reflect — Session Retrospective

## Overview

Analyze the current conversation session, categorize where time and attention were spent, identify friction points and repo improvements, and generate a polished HTML report styled after the WIRED design system. Open the report in the default browser when complete.

## When to Use

- User explicitly asks for a session review or retrospective
- User types `/reflect`
- Need to identify what slowed the agent down during a session
- Need actionable recommendations to improve the repository

## Core Pattern

The agent follows a three-phase analysis and generates a single HTML file:

1. **Scan & Categorize** — bucket tool calls and reasoning into effort categories
2. **Detect Friction** — identify patterns that indicate repo/workflow issues
3. **Summarize & Report** — compute metrics, fill WIRED HTML template, write to disk, open browser
```

**Verification:** File exists and contains valid YAML frontmatter.

- [ ] **Step 3: Commit**

```bash
cd /Users/whodevil/src/skills
git add reflect/SKILL.md
git commit -m "feat: create reflect skill foundation with frontmatter and overview"
```

---

## Chunk 2: Analysis Instructions

### Task 2: Write Phase 1 — Scan & Categorize

**Files:**
- Modify: `/Users/whodevil/src/skills/reflect/SKILL.md`

- [ ] **Step 1: Append Phase 1 instructions**

Add after the Core Pattern section:

```markdown
## Analysis Instructions

### Phase 1: Scan & Categorize

Scan the conversation context for tool calls and reasoning patterns. Bucket each into one of these categories based on signals:

| Category | Signals |
|----------|---------|
| **Coding** | `Write`, `Edit`, `read` on source files; code generation blocks; refactoring commands |
| **Debugging** | `systematic-debugging` skill invocations; error logs; repeated tool calls on the same file; `bash` debugging commands |
| **Research** | `websearch`, `webfetch`, `codesearch`, `context7_query-docs`; reading docs or external resources |
| **Testing** | `test-driven-development` skill; test execution commands; assertion failures; `pytest`, `jest`, etc. |
| **Waiting** | `bash` timeouts; external process waits; sleep-like patterns |
| **Misc** | User greetings, clarifying questions, plan updates, meta-discussion |

**Heuristics:**
- Estimate tool-call counts per category by scanning the conversation transcript for tool-usage markers (e.g., "`Write`", "`Edit`", "`websearch`"). Approximate counts, not structured log parses.
- Layer qualitative "attention weight" by considering message length and reasoning depth.
- Detect task boundaries: a new user request following a completion message; a shift from planning to execution marked by `TodoWrite` or plan presentation; a cluster of tool calls following a reasoning block.
- Note each task boundary: user request → agent plan → execution → completion or abandonment.
```

**Verification:** Section added correctly with table formatting intact.

- [ ] **Step 2: Commit**

```bash
git add reflect/SKILL.md
git commit -m "feat: add Phase 1 scan and categorize instructions"
```

---

### Task 3: Write Phase 2 — Detect Friction Points

**Files:**
- Modify: `/Users/whodevil/src/skills/reflect/SKILL.md`

- [ ] **Step 1: Append Phase 2 instructions**

Add after Phase 1:

```markdown
### Phase 2: Detect Friction Points

Look for patterns indicating the repo or workflow could be improved:

| Friction Type | Signals | Severity |
|---------------|---------|----------|
| **Missing Docs/Config** | Failed `read`/`glob` searches for conventions; repeated `grep` for "how do we do X?" | Medium |
| **Code Smells / Churn** | Repeated `edit` on the same file; very long files with many edits; absence of tests when test files exist | High |
| **Error Patterns** | Same error type recurring; tool failures followed by retries; `systematic-debugging` triggered multiple times | High |
| **Structural Gaps** | No `AGENTS.md`/`CLAUDE.md`; no test files; no `.gitignore`; inconsistent naming; missing CI config | Medium |
| **Dependency Issues** | Repeated `npm install`/`pip install`; missing lockfile; version conflicts | Low |

For each friction point, record:
- **What happened** — concrete evidence from the conversation
- **Why it mattered** — impact on velocity or correctness
- **Recommended fix** — specific, actionable recommendation
```

- [ ] **Step 2: Commit**

```bash
git add reflect/SKILL.md
git commit -m "feat: add Phase 2 friction point detection"
```

---

### Task 4: Write Phase 3 — Summarize Success Metrics

**Files:**
- Modify: `/Users/whodevil/src/skills/reflect/SKILL.md`

- [ ] **Step 1: Append Phase 3 instructions**

Add after Phase 2:

```markdown
### Phase 3: Summarize Success Metrics

- **Tasks initiated vs. completed** — with brief descriptions of each
- **Per-task effort estimate** — approximate attention investment based on message count, reasoning depth, and tool calls within each task boundary:
  - `Low` — few messages, simple execution
  - `Medium` — moderate back-and-forth, some complexity
  - `High` — complex, many iterations, debugging required
- **Key decisions made** — major architectural or technical choices, with rationale
- **Attention distribution** — approximate percentage of effort per category (derived from tool-call counts)
```

- [ ] **Step 2: Commit**

```bash
git add reflect/SKILL.md
git commit -m "feat: add Phase 3 success metrics summarization"
```

---

## Chunk 3: HTML Template

### Task 5: Write Design Tokens and CSS Rules

**Files:**
- Modify: `/Users/whodevil/src/skills/reflect/SKILL.md`

- [ ] **Step 1: Append WIRED template introduction and design tokens**

Add after Phase 3:

```markdown
## WIRED HTML Template

The skill embeds a complete, self-contained HTML document. Copy this template, replace all `{{PLACEHOLDER}}` instances with computed HTML fragments, and write the result to disk.

### HTML Escaping Rules

Inject **two kinds of content** with different escaping:

1. **Structural markup fragments** (`<div class="bar-row">`, `<tr>`, `<li>`): Injected **raw** (unescaped). These are generated by the agent and are trusted HTML.

2. **User-facing textual content** inside those fragments (tool outputs, error messages, file paths, code snippets, conversation quotes): Must be **HTML-escaped** before insertion:
   - `&` → `&amp;`
   - `<` → `&lt;`
   - `>` → `&gt;`
   - `"` → `&quot;`

### Design Tokens

| Token | Value | Usage |
|-------|-------|-------|
| `background` | `#FDFCF8` | Page background (paper-white) |
| `text-primary` | `#1A1A1A` | Body text, headlines |
| `text-secondary` | `#595959` | Captions, metadata, secondary info |
| `accent` | `#0A6ECC` | Links, highlights, ink-blue accent only |
| `border` | `#E2E2E2` | Section dividers, table borders |
| `kicker` | `ui-monospace, "SF Mono", Menlo, monospace` | Category tags, timestamps, labels |
| `display` | `Georgia, "Times New Roman", Times, serif` | Headlines, section titles |
| `body` | `Georgia, "Times New Roman", Times, serif` | Body copy, long-form text |

### CSS Rules

- No `border-radius` anywhere; sharp, print-style corners.
- Hierarchy via borders (`1px solid #E2E2E2`), weight (`font-weight: 700` for headlines), and whitespace — no drop shadows.
- Generous line height (`1.6`) for body text; tight leading (`1.1`) for display headlines.
- Max content width `680px`, centered, with generous top/bottom padding (`64px`).
```

- [ ] **Step 2: Commit**

```bash
git add reflect/SKILL.md
git commit -m "feat: add WIRED design tokens and CSS rules"
```

---

### Task 6: Write Complete HTML Template

**Files:**
- Modify: `/Users/whodevil/src/skills/reflect/SKILL.md`

- [ ] **Step 1: Append the complete HTML template**

Add after CSS Rules:

```markdown
### Complete HTML Template

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Session Retrospective — {{PROJECT_NAME}}</title>
  <style>
    :root {
      --bg: #FDFCF8;
      --text: #1A1A1A;
      --text-secondary: #595959;
      --accent: #0A6ECC;
      --border: #E2E2E2;
      --kicker: ui-monospace, "SF Mono", Menlo, monospace;
      --display: Georgia, "Times New Roman", Times, serif;
      --body: Georgia, "Times New Roman", Times, serif;
    }
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      background: var(--bg);
      color: var(--text);
      font-family: var(--body);
      line-height: 1.6;
      padding: 64px 24px;
    }
    .container {
      max-width: 680px;
      margin: 0 auto;
    }
    .kicker {
      font-family: var(--kicker);
      font-size: 12px;
      text-transform: uppercase;
      letter-spacing: 0.05em;
      color: var(--text-secondary);
      margin-bottom: 8px;
    }
    h1 {
      font-family: var(--display);
      font-size: 42px;
      font-weight: 700;
      line-height: 1.1;
      margin-bottom: 16px;
    }
    .meta {
      font-family: var(--kicker);
      font-size: 12px;
      color: var(--text-secondary);
      margin-bottom: 48px;
    }
    section {
      border-top: 1px solid var(--border);
      padding: 32px 0;
    }
    h2 {
      font-family: var(--display);
      font-size: 24px;
      font-weight: 700;
      line-height: 1.2;
      margin-bottom: 16px;
    }
    .summary-text {
      font-size: 18px;
      line-height: 1.5;
      color: var(--text-secondary);
    }
    .bar-row {
      display: flex;
      align-items: center;
      gap: 12px;
      margin-bottom: 12px;
    }
    .bar-label {
      font-family: var(--kicker);
      font-size: 12px;
      text-transform: uppercase;
      width: 100px;
      flex-shrink: 0;
    }
    .bar-track {
      flex: 1;
      height: 8px;
      background: var(--border);
      position: relative;
    }
    .bar-fill {
      height: 100%;
      background: var(--accent);
    }
    .bar-value {
      font-family: var(--kicker);
      font-size: 12px;
      width: 48px;
      text-align: right;
    }
    table {
      width: 100%;
      border-collapse: collapse;
      font-size: 14px;
    }
    th, td {
      padding: 12px 8px;
      text-align: left;
      border-bottom: 1px solid var(--border);
    }
    th {
      font-family: var(--kicker);
      font-size: 11px;
      text-transform: uppercase;
      letter-spacing: 0.05em;
      color: var(--text-secondary);
    }
    .badge {
      display: inline-block;
      font-family: var(--kicker);
      font-size: 11px;
      text-transform: uppercase;
      padding: 2px 8px;
      border: 1px solid var(--text);
      margin-right: 8px;
    }
    .badge-high { border-color: #DC2626; color: #DC2626; }
    .badge-medium { border-color: var(--text-secondary); color: var(--text-secondary); }
    .badge-low { border-color: var(--border); color: var(--text-secondary); }
    ol, ul {
      padding-left: 24px;
    }
    li {
      margin-bottom: 16px;
    }
    pre {
      background: #f5f5f5;
      padding: 12px;
      overflow-x: auto;
      font-family: var(--kicker);
      font-size: 13px;
      line-height: 1.4;
      margin: 8px 0;
    }
    code {
      font-family: var(--kicker);
      font-size: 13px;
    }
    footer {
      border-top: 1px solid var(--border);
      padding-top: 24px;
      margin-top: 48px;
      font-family: var(--kicker);
      font-size: 11px;
      color: var(--text-secondary);
    }
  </style>
</head>
<body>
  <div class="container">
    <div class="kicker">Session Retrospective</div>
    <h1>{{PROJECT_NAME}}</h1>
    <div class="meta"><time>{{TIMESTAMP}}</time></div>

    <section id="summary">
      {{EXECUTIVE_SUMMARY}}
    </section>

    <section id="attention">
      <h2>Estimated Attention Distribution</h2>
      {{ATTENTION_BREAKDOWN_BARS}}
    </section>

    <section id="tasks">
      <h2>Task Log</h2>
      <table>
        <thead>
          <tr>
            <th>Task</th>
            <th>Status</th>
            <th>Effort</th>
            <th>Outcome</th>
          </tr>
        </thead>
        <tbody>
          {{TASK_LOG_ROWS}}
        </tbody>
      </table>
    </section>

    <section id="friction">
      <h2>Friction Points &amp; Recommendations</h2>
      <ol>
        {{FRICTION_POINTS}}
      </ol>
    </section>

    <section id="decisions">
      <h2>Key Decisions</h2>
      <ul>
        {{KEY_DECISIONS}}
      </ul>
    </section>

    <footer>
      Generated by reflect skill &middot; {{TIMESTAMP}}
    </footer>
  </div>
</body>
</html>
```
```

**Verification:** Template is complete with all seven sections and placeholder markers.

- [ ] **Step 2: Commit**

```bash
git add reflect/SKILL.md
git commit -m "feat: add complete WIRED HTML template"
```

---

## Chunk 4: Template Filling and Placeholder Rules

### Task 7: Write Placeholder Filling Instructions

**Files:**
- Modify: `/Users/whodevil/src/skills/reflect/SKILL.md`

- [ ] **Step 1: Append placeholder conventions and filling rules**

Add after the template:

```markdown
### Placeholder Filling Guide

| Placeholder | Content Type | Surrounding Markup | Fallback |
|-------------|--------------|-------------------|----------|
| `{{PROJECT_NAME}}` | Plain text string (HTML-escaped) | Inside `<h1>` masthead | `Unknown Project` |
| `{{TIMESTAMP}}` | Plain text string (HTML-escaped) | Inside `<time>` element | Current timestamp from `date +%Y-%m-%d-%H-%M-%S` |
| `{{EXECUTIVE_SUMMARY}}` | HTML paragraph(s) | Inside `<section id="summary">` | Strip section |
| `{{ATTENTION_BREAKDOWN_BARS}}` | HTML `<div class="bar-row">` elements | Inside `<section id="attention">` | Strip section |
| `{{TASK_LOG_ROWS}}` | HTML `<tr>` elements | Inside `<tbody>` of task table | Strip section |
| `{{FRICTION_POINTS}}` | HTML `<li>` elements | Inside `<ol>` in `<section id="friction">` | Strip section |
| `{{KEY_DECISIONS}}` | HTML `<li>` elements | Inside `<ul>` in `<section id="decisions">` | Strip section |

**Project Name Source:** Derive from current working directory basename (e.g., `skills` if in `/Users/whodevil/src/skills`). If unavailable or home directory, use `Unknown Project`.

**Stripping Rule:** If a section-wrapped placeholder has no content, remove the entire `<section id="...">...</section>` block. `{{PROJECT_NAME}}` and `{{TIMESTAMP}}` are never stripped — they always have fallback values.

**Bar Chart Markup:**
Each category becomes:
```html
<div class="bar-row">
  <span class="bar-label">Coding</span>
  <div class="bar-track">
    <div class="bar-fill" style="width: 45%;"></div>
  </div>
  <span class="bar-value">45%</span>
</div>
```
Percentages are approximate proportions based on tool-call counts, not precise measurements.

**Multi-line content** (code snippets, error logs) is wrapped in `<pre><code>` blocks with escaping rules applied.
```

- [ ] **Step 2: Commit**

```bash
git add reflect/SKILL.md
git commit -m "feat: add placeholder filling guide and conventions"
```

---

## Chunk 5: Execution Flow and Error Handling

### Task 8: Write Execution Instructions

**Files:**
- Modify: `/Users/whodevil/src/skills/reflect/SKILL.md`

- [ ] **Step 1: Append execution flow section**

Add after placeholder guide:

```markdown
## Execution Flow

When invoked, follow this sequence:

1. **Analyze the conversation context** using the three-phase method above.
2. **Build HTML fragments** for each placeholder based on the analysis.
3. **Copy the template** and replace all `{{PLACEHOLDER}}` markers with fragments.
4. **Strip empty sections** according to the stripping rule.
5. **Get timestamp** via `bash` tool: `date +%Y-%m-%d-%H-%M-%S`
6. **Write the HTML** to `/tmp/reflect-<timestamp>.html`
7. **Open the browser** via `bash` tool:
   - macOS: `open /tmp/reflect-<timestamp>.html`
   - Linux: `xdg-open /tmp/reflect-<timestamp>.html`
   - Windows: **out of scope** — report the file path to the user instead
```

- [ ] **Step 2: Commit**

```bash
git add reflect/SKILL.md
git commit -m "feat: add execution flow instructions"
```

---

### Task 9: Write Error Handling

**Files:**
- Modify: `/Users/whodevil/src/skills/reflect/SKILL.md`

- [ ] **Step 1: Append error handling section**

Add after execution flow:

```markdown
## Error Handling

| Scenario | Behavior |
|----------|----------|
| **Very short session** (< 5 messages) | Generate a minimal report: "Session too brief for meaningful analysis." Still styled, still opens browser. |
| **No tool calls found** | Report says "No tool activity detected — session may be purely conversational." Categories show 100% Misc. |
| **Browser open fails** | Report the file path in the conversation so the user can open it manually. |
| **Unsupported OS (Windows)** | Skip browser-open. Report the file path in the conversation. |
| **Template variable missing** | Strip out the unfilled placeholder and its surrounding `<section>` wrapper. Never crash HTML generation. |
| **`/tmp` not writable** | Attempt to write to the current working directory as fallback. If that also fails, output the raw HTML in a conversation message block. |
```

- [ ] **Step 2: Commit**

```bash
git add reflect/SKILL.md
git commit -m "feat: add error handling scenarios"
```

---

## Chunk 6: Testing (RED-GREEN-REFACTOR)

### Task 10: RED Phase — Baseline Test Without Skill

**Files:**
- No files modified (this is a test)

- [ ] **Step 1: Create synthetic conversation for testing**

Create a text file `/Users/whodevil/src/skills/test-conversation.md` simulating a session:

```markdown
# Synthetic Test Conversation

## Message 1 (User)
Fix the bug in auth.py

## Message 2 (Agent)
I'll investigate the authentication bug.

## Message 3 (Agent - tool)
read /project/auth.py
read /project/tests/test_auth.py

## Message 4 (Agent - reasoning)
Looking at auth.py, I see the issue: the password hash comparison uses `==` instead of a constant-time comparison.

## Message 5 (Agent - tool)
Edit /project/auth.py:45-50
Replace with secure comparison

## Message 6 (Agent - tool)
bash: pytest tests/test_auth.py -v

## Message 7 (Agent - tool)
Test failed: AssertionError in test_login_valid

## Message 8 (Agent - tool)
systematic-debugging skill invoked

## Message 9 (Agent - tool)
read /project/auth.py
read /project/tests/test_auth.py

## Message 10 (Agent - tool)
Edit /project/auth.py:45-50
Fix logic error

## Message 11 (Agent - tool)
bash: pytest tests/test_auth.py -v

## Message 12 (Agent)
Tests pass. Bug is fixed.
```

- [ ] **Step 2: Dispatch subagent WITHOUT the skill**

Use `Task` tool to dispatch a subagent with this prompt:

```
You are an AI assistant analyzing a conversation session.

Read the conversation at /Users/whodevil/src/skills/test-conversation.md and produce a retrospective report covering:
1. Where time was spent (categories: Coding, Debugging, Research, Testing, Waiting, Misc)
2. Any friction points or repo improvements you noticed
3. Key decisions made

Output your analysis as plain text.
```

- [ ] **Step 3: Observe baseline behavior**

Document what the subagent produces:
- Did it categorize correctly?
- Did it detect the repeated `Edit` on auth.py (code smell)?
- Did it note the missing constant-time comparison initially?
- Was the output plain text (no HTML)?
- Was there no mention of opening a browser?

Expected: Likely misses categories, produces plain text, no styling, no browser opening.

- [ ] **Step 4: Save baseline observation**

```bash
cd /Users/whodevil/src/skills
git add test-conversation.md
git commit -m "test: add synthetic conversation for RED phase baseline"
```

---

### Task 11: GREEN Phase — Verify Skill Works

**Files:**
- Modify: `/Users/whodevil/src/skills/reflect/SKILL.md` (if refinements needed)

- [ ] **Step 1: Dispatch subagent WITH the skill**

Use `Task` tool to dispatch a subagent with:

```
You are an AI assistant with the reflect skill loaded.

The skill is at /Users/whodevil/src/skills/reflect/SKILL.md. Read it and follow its instructions to analyze the conversation at /Users/whodevil/src/skills/test-conversation.md.

Generate the HTML report and attempt to open it in the browser (or at least write it to /tmp/ and report the path if browser open fails).
```

- [ ] **Step 2: Verify compliance**

Check the generated HTML at `/tmp/reflect-*.html`:
- Does it contain WIRED design tokens (`#FDFCF8`, `#0A6ECC`, Georgia font)?
- Does it have all seven sections?
- Are categories identified (Coding: auth.py edits, Testing: pytest runs, Debugging: systematic-debugging)?
- Is the repeated `Edit` on auth.py flagged as friction?
- Does it attempt to open the browser?

- [ ] **Step 3: Fix any issues**

If the subagent misses categories or fails to generate HTML, refine the analysis instructions in `reflect/SKILL.md`. Re-test.

- [ ] **Step 4: Commit GREEN phase results**

```bash
git add reflect/SKILL.md
git commit -m "test: verify skill generates WIRED HTML and opens browser (GREEN phase)"
```

---

### Task 12: REFACTOR Phase — Edge Cases

**Files:**
- Modify: `/Users/whodevil/src/skills/reflect/SKILL.md` (if loopholes found)

- [ ] **Step 1: Test very short session**

Create `/Users/whodevil/src/skills/test-short-conversation.md`:

```markdown
# Short Session

## Message 1 (User)
Hello

## Message 2 (Agent)
Hi! How can I help?
```

Dispatch subagent with skill loaded. Verify it generates a minimal report: "Session too brief for meaningful analysis."

- [ ] **Step 2: Test no-tool-calls session**

Create `/Users/whodevil/src/skills/test-no-tools-conversation.md`:

```markdown
# No Tools Session

## Message 1 (User)
What is the best way to structure a Python project?

## Message 2 (Agent)
Here are some recommendations...
```

Verify report says "No tool activity detected — session may be purely conversational." with 100% Misc.

- [ ] **Step 3: Close loopholes**

If the subagent:
- Crashes on empty sections → strengthen stripping rule language
- Fails to escape HTML characters → add explicit escaping examples
- Misses task boundaries → clarify boundary detection heuristics
- Fails to open browser on Linux → verify `xdg-open` command

Add explicit counters to the skill text.

- [ ] **Step 4: Commit refactor phase**

```bash
git add reflect/SKILL.md
git commit -m "refactor: close loopholes from edge case testing"
```

- [ ] **Step 5: Clean up test files**

```bash
rm /Users/whodevil/src/skills/test-*.md
git add -A
git commit -m "chore: remove test conversation files"
```

---

## Chunk 7: Final Verification

### Task 13: Validate Complete Skill

**Files:**
- Read: `/Users/whodevil/src/skills/reflect/SKILL.md`

- [ ] **Step 1: Run final validation checklist**

Verify the skill contains:
- [ ] YAML frontmatter with `name: reflect` and proper description
- [ ] Overview section with purpose and when to use
- [ ] Phase 1: Scan & Categorize with category table
- [ ] Phase 2: Detect Friction Points with severity table
- [ ] Phase 3: Summarize Success Metrics with effort estimates
- [ ] Complete WIRED HTML template with inline CSS
- [ ] Placeholder filling guide with stripping rules
- [ ] Execution flow with browser open commands
- [ ] Error handling table with 6 scenarios
- [ ] No TODOs, TBDs, or placeholder text remaining

- [ ] **Step 2: Manual HTML rendering test**

Copy the template from the skill, fill with dummy data, and verify it renders correctly in a browser:

```bash
# Create a test HTML file with sample data
cat > /tmp/reflect-test.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Session Retrospective — Test Project</title>
  <style>
    :root {
      --bg: #FDFCF8;
      --text: #1A1A1A;
      --text-secondary: #595959;
      --accent: #0A6ECC;
      --border: #E2E2E2;
      --kicker: ui-monospace, "SF Mono", Menlo, monospace;
      --display: Georgia, "Times New Roman", Times, serif;
      --body: Georgia, "Times New Roman", Times, serif;
    }
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      background: var(--bg);
      color: var(--text);
      font-family: var(--body);
      line-height: 1.6;
      padding: 64px 24px;
    }
    .container { max-width: 680px; margin: 0 auto; }
    .kicker {
      font-family: var(--kicker);
      font-size: 12px;
      text-transform: uppercase;
      letter-spacing: 0.05em;
      color: var(--text-secondary);
      margin-bottom: 8px;
    }
    h1 {
      font-family: var(--display);
      font-size: 42px;
      font-weight: 700;
      line-height: 1.1;
      margin-bottom: 16px;
    }
    .meta {
      font-family: var(--kicker);
      font-size: 12px;
      color: var(--text-secondary);
      margin-bottom: 48px;
    }
    section {
      border-top: 1px solid var(--border);
      padding: 32px 0;
    }
    h2 {
      font-family: var(--display);
      font-size: 24px;
      font-weight: 700;
      line-height: 1.2;
      margin-bottom: 16px;
    }
    .bar-row {
      display: flex;
      align-items: center;
      gap: 12px;
      margin-bottom: 12px;
    }
    .bar-label {
      font-family: var(--kicker);
      font-size: 12px;
      text-transform: uppercase;
      width: 100px;
      flex-shrink: 0;
    }
    .bar-track {
      flex: 1;
      height: 8px;
      background: var(--border);
      position: relative;
    }
    .bar-fill {
      height: 100%;
      background: var(--accent);
    }
    .bar-value {
      font-family: var(--kicker);
      font-size: 12px;
      width: 48px;
      text-align: right;
    }
    table {
      width: 100%;
      border-collapse: collapse;
      font-size: 14px;
    }
    th, td {
      padding: 12px 8px;
      text-align: left;
      border-bottom: 1px solid var(--border);
    }
    th {
      font-family: var(--kicker);
      font-size: 11px;
      text-transform: uppercase;
      letter-spacing: 0.05em;
      color: var(--text-secondary);
    }
    .badge {
      display: inline-block;
      font-family: var(--kicker);
      font-size: 11px;
      text-transform: uppercase;
      padding: 2px 8px;
      border: 1px solid var(--text);
      margin-right: 8px;
    }
    .badge-high { border-color: #DC2626; color: #DC2626; }
    ol, ul { padding-left: 24px; }
    li { margin-bottom: 16px; }
    footer {
      border-top: 1px solid var(--border);
      padding-top: 24px;
      margin-top: 48px;
      font-family: var(--kicker);
      font-size: 11px;
      color: var(--text-secondary);
    }
  </style>
</head>
<body>
  <div class="container">
    <div class="kicker">Session Retrospective</div>
    <h1>Test Project</h1>
    <div class="meta"><time>2026-04-24-12-00-00</time></div>

    <section id="summary">
      <p class="summary-text">This is a test of the reflect skill HTML template rendering.</p>
    </section>

    <section id="attention">
      <h2>Estimated Attention Distribution</h2>
      <div class="bar-row">
        <span class="bar-label">Coding</span>
        <div class="bar-track"><div class="bar-fill" style="width: 60%;"></div></div>
        <span class="bar-value">60%</span>
      </div>
      <div class="bar-row">
        <span class="bar-label">Testing</span>
        <div class="bar-track"><div class="bar-fill" style="width: 40%;"></div></div>
        <span class="bar-value">40%</span>
      </div>
    </section>

    <section id="tasks">
      <h2>Task Log</h2>
      <table>
        <thead>
          <tr><th>Task</th><th>Status</th><th>Effort</th><th>Outcome</th></tr>
        </thead>
        <tbody>
          <tr><td>Fix auth bug</td><td>Completed</td><td>High</td><td>Success</td></tr>
        </tbody>
      </table>
    </section>

    <footer>
      Generated by reflect skill &middot; 2026-04-24-12-00-00
    </footer>
  </div>
</body>
</html>
EOF

# Open in browser
open /tmp/reflect-test.html
```

- [ ] **Step 3: Final commit**

```bash
git add -A
git commit -m "feat: complete reflect skill with WIRED template and testing"
```

---

## Execution Handoff

After completing all tasks above:

1. **Symlink the skill** to the opencode skills directory:
   ```bash
   ln -s /Users/whodevil/src/skills/reflect ~/.config/opencode/skills/reflect
   ```

2. **Test end-to-end** by typing `/reflect` in an opencode session.

3. **Verify** the browser opens with a properly styled WIRED retrospective report.

Plan complete and saved to `docs/superpowers/plans/2026-04-24-reflect-skill-implementation.md`. Ready to execute?
