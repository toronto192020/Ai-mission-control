# Tapestry: The Quiet Guardian Automation Playbook

This playbook provides the comprehensive, copy-pasteable implementation guide for the **Tapestry** automation system. Tapestry serves as the orchestrator for the "Quiet Guardian" architecture, unifying intent capture from edge devices (Rabbit R1, OpenClaw CLI) and dispatching tasks to specialized autonomous agents (Skyvern, MultiOn, UFO³) via an n8n backbone. It is designed to be highly resilient, context-aware, and secure.

## 1. System Architecture Overview

The Tapestry system is composed of four distinct layers, working in concert to handle complex, multi-step workflows autonomously.

- **Interface Layer (Intent Capture):** The Rabbit R1 and OpenClaw CLI act as the primary interfaces. They capture natural language intent (e.g., voice commands or text) and format it into a standardized JSON payload.
- **Orchestration Layer (n8n & Tailscale):** The core routing engine is n8n, exposed securely to the internet via a Tailscale Funnel. It receives the structured intent via a single `Tapestry Webhook` endpoint, evaluates the request, and branches the workflow to the appropriate agent.
- **Agent Execution Layer:**
  - **UFO³:** Deployed for macOS desktop automation, handling local file parsing, legacy application interaction, and unstructured document processing (e.g., extracting data from BlueCare PDFs or Excel spreadsheets).
  - **MultiOn:** Utilized for navigating authenticated web portals that require complex session management and interaction (e.g., SPER, Centrelink).
  - **Skyvern:** Tasked with autonomous browser workflows, particularly form filling, status checks, and straightforward transactions (e.g., TMR rego checks).
- **Observability Layer (UniFi Site Manager):** The UniFi Gateway Ultra provides continuous network health metrics via the Site Manager API. This ensures n8n only dispatches web-dependent tasks when the internet connection is stable, preventing task failures due to "sick" networks.

## 2. Infrastructure Setup: Tailscale & OpenClaw

The first step is establishing a secure, publicly accessible endpoint for the n8n orchestrator without exposing internal ports directly to the internet.

### 2.1. Exposing n8n via Tailscale Funnel

Run these commands on the host machine running n8n (e.g., the MSI Cubi NUC). Ensure n8n is actively listening on `0.0.0.0:5678` or `localhost:5678`.

1. Enable the Funnel feature in your Tailscale admin console (this requires configuring HTTPS certificates and Access Control Lists).
2. Execute the Funnel command:

```bash
# Expose port 5678 securely
tailscale funnel 5678

# Alternatively, to run it in the background on port 443 (if permitted by ACLs):
# tailscale funnel --https=443 5678
```

This command will output a public URL, which will look similar to:
`https://andrewsanders-tailXXXX.ts.net/`

Your **Tapestry Webhook URL** will append the specific n8n path:
`https://andrewsanders-tailXXXX.ts.net/webhook/tapestry`

### 2.2. Configuring OpenClaw CLI

Configure the OpenClaw host to route intents to the new Tapestry Webhook. Edit the OpenClaw configuration file located at `~/.openclaw/config.yaml`.

```yaml
# ~/.openclaw/config.yaml
gateway:
  mode: remote
  listen_addr: 0.0.0.0
  port: 18789
  # Replace with your Tailscale Funnel URL or internal IP if running locally
  orchestrator_url: "https://andrewsanders-tailXXXX.ts.net/webhook/tapestry"

skills:
  - name: "Situational_Awareness"
    path: "~/.openclaw/skills/situational_awareness.md"
  - name: "Tapestry_Orchestrator"
    path: "~/.openclaw/skills/tapestry_orchestrator.md"
```

After saving the configuration, restart the OpenClaw service to apply the changes:

```bash
openclaw restart
```

## 3. n8n Orchestration: Wiring the Agents

The n8n platform acts as the brain, receiving the standardized JSON payload and directing traffic based on the `intent` field.

### 3.1. The Webhook Entry Point

1. Open the n8n GUI and create a new workflow named **"Tapestry Orchestrator"**.
2. Add a **Webhook** node as the trigger:
   - **Path:** `tapestry`
   - **Method:** `POST`
   - **Respond:** Set to "Immediately" (returning a 202 Accepted) or after the branch selection completes, depending on whether you need synchronous feedback.

