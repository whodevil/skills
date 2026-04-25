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
