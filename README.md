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

The solution allows a user to initiate an approval request directly from Slack with `/approve-request`. n8n validates the incoming Slack request, immediately acknowledges the command, posts an interactive **Block Kit** approval card, and waits for the decision-maker to select **Approve** or **Reject**. The callback is authenticated again before the original Slack message is dynamically updated with the final decision and decision-maker identity.

The accompanying technical report describes the project as an enterprise-grade approval workflow and a sanitized template suitable for public repository distribution. fileciteturn4file0L2-L15

## Why This Project Matters

This project demonstrates a practical automation pattern for approval-driven business processes:

- Reduce manual approval communication.
- Keep approval actions inside Slack.
- Provide an explicit, auditable decision state.
- Authenticate inbound Slack events before changing workflow state.
- Protect webhook endpoints against stale/replayed requests.
- Keep credentials and signing secrets outside the public workflow export.

## Architecture

```text
┌──────────────────────┐
│       Slack User     │
└──────────┬───────────┘
           │ /approve-request
           ▼
┌──────────────────────────────┐
│ Slash Command Webhook (n8n)  │
└────────────┬─────────────────┘
             ▼
┌──────────────────────────────┐
│ HMAC-SHA256 Signature Check   │
│ + 300s Replay Protection      │
└────────────┬─────────────────┘
             ▼
┌──────────────────────────────┐
│ Immediate HTTP 200 Response  │
│ "Processing your request..." │
└────────────┬─────────────────┘
             ▼
┌──────────────────────────────┐
│ Slack Block Kit Approval Card │
│       [Approve] [Reject]     │
└────────────┬─────────────────┘
             │ button click
             ▼
┌──────────────────────────────┐
│ Interaction Webhook (n8n)    │
└────────────┬─────────────────┘
             ▼
┌──────────────────────────────┐
│ HMAC-SHA256 Signature Check  │
│ + Timestamp Validation        │
└────────────┬─────────────────┘
             ▼
       ┌───────────────┐
       │ Approve/Reject│
       │    Switch     │
       └───────┬───────┘
          ┌────┴────┐
          ▼         ▼
     ┌─────────┐ ┌─────────┐
     │Approved │ │Rejected │
     └────┬────┘ └────┬────┘
          └──────┬─────┘
                 ▼
      ┌─────────────────────┐
      │ Update Slack Message│
      │ + Decision + User ID│
      └─────────────────────┘
```

## Core Workflow

| Stage | n8n Node | Type | Functional Purpose |
|---|---|---|---|
| 1 | **Slash Command Webhook** | Webhook (POST) | Receives `/approve-request` payloads from Slack. |
| 2 | **Verify Slash Signature** | Code (JavaScript) | Validates Slack's `x-slack-signature` using HMAC-SHA256. |
| 3 | **Respond Processing** | Respond to Webhook | Immediately acknowledges the request with `Processing your request...`. |
| 4 | **Send Approval Message** | Slack | Posts the interactive Block Kit approval card. |
| 5 | **Interaction Webhook** | Webhook (POST) | Receives Slack interactive button callbacks. |
| 6 | **Verify Button Signature** | Code (JavaScript) | Authenticates the interactive callback before processing it. |
| 7 | **Approve or Reject** | Switch | Routes execution according to the interaction value. |
| 8 | **Mark Approved** | Slack | Updates the original message with the approval result and Slack user. |
| 9 | **Mark Rejected** | Slack | Updates the original message with the rejection result and Slack user. |

The workflow export confirms the webhook, signature-verification, response, Slack, switch, and message-update nodes described above. fileciteturn4file1L5-L10 fileciteturn4file1L37-L64

## Interactive Slack UI

The approval card uses Slack Block Kit action buttons with distinct action/value pairs:

| Action | `action_id` | `value` | Result |
|---|---|---|---|
| Approve | `approve` | `approve` | Updates the request to **Request Approved** |
| Reject | `reject` | `reject` | Updates the request to **Request Rejected** |

The workflow export defines these button values and routes them through the **Approve or Reject** Switch node. fileciteturn4file1L52-L61 fileciteturn4file1L112-L165

## Security Design

### HMAC-SHA256 Verification

Slack requests are authenticated using the request timestamp and raw request body. The core signing construction is:

```javascript
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
```

The workflow also checks the request timestamp and rejects requests outside a **300-second / 5-minute window**, providing replay-attack protection. The technical report documents the same HMAC construction and timestamp protection. fileciteturn4file0L21-L25 fileciteturn4file0L56-L64

### Security Principles

- Validate inbound Slack signatures before processing events.
- Use the raw request body when calculating the signature.
- Enforce timestamp freshness.
- Use timing-safe signature comparison.
- Never commit live credentials or signing secrets.
- Use sanitized placeholders in public workflow exports.

> **Security warning:** Never paste your Slack Signing Secret, Bot Token, Gemini/API keys, or other credentials into this README, the workflow JSON, screenshots, or public Git history.

## Request Flow

### 1. Slash Command

