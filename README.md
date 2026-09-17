# Slack Interactive Approval Workflow

<p align="center">
  <strong>Secure, interactive Slack approvals powered by n8n</strong><br>
  Slash Commands • Block Kit • Webhooks • HMAC-SHA256 • Automated Decision Routing
</p>

<p align="center">
  <a href="https://n8n.io/"><img src="https://img.shields.io/badge/n8n-Workflow%20Automation-ff6d5a?logo=n8n&logoColor=white" alt="n8n"></a>
  <a href="https://api.slack.com/"><img src="https://img.shields.io/badge/Slack-API-4A154B?logo=slack&logoColor=white" alt="Slack API"></a>
  <img src="https://img.shields.io/badge/Security-HMAC--SHA256-2ea44f" alt="HMAC-SHA256 Security">
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-blue.svg" alt="MIT License"></a>
</p>

<p align="center">
  <a href="https://www.loom.com/share/a183df2c099d4aca970ed91d1a74a347">▶ Watch the Loom Demo</a>
  •
  <a href="https://github.com/shaikshahid777/n8n-slack-interactive-approval">View Repository</a>
</p>

---

## Overview

**Slack Interactive Approval Workflow** is a secure, event-driven approval automation built with **n8n** and the **Slack API**.

A user initiates an approval request directly from Slack with `/approve-request`. n8n validates the inbound request, immediately acknowledges it, posts an interactive **Slack Block Kit** approval card, and processes the decision-maker's button click. The callback is authenticated before the original Slack message is updated with the final decision and decision-maker identity.

## Key Capabilities

- ⚡ **Slash Command Trigger** — `/approve-request` starts the approval flow from Slack.
- 🧩 **Interactive Block Kit UI** — Approve and Reject buttons provide an in-Slack decision experience.
- 🔐 **HMAC-SHA256 Verification** — authenticates inbound Slack requests using the raw body and signing secret.
- 🛡️ **Replay Protection** — rejects requests outside the 300-second timestamp window.
- 🔀 **Decision Routing** — a Switch node routes `approve` and `reject` actions.
- 🔄 **Dynamic State Synchronization** — updates the original Slack message with the final state.
- 👤 **Decision Attribution** — records the Slack user ID associated with the decision.
- 🧼 **Sanitized Public Template** — no live credentials or secrets are intended to be stored in the repository.

## Architecture

```text
┌──────────────────────┐
│      Slack User      │
└──────────┬───────────┘
           │ /approve-request
           ▼
┌──────────────────────────────┐
│ Slash Command Webhook (n8n)  │
└────────────┬─────────────────┘
             ▼
┌──────────────────────────────┐
│ HMAC-SHA256 + Replay Check   │
│      300-second window       │
└────────────┬─────────────────┘
             ▼
┌──────────────────────────────┐
│ Immediate HTTP 200 Response  │
│ "Processing your request..." │
└────────────┬─────────────────┘
             ▼
┌──────────────────────────────┐
│ Slack Block Kit Approval Card│
│       [Approve] [Reject]     │
└────────────┬─────────────────┘
             │ button click
             ▼
┌──────────────────────────────┐
│ Interaction Webhook (n8n)    │
└────────────┬─────────────────┘
             ▼
┌──────────────────────────────┐
│ HMAC-SHA256 + Timestamp Check│
└────────────┬─────────────────┘
             ▼
       ┌───────────────┐
       │ Approve/Reject│
       │    Switch     │
       └───────┬───────┘
          ┌────┴────┐
          ▼         ▼
     ┌─────────┐ ┌─────────┐
     │ Approved│ │ Rejected│
     └────┬────┘ └────┬────┘
          └──────┬─────┘
                 ▼
      ┌─────────────────────┐
      │ Update Slack Message│
      │ + Decision + User ID│
      └─────────────────────┘
```

## n8n Node Matrix

| Node Name | Type | Functional Purpose |
|---|---|---|
| **Slash Command Webhook** | Webhook (POST) | Receives `/approve-request` payloads from Slack. |
| **Verify Slash Signature** | Code (JavaScript) | Validates the Slack request signature using HMAC-SHA256. |
| **Respond Processing** | Respond to Webhook | Immediately returns `Processing your request...`. |
| **Send Approval Message** | Slack | Posts the interactive Block Kit approval card. |
| **Interaction Webhook** | Webhook (POST) | Receives interactive Slack button callbacks. |
| **Verify Button Signature** | Code (JavaScript) | Authenticates the button-click callback. |
| **Approve or Reject** | Switch | Routes execution by the interaction value. |
| **Mark Approved** | Slack | Updates the original message with the approval result. |
| **Mark Rejected** | Slack | Updates the original message with the rejection result. |

## Slack Interaction Model

The Block Kit card uses explicit action/value pairs:

| Action | `action_id` | `value` | Final State |
|---|---|---|---|
| Approve | `approve` | `approve` | `Request Approved` |
| Reject | `reject` | `reject` | `Request Rejected` |

The interaction handler uses the callback's user ID, channel ID, message timestamp, and action value to update the correct Slack message.

## Security Implementation

### HMAC-SHA256 Signature Verification

Slack signatures are calculated from the version prefix, request timestamp, and exact raw request body:

