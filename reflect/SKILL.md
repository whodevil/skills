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