### 3.2. Routing Logic (Switch Node)

Add a **Switch** node (or an **If** node) immediately following the Webhook. This node evaluates `body.intent` to determine the execution path.

- **Condition 1 (Skyvern):** If `body.intent` contains keywords like "rego", "registration", or "TMR", branch to the Skyvern execution path.
- **Condition 2 (MultiOn):** If `body.intent` contains "SPER", "enforcement order", or "fine", branch to the MultiOn execution path.
- **Condition 3 (UFO³):** If `body.intent` contains "BlueCare", "Care Schedule", or "PDF", branch to the UFO³ execution path.

### 3.3. Agent Configurations

#### Skyvern (Autonomous Browsing)

Skyvern handles straightforward web tasks.

1. Install the `n8n-nodes-skyvern` community node via the n8n UI.
2. Configure the node with your Skyvern API key.
3. Use the **"Dispatch Task"** operation.
4. Pass instructions dynamically based on the payload. For example:
   > "Go to the TMR Queensland rego portal, log in with saved credentials, check the registration status for plate `{{ $json.body.parameters.plate }}`, and return the status text."

#### MultiOn (Authenticated Portals)

MultiOn manages complex, authenticated sessions.

1. Add an **HTTP Request** node.
2. Configure it to POST to the MultiOn API endpoint (`https://api.multion.ai/v1/sessions`).
3. Include your MultiOn API key in the headers.
4. Pass the task description in the body:
   > "Log into the SPER portal, navigate to the enforcement orders section, and return the latest enforcement order number and current balance."

#### UFO³ (macOS Desktop Automation)

UFO³ interacts with the local operating system and files.

1. Add an **Execute Command** node (running on the n8n host, or via SSH if UFO³ is on a separate Mac).
2. Execute the UFO CLI command, passing the required instructions:
   ```bash
   ufo run "Open the latest BlueCare PDF in the Downloads folder, detect the Amygdala routine schedule, and summarize it into concise bullet points."
   ```
3. Ensure the node captures and returns the standard output (stdout) containing the summary.

### 3.4. Final Notification

Conclude each branch with a notification node (e.g., Email, Telegram, or a push notification service) to deliver the result of the agent's execution back to the user.

## 4. Observability: UniFi Network Health

To prevent tasks from failing due to connectivity issues, Tapestry integrates with the UniFi Gateway Ultra to monitor network health.

1. Enable **UniFi Site Manager** and generate an API key via your Ubiquiti account.
2. Create a separate n8n workflow that runs on a schedule (e.g., every 5 minutes using a **Schedule Trigger** node).
3. Use an **HTTP Request** node to query the UniFi Site Manager API for Internet Health Metrics.
4. Evaluate the response. If the network health is poor or disconnected, update a global variable or database flag (e.g., `WAN_STATUS = sick`).
5. In the main Tapestry Orchestrator workflow, add an **If** node before dispatching to Skyvern or MultiOn to check `WAN_STATUS`. If the network is "sick", pause the workflow or notify the user that the task is queued until connectivity is restored.

## 5. Rabbit R1 Integration Pattern

The Rabbit R1 device provides the voice-activated edge interface for Tapestry.

1. Access the **Rabbit Hole Teach Mode**.
2. Record a new action:
   - Open the browser.
   - Navigate to your Tailscale Funnel URL (`https://andrewsanders-tailXXXX.ts.net/webhook/tapestry`).
   - Configure it to send a POST request containing the recognized intent in the standardized JSON format.
3. Associate this action with a specific voice phrase, such as: **"Command Center: [Intent]"**.
4. When you speak to the R1, it will transcribe the intent, package it into the JSON payload, and fire the webhook.

**Recommended Initial Test Flow:**
Start with a read-only task to verify the pipeline.
- **Voice Command:** "Command Center: Check my TMR rego status and report the result."
- **Execution:** The R1 sends the payload, n8n routes it to Skyvern, Skyvern checks the TMR website without making payments, and n8n emails the result.

---
*The Tapestry system is now ready to orchestrate the Quiet Guardian architecture.*
