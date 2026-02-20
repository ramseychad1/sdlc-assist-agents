# SOP: Vertex AI ADK Agent Setup
## From GCP Project Creation to Cloud Deployment

**Purpose:** Step-by-step guide for setting up a Google Vertex AI Agent using the Agent Development Kit (ADK) on a Mac, including local testing and cloud deployment to Vertex AI Agent Engine.  
**Stack Context:** Spring Boot 3.4 (Java 21), Angular 21, PostgreSQL — AI layer being added via Vertex AI ADK  
**Last Validated:** February 2026 | ADK Version: 1.25.0

---

## Can I Do This All Through the GCP Console?

Short answer: **partially, but not fully.**

| Task | Console | Terminal/ADK |
|---|---|---|
| Create GCP project | ✅ Yes | ✅ Yes |
| Enable APIs | ✅ Yes | ✅ Yes |
| Create service accounts | ✅ Yes | ✅ Yes |
| Write agent.py code | No | ✅ Required |
| Local testing with adk web | No | ✅ Required |
| Deploy agent to cloud | No | ✅ Required |
| View deployed agents | ✅ Yes | ✅ Yes |
| Monitor usage and logs | ✅ Yes | ✅ Yes |
| Test deployed agent | ✅ Yes (built-in chat UI) | ✅ Yes |

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
Follow the wizard - it will ask you to log in with your Google account and select a project. You can select any project for now; you will set the correct one in Phase 3.

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
4. Fill in:
   - Project name: sdlc-assist (or your preferred name)
   - Organization: select your org if applicable
5. Click Create and wait about 30 seconds
6. Select your new project in the top dropdown

### 3b - Link Billing

1. In the left nav go to Billing
2. Link your existing billing account to this project

NOTE: Without billing linked, API calls will fail even for free-tier operations.

### 3c - Enable Required APIs

Run these commands in your terminal:
```bash
gcloud config set project sdlc-assist
gcloud services enable aiplatform.googleapis.com
gcloud services enable dialogflow.googleapis.com
gcloud services enable storage.googleapis.com
```

Each takes about 30 seconds and returns "Operation finished successfully" when done.

Verify your project is set correctly:
```bash
gcloud config get project
```
Should return your project ID (e.g., sdlc-assist).

### 3d - Create a Service Account

This is the identity your Spring Boot backend uses to authenticate to GCP.

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

NOTE: The downloaded .json file is a credential key. Never commit it to Git. Store it somewhere safe and note the full path - you will need it in Phase 5.

**Verify the service account was created:**
```bash
gcloud iam service-accounts list
```
You should see sdlc-assist-backend in the list.

---

## Phase 4: Install ADK

```bash
pip install google-adk
```

Install the extensions package:
```bash
pip install 'google-adk[extensions]'
```

NOTE: Use single quotes around google-adk[extensions] - zsh will throw an error without them.

Verify ADK installed correctly:
```bash
adk --version
```
Should return a version number (e.g., adk, version 1.25.0).

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

NOTE: Folder and agent names must use underscores only - no hyphens. ADK will reject names like prd-generation-agent. Use prd_generation_agent instead.

### 5b - Create the .env file

Find the full path to your JSON key file:
```bash
find ~ -name "sdlc-assist-key.json" 2>/dev/null
```

Then write the .env file using the terminal (avoids smart quote issues from copy-paste):
```bash
cat > .env << 'EOF'
GOOGLE_CLOUD_PROJECT=sdlc-assist
GOOGLE_CLOUD_LOCATION=us-central1
GOOGLE_APPLICATION_CREDENTIALS=/Users/YOURUSERNAME/path/to/sdlc-assist-key.json
GOOGLE_GENAI_USE_VERTEXAI=1
EOF
```
Replace the credentials path with your actual path from the find command above.

Verify it looks correct:
```bash
cat .env
```

You should see exactly 4 lines, each on its own line with no content jammed together.

NOTE: GOOGLE_GENAI_USE_VERTEXAI=1 is critical - without it, ADK routes to the consumer Gemini API instead of Vertex AI and will fail with a missing API key error.

