# Reflect Skill Design

**Date:** 2026-04-24
**Topic:** Session Retrospective Analysis Skill (`retro`)
**Approach:** A — Prompt-Based Analysis with Embedded WIRED Template

---

## 1. Goal

Create an opencode skill that, when invoked via `/retro`, causes the agent to review the current session, analyze where its time and attention were spent, identify improvements to the repo that would have helped it perform better, and output the findings as a polished HTML document styled after the WIRED design system. The final HTML is opened in the default browser.

---

## 2. Architecture

### 2.1 Skill Location & Structure

The skill lives under the project root at:

```
retro/
  SKILL.md
```

It is a single, self-contained file. No companion scripts or external dependencies are required. The user will symlink this directory into `~/.config/opencode/skills/` after deployment.

### 2.2 Skill Type

This is a **technique skill** — a how-to guide that instructs the agent on how to perform session retrospective analysis and generate a styled report. It follows the `writing-skills` discipline: the SKILL.md itself is the "production code" and the test is a pressure scenario where a subagent simulates a conversation and is asked to reflect.

### 2.3 Execution Flow (Agent Perspective)

When the user types `/retro`, the agent:

1. **Receives the skill** via the standard opencode skill-loading mechanism.
2. **Reads the SKILL.md**, which contains:
   - Analysis instructions (what to scan for, how to categorize).
   - The complete WIRED HTML template (inline CSS, layout structure).
3. **Performs analysis** over the current conversation context.
4. **Fills the template** with computed findings.
5. **Writes the HTML** to `docs/retro/YYYY-MM-DD-HHMMSS-<prefix>.html`.
6. **Opens the browser** via a `bash` tool call (`open docs/retro/YYYY-MM-DD-HHMMSS-<prefix>.html` on macOS, `xdg-open docs/retro/YYYY-MM-DD-HHMMSS-<prefix>.html` on Linux).

The prefix is derived from the session's primary topic or project name (e.g., `skills`, `retro-skill`, `auth-bug-fix`). The timestamp is obtained via a `bash` tool call: `date +%Y-%m-%d-%H%M%S`.

---

## 3. Components

### 3.1 YAML Frontmatter

```yaml
---
name: retro
description: Use when the user wants to review the current session, analyze time spent, identify repo improvements, or generate a retrospective report.
---
```

Constraints:
- `name`: letters, numbers, hyphens only.
- `description`: starts with "Use when...", third person, no workflow summary, under 500 characters.

### 3.2 Analysis Instructions

The core of the skill is a structured prompt that tells the agent how to analyze the conversation. It is broken into three phases:

#### Phase 1: Scan & Categorize

The agent scans the conversation for tool calls and reasoning patterns, then buckets them into:

| Category | Signals |
|----------|---------|
| **Coding** | `Write`, `Edit`, `read` on source files; code generation blocks; refactoring commands |
| **Debugging** | `systematic-debugging` skill invocations; error logs; repeated tool calls on the same file; `bash` debugging commands |
| **Research** | `websearch`, `webfetch`, `codesearch`, `context7_query-docs`; reading docs or external resources |
| **Testing** | `test-driven-development` skill; test execution commands; assertion failures; `pytest`, `jest`, etc. |
| **Waiting** | `bash` timeouts; external process waits; sleep-like patterns |
| **Misc** | User greetings, clarifying questions, plan updates, meta-discussion |

Heuristics:
- Estimate tool-call counts per category by heuristically scanning the conversation transcript for tool-usage markers (e.g., "`Write`", "`Edit`", "`websearch`"). This is an approximate count, not a structured log parse.
- Layer qualitative "attention weight" on top by considering message length and reasoning depth.
- Detect task boundaries: a new user request following a completion message; a shift from planning to execution marked by `TodoWrite` or plan presentation; a cluster of tool calls following a reasoning block.
- Note task boundaries: user request → agent plan → execution → completion or abandonment.

#### Phase 2: Detect Friction Points

