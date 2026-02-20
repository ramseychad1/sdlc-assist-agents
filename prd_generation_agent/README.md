# PRD Generation Agent

An AI agent that transforms raw project inputs — meeting transcripts, notes, and documents — into structured Product Requirements Documents (PRDs) ready for enterprise software engineering teams.

Built with [Google ADK](https://google.github.io/adk-docs/) and powered by Gemini 2.0 Flash.

## What it does

Upload source material (meeting notes, feature briefs, discovery docs) and the agent produces a fully structured PRD in markdown with:

- **Executive Summary** — product vision and MVP scope
- **Epics** — strategic goals grouped by user-facing capability
- **Stories** — user needs written in Given/When/Then format with Acceptance Criteria
- **Tasks** — concrete work items tagged by component (Frontend, Backend, Database, Testing, Design)

Every Epic, Story, and Task includes a Jira-ready Summary field (under 80 characters).

## Scope

The agent focuses strictly on **product and user-facing requirements**. It deliberately excludes:

- Infrastructure, DevOps, and CI/CD concerns
- Database schema and backend architecture decisions
- Effort estimates, story points, or timelines
- Sub-tasks (the dev team breaks tasks down further)

Ambiguous items are flagged inline with `{confirm with PM}`.

## Project structure

```
prd_generation_agent/
├── agent.py        # Agent definition and prompt
├── __init__.py
└── README.md
```

## Setup

1. Install the Google ADK:
   ```bash
   pip install google-adk
   ```

2. Run the agent with the ADK CLI or web UI:
   ```bash
   adk run prd_generation_agent
   # or
   adk web
   ```
