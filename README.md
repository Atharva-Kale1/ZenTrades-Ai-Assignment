# Retell Agent Setup (Manual Import)

If the Retell API is not available on the free tier, the generated Agent Draft Spec JSON can be manually imported into the Retell UI.

The pipeline automatically generates an agent configuration file at:
`outputs/accounts/<account_id>/v1/agent_spec.json`

or after onboarding updates:
`outputs/accounts/<account_id>/v2/agent_spec.json`

## Manual Steps to Create the Agent in Retell

1. **Create a Retell account**
   - Go to: [https://retellai.com](https://retellai.com)
   - Sign in or create a free account.

2. **Open the Agents Dashboard**
   - Navigate to: **Dashboard → Agents**
   - Click: **Create New Agent**

3. **Configure the Agent**
   - Fill the fields using values from the generated `agent_spec.json` file.
   - **Agent Name**
     - Copy: `agent_name`
     - Example: *Seattle Plumbing Voice Assistant*
   - **Voice Style**
     - Select a basic voice that matches: `voice_style`
     - Example: *professional-friendly*
   - **System Prompt**
     - Copy the entire value from `system_prompt` and paste it into the Agent System Prompt field.
   - **Key Variables**
     - Add the values from `key_variables` (e.g., `timezone`, `business_hours`, `office_address`, `emergency_routing`).
     - These can be added to the prompt or stored in Retell variables if available.
   - **Call Transfer Protocol**
     - Use the values from `call_transfer_protocol` to configure: `retry_limit`, `timeout`, and `escalation order`.
   - **Fallback Protocol**
     - Copy the text from `fallback_protocol` and paste it into the fallback handling section of the agent.
     - Example: *If transfer fails reassure the caller someone will call shortly.*
   - **Tool Invocation Placeholders**
     - The following placeholders are included in the spec and represent actions that would normally trigger backend tools. These should not be mentioned to the caller.
     - Example placeholders: `transfer_call`, `create_callback_request`, `lookup_customer`
     - These can be implemented later using backend integrations.

4. **Save the Agent**
   - Click **Save Agent**. The agent will now be ready to handle calls.

## Versioning Behavior

When onboarding updates are processed, the pipeline generates:
`outputs/accounts/<account_id>/v2/agent_spec.json`

**To update the Retell agent:**
1. Open the existing agent in the Retell dashboard.
2. Replace the system prompt and settings with values from the v2 spec.
3. Save the updated agent.

---

# Zero-Cost Automation Pipeline

**Demo Call → Retell Agent Draft → Onboarding Update → Agent Revision**

This project builds an automated workflow that converts customer demo calls into Retell voice agents, then updates those agents after onboarding calls.

The system is built using n8n automation workflows, runs locally, and requires zero paid APIs or services.

## Architecture Overview

The system contains two pipelines:
- **Pipeline A:** Demo Call → Account Memo → Retell Agent Draft
- **Pipeline B:** Onboarding Call → Memo Patch → Agent Update → Changelog

All generated assets are stored locally in a versioned account directory structure.

## Workflow Architecture

### Pipeline A — Demo Call → Preliminary Agent

**Purpose:** Convert a demo call transcript into a structured Account Memo JSON and a Retell Agent Draft Spec.

**Flow:**
```
Demo Call Transcript
        ↓
Read Files from Disk
        ↓
Analyze Document (Gemini)
        ↓
Code Node (Generate Account Memo JSON)
        ↓
Generate Retell Agent Spec
        ↓
Save memo.json
        ↓
Save agent_spec.json
```

**Output:** `outputs/accounts/<account_id>/v1/`
- `memo.json`
- `agent_spec.json`

**Example:** `outputs/accounts/ACC-seattle-plumbing/`
```
v1/
  memo.json
  agent_spec.json
```

### Pipeline B — Onboarding → Agent Modification

**Purpose:** Update the previously generated agent configuration using onboarding call data.

**Flow:**
```
Onboarding Transcript
        ↓
Analyze Document (Gemini)
        ↓
Load Existing Memo (v1)
        ↓
Merge Old + New Data
        ↓
Patch + Diff Generator
        ↓
Generate Updated Memo
        ↓
Generate Updated Agent Spec
        ↓
Save v2 Outputs
        ↓
Save Change Log
```

**Output:** `outputs/accounts/<account_id>/`
- `v1/`
  - `memo.json`
  - `agent_spec.json`
- `v2/`
  - `memo.json`
  - `agent_spec.json`
- `changes.json`

**Example:** `outputs/accounts/ACC-seattle-plumbing/`
```
v1/
  memo.json
  agent_spec.json
v2/
  memo.json
  agent_spec.json
changes.json
```

## Data Flow

```
Demo Calls (5 files)
        ↓
    Pipeline A
        ↓
Account Memo JSON
        ↓
Retell Agent Draft


Onboarding Calls (5 files)
        ↓
    Pipeline B
        ↓
   Memo Patch
        ↓
  Agent Spec v2
        ↓
   Change Log
```

## Setup Instructions

### 1. Install n8n

Run locally with Docker:
```bash
docker run -it --rm \
-p 5678:5678 \
-v ~/.n8n:/home/node/.n8n \
-v ./data:/home/node/.n8n-files \
n8nio/n8n
```
Open: `http://localhost:5678`

### 2. Import Workflows

Import the workflows from:
- `/workflows/pipeline-a.json`
- `/workflows/pipeline-b.json`

In n8n: **Workflows → Import**

### 3. Add Dataset

Place transcripts here:
- `/home/node/.n8n-files/input/demo_calls/`
- `/home/node/.n8n-files/input/onboarding_calls/`

Example:
- `input/demo_calls/demo1.txt`
- `input/demo_calls/demo2.txt`
- `input/onboarding_calls/onboarding1.txt`

### 4. Run Pipeline

Click: **Execute Workflow**
- Pipeline A processes demo calls.
- Pipeline B processes onboarding calls.

## Storage Structure

Outputs are stored in: `/home/node/.n8n-files/outputs/accounts/`

Structure:
```
accounts/
  ACC-company-1/
      v1/
      v2/
      changes.json
```

## Account Memo Schema

Each account memo includes:
- `account_id`
- `company_name`
- `business_hours`
- `office_address`
- `services_supported`
- `emergency_definition`
- `emergency_routing_rules`
- `non_emergency_routing_rules`
- `call_transfer_rules`
- `integration_constraints`
- `after_hours_flow_summary`
- `office_hours_flow_summary`
- `questions_or_unknowns`
- `notes`

## Retell Agent Draft Spec

Generated agent spec includes:
- `agent_name`
- `voice_style`
- `system_prompt`
- `key_variables`
- `tool_invocation_placeholders`
- `call_transfer_protocol`
- `fallback_protocol`
- `version`

Example:
```json
{
 "agent_name": "Seattle Plumbing Voice Assistant",
 "voice_style": "professional-friendly",
 "version": "v1"
}
```

## Versioning and Diff

Pipeline B generates:
- `v2 memo.json`
- `v2 agent_spec.json`
- `changes.json`

Example change log:
```json
{
 "business_hours": {
  "old": "9am-5pm",
  "new": "8am-6pm",
  "reason": "Updated during onboarding call"
 }
}
```

## LLM Usage (Zero Cost)

Extraction uses Google Gemini free-tier API.

API key location: [Google AI Studio](https://aistudio.google.com/app/apikey)
Add in n8n credentials: **Gemini API Key**

## Known Limitations

- Transcript quality affects extraction accuracy.
- Rule-based patching assumes flat JSON schema.
- Some nested updates may require deeper diff logic.
- Manual import of agent spec may be required if Retell API is unavailable.

## Improvements with Production Access

With production resources the system could include:
- Direct Retell API integration
- Structured database storage (Supabase/Postgres)
- Vector search for transcript understanding
- Automatic account lookup
- Dashboard UI for agent management
- Robust nested JSON diff engine
- Monitoring and logging

## Technologies Used

- n8n
- Node.js
- Google Gemini
- Local File Storage
- JSON Schema
