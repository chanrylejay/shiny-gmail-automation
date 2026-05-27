# SHINY GMAIL AUTOMATION V4.0 LEAN — COMPLETE DEPLOYMENT GUIDE

**Date:** May 2026 | **Architecture Version:** V4.0 Lean | **60 Executable Nodes** | **3 Workflows**  
**Stack:** n8n + Gemini + Neon PostgreSQL + Gmail OAuth2 + Telegram Bot API + ngrok

---

## TABLE OF CONTENTS

- [Step 0 — Gather Your Credentials & IDs](#step-0--gather-your-credentials--ids)
- [Step 1 — Create Gmail Labels](#step-1--create-gmail-labels)
- [Step 2 — Set n8n Environment Variables](#step-2--set-n8n-environment-variables)
- [Step 3 — Set Up ngrok for Telegram](#step-3--set-up-ngrok-for-telegram)
- [Step 4 — Run the Database Schema](#step-4--run-the-database-schema)
- [Step 5 — Create Credentials in n8n](#step-5--create-credentials-in-n8n)
- [File-by-File Import Instructions](#file-by-file-import-instructions)
- [Post-Import: Connecting Credentials & Replacing Placeholders](#post-import-connecting-credentials--replacing-placeholders)
- [Known Import Issues](#known-import-issues)
- [Activation Order](#activation-order)
- [Deployment Checklist](#deployment-checklist)
- [Verification & Smoke Tests](#verification--smoke-tests)
- [Troubleshooting](#troubleshooting)
- [Monthly Cost Estimate](#monthly-cost-estimate)
- [Architecture Summary](#architecture-summary)
- [Environment Variable Quick Reference](#environment-variable-quick-reference)
- [Sample Startup Script](#sample-startup-script)

---

## STEP 0 — GATHER YOUR CREDENTIALS & IDS

Before importing anything into n8n, collect **all** required credentials and IDs. You will paste these into either n8n credentials, environment variables, or workflow settings during deployment.

> [!TIP]
> Put these values in a temporary notepad file while deploying, then delete it after setup. Never commit credentials to GitHub.

### 1. Gmail OAuth2 Credentials

**Where to get it:**

1. Go to <https://console.cloud.google.com>
2. Create a new project or use an existing one.
3. Go to **APIs & Services** → **Library**.
4. Search for **Gmail API**.
5. Click **Enable**.
6. Go to **APIs & Services** → **Credentials**.
7. Click **Create Credentials** → **OAuth 2.0 Client ID**.
8. Choose **Web application**.
9. Add this Authorized redirect URI if you run n8n locally:

```text
http://localhost:5678/rest/oauth2-credential/callback
```

If your n8n instance runs on another host or port, adjust the URL to match your n8n base URL.

**What they look like:**

```text
Client ID:     123456789-abcdef.apps.googleusercontent.com
Client Secret: GOCSPX-xxxxx
```

### 2. Telegram Bot Token

**Where to get it:**

1. Open Telegram.
2. Message **@BotFather**.
3. Type `/newbot`.
4. Follow the prompts.
5. Copy the bot token.

**What it looks like:**

```text
7123456789:AAH...
```

### 3. Telegram Chat ID

Workflow C only accepts commands from your authorized chat ID. This protects the bot from random Telegram users.

#### Option A — Use @userinfobot

1. Open Telegram.
2. Search for **@userinfobot**.
3. Send any message.
4. It replies with your chat ID.

#### Option B — Use getUpdates

> [!NOTE]
> This method only works when your bot does **not** already have an active webhook. If Workflow C is already active, use Option A.

1. Send any message to your bot.
2. Open this URL in your browser:

```text
https://api.telegram.org/bot<YOUR_TOKEN_HERE>/getUpdates
```

3. Look for this field:

```json
"chat":{"id":1234567890}
```

That number is your Telegram chat ID.

### 4. Gemini API Key

**Where to get it:**

1. Go to <https://aistudio.google.com/apikey>.
2. Click **Create API Key**.
3. Choose your Google Cloud project or create a new one.
4. Copy the API key.

**What it looks like:**

```text
AIzaSy...
```

> [!WARNING]
> This is a **Google AI Studio** API key, not a generic Google Cloud API key. Make sure you are on `aistudio.google.com`.

> [!CAUTION]
> If your goal is to stay free-tier, be careful with billing settings. Free-tier availability and quota behavior can vary by model, region, and project.

### 5. Postgres / Neon Connection Details

**Where to get it:**

1. Go to <https://console.neon.tech>.
2. Open your project.
3. Click **Connection Details**.

You need:

```text
Host
Database
User
Password
SSL required/enabled
```

Typical Neon host example:

```text
ep-xxx.ap-southeast-1.aws.neon.tech
```

### 6. Error Workflow ID / Supervisor Workflow ID Optional

If you have a separate error-monitoring workflow, such as Supervisor V23, note its workflow ID.

You can set it as the **Error Workflow** for all three Shiny Gmail workflows after import.

If you do not have one, skip this. V4.0 still has internal failure handling for normal operational errors.

---

## STEP 1 — CREATE GMAIL LABELS

Shiny Gmail V4.0 uses **7 custom Gmail labels**.

These must exist before the first real run.

### How to create a label in Gmail

1. Open <https://mail.google.com>.
2. In the left sidebar, scroll down.
3. Click **+ Create new label**.
4. Type the label exactly as shown.
5. Click **Create**.

### Create these labels exactly

```text
priority
finances
accounts-subscriptions
account-security
unimportant
needs-review
processed
```

| # | Label Name | Purpose |
|---|---|---|
| 1 | `priority` | Real human conversations and important replies |
| 2 | `finances` | Receipts, orders, invoices, billing, payments, refunds, and money records |
| 3 | `accounts-subscriptions` | Service updates, subscriptions, apps, tools, healthchecks, quotas, and platform notices |
| 4 | `account-security` | OTPs, login alerts, password resets, verification codes, and suspicious activity |
| 5 | `unimportant` | Newsletters, promotions, coupons, cold outreach, and low-value clutter |
| 6 | `needs-review` | Ambiguous emails or AI failures that need your eyes |
| 7 | `processed` | Operational source of truth — marks emails as done |

> [!WARNING]
> Do **not** create a Gmail label named `important`. Gmail reserves that as a system label. This project uses `priority` instead.

> [!WARNING]
> Label names must be exact. Use lowercase and hyphens exactly as shown.

> [!TIP]
> The `processed` label is the most important operational label. Workflow A searches for `is:inbox -label:processed`, so emails with `processed` are ignored on future runs.

### Clean-slate reprocessing note

If you are migrating from an old V2.6 setup and want old inbox emails to be processed again:

1. Delete or remove old labels like `receipts`, `email-subs`, and `please-review` if you no longer want them.
2. Remove the old `processed` label from emails you want V4.0 to process again.
3. Make sure those emails are in the inbox.

> [!NOTE]
> Deleting a Gmail label does **not** delete emails. It only removes the label.

---

## STEP 2 — SET n8n ENVIRONMENT VARIABLES

Set these variables before starting n8n.

### PowerShell Windows

```powershell
# Shiny Gmail required/recommended variables
$env:GMAIL_ACCOUNT = "primary"
$env:GEMINI_API_KEY = "YOUR_GEMINI_API_KEY_HERE"
$env:GEMINI_MODEL = "gemini-3.1-flash-lite"
$env:GEMINI_TIMEOUT = "15000"
$env:TELEGRAM_CHAT_ID = "YOUR_CHAT_ID_HERE"

# n8n Code node requirements
$env:NODE_FUNCTION_ALLOW_BUILTIN = "crypto"
$env:N8N_BLOCK_ENV_ACCESS_IN_NODE = "false"

# Optional tuning
$env:GMAIL_BATCH_SIZE = "50"
$env:WAIT_BETWEEN_EMAILS = "5"
```

### Linux / macOS

```bash
export GMAIL_ACCOUNT="primary"
export GEMINI_API_KEY="YOUR_GEMINI_API_KEY_HERE"
export GEMINI_MODEL="gemini-3.1-flash-lite"
export GEMINI_TIMEOUT="15000"
export TELEGRAM_CHAT_ID="YOUR_CHAT_ID_HERE"

export NODE_FUNCTION_ALLOW_BUILTIN="crypto"
export N8N_BLOCK_ENV_ACCESS_IN_NODE="false"

export GMAIL_BATCH_SIZE="50"
export WAIT_BETWEEN_EMAILS="5"
```

> [!IMPORTANT]
> Restart n8n completely after changing environment variables. n8n reads them at startup.

> [!WARNING]
> Windows users should use `$env:VAR = "value"`, not `export VAR=value`.

### Optional tuning variables explained

| Variable | Default | Recommended | Purpose |
|---|---:|---:|---|
| `GMAIL_BATCH_SIZE` | `50` | `3` for testing, `50` for normal use | Max emails fetched per run |
| `WAIT_BETWEEN_EMAILS` | `5` | `5` to `15` | Delay between Gemini-using emails |
| `GEMINI_TIMEOUT` | `15000` | `15000` | Gemini HTTP timeout in milliseconds |

### Choosing a Gemini model

Recommended model order:

| Model | Notes |
|---|---|
| `gemini-3.1-flash-lite` | Recommended default for V4.0 |
| `gemini-3.5-flash` | Forward-compatible in V4.0; temperature is omitted automatically |
| `gemini-2.5-flash-lite` | Good fallback |
| `gemini-2.5-flash` | Capable fallback |
| `gemini-2.0-flash` | Older fallback |

Quick PowerShell test:

```powershell
$key = "YOUR_API_KEY"
$url = "https://generativelanguage.googleapis.com/v1beta/models/gemini-3.1-flash-lite:generateContent?key=$key"
$body = '{"contents":[{"parts":[{"text":"Say hello"}]}]}'

Invoke-RestMethod -Uri $url -Method POST -Body $body -ContentType "application/json"
```

If you get a valid response, the model works for your key. If you get `429`, try a different model or increase wait time.

---

## STEP 3 — SET UP NGROK FOR TELEGRAM

Telegram requires HTTPS webhook URLs. Local n8n on `http://localhost:5678` is not reachable by Telegram unless you use a tunnel.

### Setup summary

1. Create a free ngrok account.
2. Download ngrok.
3. Run your auth token command.
4. Claim or copy your static HTTPS domain.
5. Set `WEBHOOK_URL`.
6. Start ngrok before n8n.

### PowerShell example

```powershell
$env:WEBHOOK_URL = "https://your-static-domain.ngrok-free.dev"

Start-Process -FilePath "ngrok" -ArgumentList "http", "5678", "--domain", "your-static-domain.ngrok-free.dev" -WindowStyle Minimized
Start-Sleep -Seconds 3

npx n8n
```

> [!IMPORTANT]
> Both n8n and ngrok must stay running for Telegram commands to work.

> [!NOTE]
> ngrok is mainly required for Workflow C inbound Telegram webhooks. Outgoing Telegram messages from Workflow A work without ngrok.

---

## STEP 4 — RUN THE DATABASE SCHEMA

### Fresh V4.0 install

Use:

```text
database/schema.sql
```

This should contain the V4.0 schema.

### Existing V2.6 database choices

If you want to preserve old rules and history:

```text
Use migration_v2_6_to_v4_0.sql
```

If you want a clean slate and want inbox emails reprocessed under V4.0:

```text
Create a fresh database or clear old tables, then run the V4.0 schema.
```

### Neon SQL Editor steps

1. Open <https://console.neon.tech>.
2. Open your project.
3. Click **SQL Editor**.
4. Paste the full V4.0 schema.
5. Click **Run**.

### Expected verification

```text
3 tables:
- gmail_workflow_runs
- gmail_message_ledger
- gmail_sender_rules

3 unique constraints:
- workflow run primary key
- message ledger account/message unique key
- sender rule unique key

5 performance indexes:
- workflow run guard index
- message ledger exact lookup index
- message ledger thread reuse index
- message ledger sender history index
- sender rules enabled/priority index
```

---

## STEP 5 — CREATE CREDENTIALS IN n8n

Create these credentials before importing workflows.

### Credential 1: Gmail OAuth2

| Field | Value |
|---|---|
| Credential type | Gmail OAuth2 |
| Suggested name | Gmail OAuth2 |
| Client ID | Your Google OAuth Client ID |
| Client Secret | Your Google OAuth Client Secret |

After saving, click **Sign in with Google** and authorize Gmail read/modify/labels access.

### Credential 2: Neon Postgres

| Field | Value |
|---|---|
| Credential type | Postgres |
| Suggested name | Neon Postgres |
| Host | Your Neon host |
| Database | Your database name, often `neondb` |
| User | Your Neon username |
| Password | Your Neon password |
| SSL | Enabled / required |

Click **Test Connection**.

### Credential 3: Telegram Bot API

| Field | Value |
|---|---|
| Credential type | Telegram |
| Suggested name | Telegram Bot API |
| Bot Token | Token from BotFather |

---

## FILE-BY-FILE IMPORT INSTRUCTIONS

Import workflows one at a time.

> [!IMPORTANT]
> Do not activate until all workflows are imported, credentials are connected, and Workflow A has Workflow B's real ID.

### File 1: Workflow B — Single Email Processor Child 19 nodes

Import this first.

```text
workflows/workflow_b_email_processor.json
```

Connect credentials:

| Node | Credential |
|---|---|
| Check Ledger Completed | Neon Postgres |
| Gemini Classify Email | No n8n credential; uses `$env.GEMINI_API_KEY` |
| Apply Category Label | Gmail OAuth2 |
| Apply Processed Label | Gmail OAuth2 |
| Archive Email | Gmail OAuth2 |
| Upsert Ledger Completed | Neon Postgres |
| Upsert Ledger Failed | Neon Postgres |

Copy Workflow B's ID from the URL:

```text
http://localhost:5678/workflow/WORKFLOW_B_ID_HERE
```

### File 2: Workflow C — Telegram Rules Manager 16 nodes

```text
workflows/workflow_c_rules_manager.json
```

Connect credentials:

| Node | Credential |
|---|---|
| Telegram Trigger | Telegram Bot API |
| Upsert Rule | Neon Postgres |
| Disable Rule | Neon Postgres |
| Query Rules | Neon Postgres |
| Query Stats | Neon Postgres |
| Query Recent | Neon Postgres |
| Query Explain | Neon Postgres |
| Send Telegram Reply | Telegram Bot API |

### File 3: Workflow A — Daily Gmail Orchestrator 25 nodes

Import this last.

```text
workflows/workflow_a_orchestrator.json
```

Connect credentials:

| Node | Credential |
|---|---|
| Simple Run Guard Query | Neon Postgres |
| Mark Run Started | Neon Postgres |
| Get Gmail Labels | Gmail OAuth2 |
| Get Sender Rules | Neon Postgres |
| Fetch Inbox Emails | Gmail OAuth2 |
| Mark Run Finished | Neon Postgres |
| Send Telegram Summary | Telegram Bot API |
| Send Immediate Telegram Notice | Telegram Bot API |

Then open **Call Single Email Processor** and replace:

```text
REPLACE_WITH_WORKFLOW_B_ID
```

with Workflow B's actual ID.

---

## POST-IMPORT: CONNECTING CREDENTIALS & REPLACING PLACEHOLDERS

Check these before testing:

- Workflow B ID has been pasted into Workflow A.
- All Gmail nodes have Gmail OAuth2 credentials selected.
- All Postgres nodes have Neon Postgres credentials selected.
- All Telegram nodes have Telegram Bot API credentials selected.
- Supervisor/error workflow is set on all three workflows if you use one.
- Any harmless `Replace Me` import artifact node is deleted.

> [!TIP]
> If a node still says **Select Credential**, it is not ready.

---

## KNOWN IMPORT ISSUES

n8n imports can sometimes corrupt internal output mappings on multi-output nodes. The canvas may look correct while execution routes incorrectly.

### Symptoms

- `/rules` returns fallback help.
- Daily workflow finds emails but processes 0.
- Workflow jumps to final summary too early.
- IF node routes true to false or false to true.
- SplitInBatches sends data to done too early.
- A node throws `compareOperationFunctions[compareData.operation] is not a function`.

### Nodes to verify if behavior looks wrong

| Workflow | Node | Type |
|---|---|---|
| Workflow A | Env OK? | IF |
| Workflow A | Active Run? | IF |
| Workflow A | No Work? | IF |
| Workflow A | Split Emails | SplitInBatches |
| Workflow A | Used Gemini? | IF |
| Workflow B | Input Failed? | IF |
| Workflow B | Needs Gemini? | IF |
| Workflow B | Label Ready? | IF |
| Workflow C | Route Command | Switch |

### Fix

For the affected node only:

1. Note its settings.
2. Delete it.
3. Add a new node of the same type.
4. Reconfigure it.
5. Redraw connections.
6. Save and retest.

---

## ACTIVATION ORDER

Activate in this exact order:

1. **Workflow B** — Email Processor
2. **Workflow C** — Telegram Rules Manager
3. **Workflow A** — Daily Orchestrator

> [!WARNING]
> Workflow A calls Workflow B. If Workflow B is inactive or the ID is wrong, emails will fail.

---

## DEPLOYMENT CHECKLIST

### Pre-import

- [ ] Gmail OAuth2 Client ID and Secret ready
- [ ] Telegram Bot Token ready
- [ ] Telegram Chat ID ready
- [ ] Gemini API Key ready and tested
- [ ] Neon Postgres connection details ready
- [ ] All 7 Gmail labels created
- [ ] Environment variables set
- [ ] ngrok installed and running
- [ ] V4.0 schema executed
- [ ] Schema verification passed: 3 tables, 3 unique constraints, 5 indexes

### Post-import

- [ ] Workflow B ID pasted into Workflow A
- [ ] All credentials connected
- [ ] Error Workflow set if using Supervisor V23
- [ ] n8n restarted

### Smoke tests

- [ ] `/help` works
- [ ] `/rules` works
- [ ] `/stats` works
- [ ] `/recent` works
- [ ] `/recent failed` works
- [ ] `/recent warnings` works
- [ ] `/explain sender@example.com` works
- [ ] Small batch test with 1-3 emails passes
- [ ] Gmail category labels applied correctly
- [ ] `processed` label applied correctly
- [ ] Emails archived correctly
- [ ] Ledger entries created

---

## VERIFICATION & SMOKE TESTS

Set low batch size during tests:

```powershell
$env:GMAIL_BATCH_SIZE = "3"
```

Restart n8n after changing it.

### Test 1: Telegram commands

Run:

```text
/help
/rules
/stats
/recent
/recent failed
/recent warnings
/whitelist test@example.com
/rules
/unwhitelist test@example.com
/explain test@example.com
```

### Test 2: Workflow A small batch

Run Workflow A manually.

Pass if:

- Telegram summary arrives.
- Emails receive one category label.
- Emails receive `processed`.
- Emails are archived.
- Ledger rows are created.

### Test 3: Database verification

Run in Neon:

```sql
SELECT run_id, status, total_discovered, completed_count, failed_count, review_count, started_at
FROM gmail_workflow_runs
ORDER BY started_at DESC
LIMIT 5;

SELECT gmail_message_id, category, source, confidence, status, updated_at
FROM gmail_message_ledger
ORDER BY updated_at DESC
LIMIT 10;
```

### Test 4: Sender rule match

```text
/whitelist your-own-email@gmail.com
```

Send yourself an email, run Workflow A, and confirm:

- Category is `priority`
- Source is `whitelist`

Clean up:

```text
/unwhitelist your-own-email@gmail.com
```

### Test 5: Domain rule match

```text
/blacklistdomain example.com
```

Run Workflow A against an email from that domain/subdomain and confirm:

- Category is `unimportant`
- Source is `blacklist`

Clean up:

```text
/unblacklistdomain example.com
```

### Test 6: Account security classification

Send yourself a test email with a subject like:

```text
Your verification code is 123456
```

Pass if it becomes `account-security`, or `needs-review` if uncertain.

---

## TROUBLESHOOTING

### Gmail says you cannot create `important`

Use `priority`. Gmail reserves `important`.

### Telegram Rules Manager cannot activate

Set `WEBHOOK_URL`, start ngrok, restart n8n, then activate Workflow C again.

### Telegram commands only return fallback help

Check `TELEGRAM_CHAT_ID`. If it is correct, inspect Workflow C execution trace. If Route Command sends everything to fallback, recreate the Switch node.

### Daily workflow finds emails but completes 0

Recreate **Split Emails** and wire:

- Loop output → Call Single Email Processor
- Done output → Build Final Summary

### Workflow jumps to no-work summary even when emails exist

Recreate **No Work?** IF node.

### Gemini returns 429 Too Many Requests

Try:

```powershell
$env:WAIT_BETWEEN_EMAILS = "10"
$env:GMAIL_BATCH_SIZE = "10"
```

Restart n8n. You can also try another Gemini model.

### Run guard blocks new runs

```sql
UPDATE gmail_workflow_runs
SET status = 'failed', finished_at = NOW(), updated_at = NOW()
WHERE status = 'running';
```

### `$env is not defined`

```powershell
$env:N8N_BLOCK_ENV_ACCESS_IN_NODE = "false"
```

Restart n8n.

### `crypto.randomUUID is not a function`

```powershell
$env:NODE_FUNCTION_ALLOW_BUILTIN = "crypto"
```

Restart n8n.

### Batch full warning appears

Workflow A fetched exactly `GMAIL_BATCH_SIZE` emails. More may be waiting. Run again manually or increase batch size.

### Duplicate email processing

Check whether the `processed` label was manually removed from the email.

---

## MONTHLY COST ESTIMATE

| Component | Monthly Cost | Notes |
|---|---|---|
| Gemini API | $0.00 | Free tier should cover typical personal email volume |
| Neon PostgreSQL | $0.00 | Free tier is enough for personal usage |
| Gmail API | $0.00 | Free for personal use |
| Telegram Bot API | $0.00 | Free |
| n8n self-hosted | $0.00 | Runs on your own machine |
| ngrok | $0.00 | Free tier is enough for personal use |
| **TOTAL** | **$0.00** | Free tier everything |

---

## ARCHITECTURE SUMMARY

| Workflow | Nodes | Purpose | Runs When |
|---|---:|---|---|
| A — Daily Orchestrator | 25 | Fetch inbox, apply rules, batch emails, call child, summarize | Daily at 6:00 AM Asia/Manila or manual test |
| B — Email Processor | 19 | Classify one email, apply labels, archive, update ledger | Called by A per email |
| C — Telegram Rules Manager | 16 | Manage rules and inspect history via Telegram | On Telegram message |

### System totals

| Metric | Value |
|---|---:|
| Total executable nodes | 60 |
| Workflows | 3 |
| Database tables | 3 |
| Performance indexes | 5 |
| Gmail labels | 7 |
| External APIs | Gmail, Gemini, Telegram |

### Source of truth

```text
Gmail processed label = operational source of truth
Postgres ledger = audit / idempotency / thread-reuse helper
Postgres rules = sender/domain control plane
Postgres runs = run tracking and summary history
```

### Gmail commit order

```text
1. Apply CATEGORY label
2. Apply PROCESSED label
3. Archive email by removing INBOX label
4. Upsert ledger completed
```

Archive failure is a warning because the email already has category + processed labels.

### Classification priority

```text
1. Exact sender rules
2. Domain rules
3. Exact ledger replay
4. Safe same-domain thread reuse
5. Gemini AI classification
6. needs-review fallback for uncertainty or failure
```

---

## ENVIRONMENT VARIABLE QUICK REFERENCE

### Required / Recommended

| Variable | Example Value | Purpose |
|---|---|---|
| `GMAIL_ACCOUNT` | `primary` | Gmail account identifier; defaults to primary if omitted |
| `GEMINI_API_KEY` | `AIzaSy...` | Google AI Studio API key |
| `GEMINI_MODEL` | `gemini-3.1-flash-lite` | Gemini model name |
| `TELEGRAM_CHAT_ID` | `1234567890` | Authorized Telegram chat ID |
| `NODE_FUNCTION_ALLOW_BUILTIN` | `crypto` | Allows crypto module in Code nodes |
| `N8N_BLOCK_ENV_ACCESS_IN_NODE` | `false` | Allows `$env` access in Code nodes |
| `WEBHOOK_URL` | `https://xxx.ngrok-free.dev` | Public HTTPS URL for Telegram webhooks |

### Optional

| Variable | Default | Purpose |
|---|---|---|
| `GMAIL_BATCH_SIZE` | `50` | Max emails fetched per run |
| `WAIT_BETWEEN_EMAILS` | `5` | Seconds between Gemini-using email processing |
| `GEMINI_TIMEOUT` | `15000` | Gemini HTTP request timeout in milliseconds |

---

## SAMPLE STARTUP SCRIPT

Save as `start-n8n.ps1` and run with:

```powershell
.\start-n8n.ps1
```

```powershell
# Shiny Gmail Automation V4.0 Lean startup

$env:NODE_FUNCTION_ALLOW_BUILTIN = "crypto"
$env:N8N_BLOCK_ENV_ACCESS_IN_NODE = "false"

$env:WEBHOOK_URL = "https://your-static-domain.ngrok-free.dev"

$env:GMAIL_ACCOUNT = "primary"
$env:GEMINI_API_KEY = "YOUR_GEMINI_API_KEY_HERE"
$env:GEMINI_MODEL = "gemini-3.1-flash-lite"
$env:GEMINI_TIMEOUT = "15000"
$env:TELEGRAM_CHAT_ID = "YOUR_CHAT_ID_HERE"

$env:GMAIL_BATCH_SIZE = "50"
$env:WAIT_BETWEEN_EMAILS = "5"

Start-Process -FilePath "ngrok" -ArgumentList "http", "5678", "--domain", "your-static-domain.ngrok-free.dev" -WindowStyle Minimized
Start-Sleep -Seconds 3

npx n8n
```

> [!WARNING]
> Replace `YOUR_GEMINI_API_KEY_HERE`, `YOUR_CHAT_ID_HERE`, and `your-static-domain.ngrok-free.dev` before first use.

---

## DEPLOYMENT COMPLETE

If all smoke tests pass, your Shiny Gmail Automation V4.0 Lean is ready for production.

- Workflow A runs daily at 6:00 AM Asia/Manila.
- Workflow B processes one email safely at a time.
- Workflow C lets you manage rules from Telegram.
- Emails that fail classification go to `needs-review`.
- Archive failures become warnings, not silent failures.
- Your Gmail gets cleaner every day.

Enjoy your clean inbox! 🚀
