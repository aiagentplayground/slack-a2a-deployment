# Let's deploy slackbot-agent 

Shoutout: The following is a slackbot is from @Matcham89 located at https://github.com/Matcham89/slackbot-agent. All credit goes to him.

A secure, production-ready Slack bot that connects your workspace to Kagent's Kubernetes AI agents using the A2A (Agent2Agent) protocol.

![slackbot kagent sync](./image/slackkagent.gif)

# Deploy kagent.dev 

Open-source tools for DevOps and platform engineers to build, deploy, and run AI-powered solutions in Kubernetes. From intelligent agents to MCP servers. Visit kagent.dev

## Prerequisites 
  * kagent.dev deployed 
  * access to your k8s environment

# Configuration of Environment variables
To deploy the bot we will require the following variables from their slack configuration.

```bash
SLACK_BOT_TOKEN=xoxb-your-bot-token
SLACK_APP_TOKEN=xapp-your-app-token
KAGENT_BASE_URL=http://kagent-controller.kagent.svc.cluster.local:8083
KAGENT_NAMESPACE=kagent
KAGENT_AGENT_NAME=slackbot-k8s-agent
```


# Deploy Slack App Setup

## Required OAuth Scopes:

  * app_mentions:read - Detect @mentions
  * chat:write - Post messages
  * channels:history - Read channel history

## Required Settings:

  * Socket Mode - Enabled with connections:write scope
  * Event Subscriptions - Enabled with app_mention event
  * Request URL - Leave blank (Socket Mode doesn't need it)

# Usage

## Basic Interaction

```bash
# Invite the bot to a channel
/invite @kagent

# Ask questions
@kagent list all namespaces
@kagent what pods are running in production?
@kagent show me logs for the first pod#
```

# Deploy secrets in your k8s environment 

```bash
kubectl create secret generic slack-credentials -n kagent \
  --from-literal=SLACK_BOT_TOKEN=$SLACK_BOT_TOKEN \
  --from-literal=SLACK_APP_TOKEN=$SLACK_APP_TOKEN \
  --from-literal=SLACK_TEAM_ID=$SLACK_TEAM_ID \
  --from-literal=SLACK_CHANNEL_IDS=$SLACK_CHANNEL_IDS 
```

# Deploy slackbot in your k8s environment

```bash
kubectl apply -f slackbot-deployment.yaml
```

# Deploy MCP and Agent

Note, in side the in the agent-deployment.yaml configuraiton i have specified the default slack channel

```
    systemMessage: |-
        You're an expert Kubernetes agent that uses tools to answer users questions and help them with their Kubernetes clusters.
        
        ## Slack Integration Rules (IMPORTANT)
        - Default Slack channel ID: C04QDQ6PJHF
        - ALWAYS use channel_id "C04QDQ6PJHF" when calling slack_post_message unless the user explicitly provides a different channel ID
        - Never ask the user which channel to use - default to C04QDQ6PJHF
```

## Deploy it.

```bash
kubectl apply -f mcp-deployment.yaml
kubectl apply -f agent-deployment.yaml
```



