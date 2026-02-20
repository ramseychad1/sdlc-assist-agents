# SDLC Assist Agents

An AI-powered platform that automates the generation of product requirements documents, design systems, and UI screen prototypes for enterprise software engineering teams. Built on Google ADK and Gemini 2.0 Flash, running on Google Vertex AI.

## Overview

SDLC Assist orchestrates a pipeline of specialized agents that transform raw project inputs (meeting notes, requirements) into production-ready design and engineering deliverables.

```
Raw Input (notes, transcripts)
        │
        ▼
┌─────────────────────┐
│  PRD Generation     │  → Structured PRD (Epics, Stories, Tasks)
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│  Design System      │  → Design tokens + component specs (JSON/CSS)
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│  Screen Extraction  │  → Inventory of all required UI screens (JSON)
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│  Screen Generation  │  → High-fidelity HTML/CSS prototypes
└─────────────────────┘
```

## Agents

### PRD Generation Agent
Transforms raw project inputs into structured, enterprise-grade Product Requirements Documents.

- Generates Epics, Stories (Given/When/Then), and Tasks from source materials
- Produces Jira-ready summaries (≤80 characters) for every item
- Flags ambiguous requirements with `{confirm with PM}`
- Excludes infrastructure, DevOps, CI/CD, and effort estimates by design

### Design System Agent
Generates complete, production-ready design systems from a PRD and design template metadata.

- Customizes design tokens: colors, typography, spacing, border radius, shadows
- Generates specs for core UI components (buttons, inputs, cards, tables, navigation)
- Outputs self-contained HTML/CSS examples using CSS custom properties
- Ensures WCAG AA color contrast compliance

### Screen Extraction Agent
Analyzes a PRD and produces a complete inventory of distinct UI screens to be designed.

- Identifies all implied screens (empty states, confirmations, auth flows)
- Groups screens by Epic with complexity scoring (low / medium / high)
- Outputs structured JSON with screen type, user role, and design guidance notes

### Screen Generation Agent
Generates high-fidelity HTML/CSS screen prototypes from screen definitions and the design system.

- Produces complete, standalone HTML5 documents with inline CSS
- Applies design tokens as CSS custom properties
- Uses domain-specific, realistic content (no lorem ipsum)
- Follows screen-type-specific layouts: dashboard, list, detail, form, modal, auth, wizard, etc.
- Static prototypes only — no JavaScript logic or backend calls

## Tech Stack

| Layer | Technology |
|-------|-----------|
| AI Model | Gemini 2.0 Flash |
| Agent Framework | Google ADK (Agent Development Kit) |
| Agent Runtime | Google Vertex AI Agent Engine |
| Agent Language | Python 3.12 |
| Backend | Spring Boot 3.4 (Java 21) |
| Frontend | Angular 21 |
| Database | PostgreSQL |
| Infrastructure | Google Cloud Platform |

## Prerequisites

- Python 3.12+
- Google Cloud CLI (`gcloud`)
- A GCP project with the following APIs enabled:
  - Vertex AI (`aiplatform.googleapis.com`)
  - Dialogflow (`dialogflow.googleapis.com`)
  - Cloud Storage (`storage.googleapis.com`)
- A GCP service account with `roles/aiplatform.user`

## Setup

**1. Install Google ADK**
```bash
pip install google-adk
pip install 'google-adk[extensions]'   # single quotes required for zsh
adk --version
```

**2. Configure environment variables**

Copy `prd_generation_agent/.env.example` to `.env` in the project root and fill in your values:
```bash
GOOGLE_CLOUD_PROJECT=your-gcp-project-id
GOOGLE_CLOUD_LOCATION=us-central1
GOOGLE_APPLICATION_CREDENTIALS=/path/to/your-service-account-key.json
GOOGLE_GENAI_USE_VERTEXAI=1
```

> `GOOGLE_GENAI_USE_VERTEXAI=1` is required to route requests through Vertex AI instead of the consumer Gemini API.

**3. Run locally**
```bash
adk web
# Opens dev UI at http://127.0.0.1:8000
```

Or run a specific agent directly:
```bash
adk run prd_generation_agent
```

**4. Deploy to Vertex AI**
```bash
adk deploy agent_engine prd_generation_agent \
  --project=your-gcp-project-id \
  --region=us-central1
```

## Project Structure

```
sdlc-assist-agents/
├── .gitignore
├── prd_generation_agent/
│   ├── __init__.py
│   ├── agent.py
│   ├── .env.example
│   └── README.md
├── design_system_agent/
│   ├── __init__.py
│   └── agent.py
├── screen_extraction_agent/
│   ├── __init__.py
│   └── agent.py
├── screen_generation_agent/
│   ├── __init__.py
│   └── agent.py
├── Google Agents Setup Guide/
│   └── VERTEX-AI-ADK-AGENT-SETUP-SOP.md
└── technical-design-agents/
    └── # WORKSPACE RULES — Java Engineering
```

## Important Notes

- Agent folder names must use underscores (e.g., `prd_generation_agent`), not hyphens
- The root agent variable in each `agent.py` must be named exactly `root_agent`
- Never commit `.env` files or service account JSON keys — both are covered by `.gitignore`
- For detailed setup instructions, troubleshooting, and GCP configuration steps, see the [Setup Guide](Google%20Agents%20Setup%20Guide/VERTEX-AI-ADK-AGENT-SETUP-SOP.md)