### 5c - Create the agent file

Use the terminal method below - it avoids all copy-paste character issues:

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

If using VS Code instead:
```bash
code prd_generation_agent/agent.py
```
Paste the code above into VS Code and save with Cmd+S.

NOTE: Do NOT use TextEdit or copy-paste from a browser into Sublime - both can insert smart quote characters that Python cannot parse. If you must use Sublime, use the terminal cat method above instead.

---

## Phase 6: Run and Test Locally

### 6a - Export environment variables

Before running ADK, export the variables directly in your terminal session:
```bash
export GOOGLE_CLOUD_PROJECT=sdlc-assist
export GOOGLE_CLOUD_LOCATION=us-central1
export GOOGLE_APPLICATION_CREDENTIALS=/Users/YOURUSERNAME/path/to/sdlc-assist-key.json
```

### 6b - Start the ADK web server

Make sure you are in the sdlc-assist-agents folder (one level ABOVE the agent folder):
```bash
cd ~/sdlc-assist-agents
adk web
```

You should see:
```
+-----------------------------------------------------------------------------+
| ADK Web Server started                                                      |
| For local testing, access at http://127.0.0.1:8000.                        |
+-----------------------------------------------------------------------------+
```

### 6c - Test in browser

1. Open http://127.0.0.1:8000 in your browser
2. You should see the ADK Dev UI with prd_generation_agent in the dropdown
3. A Session ID will be auto-generated at the top
4. Type a test message in the chat input on the right side:

```
Generate a PRD for a mobile app that helps users track their daily water intake.
```

5. The agent should respond with a structured PRD in Markdown format

NOTE: The agent is running entirely on your local Mac at this point. It disappears when you hit Ctrl+C. Nothing has been deployed to GCP yet.

---

## Phase 7: Deploy to Vertex AI Agent Engine

This is the step that pushes your agent to Google Cloud where it runs 24/7 and is accessible via an HTTPS endpoint.

### 7a - Run the deploy command

From your sdlc-assist-agents folder:
```bash
adk deploy agent_engine prd_generation_agent \
  --project=sdlc-assist \
  --region=us-central1
```

This takes 2-5 minutes. You will see output like:
```
Staging all files...
Copying agent source code complete.
Resolving files and dependencies...
Initializing Vertex AI...
Vertex AI initialized.
Deploying to agent engine...
Created agent engine: projects/127687886386/locations/us-central1/reasoningEngines/454356687504015360
Cleaning up the temp folder...
```

NOTE: You must pass --project and --region explicitly. Without them the deploy fails with "No project/region or api_key provided."

### 7b - Save your Resource ID

The long number at the end of the reasoningEngines/ path is your agent's Resource ID. Save it - you will need it when calling the agent from Spring Boot:

```
projects/127687886386/locations/us-central1/reasoningEngines/454356687504015360
                                                              ^^^^^^^^^^^^^^^^^^
                                                              This is your Resource ID
```

Store this in your application.yml or environment config for your Spring Boot backend.

### 7c - Verify in the GCP Console

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Make sure sdlc-assist is selected in the top project dropdown
3. Search for Vertex AI in the search bar
4. In the left nav look for Agent Engine
5. Or navigate directly to:
```
https://console.cloud.google.com/vertex-ai/agents?project=sdlc-assist
```

You should see prd_generation_agent listed with a green active status. The console also shows usage metrics, logs, and a built-in chat UI for testing the deployed agent directly.

---

## Phase 8: GitHub Setup

NOTE: This can also be done through your IDE (IntelliJ, VS Code, etc.) if you prefer a UI over terminal commands.

### 8a - Create .gitignore FIRST

Before pushing anything, protect your secrets:
```bash
cat > ~/sdlc-assist-agents/.gitignore << 'EOF'
.env
*.json
.adk/
__pycache__/
EOF
```

This blocks:
- .env - contains your project ID and credentials path
- *.json - blocks your service account key file
- .adk/ - local session database, not needed in source control
- __pycache__/ - Python compiled files, never commit these

