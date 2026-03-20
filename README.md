# Kagent Slack Bot with HITL Approval Support

Shoutout: The original slackbot is from @Matcham89 located at https://github.com/Matcham89/slackbot-agent. All credit goes to him.

A Slack bot that connects your workspace to Kagent's Kubernetes AI agents using the A2A (Agent2Agent) protocol. Supports **Human-in-the-Loop (HITL)** approvals — when an agent hits a `requireApproval` tool (like `k8s_create_resource` or `create_pull_request`), the bot posts interactive Approve / Deny buttons directly in Slack.

![slackbot kagent sync](./image/slackkagent.gif)

## Features

- **A2A protocol** — JSON-RPC 2.0 `message/send` with conversation context
- **HITL approvals** — Block Kit Approve / Deny buttons for `requireApproval` tools
- **ask_user support** — Choice buttons or free-text replies for agent questions
- **Thread context** — Multi-turn conversations via Slack threads
- **Text-based approve/deny** — Type "approve" or "deny" in threads as an alternative to buttons
- **Message chunking** — Long responses split at 3000 chars
- **Health probes** — Writes `/tmp/bot-healthy` for K8s liveness/readiness

## Prerequisites

- [kagent.dev](https://kagent.dev) deployed on your K8s cluster
- A Slack app with Socket Mode enabled

## Slack App Setup

### Required OAuth Scopes

- `app_mentions:read` — Detect @mentions
- `chat:write` — Post messages
- `channels:history` — Read channel history

### Required Settings

- **Socket Mode** — Enabled with `connections:write` scope
- **Event Subscriptions** — Enabled with `app_mention` and `message.channels` events
- **Interactivity** — Enabled (required for HITL button clicks)
- **Request URL** — Leave blank (Socket Mode doesn't need it)

## Configuration

```bash
SLACK_BOT_TOKEN=xoxb-your-bot-token
SLACK_APP_TOKEN=xapp-your-app-token

# Option 1: Full URL
KAGENT_A2A_URL=http://kagent-controller.kagent.svc.cluster.local:8083/api/a2a/kagent/k8s-agent

# Option 2: Separate components
KAGENT_BASE_URL=http://kagent-controller.kagent.svc.cluster.local:8083
KAGENT_NAMESPACE=kagent
KAGENT_AGENT_NAME=slackbot-k8s-agent
```

## Deploy

### 1. Create secrets

```bash
kubectl create secret generic slack-credentials -n kagent \
  --from-literal=SLACK_BOT_TOKEN=$SLACK_BOT_TOKEN \
  --from-literal=SLACK_APP_TOKEN=$SLACK_APP_TOKEN
```

### 2. Deploy the bot

```bash
kubectl apply -f slackbot-deployment.yaml
```

### 3. Deploy MCP and Agent

```bash
kubectl apply -f mcp-deployment.yaml
kubectl apply -f agent-deployment.yaml
```

## HITL Approval Flow

```
1. User @mentions the bot with a request
2. Bot sends message/send to kagent A2A endpoint
3. Agent decides to call a requireApproval tool
4. Kagent returns status.state = "input-required"
5. Bot posts Slack message with Approve / Deny buttons
6. User clicks Approve or Deny (or types "approve"/"deny" in thread)
7. Bot sends DataPart with decision_type to kagent
8. Agent resumes and bot streams the continued response
```

## Usage

```
# Invite the bot to a channel
/invite @kagent

# Ask questions — responses appear in threads
@kagent list all namespaces
@kagent create a deployment for nginx in staging

# When approval is needed, click the buttons or reply:
approve / deny
```

## Build

```bash
cd slackbot
# Set your registry (default: docker.io/sebbycorp)
export REGISTRY=docker.io/youruser
./build.sh
```