```text
User → Slack → POST /approve-request → n8n
```

The webhook validates the request and returns the immediate acknowledgement:

```text
Processing your request...
```

The exported workflow uses `responseMode: responseNode` and a dedicated Respond to Webhook node for this acknowledgement. fileciteturn4file1L5-L10 fileciteturn4file1L37-L45

### 2. Approval Message

n8n sends a Block Kit message containing:

```text
New Approval Request

[ Approve ]  [ Reject ]
```

### 3. Interactive Callback

```text
Slack Button Click
       ↓
POST /slack-interactions
       ↓
Verify Signature
       ↓
Read action/value/user/channel/message timestamp
       ↓
Approve or Reject
       ↓
Update original Slack message
```

The exported workflow uses `slack-interactions` for the callback endpoint and updates the original message using its channel ID and message timestamp. fileciteturn4file1L81-L106 fileciteturn4file1L177-L220

## Setup & Deployment

### Prerequisites

- A running n8n instance.
- A Slack workspace where you can create/configure an app.
- A Slack app with a bot user and the required permissions.
- Secure access to the Slack Signing Secret.

### Step 1 — Import the Workflow

1. Open n8n.
2. Go to **Workflows → Import from File**.
3. Select `Slack_Interactive_Approval.json`.
4. Review the imported nodes and expressions.

The project report documents the same import process. fileciteturn4file0L67-L70

### Step 2 — Configure Slack Credentials

Configure the Slack OAuth/Bot Token credential in n8n and attach it to the Slack nodes.

Do **not** store the Bot Token directly inside the workflow JSON or README.

### Step 3 — Configure Signature Verification

Provide the Slack Signing Secret through a secure secret/environment mechanism supported by your n8n deployment.

The public JSON in this repository is sanitized and must not contain a real secret.

### Step 4 — Configure Slack Slash Command

In your Slack App configuration:

1. Create the `/approve-request` Slash Command.
2. Copy the active n8n webhook URL.
3. Set that URL as the Slash Command Request URL.

### Step 5 — Configure Interactivity

Under Slack App **Interactivity & Shortcuts**:

1. Enable interactivity.
2. Set the n8n interaction webhook URL.
3. Save the Slack App configuration.

### Step 6 — Activate & Test

Activate the workflow and execute:

```text
/approve-request
```

Verify that:

- Slack receives the immediate processing acknowledgement.
- The approval card appears.
- **Approve** updates the original message to the approved state.
- **Reject** updates the original message to the rejected state.
- The decision-maker's Slack user ID is included in the final status.
- Invalid or stale signed requests are rejected.

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
    ├── ... n8n workflow/execution evidence
    └── ... project screenshots
```

The repository is intentionally organized around the sanitized workflow export, technical report, and visual assessment evidence.

## Demonstration

### Loom Video

▶ **[Watch the complete project demonstration](https://www.loom.com/share/a183df2c099d4aca970ed91d1a74a347)**

The demonstration accompanies the repository as evidence of the implemented Slack interaction and n8n workflow behavior.

## Documentation

- **Workflow:** `Slack_Interactive_Approval.json`
- **Technical Report:** `Slack_Interactive_Approval_Project_Report.pdf`
- **Visual Evidence:** `Screenshots/`
- **Demo:** Loom video linked above

The technical report covers the executive summary, capabilities, workflow node matrix, security verification, replay protection, and deployment procedure. fileciteturn4file0L7-L27 fileciteturn4file0L67-L75

## Troubleshooting Checklist

### Slack does not receive the response

- Confirm the Slash Command Request URL is correct.
- Confirm the n8n workflow is active.
- Check that the webhook returns an immediate response.
- Review the n8n execution log.

### Signature verification fails

- Confirm the Signing Secret is correct.
- Confirm the raw request body is used.
- Confirm `x-slack-signature` and `x-slack-request-timestamp` are received.
- Check that the request timestamp is within the allowed 300-second window.

### Button clicks do not update the message

- Confirm Slack Interactivity is enabled.
- Confirm the interaction Request URL points to the correct n8n webhook.
- Confirm the Slack credential has permission to update the message.
- Verify the callback contains the expected `approve` or `reject` value.

## Project Status

**Status:** Completed assessment project / sanitized public template

The accompanying report describes the workflow as **Production Ready / Sanitized Template** and documents HMAC-SHA256 verification, replay protection, dynamic state synchronization, and public-safe secret handling. fileciteturn4file0L2-L6

> **Note:** “Production Ready / Sanitized Template” describes the project documentation. Before production use, deploy it with your organization's own Slack app, credentials, secret management, logging, access controls, and operational policies.

## License

This project is released under the **MIT License**. See [`LICENSE`](LICENSE) for the complete license text.

## Author

**Shaik Mohammad Shaheed**

Workflow Automation & DevOps  
Built with **n8n + Slack API + Block Kit + Webhooks**

---

<p align="center">
  <sub>Built as a practical demonstration of secure workflow automation, interactive Slack interfaces, and event-driven DevOps patterns.</sub>
</p>
