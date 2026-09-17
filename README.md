# Slack Interactive Approval Workflow with n8n

An automated Slack approval workflow built with **n8n**, **Slack API**, **Slack Block Kit**, and **Webhooks**. The workflow lets users trigger an approval request from Slack with `/approve-request`, presents interactive **Approve** and **Reject** buttons, validates incoming Slack requests using **HMAC-SHA256**, and updates the original Slack message with the final decision.

## Project Overview

This project demonstrates an end-to-end interactive approval workflow using n8n as the automation engine. It includes request authentication, immediate webhook acknowledgement, interactive Block Kit controls, decision routing, and message updates.

According to the project report, the workflow is designed as a sanitized/public-safe template with credentials and secrets kept out of the repository. The report describes the implementation as using Slack API, Block Kit, Webhooks, and HMAC-SHA256 verification.

## Key Features

- **Slash Command Trigger** — `/approve-request` starts an approval request from a Slack channel.
- **Immediate Acknowledgement** — responds with `Processing your request...` so Slack receives an immediate response within its timeout window.
- **Interactive Block Kit UI** — posts an approval card containing **Approve** and **Reject** buttons.
- **HMAC-SHA256 Verification** — verifies Slack request signatures using the raw request body, timestamp, and signing secret.
- **Replay Protection** — rejects requests whose timestamps are more than 5 minutes (300 seconds) from the current time.
- **Approve/Reject Routing** — routes the interaction according to the button value.
- **Dynamic Slack Update** — updates the original message to show `Request Approved` or `Request Rejected` and the Slack user who made the decision.
- **Sanitized Export** — the repository is intended to contain a public-safe workflow template rather than live secrets.

## Workflow Architecture

```text
Slack /approve-request
        |
        v
Slash Command Webhook
        |
        v
Verify Slash Signature
        |
        v
Respond: Processing your request...
        |
        v
Send Approval Message
        |
        |  Approve / Reject button click
        v
Interaction Webhook
        |
        v
Verify Button Signature
        |
        v
Approve or Reject (Switch)
       / \
      /   \
 Approve  Reject
    |        |
    v        v
Mark Approved   Mark Rejected
      \        /
       \      /
        v    v
   Update original Slack message
```

The exported workflow contains the Slash Command webhook at `approve-request` and the interaction webhook at `slack-interactions`, with raw-body handling enabled for signature verification.

## n8n Node Matrix

| Node | Type | Purpose |
|---|---|---|
| Slash Command Webhook | Webhook (POST) | Receives `/approve-request` payloads from Slack |
| Verify Slash Signature | Code (JavaScript) | Validates the Slack request signature using HMAC-SHA256 |
| Respond Processing | Respond to Webhook | Returns `Processing your request...` immediately |
| Send Approval Message | Slack | Sends the interactive Block Kit approval message |
| Interaction Webhook | Webhook (POST) | Receives Slack button interaction callbacks |
| Verify Button Signature | Code (JavaScript) | Authenticates button-click callbacks |
| Approve or Reject | Switch | Routes the request by action value |
| Mark Approved | Slack | Updates the original message with the approval result |
| Mark Rejected | Slack | Updates the original message with the rejection result |

## Security

Slack request verification follows the standard signing pattern:

```javascript
const baseString = 'v0:' + timestamp + ':' + rawBody;
const expected = 'v0=' + crypto
  .createHmac('sha256', signingSecret)
  .update(baseString)
  .digest('hex');
```

The workflow also performs timestamp/replay protection and uses a timing-safe signature comparison. The project report specifies that secrets should be externalized and that the public export should not expose credentials.

> **Important:** Never commit your real Slack Signing Secret, Bot Token, Gemini/API keys, webhook secrets, or other credentials to GitHub. Use n8n credentials or secure environment configuration for live deployments.

## Slack Block Kit

The approval message contains two interactive buttons:

- `action_id`: `approve`
- `value`: `approve`
- `action_id`: `reject`
- `value`: `reject`

The interactive callback is parsed to obtain the Slack user, selected action/value, channel, and original message timestamp before the workflow updates the message.

## Files

Suggested repository structure:

```text
.
├── README.md
├── Slack Interactive Approval.json
├── Slack_Interactive_Approval_Project_Report.pdf
├── screenshots/
│   └── ...
└── docs/
    └── ...
```

The main workflow export is the JSON file included in this repository.

## Setup

1. Import the workflow JSON into n8n.
2. Configure your Slack OAuth/Bot Token credential in n8n.
3. Configure the Slack Signing Secret securely in your deployment.
4. Create the `/approve-request` Slash Command in your Slack app and point it to the active n8n webhook URL.
5. Configure Slack **Interactivity & Shortcuts** to use the interaction webhook URL.
6. Activate the workflow.
7. Run `/approve-request` in Slack and test both **Approve** and **Reject**.

## Demo

Loom demonstration:

https://www.loom.com/share/a183df2c099d4aca970ed91d1a74a347

## Project Report

The accompanying report covers the executive summary, key capabilities, node architecture, HMAC-SHA256 security design, replay protection, and deployment steps.

## Assessment Evidence

This repository is intended to support the assessment submission with the workflow export, project documentation, screenshots, and demonstration video.

## Author

**Shaik Mohammad Shaheed**

Category: Workflow Automation & DevOps  
Platform: n8n Automation Engine  
Integration: Slack API, Block Kit, Webhooks  
Security: HMAC-SHA256 Verification
