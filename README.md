
# Restaurant WhatsApp Automation 

**An n8n automation system for restaurant ordering, WhatsApp customer support, delivery updates, promotional broadcasts, and WhatsApp inbox management.**

This project connects a restaurant website to **WhatsApp**, using **n8n** as the automation layer. It sends customers order notifications, handles incoming text and media messages with an AI agent, reads menu and order information from **Supabase**, and stores WhatsApp conversations and media for an inbox/dashboard.

> **Project type:** n8n workflow export  
> **Integrations:** Evolution API, Supabase, OpenRouter, Google Gemini, Gmail  
> **Primary use case:** Restaurant operations and customer communication

---

## Contents

- [What This Workflow Does](#what-this-workflow-does)
- [Workflow Modules](#workflow-modules)
- [Integrations and Services](#integrations-and-services)
- [Architecture](#architecture)
- [Webhook Endpoints](#webhook-endpoints)
- [Requirements](#requirements)
- [Setup Guide](#setup-guide)
- [Configuration and Credentials](#configuration-and-credentials)
- [Security Checklist](#security-checklist)
- [Important Checks Before Production](#important-checks-before-production)
- [Troubleshooting](#troubleshooting)
- [Project Contents](#project-contents)

---

## What This Workflow Does

The automation is organized into several connected modules:

1. **Order notifications** — sends WhatsApp confirmations and status updates to customers.
2. **Large-order alerts** — notifies the restaurant owner by email and WhatsApp when an order reaches the configured quantity threshold.
3. **WhatsApp AI assistant** — responds to customer questions, uses restaurant menu data, and can look up order information.
4. **Voice and image understanding** — retrieves incoming media from Evolution API and uses Google Gemini to transcribe audio or analyze images.
5. **Delivery and offer notifications** — provides webhook entry points for delivery events and promotional broadcasts.
6. **WhatsApp inbox storage** — records incoming messages, tracks message statuses, and stores media in a private Supabase Storage bucket.
7. **Human handoff** — accepts staff replies and sends them to the customer through WhatsApp.
8. **Manual data reset** — provides a manual-only workflow for clearing WhatsApp message/contact data and stored media.

---

## Workflow Modules

### 1. New Order — `order.created`

**Webhook:** `POST /res-created`

When the restaurant website creates an order:

- n8n receives the order payload and acknowledges the request.
- The workflow totals the quantities in `order_items`.
- Orders below 15 items follow the normal customer-notification branch.
- Orders with 15 items or more follow the large-order branch.
- The customer receives an order summary containing item details, extras, totals, and an invoice link.
- Large orders trigger an owner alert by email and WhatsApp.

### 2. Order Ready — `order.status_changed`

**Webhook:** `POST /res-ready`

When an order becomes ready:

- The workflow checks for the `ready` status.
- It creates a customer-facing WhatsApp message.
- The message distinguishes between restaurant pickup and home delivery.
- The customer's phone number is normalized before the message is sent.

### 3. Order Cancelled

**Webhook:** `POST /res-cancelled`

When an order is cancelled:

- The workflow checks for the `cancelled` status.
- It prepares a cancellation notification with the order reference and invoice link.
- It normalizes the customer's phone number and sends the WhatsApp message.

### 4. Cancellation Request

**Webhook:** `POST /res-cancel-request`

This endpoint is documented in the workflow for customer cancellation requests and owner/admin cancellation events.

The intended notifications include:
- A confirmation or update for the customer.
- An alert to the owner when appropriate.

**Production note:** The current message-building code contains a temporary unconditional test condition. Review and replace that condition with explicit event/status checks before enabling this route in production.

### 5. WhatsApp AI Assistant

**Webhook:** `POST /whatsapp`

Evolution API forwards incoming WhatsApp messages to n8n.

The workflow:
- Extracts message content and sender information.
- Filters out messages sent by the connected account and unsupported group/broadcast traffic.
- Routes text messages to the AI agent.
- Routes audio and image messages through the media-processing branch.
- Uses conversation memory to retain recent context for each sender.
- Uses Supabase tools to retrieve menu items, menu categories, orders, and order-status history.
- Sends the generated response through Evolution API.

The AI model connection is configured through OpenRouter. The workflow also includes Google Gemini nodes for audio transcription and image analysis.

### 6. Voice and Image Processing

For supported incoming media:

1. Retrieve the media payload from Evolution API.
2. Check that media data is available.
3. Convert Base64 data into a binary file.
4. Send audio to the Gemini transcription node.
5. Send images to the Gemini image-analysis node.
6. Convert the result into text for the AI assistant.

If media retrieval fails, the workflow follows its error/return branch rather than treating the media as successfully processed.

### 7. Delivery Notifications

**Webhook:** `POST /res-Delivery`

A separate webhook is provided for delivery-related events. The workflow includes phone cleanup and message/image sending branches.

Connect this endpoint to the relevant delivery event in the restaurant application and verify the expected payload before production use.

### 8. Promotional Offers

**Webhook:** `POST /res-offers`

The offers module prepares recipients and optional media, then processes the recipients in a loop with a wait step between sends.

Use this module for authorized restaurant promotional messages. Test with a small internal recipient list before using a customer list.

### 9. WhatsApp Inbox and Media Storage

The inbox receiver uses:

**Webhook:** `POST /whatsapp`

The workflow supports:
- Incoming message logging in PostgreSQL/Supabase.
- Duplicate-message detection.
- Message-type classification.
- Marking incoming messages as read.
- Retrieving images, voice notes, videos, and files from Evolution API.
- Uploading media to a private Supabase Storage bucket.
- Recording successful or failed media uploads.
- Tracking outgoing message statuses such as sent, delivered, read, played, and failed.

The inbox/storage module stores conversations and media; the AI response module is a separate part of the automation.

### 10. Human Reply

**Webhook:** `POST /res-human-reply`

This endpoint accepts a staff reply from the restaurant dashboard, normalizes the destination number, sends the message through Evolution API, and logs the outgoing reply.

### 11. Manual WhatsApp Data Reset

**Webhook:** `POST /res-wipe-all`

The workflow includes a destructive reset path that can:
- List stored WhatsApp media objects.
- Delete those objects from Supabase Storage.
- Clear WhatsApp message and contact records.

This operation is destructive and should be restricted to authorized administrators. Do not expose it as a public, unauthenticated endpoint.

---

## Integrations and Services

| Service | Role |
|---|---|
| **n8n** | Workflow orchestration, webhooks, routing, and automation |
| **Evolution API v2** | WhatsApp message sending, receiving, and media retrieval |
| **Supabase / PostgreSQL** | Menu, order, conversation, contact, and message data |
| **Supabase Storage** | Private storage for WhatsApp media |
| **OpenRouter** | Chat model provider for the AI assistant |
| **Google Gemini** | Voice transcription and image analysis |
| **Gmail** | Large-order email notifications |

---

## Architecture

```text
                   RESTAURANT WEBSITE
                           |
             Order / Status / Delivery Events
                           |
                           v
                    +-------------+
                    |     n8n     |
                    |  Webhooks   |
                    +-------------+
                      |    |    |
          +-----------+    |    +----------------+
          |                |                     |
          v                v                     v
   Order Notifications  WhatsApp AI        Offers / Delivery
          |             Assistant              |
          |                |                    |
          v          +-----+------+             v
      Evolution API  |            |        Evolution API
          |       Supabase     OpenRouter
          |       Menu/Orders  + Gemini
          v            |
       WhatsApp        v
                  Evolution API
                       |
                    WhatsApp

   WhatsApp Incoming Messages
              |
              v
        Evolution API
              |
              v
         n8n /whatsapp
          |         |
          v         v
      PostgreSQL  Media Retrieval
      Message Log      |
                       v
               Supabase Storage
```

---

## Webhook Endpoints

The following paths are defined in the supplied workflow export. Add your own n8n production base URL when configuring the restaurant application.

| Purpose | Method | Path |
|---|---|---|
| New order | POST | `/res-created` |
| Order ready | POST | `/res-ready` |
| Order cancelled | POST | `/res-cancelled` |
| Cancellation request / owner cancellation | POST | `/res-cancel-request` |
| Delivery event | POST | `/res-Delivery` |
| Promotional offers | POST | `/res-offers` |
| Incoming WhatsApp messages | POST | `/whatsapp` |
| Staff / human reply | POST | `/res-human-reply` |
| Manual WhatsApp data reset | POST | `/res-wipe-all` |

Use the **Production URL** of each activated n8n webhook in connected services. Do not use `/webhook-test` URLs for live traffic.

---

## Requirements

Before importing and activating the workflow, prepare:

- A running n8n instance.
- Evolution API v2 with a connected WhatsApp instance.
- A Supabase project with the required database tables and Storage bucket.
- PostgreSQL connection details for the Supabase database.
- An OpenRouter API account and chat model.
- Google Gemini credentials for audio and image processing.
- Gmail credentials for large-order email alerts.
- The restaurant website configured to send the required webhook events.

---

## Setup Guide

### 1. Import the Workflow

1. Open your n8n instance.
2. Create a new workflow or open the target workflow.
3. Use **Import from File** and select the supplied n8n JSON export.
4. Review the imported nodes and connections.
5. Save the workflow without activating it yet.

### 2. Prepare Supabase

1. Open your Supabase project.
2. Run the project's WhatsApp database setup SQL in the Supabase SQL Editor. The workflow notes refer to `whatsapp_supabase_setup.sql`.
3. Confirm that the required tables exist, including the WhatsApp configuration, messages, and contacts tables.
4. Create or verify the private media bucket (the workflow configuration uses `whatsapp-media` as the example).
5. Confirm that the database and Storage credentials point to the same Supabase project.

The supplied workflow notes specify these configuration values in `whatsapp.configurations`:

| Setting | Description |
|---|---|
| `evolution_url` | Evolution API base URL, without a trailing slash |
| `supabase_url` | Supabase project URL |
| `media_bucket` | Private Storage bucket name |

The configuration row is expected to use `environment = production` and `is_secret = false` for these non-secret values.

### 3. Configure n8n Credentials

Create or select the required credentials in n8n:

| Credential type | Used for |
|---|---|
| Postgres | Reading and updating WhatsApp database records |
| Header Auth | Evolution API requests using the `apikey` header |
| Supabase API | Uploading and deleting media in Supabase Storage |
| OpenRouter | AI chat model |
| Google Gemini | Voice transcription and image analysis |
| Gmail OAuth2 | Large-order email notifications |

Keep API keys and passwords inside n8n Credentials. Do not paste them into Code nodes, workflow notes, README files, or source control.

### 4. Configure Evolution API

1. Confirm that the WhatsApp instance is connected.
2. Configure the Evolution API webhook to use the **Production URL** of the n8n `/whatsapp` webhook.
3. Enable the message events required by the workflow: `MESSAGES_UPSERT` and `MESSAGES_UPDATE`.
4. Keep media Base64 disabled in the webhook if using the workflow's separate media-retrieval step.
5. Confirm that Evolution API stores incoming messages so the media retrieval endpoint can access them.

The workflow notes describe the Evolution webhook configuration endpoint as:

```text
POST {evolution_url}/webhook/set/{instance}
```

Use your own Evolution API host, instance name, and API credential.

### 5. Configure AI Services

- Select the intended OpenRouter chat model in the **OpenRouter Chat Model** node.
- Configure the Google Gemini credential in the transcription and image-analysis nodes.
- Review the AI Agent instructions and make sure the restaurant name, menu behavior, language, and customer-ordering rules match your business.
- Verify that the Supabase tools can read the correct menu and order records.

### 6. Configure Website Events

Set the restaurant website to call the relevant n8n webhook when an order is created, updated, cancelled, or marked ready.

Make sure the payload contains the fields expected by the workflow, such as:
- `event`
- `payload.id`
- `payload.order_no`
- `payload.customer_name`
- `payload.phone`
- `payload.order_type`
- `payload.order_items`
- `payload.total`
- `payload.status`

The exact fields required vary by event. Test each event using a real example payload from your application before activating the workflow.

### 7. Test Before Activation

Test each module independently:

- Create a normal order and verify the customer confirmation.
- Create an order at or above the large-order threshold and verify the owner alerts.
- Change an order to `ready` and `cancelled`.
- Submit a cancellation request.
- Send a text message to the connected WhatsApp number.
- Send an audio message and an image.
- Confirm that messages and media appear in the inbox database and Storage.
- Test a delivery event and an offer using internal test recipients.
- Test a staff reply from the dashboard.
- Verify that the reset workflow is restricted and is not triggered accidentally.

Activate the workflow only after the tests pass.

---

## Configuration and Credentials

The workflow uses credentials and settings managed by n8n and Supabase. The exact credential names may differ between installations.

**Never commit real credentials.** Before sharing or publishing a workflow export, inspect it for:
- Evolution API keys.
- Supabase service-role keys.
- Database passwords.
- OpenRouter and Gemini API keys.
- Gmail OAuth tokens.
- Private webhook URLs or secrets.

If a credential was included in an exported workflow or shared file, treat it as exposed and rotate it before using the workflow in production.

---

## Security Checklist

- [ ] Replace any exposed API key or password.
- [ ] Move every secret into n8n Credentials.
- [ ] Add authentication to incoming webhooks where supported.
- [ ] Use long, random webhook paths as an additional protection, not as the only protection.
- [ ] Protect the manual reset endpoint with strong authentication and authorization.
- [ ] Keep the Supabase media bucket private.
- [ ] Use least-privilege access where possible.
- [ ] Avoid retaining sensitive successful execution data longer than necessary.
- [ ] Restrict access to n8n, its credentials, and execution history.
- [ ] Test that duplicate incoming messages do not trigger duplicate AI replies.

---

## Important Checks Before Production

The supplied export includes development notes and implementation details that should be reviewed before going live:

1. **Exposed Evolution API key:** the export contains an API key value directly in HTTP Request node headers. Treat it as compromised, rotate it in Evolution API, and replace the hardcoded values with an n8n Header Auth credential.
2. **Cancellation-request test condition:** the cancellation-request message code currently has an unconditional test condition. Replace it with explicit event/status validation before production.
3. **Webhook security:** the workflow notes identify the incoming Evolution webhook as open. Configure Header Auth or another appropriate verification mechanism where your Evolution setup supports it. A long random path alone is not sufficient protection.
4. **Reset endpoint:** ensure `/res-wipe-all` cannot be called by the public or by ordinary staff accounts.
5. **Duplicate nodes and phone normalization:** the export contains multiple phone-normalization nodes and some duplicated logic. Review each branch and keep one consistent normalization path per destination.
6. **Large-order threshold:** confirm that 15 items is the intended business threshold and test the boundary values (14, 15, and 16).
7. **AI order actions:** verify the AI agent's tool permissions and order lookup behavior. Do not assume that a message sent by the AI means an order was successfully created or updated in the database.
8. **Payload compatibility:** confirm that the website sends the expected event names and nested payload structure for every webhook.

---

## Troubleshooting

| Issue | What to check |
|---|---|
| WhatsApp message is not sent | Verify the Evolution API URL, instance name, Header Auth credential, and destination number format. |
| Phone number is rejected | Check the normalization node output and confirm that the country code is present. |
| AI cannot answer menu questions | Verify Supabase credentials, table names, and the menu tools connected to the AI Agent. |
| Voice or image is not processed | Check media retrieval, Evolution message storage, Base64 conversion, and Gemini credentials. |
| Media upload fails | Confirm the Supabase URL, Storage bucket, service-role credential, and project match. |
| Incoming messages are missing | Check the Evolution webhook URL, enabled events, n8n activation, and execution logs. |
| Duplicate replies occur | Verify message deduplication and ensure the workflow ignores messages sent by the connected account. |
| Large-order alert is not triggered | Check the total quantity calculation and the IF node threshold. |
| Status is not updated | Confirm that outgoing message IDs are saved and match Evolution API's message ID. |

---

## Project Contents

The supplied file is an **n8n workflow JSON export** containing:
- Webhook trigger nodes.
- Code nodes for message formatting and phone normalization.
- HTTP Request nodes for Evolution API and Supabase Storage.
- Supabase and PostgreSQL nodes.
- AI Agent, memory, and model nodes.
- Google Gemini media-processing nodes.
- Gmail notification node.
- Sticky notes documenting setup, flow behavior, and security considerations.

---

## Maintainer

**Faris Ataa Lashen**  
AI Automation Engineer

Focus areas: n8n workflow automation, AI agents, WhatsApp integrations, RAG, and LLM-powered business workflows.