The agent looks for patterns that indicate the repo or workflow could be improved:

| Friction Type | Signals | Severity |
|---------------|---------|----------|
| **Missing Docs/Config** | Failed `read`/`glob` searches for conventions; repeated `grep` for "how do we do X?" | Medium |
| **Code Smells / Churn** | Repeated `edit` on the same file; very long files with many edits; absence of tests when test files exist | High |
| **Error Patterns** | Same error type recurring; tool failures followed by retries; `systematic-debugging` triggered multiple times | High |
| **Structural Gaps** | No `AGENTS.md`/`CLAUDE.md`; no test files; no `.gitignore`; inconsistent naming; missing CI config | Medium |
| **Dependency Issues** | Repeated `npm install`/`pip install`; missing lockfile; version conflicts | Low |

For each friction point, the agent records:
- What happened (concrete evidence from the conversation).
- Why it mattered (impact on velocity or correctness).
- Recommended fix (specific, actionable).

#### Phase 3: Summarize Success Metrics

- Tasks initiated vs. completed (with brief descriptions).
- Per-task effort estimate: approximate the agent's attention investment per task based on message count, reasoning depth, and tool calls within that task's boundary. Use qualitative labels: `Low` (few messages, simple), `Medium` (moderate back-and-forth), `High` (complex, many iterations, debugging required).
- Key decisions made and their rationale.
- Token/attention distribution across categories (qualitative estimate).

### 3.3 WIRED HTML Template

The skill embeds a complete, self-contained HTML document as a fenced code block. The agent copies this template, replaces placeholder sections with computed values, and writes the file.

#### Design Tokens (Embedded in `<style>`)

| Token | Value | Usage |
|-------|-------|-------|
| `background` | `#FDFCF8` | Page background (paper-white) |
| `text-primary` | `#1A1A1A` | Body text, headlines |
| `text-secondary` | `#595959` | Captions, metadata, secondary info |
| `accent` | `#0A6ECC` | Links, highlights, ink-blue accent only |
| `border` | `#E2E2E2` | Section dividers, table borders |
| `kicker` | `ui-monospace, SF Mono, Menlo, monospace` | Category tags, timestamps, labels |
| `display` | `Georgia, Times New Roman, Times, serif` | Headlines, section titles |
| `body` | `Georgia, Times New Roman, Times, serif` | Body copy, long-form text |

#### Layout Structure

The HTML is a single-column, broadsheet-dense layout:

1. **Masthead**: Session title ("Session Retrospective"), project name, timestamp. Kicker in mono above.
2. **Executive Summary**: 3–4 sentence overview. Large serif text.
3. **Attention Breakdown**: CSS-based horizontal bar chart showing estimated distribution derived from tool-call counts per category. Labeled as "Estimated Attention Distribution" — percentages are approximate (based on proportional tool-call counts), not precise measurements. Mono labels, ink-blue bars.
4. **Task Log**: Table of tasks (completed vs. abandoned), with effort estimates and outcomes.
5. **Friction Points & Recommendations**: Numbered list. Each item has a kicker severity badge (HIGH, MEDIUM, LOW), a headline, evidence, and a recommended fix.
6. **Key Decisions**: Bullet list of major architectural or technical choices, with rationale.
7. **Footer**: Generated-by line, mono timestamp.

#### HTML Escaping Requirement

The agent injects **two kinds of content** into the template, with different escaping rules:

1. **Structural markup fragments** (e.g., `<div class="bar-row">...`, `<tr>...`, `<li>...`): Injected **raw** (unescaped). These are generated by the agent and are trusted HTML.

2. **User-facing textual content** inside those fragments (tool outputs, error messages, file paths, code snippets, conversation quotes): Must be **HTML-escaped** before insertion:
   - `&` → `&amp;`
   - `<` → `&lt;`
   - `>` → `&gt;`
   - `"` → `&quot;`

This ensures the generated document is well-formed regardless of the raw conversation content being analyzed.

#### CSS Rules

