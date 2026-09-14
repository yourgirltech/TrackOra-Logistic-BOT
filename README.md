# Trackora — Logistics CRM Automation

A set of n8n workflows that automate the customer-facing and operational side of a logistics business, backed by Supabase.

Instead of a monolithic CRM, this repo is five independent workflows that each own one job: answering tracking questions, keeping customers informed, catching stuck shipments, capturing new leads, and segmenting customers. They share a single Supabase database, so they compose naturally — an event written by one workflow is what triggers another. Each workflow is a plain JSON file you can import into any n8n instance and wire up to your own credentials.

## Workflows

### Tracking Chatbot — `Logistic tracking Chatbot.json`

Exposes a webhook that accepts a waybill number and returns a conversational status update. When a request comes in, the workflow looks up the shipment in the `shipments` table by `waybill_id`, pulls the most recent row from `events` for that shipment (ordered by timestamp), and passes the location, event type, and timestamp to Google Gemini. Gemini turns the raw fields into a short, friendly message — two or three sentences, no jargon — which is returned as JSON to the caller. This is the piece you'd put behind a website widget, a WhatsApp bot, or a support form so customers can self-serve instead of emailing "where is my package?"

### Status Notifications — `status_notifications_workflow.json`

Emails customers automatically whenever a shipment moves to a new status. It's triggered by a webhook carrying a new `events` row (a Supabase database webhook is the intended source), then resolves the shipment and its owning customer, composes a plain-language message naming the new status and location, and sends it over SMTP. After the email goes out it writes the new status back to `shipments.current_status`, so the shipment record always reflects the latest event without a separate sync job.

### Exception Watcher — `exception_watcher_workflow.json`

Runs hourly and looks for shipments that have gone sideways — anything sitting in `customs_hold` or `failed_attempt`. For each one it derives a severity and issue type (customs holds are treated as high severity, failed delivery attempts as medium), opens a row in `tickets` with status `open`, and emails the ops inbox with the waybill, current status, severity, and destination. The goal is that no stuck shipment sits unnoticed: by the time a customer complains, there's already a ticket for it.

### Lead Intake — `lead_intake_workflow.json`

Captures inbound quote requests and makes sure nobody falls through the cracks. A webhook receives the quote form payload (company, contact, email, phone, cargo details), writes it to `leads` with status `new`, and immediately notifies the sales inbox. The workflow then waits 48 hours and re-reads the lead: if the status is still `new`, meaning nobody has touched it, it sends the prospect a friendly follow-up email. If sales has already moved the lead along, the follow-up is skipped.

### Customer Segmentation — `customer_segmentation_workflow.json`

Runs once a day to keep customer segments current. It pulls every row from `shipments`, counts them per `customer_id` in a Code node, and labels anyone with more than two shipments as `repeat`. Matching customers get their `segment` column updated in the `customers` table. It's deliberately simple, and the counting logic is the obvious place to extend — add recency, revenue, or lane-based tiers and the rest of the workflow stays the same.

## Tech Stack

- **[n8n](https://n8n.io)** — workflow automation engine; hosts and runs all five workflows
- **[Supabase](https://supabase.com)** — Postgres database and REST API used as the system of record; also the source of the database webhooks that drive the event-triggered workflows
- **[Google Gemini](https://ai.google.dev)** (`gemini-3-flash-preview` via the n8n LangChain Google Gemini node) — generates the natural-language status replies in the tracking chatbot
- **SMTP** — outbound email for customer notifications, ops alerts, and sales follow-ups

## Database Schema

Five tables in Supabase. Key columns referenced by the workflows:

| Table | Key columns |
| --- | --- |
| `customers` | `id`, `name`, `email`, `segment` (`new` / `repeat`) |
| `shipments` | `id`, `waybill_id`, `customer_id` → `customers.id`, `current_status`, `destination` |
| `events` | `id`, `shipment_id` → `shipments.id`, `event_type`, `location`, `timestamp` |
| `tickets` | `id`, `shipment_id` → `shipments.id`, `issue_type`, `severity`, `status` |
| `leads` | `id`, `company_name`, `contact_name`, `email`, `phone`, `cargo_details`, `status` |

`shipments.current_status` and `events.event_type` share a status vocabulary — `in_transit`, `customs_hold`, `failed_attempt`, `delivered`, and so on. The exception watcher keys off `customs_hold` and `failed_attempt` specifically.

## How to Import

Each JSON file is a standalone n8n workflow. In your n8n instance:

1. Go to **Workflows → Import from File** (in the workflow editor, the **⋯** menu → **Import from File**).
2. Select one of the `.json` files in this repo.
3. Repeat for each workflow you want.

**After importing, you must reconnect credentials on every node that uses them.** The credential references in these files point at credentials that exist in the original n8n instance, not yours, so they will come in empty or broken. Open each workflow and set:

- **Supabase nodes** — your own Supabase API credential (project URL + service role key)
- **Email (SMTP) nodes** — your own SMTP credential
- **Google Gemini node** (tracking chatbot only) — your own Google Gemini / PaLM API credential

While you're in there, also replace the placeholder sender and recipient addresses (`alerts@`, `ops@`, `sales@`, `notifications@yourlogistics.com`) with your real ones, and note the webhook URLs n8n generates for the webhook-triggered workflows — you'll need them to point your forms and Supabase database webhooks at the right place.

## Setup Requirements

- **A Supabase project** with the five tables above created. The workflows assume the column names listed in the schema section; if yours differ, update the field mappings in the Supabase nodes.
- **An n8n instance** — either n8n Cloud or self-hosted. The tracking chatbot uses the `@n8n/n8n-nodes-langchain` Google Gemini node, so a reasonably recent version is required.
- **SMTP credentials** for outbound email (any provider — SES, Postmark, SendGrid, Gmail SMTP, etc.). Three of the five workflows send mail.
- **A Google Gemini API key** for the tracking chatbot.
- **Supabase database webhooks** (optional but recommended) pointing at the status-notification workflow's webhook URL, so status emails fire automatically when a row lands in `events`.

Workflows ship with `active: false` and should be tested with sample data before you switch them on.

## Notes

No credentials are committed to this repo. Credential fields in the JSON are placeholders that n8n will prompt you to fill on import.
