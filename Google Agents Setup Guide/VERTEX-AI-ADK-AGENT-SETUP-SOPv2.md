# SOP: Vertex AI ADK Agent Setup
## From GCP Project Creation to Cloud Deployment

**Purpose:** Step-by-step guide for setting up a Google Vertex AI Agent using the Agent Development Kit (ADK) on a Mac, including local testing and cloud deployment to Vertex AI Agent Engine.  
**Stack Context:** Spring Boot 3.4 (Java 21), Angular 21, PostgreSQL - AI layer being added via Vertex AI ADK  
**Last Validated:** February 2026 | ADK Version: 1.25.0

---

## Can I Do This All Through the GCP Console?

Short answer: **partially, but not fully.**

| Task | Console | Terminal/ADK |
|---|---|---|
| Create GCP project | Yes | Yes |
| Enable APIs | Yes | Yes |
| Create service accounts | Yes | Yes |
| Write agent.py code | No | Required |
| Local testing with adk web | No | Required |
| Deploy agent to cloud | No | Required |
| View deployed agents | Yes | Yes |
| Monitor usage and logs | Yes | Yes |
| Test deployed agent | Yes (built-in chat UI) | Yes |

The console is a management and visibility layer. Agent development and deployment is code and terminal based.

---

## Prerequisites

Before starting, you need:
- A Google account with access to Google Cloud Console
- A Google Cloud billing account
- A Mac with Terminal access
- Sublime Text or VS Code installed (VS Code preferred for code editing)

---

## Phase 1: Python Installation

ADK requires Python for its CLI tools. This does not affect your Java development.