- No `border-radius` anywhere; sharp, print-style corners.
- Hierarchy via borders (`1px solid #E2E2E2`), weight (`font-weight: 700` for headlines), and whitespace — no drop shadows.
- Generous line height (`1.6`) for body text; tight leading (`1.1`) for display headlines.
- Max content width `680px`, centered, with generous top/bottom padding (`64px`).

#### Complete Placeholder List

The embedded template uses these placeholders. Each is replaced with raw HTML fragments (not plain text) so the agent can inject structured content directly:

| Placeholder | Content Type | Surrounding Markup |
|-------------|--------------|-------------------|
| `{{PROJECT_NAME}}` | Plain text string (HTML-escaped) | Inside `<h1>` masthead |
| `{{TIMESTAMP}}` | Plain text string (HTML-escaped) | Inside `<time>` element |
| `{{EXECUTIVE_SUMMARY}}` | HTML paragraph(s) | Inside `<section id="summary">` |
| `{{ATTENTION_BREAKDOWN_BARS}}` | HTML `<div class="bar-row">` elements | Inside `<section id="attention">` |
| `{{TASK_LOG_ROWS}}` | HTML `<tr>` elements | Inside `<tbody>` of task table |
| `{{FRICTION_POINTS}}` | HTML `<li>` or `<article>` elements | Inside `<ol>` or `<section id="friction">` |
| `{{KEY_DECISIONS}}` | HTML `<li>` elements | Inside `<ul>` or `<section id="decisions">` |

Stripping rule: if a section-wrapped placeholder is not filled, its entire surrounding `<section>` (or equivalent wrapper like `<tbody>`, `<ol>`, `<ul>`) is removed. `{{PROJECT_NAME}}` and `{{TIMESTAMP}}` are never stripped — they always have fallback values (`Unknown Project` and the current timestamp, respectively).

#### Bar Chart Markup Structure

The attention breakdown uses nested `<div>` elements with inline percentage widths:

```html
<div class="bar-row">
  <span class="bar-label">Coding</span>
  <div class="bar-track">
    <div class="bar-fill" style="width: 45%;"></div>
  </div>
  <span class="bar-value">45%</span>
</div>
```

- `.bar-track`: full-width container with a subtle bottom border.
- `.bar-fill`: ink-blue background (`#0A6ECC`), height `8px`, no radius.
- `.bar-label` and `.bar-value`: monospace, `12px`, uppercase.

#### Project Name Source

The agent derives the project name from the current working directory basename (e.g., `skills` if in `/Users/whodevil/src/skills`). If the working directory is unavailable or is the home directory, fall back to `Unknown Project`.

#### Section Wrapper Convention

Every major section uses a consistent wrapper so unfilled placeholders can be stripped cleanly:

```html
<section id="summary">
  {{EXECUTIVE_SUMMARY}}
</section>

<section id="attention">
  {{ATTENTION_BREAKDOWN_BARS}}
</section>

<section id="tasks">
  <table>...</table>
  <tbody>
    {{TASK_LOG_ROWS}}
  </tbody>
</section>

<section id="friction">
  <ol>
    {{FRICTION_POINTS}}
  </ol>
</section>

<section id="decisions">
  <ul>
    {{KEY_DECISIONS}}
  </ul>
</section>
```

Stripping rule: if a placeholder is not filled, remove the entire `<section id="...">...</section>` block (or the `<tbody>` / `<ol>` / `<ul>` wrapper if the section has other static content).

#### Complete HTML Template (Embedded in SKILL.md)

The skill embeds the following complete, self-contained template as a fenced code block. The agent copies it verbatim, replaces all `{{PLACEHOLDER}}` instances with computed HTML fragments, and writes the result to disk.

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
      Generated by retro skill &middot; {{TIMESTAMP}}
    </footer>
  </div>
</body>
</html>
```

Multi-line content (code snippets, error logs) is wrapped in `<pre><code>` blocks with the escaping rules applied.

---

## 4. Data Flow

```
User types /retro
    ↓
