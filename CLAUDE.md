# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is the program management repository for **Symphony** — Yuno's Agentic AI Orchestration Platform. It serves as the planning, documentation, and architecture hub for the Symphony system, which consists of two sibling repositories:

- `symph-front-end` — Plain vanilla HTML/CSS/JS frontend (Vite dev server)
- `symph-back-end` — FastAPI backend running LangGraph agents

## Architecture

Symphony is a multi-agent AI platform. The tech stack decisions recorded here:

- **Frontend**: Vite + vanilla HTML/CSS/JS (planned migration path to React)
- **Backend**: FastAPI + LangGraph agents, running in a Python virtualenv named `symphony`
- **Agent system**: Agents are defined with a structured template covering system prompt, responsibilities, decision framework, quality checks, output format, edge cases, and persistent memory (`Memory.md`)

## Development Commands

### Frontend (`symph-front-end`)
```bash
npm run dev        # Start Vite dev server at http://localhost:5173/
# Quit: q + Enter, or Ctrl+C
```

### Backend (`symph-back-end`)
```bash
mkvirtualenv symphony   # One-time setup
workon symphony
pip install fastapi uvicorn

workon symphony
fastapi dev              # Start server at http://127.0.0.1:8000
                         # API docs at http://127.0.0.1:8000/docs
# Quit: Ctrl+C
```

## Agent Definition Structure

When designing or documenting agents for Symphony, use this template (from `doc/AgentDefinition.md`):

```
- Name:
- Description:
- Model: (Sonnet / Haiku / Inherit from Parent)
- Memory:

## Project Context
## Your responsibilities
## Decision Framework
## Quality Checks
## Output Format
## Edge cases to handle
## Persistent Agent Memory
## Memory.md
```