**Step 1 - Download Python**
Go to [python.org/downloads](https://python.org/downloads) and download Python 3.12.x.

**Step 2 - Run the installer**
CRITICAL: On the first installer screen, check "Add Python to PATH" before clicking Install Now. Missing this step breaks everything.

**Step 3 - Verify installation**
Open a NEW terminal window (must be new, not existing) and run:
```bash
python --version
pip --version
```
Both should return version numbers. If either fails, Python was not added to PATH - reinstall and check the box.

---

## Phase 2: Google Cloud CLI Installation

**Step 1 - Download and install**
Go to [cloud.google.com/sdk/docs/install](https://cloud.google.com/sdk/docs/install), download the installer for your OS, and run it.

**Step 2 - Initialize**
```bash
gcloud init
```
Follow the wizard - it will ask you to log in with your Google account and select a project.

**Step 3 - Set up Application Default Credentials**
```bash
gcloud auth application-default login
```
A browser window opens. Log in with your Google account. This allows local tools to authenticate to GCP automatically.

---

## Phase 3: GCP Project Setup

### 3a - Create the Project

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Click the project dropdown at the top of the page
3. Click New Project
4. Fill in Project name (e.g., sdlc-assist) and Organization if applicable
5. Click Create and wait about 30 seconds
6. Select your new project in the top dropdown

### 3b - Link Billing

Go to Billing in the left nav and link your existing billing account to this project.

NOTE: Without billing linked, API calls will fail even for free-tier operations.

### 3c - Enable Required APIs

```bash
gcloud config set project sdlc-assist
gcloud services enable aiplatform.googleapis.com
gcloud services enable dialogflow.googleapis.com
gcloud services enable storage.googleapis.com
```

Verify your project is set correctly:
```bash
gcloud config get project
```

### 3d - Create a Service Account

**Create the account:**
```bash
gcloud iam service-accounts create sdlc-assist-backend \
  --display-name="SDLC Assist Backend" \
  --description="Service account for SDLC Assist Spring Boot backend"
```

**Grant Vertex AI access:**
```bash
gcloud projects add-iam-policy-binding sdlc-assist \
  --member="serviceAccount:sdlc-assist-backend@sdlc-assist.iam.gserviceaccount.com" \
  --role="roles/aiplatform.user"
```

**Create and download the JSON key:**
```bash
gcloud iam service-accounts keys create ~/sdlc-assist-key.json \
  --iam-account=sdlc-assist-backend@sdlc-assist.iam.gserviceaccount.com
```

NOTE: Never commit the .json key file to Git. Store it somewhere safe outside any Git repository.

**Verify the service account was created:**
```bash
gcloud iam service-accounts list
```

---

## Phase 4: Install ADK

```bash
pip install google-adk
pip install 'google-adk[extensions]'
```

NOTE: Use single quotes around google-adk[extensions] - zsh will throw an error without them.

Verify:
```bash
adk --version
```

---

## Phase 5: Create Your Agent Project

### 5a - Create the folder structure

```bash
mkdir ~/sdlc-assist-agents
cd ~/sdlc-assist-agents
mkdir prd_generation_agent
touch prd_generation_agent/__init__.py
touch prd_generation_agent/agent.py
touch .env
```

NOTE: Folder and agent names must use underscores only - no hyphens. ADK will reject names like prd-generation-agent.

### 5b - Create the .env file

Find the full path to your JSON key file:
```bash
find ~ -name "sdlc-assist-key.json" 2>/dev/null
```

Write the .env file using the terminal:
```bash
cat > .env << 'EOF'
GOOGLE_CLOUD_PROJECT=sdlc-assist
GOOGLE_CLOUD_LOCATION=us-central1
GOOGLE_APPLICATION_CREDENTIALS=/Users/YOURUSERNAME/path/to/sdlc-assist-key.json
GOOGLE_GENAI_USE_VERTEXAI=1
EOF
```

Verify it looks correct (should be exactly 4 lines, nothing jammed together):
```bash
cat .env
```

NOTE: GOOGLE_GENAI_USE_VERTEXAI=1 is critical - without it ADK routes to consumer Gemini API and fails.

### 5c - Create the agent file

Use the terminal method to avoid copy-paste character issues:

```bash
cat > prd_generation_agent/agent.py << 'EOF'
from google.adk.agents import Agent

root_agent = Agent(
    name="prd_generation_agent",
    model="gemini-2.0-flash",
    description="Generates PRDs from uploaded project documents.",
    instruction="""
You are an expert Product Manager specializing in creating Product Requirements Documents.

When given source documents or project descriptions, you will:
1. Analyze all provided content thoroughly
2. Extract key requirements, goals, and constraints
3. Structure a professional PRD with these sections:
   - Executive Summary
   - Problem Statement
   - Goals and Success Metrics
   - User Personas
   - Functional Requirements
   - Non-Functional Requirements
   - Out of Scope
   - Open Questions

Be thorough and professional. Format output in clean Markdown.
""",
)
EOF
```

NOTE: Do NOT use TextEdit or copy-paste from a browser into Sublime - both can insert smart quote characters that Python cannot parse.

---

## Phase 6: Run and Test Locally

### 6a - Export environment variables

```bash
export GOOGLE_CLOUD_PROJECT=sdlc-assist
export GOOGLE_CLOUD_LOCATION=us-central1
export GOOGLE_APPLICATION_CREDENTIALS=/Users/YOURUSERNAME/path/to/sdlc-assist-key.json
```

### 6b - Start the ADK web server

IMPORTANT: Run from the sdlc-assist-agents folder, one level ABOVE the agent folder.

```bash
cd ~/sdlc-assist-agents
adk web
```

### 6c - Test in browser

1. Open http://127.0.0.1:8000
2. Confirm prd_generation_agent appears in the dropdown
3. Note the Session ID at the top - this is your conversation state identifier
4. Type a test message and confirm the agent responds

NOTE: At this point the agent is running on your Mac only. Nothing is deployed to GCP yet.

---

## Phase 7: Deploy to Vertex AI Agent Engine

### 7a - BEFORE deploying: check if agent already exists

Always check what is already deployed before running a deploy command to avoid creating duplicates.

**Query deployed agents via REST API:**
```bash
curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  "https://us-central1-aiplatform.googleapis.com/v1/projects/sdlc-assist/locations/us-central1/reasoningEngines" \
  | grep -o '"name": "[^"]*"'
```

This returns just the name lines, one per deployed agent. The Resource ID is the number at the end:
```
"name": "projects/127687886386/locations/us-central1/reasoningEngines/454356687504015360"
                                                                        ^^^^^^^^^^^^^^^^^^
                                                                        This is your Resource ID
```

**Alternatively, check the GCP Console:**
```
https://console.cloud.google.com/vertex-ai/agents?project=sdlc-assist
```

### 7b - First time deploy (creates new agent)

Only run this if no agent exists yet:
```bash
adk deploy agent_engine prd_generation_agent \
  --project=sdlc-assist \
  --region=us-central1
```

This takes 2-5 minutes. On success you will see:
```
Created agent engine: projects/127687886386/locations/us-central1/reasoningEngines/454356687504015360
```

NOTE: You must pass --project and --region explicitly. Without them the deploy fails.

### 7c - Save your Resource ID immediately

The number at the end of reasoningEngines/ is your Resource ID. Save it to your README:

```bash
cat > ~/sdlc-assist-agents/README.md << 'EOF'
# SDLC Assist Agents

## Deployed Resource IDs
prd_generation_agent: 454356687504015360

## Update Deploy Command (use this for all future deploys)
adk deploy agent_engine prd_generation_agent \
  --project=sdlc-assist \
  --region=us-central1 \
  --resource_id=454356687504015360
EOF
```

### 7d - All future deploys: update existing agent

After the first deploy, always use --resource_id to update instead of create:
```bash
adk deploy agent_engine prd_generation_agent \
  --project=sdlc-assist \
  --region=us-central1 \
  --resource_id=YOUR_RESOURCE_ID
```

### 7e - Verify in the GCP Console

Navigate to:
```
https://console.cloud.google.com/vertex-ai/agents?project=sdlc-assist
```

You should see prd_generation_agent listed with a green active status, usage metrics, logs, and a built-in chat UI for testing.

---

## Phase 8: GitHub Setup

NOTE: Can also be done through your IDE.

### 8a - Create .gitignore FIRST (before pushing anything)

```bash
cat > ~/sdlc-assist-agents/.gitignore << 'EOF'
.env
*.json
.adk/
__pycache__/
EOF
```

CRITICAL: Never push .env or .json key files to GitHub. Exposed GCP credentials result in unauthorized usage and unexpected charges.

### 8b - Initialize and push

```bash
cd ~/sdlc-assist-agents
git init
git add .
git commit -m "Initial commit: prd_generation_agent"
git remote add origin https://github.com/YOURUSERNAME/sdlc-assist-agents.git
git push -u origin main
```

Safe to commit: agent.py, __init__.py, .gitignore, README.md, this SOP file.

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| invalid character U+2014 in agent.py | Smart quote from copy-paste | Rewrite file using terminal cat method |
| Found invalid agent name | Hyphens in agent name or folder | Rename folder and update name= in agent.py to underscores |
| Missing key inputs argument, provide api_key | GOOGLE_GENAI_USE_VERTEXAI=1 missing | Add that line to .env and restart |
| Model vertexai/gemini-2.0-flash not found | litellm prefix syntax not supported | Use model="gemini-2.0-flash" with GOOGLE_GENAI_USE_VERTEXAI=1 in .env |
| no such file or directory in terminal | Wrong working directory | Run adk web from sdlc-assist-agents not from inside agent folder |
| .env content jammed on one line | echo >> did not add newline | Rewrite whole file using cat > .env EOF method |
| adk: command not found | Python PATH not set | Reinstall Python and check Add to PATH |
| No project/region or api_key provided | Missing flags on deploy | Always pass --project and --region to adk deploy |
| zsh: no matches found: google-adk[extensions] | zsh interprets brackets | Use single quotes: pip install 'google-adk[extensions]' |
| Accidentally created duplicate agents | Ran deploy without --resource_id | Delete duplicate with delete command (see cheat sheet), always query first |

---

## Final Folder Structure

```
sdlc-assist-agents/
├── .env                          <- 4 environment variables (NOT in Git)
├── .gitignore                    <- blocks .env, *.json, .adk/, __pycache__/
├── README.md                     <- stores your Resource IDs and deploy commands
└── prd_generation_agent/
    ├── __init__.py               <- empty file, required by Python
    ├── agent.py                  <- agent definition with name, model, instruction
    └── .adk/
        └── session.db            <- auto-created by ADK on first run (NOT in Git)
```

---

## What's Running Where

| Component | Where it runs | Started by |
|---|---|---|
| adk web local server | Your Mac only | adk web in terminal |
| Deployed agent | Vertex AI Agent Engine (GCP) | adk deploy agent_engine |
| Session state (local) | prd_generation_agent/.adk/session.db | Auto during adk web |
| Session state (cloud) | Vertex AI managed infrastructure | Auto after deploy |
| Agent instructions | Inside agent.py locally / Agent Engine in cloud | Your code |

---

## Key Concepts for Claude Context Handoffs

- **ADK (Agent Development Kit):** Google open-source framework for building and running AI agents locally and in production
- **Agent Engine:** Vertex AI managed runtime where agents are deployed - runs 24/7, scales automatically, accessible via HTTPS
- **ReasoningEngine:** The underlying GCP resource type for deployed agents - Agent Engine renamed from Reasoning Engine in 2025 but the API still uses reasoningEngines in URLs
- **adk web:** Local development server only - agent runs on your Mac, dies when terminal closes, nothing in the cloud
- **adk deploy agent_engine:** The command that pushes the agent to Google Cloud
- **Resource ID:** The long number in the deploy output (reasoningEngines/454356687504015360) - required for all future update deploys and for Spring Boot integration
- **Session ID:** Auto-generated UUID that maintains conversation state per user/project. Store in your database (projects table) to allow users to resume sessions
- **GOOGLE_GENAI_USE_VERTEXAI=1:** Environment variable that tells ADK to route through Vertex AI instead of consumer Gemini API
- **model="gemini-2.0-flash":** Use without any prefix when GOOGLE_GENAI_USE_VERTEXAI=1 is set
- **root_agent:** Required variable name ADK looks for in agent.py - must be named exactly this
- **Folder naming:** ADK uses folder name as agent module name - underscores only, no hyphens

---

## Next Steps (Not Yet Completed)

1. **Spring Boot integration** - Wire the deployed Agent Engine endpoint and Resource ID into the Java backend using the Vertex AI Java SDK
2. **Session ID persistence** - Store Vertex AI session IDs in the projects table in PostgreSQL
3. **Hybrid SSE** - Update Spring Boot SSE endpoint to emit progress events while waiting for agent response
4. **Swap in real prompts** - Replace placeholder instruction in agent.py with full production prompt from planning-analysis.txt
5. **Additional agents** - Repeat Phases 5-7 for design_system_agent, prototype_agent, etc.

---

---

# ADK & gcloud Command Cheat Sheet

## Querying Deployed Agents

**View in console (easiest for humans):**
```
https://console.cloud.google.com/vertex-ai/agents?project=sdlc-assist
```

**List all deployed agents - clean terminal output:**
```bash
curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  "https://us-central1-aiplatform.googleapis.com/v1/projects/sdlc-assist/locations/us-central1/reasoningEngines" \
  | grep -o '"name": "[^"]*"'
```
Outputs just the name lines, one per deployed agent:
```
"name": "projects/127687886386/locations/us-central1/reasoningEngines/454356687504015360"
"name": "projects/127687886386/locations/us-central1/reasoningEngines/2165724545904803840"
```
The Resource ID is the last number on each line.

**Get details of one specific agent - clean terminal output:**
```bash
curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  "https://us-central1-aiplatform.googleapis.com/v1/projects/sdlc-assist/locations/us-central1/reasoningEngines/YOUR_RESOURCE_ID" \
  | grep -o '"name": "[^"]*"'
```

---

## Deploying Agents

**First time - creates a brand new agent:**
```bash
adk deploy agent_engine prd_generation_agent \
  --project=sdlc-assist \
  --region=us-central1
```
NOTE: Every run without --resource_id creates a NEW agent. You will accumulate duplicates and unnecessary costs.

**All subsequent deploys - updates existing agent:**
```bash
adk deploy agent_engine prd_generation_agent \
  --project=sdlc-assist \
  --region=us-central1 \
  --resource_id=YOUR_RESOURCE_ID
```

**Delete an agent (remove duplicates or decommission):**
```bash
adk deploy agent_engine --delete \
  --project=sdlc-assist \
  --region=us-central1 \
  --resource_id=YOUR_RESOURCE_ID
```

**Golden rule: always query before you deploy.**

---

## Local Development

**Start local web UI (run from sdlc-assist-agents folder, NOT inside agent folder):**
```bash
cd ~/sdlc-assist-agents
adk web
```

**Start local web UI connected to cloud session state:**
```bash
adk web --session_service_uri=agentengine://YOUR_RESOURCE_ID
```
This lets you test locally but using the real cloud-managed session state instead of the local session.db file.

**Run agent in terminal without browser UI:**
```bash
adk run prd_generation_agent
```

**Start an API server instead of web UI (useful for testing Spring Boot integration locally):**
```bash
adk api_server prd_generation_agent
```
Starts a FastAPI server at http://127.0.0.1:8000 with REST endpoints your Spring Boot app can call during local development.

---

## GCP Project & Auth Commands

**Set active project:**
```bash
gcloud config set project sdlc-assist
```

**Confirm active project:**
```bash
gcloud config get project
```

**List all your GCP projects:**
```bash
gcloud projects list
```

**List service accounts:**
```bash
gcloud iam service-accounts list
```

**Refresh authentication (if you get auth errors):**
```bash
gcloud auth application-default login
```

**Print current access token (used in curl commands above):**
```bash
gcloud auth print-access-token
```

**List enabled APIs:**
```bash
gcloud services list --enabled
```

---

## ADK Help

**See all ADK commands:**
```bash
adk --help
```

**See options for a specific command:**
```bash
adk deploy --help
adk web --help
adk run --help
```

---

## Deploy Workflow Decision Tree

```
Are you deploying for the first time?
    YES -> adk deploy agent_engine prd_generation_agent --project=sdlc-assist --region=us-central1
           Save the Resource ID to README.md immediately

    NO  -> Do you know your Resource ID?
               YES -> adk deploy agent_engine prd_generation_agent --project=... --region=... --resource_id=YOUR_ID
               NO  -> Query it first:
                      curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" \
                        "https://us-central1-aiplatform.googleapis.com/v1/projects/sdlc-assist/locations/us-central1/reasoningEngines" \
                        | grep -o '"name": "[^"]*"'
                      Then deploy with --resource_id
```

---

## SDLC Assist Specific Resource IDs

Store your actual values here and keep this file in your README.md too:

```
Project ID:        sdlc-assist
Project Number:    127687886386
Region:            us-central1

Deployed Agents:
  prd_generation_agent    Resource ID: 454356687504015360
  design_system_agent     Resource ID: (not yet deployed)
  prototype_agent         Resource ID: (not yet deployed)
  tech_design_agent       Resource ID: (not yet deployed)
```