```javascript
const crypto = require('crypto');

const baseString = 'v0:' + timestamp + ':' + rawBody;

const expected = 'v0=' + crypto
  .createHmac('sha256', signingSecret)
  .update(baseString)
  .digest('hex');

const valid =
  signature.length === expected.length &&
  crypto.timingSafeEqual(
    Buffer.from(expected),
    Buffer.from(signature)
  );

if (!valid) {
  throw new Error('Rejected: invalid Slack signature');
}
```

### Replay-Attack Protection

Requests are checked against the Slack request timestamp. Requests older than **300 seconds (5 minutes)** are rejected before business logic is processed.

### Secret Management

The repository is a sanitized template. **Never commit real Slack Signing Secrets, Bot Tokens, API keys, webhook secrets, or other credentials.** Configure secrets through n8n credentials or the secure environment/secret mechanism provided by your deployment.

## Request Lifecycle

### 1. Slash Command

```text
Slack → POST /approve-request → n8n
```

The workflow validates the request and immediately returns:

```text
Processing your request...
```

### 2. Approval Card

n8n posts an interactive message:

```text
New Approval Request

[ Approve ]  [ Reject ]
```

### 3. Interactive Callback

```text
Button Click
    ↓
POST /slack-interactions
    ↓
Verify Signature
    ↓
Extract action + user + channel + message timestamp
    ↓
Approve / Reject Switch
    ↓
Update Original Slack Message
```

## Setup & Deployment

### Prerequisites

- n8n instance
- Slack workspace with permission to create/configure an app
- Slack App with bot user and required scopes
- Slack Signing Secret stored securely

### 1. Import the Workflow

1. Open n8n.
2. Select **Workflows → Import from File**.
3. Import `Slack_Interactive_Approval.json`.
4. Review the workflow before activation.

### 2. Configure Slack Credentials

Configure the Slack OAuth/Bot Token credential in n8n and attach it to the Slack nodes.

### 3. Configure the Signing Secret

Provide the Slack Signing Secret through secure deployment configuration. Do not put the real secret into GitHub.

### 4. Configure `/approve-request`

In the Slack App configuration:

1. Create the `/approve-request` Slash Command.
2. Copy the active n8n webhook URL.
3. Set it as the Slash Command Request URL.

### 5. Configure Interactivity

Under **Interactivity & Shortcuts**:

1. Enable interactivity.
2. Set the n8n interaction webhook URL.
3. Save the Slack App configuration.

### 6. Activate & Test

Run:

```text
/approve-request
```

Verify the following:

- Immediate processing acknowledgement is returned.
- Approval card appears in Slack.
- Approve changes the message to the approved state.
- Reject changes the message to the rejected state.
- Decision-maker identity is included.
- Invalid signatures are rejected.
- Stale/replayed requests are rejected.

## Repository Structure

```text
n8n-slack-interactive-approval/
│
├── README.md
├── LICENSE
├── .gitignore
│
├── Slack_Interactive_Approval.json
├── Slack_Interactive_Approval_Project_Report.pdf
│
└── Screenshots/
    ├── ... Slack UI interaction evidence
    ├── ... n8n workflow evidence
    └── ... execution screenshots
```

## Demonstration

### Loom Video

▶ **[Watch the complete project demonstration](https://www.loom.com/share/a183df2c099d4aca970ed91d1a74a347)**

The video demonstrates the implemented workflow and provides visual evidence for the project assessment.

## Project Documentation

| Artifact | Purpose |
|---|---|
| `Slack_Interactive_Approval.json` | Sanitized n8n workflow export |
| `Slack_Interactive_Approval_Project_Report.pdf` | Technical project report |
| `Screenshots/` | Slack and n8n implementation evidence |
| Loom Demo | End-to-end visual demonstration |

## Troubleshooting

### Slack does not receive the response

- Confirm the Slash Command Request URL.
- Confirm the n8n workflow is active.
- Check the n8n execution log.
- Confirm the webhook returns immediately.

### Signature verification fails

- Confirm the Signing Secret.
- Confirm the raw request body is used.
- Confirm `x-slack-signature` and `x-slack-request-timestamp` are present.
- Check the 300-second timestamp window.

### Button clicks do not update the message

- Confirm Slack Interactivity is enabled.
- Confirm the interaction Request URL is correct.
- Confirm the Slack credential has the required permissions.
- Confirm the callback contains `approve` or `reject`.

## Project Status

**Status:** Completed assessment project / Sanitized public template

The implementation focuses on secure request verification, interactive Slack decision handling, replay protection, dynamic message state synchronization, and public-safe credential handling.

> **Production note:** Before using this workflow in a real organization, configure organization-specific access controls, secret management, monitoring, logging, error handling, and operational policies.

## License

Released under the **MIT License**. See [`LICENSE`](LICENSE) for the complete license text.

## Author

**Shaik Mohammad Shaheed**  
Workflow Automation & DevOps

**Technology Stack:** n8n • Slack API • Slack Block Kit • Webhooks • JavaScript • HMAC-SHA256

---

<p align="center">
  <sub>Built as a practical demonstration of secure workflow automation, interactive Slack interfaces, and event-driven DevOps patterns.</sub>
</p>