CRITICAL: Never push your .env or .json key file to GitHub. Exposed GCP credentials can result in unauthorized usage and unexpected charges.

### 8b - Initialize and push

```bash
cd ~/sdlc-assist-agents
git init
git add .
git commit -m "Initial commit: prd_generation_agent"
git remote add origin https://github.com/YOURUSERNAME/sdlc-assist-agents.git
git push -u origin main
```

Safe files to commit:
- prd_generation_agent/agent.py
- prd_generation_agent/__init__.py
- .gitignore
- This SOP .md file

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| invalid character U+2014 in agent.py | Smart quote from copy-paste | Rewrite file using terminal cat method |
| Found invalid agent name | Hyphens in agent name or folder name | Rename folder and update name= in agent.py to use underscores |
| Missing key inputs argument, provide api_key | GOOGLE_GENAI_USE_VERTEXAI=1 missing from .env | Add that line to .env and restart |
| Model vertexai/gemini-2.0-flash not found | litellm prefix syntax not supported | Use model="gemini-2.0-flash" with GOOGLE_GENAI_USE_VERTEXAI=1 in .env |
| no such file or directory in terminal | Wrong working directory | Check pwd - run adk web from sdlc-assist-agents not from inside the agent folder |
| .env content jammed on one line | echo >> did not add a newline | Rewrite the whole file using the cat > .env EOF method |
| adk: command not found | Python PATH not set | Reinstall Python and check Add to PATH |
| No project/region or api_key provided | Missing flags on deploy command | Always pass --project and --region to adk deploy |
| zsh: no matches found: google-adk[extensions] | zsh interprets brackets | Use single quotes: pip install 'google-adk[extensions]' |

---

## Final Folder Structure

When everything is set up correctly, your structure should look like this:

```
sdlc-assist-agents/
├── .env                          <- 4 environment variables (NOT in Git)
├── .gitignore                    <- blocks .env, *.json, .adk/, __pycache__/
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
| Session state (local) | prd_generation_agent/.adk/session.db | Auto, during adk web |
| Session state (cloud) | Vertex AI managed infrastructure | Auto, after deploy |
| Agent instructions | Inside agent.py locally / Agent Engine in cloud | Your code |

---

## Key Concepts for Claude Context Handoffs

- **ADK (Agent Development Kit):** Google's open-source framework for building and running AI agents locally and in production
- **Agent Engine:** The Vertex AI managed runtime where agents are deployed for production - runs 24/7, scales automatically, accessible via HTTPS
- **adk web:** Local development server only - agent runs on your Mac, dies when terminal closes, nothing in the cloud
- **adk deploy agent_engine:** The command that actually pushes the agent to Google Cloud
- **Resource ID:** The long number in the deploy output (reasoningEngines/454356687504015360) - this is how Spring Boot identifies which agent to call
- **Session ID:** Auto-generated UUID that maintains conversation state per user/project. Store this in your database (projects table) to allow users to resume sessions
- **GOOGLE_GENAI_USE_VERTEXAI=1:** Environment variable that tells ADK to route through Vertex AI instead of the consumer Gemini API
- **model="gemini-2.0-flash":** Model identifier when using Vertex AI routing via the env var - do NOT use vertexai/ prefix
- **root_agent:** The required variable name ADK looks for in agent.py - must be named exactly this
- **Folder naming:** ADK uses the folder name as the agent module name - must use underscores only, no hyphens

---

## Next Steps (Not Yet Completed)

1. **Spring Boot integration** - Wire the deployed Agent Engine endpoint and Resource ID into the Java backend using the Vertex AI Java SDK
2. **Session ID persistence** - Store Vertex AI session IDs in the projects table in PostgreSQL so users can resume sessions
3. **Hybrid SSE** - Update Spring Boot SSE endpoint to emit progress events while waiting for agent response
4. **Swap in real prompts** - Replace the placeholder instruction in agent.py with the full production prompt from planning-analysis.txt
5. **Additional agents** - Repeat Phases 5-7 for design_system_agent, prototype_agent, etc.
6. **GitHub push** - Push agent code to GitHub repository (Phase 8 above)