Agent loads retro/SKILL.md
    ↓
Agent scans conversation context
    ├── Categorize tool calls and reasoning
    ├── Detect friction patterns
    └── Compute success metrics
    ↓
Agent fills WIRED HTML template
    ├── Replace title/timestamp placeholders
    ├── Inject category percentages into bar chart
    ├── Build task log rows
    ├── Build friction point list
    └── Build key decisions list
    ↓
Agent writes HTML to docs/retro/<prefix>-<timestamp>.html
    ↓
Agent opens browser via bash tool
    └── macOS: open <file>
    └── Linux: xdg-open <file>
```

---

## 5. Error Handling

| Scenario | Behavior |
|----------|----------|
| **Very short session** (< 5 messages) | Generate a minimal report: "Session too brief for meaningful analysis." Still styled, still opens browser. |
| **No tool calls found** | Report says "No tool activity detected — session may be purely conversational." Categories show 100% Misc. |
| **Browser open fails** | Report the file path in the conversation so the user can open it manually. |
| **Unsupported OS (Windows)** | Skip the browser-open step entirely. Report the file path in the conversation so the user can open it manually. |
| **Template variable missing** | Strip out the unfilled placeholder and its surrounding section wrapper (`<section>` or `<div>`); never crash the HTML generation. |
| **`docs/retro/` does not exist** | Create the directory via `bash` tool: `mkdir -p docs/retro` before writing. |
| **`docs/retro/` not writable** | Attempt to write to the current working directory as fallback. If that also fails, output the raw HTML in a conversation message block so the user can save it manually. |

---

## 6. Testing Strategy

This is a **technique skill**, so testing follows the `writing-skills` RED-GREEN-REFACTOR cycle.

### RED (Baseline)

1. Create a synthetic conversation log with known patterns:
   - 10 `Edit` calls (coding)
   - 3 `systematic-debugging` invocations (debugging)
   - 5 `websearch` calls (research)
   - 2 test failures (testing)
   - 1 missing `AGENTS.md` search (friction)
2. Dispatch a subagent WITHOUT the skill. Ask it to analyze the conversation and produce a report.
3. Observe baseline behavior: likely misses categories, produces plain text, no styling, no browser opening.

### GREEN (Write Skill)

4. Write the SKILL.md with the analysis instructions and WIRED template.
5. Re-run the subagent WITH the skill loaded. Verify:
   - Categories are correctly identified.
   - Friction points are surfaced.
   - HTML is generated and contains WIRED design tokens.

### REFACTOR (Close Loopholes)

6. Test edge cases (short session, no tool calls). Ensure graceful output.
7. Verify the subagent attempts to open the browser (or at least provides the path).
8. Check that the HTML renders correctly in a real browser (manual verification).

---

## 7. File Output

The generated HTML is written to:

```
docs/retro/YYYY-MM-DD-HHMMSS-<prefix>.html
```

The `docs/retro/` directory is created if it does not exist. The prefix is derived from the session's primary topic or project name (e.g., `skills`, `retro-skill`). This keeps reports organized and discoverable within the repository.

---

## 8. Dependencies

- **None.** The skill is entirely self-contained.
- The agent relies on its own conversation context (already loaded).
- Browser opening uses standard OS commands (`open` for macOS, `xdg-open` for Linux). **Windows is out of scope** for this version.
- Placeholder convention in the embedded template: use `{{SECTION_NAME}}` (e.g., `{{EXECUTIVE_SUMMARY}}`, `{{TASK_LOG_ROWS}}`) so the agent can reliably detect and replace sections. If a placeholder is not filled, it is stripped out during generation.

---

## 9. Approval Status

| Section | Approved |
|---------|----------|
| Architecture & Skill Structure | ✅ |
| Analysis Methodology | ✅ |
| HTML Output & WIRED Styling | ✅ |
| Testing Strategy | ✅ |

**Overall Design:** **APPROVED** by user on 2026-04-24.
