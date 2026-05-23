# SHINY GMAIL AUTOMATION V2.6 LEAN — COMPLETE DEPLOYMENT GUIDE

**Date:** May 2026 | **Architecture Version:** V2.6 Lean | **54 Nodes** | **3 Workflows**
**Stack:** n8n + Gemini + Neon PostgreSQL + Gmail OAuth2 + Telegram Bot API + ngrok

---

## TABLE OF CONTENTS

- [Step 0 — Gather Your Credentials & IDs](#step-0-gather-your-credentials--ids)
- [Step 1 — Create Gmail Labels](#step-1-create-gmail-labels)
- [Step 2 — Set n8n Environment Variables](#step-2-set-n8n-environment-variables)
- [Step 3 — Set Up ngrok for Telegram](#step-3-set-up-ngrok-for-telegram)
- [Step 4 — Run the Database Schema](#step-4-run-the-database-schema)
- [Step 5 — Create Credentials in n8n](#step-5-create-credentials-in-n8n)
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

## STEP 0: GATHER YOUR CREDENTIALS & IDs

Before importing anything into n8n, collect ALL of these. You will need every single one during the import process.

### 1. Gmail OAuth2 Credentials

**Where to get it:**
- Go to https://console.cloud.google.com
- Create a new project (or use an existing one)
- Go to **APIs & Services** → **Library** → Search for **Gmail API** → **Enable** it
- Go to **APIs & Services** → **Credentials** → **Create Credentials** → **OAuth 2.0 Client ID**
- Application type: **Web application**
- Add an Authorized redirect URI: `http://localhost:5678/rest/oauth2-credential/callback` (adjust if your n8n runs on a different host/port)
- Copy the **Client ID** and **Client Secret**

**What they look like:**
- Client ID: `123456789-abcdef.apps.googleusercontent.com`
- Client Secret: `GOCSPX-xxxxx`

### 2. Telegram Bot Token

**Where to get it:** Open Telegram → message **@BotFather** → type `/newbot` → follow the prompts.

**What it looks like:** `7123456789:AAH...`

### 3. Telegram Chat ID

**Where to get it:**

**Option A — Use @userinfobot:**
1. Open Telegram
2. Search for **@userinfobot**
3. Send it any message (like "hi")
4. It replies with your chat ID

**Option B — Use the getUpdates API:**

> [!NOTE]
> This method only works if your bot does NOT have an active webhook. If you already activated Workflow C, use Option A instead.

Send any message to your bot, then open this URL in your browser:
`https://api.telegram.org/bot<YOUR_TOKEN_HERE>/getUpdates`
Look for `"chat":{"id":XXXXXXX}` — that number is your Chat ID.

**What it looks like:** A number like `1234567890`

### 4. Gemini API Key

**Where to get it:**
- Go to https://aistudio.google.com/apikey
- Click **Create API Key**
- Select your Google Cloud project (or create a new one)
- Copy the key

**What it looks like:** `AIzaSy...`

> [!WARNING]
> This is a Google AI Studio API key, NOT a Google Cloud API key. Make sure you're on aistudio.google.com.

> [!CAUTION]
> Do **not** link a billing account to your Google AI Studio project. Enabling billing disables the free tier entirely — every request becomes billable from the first token. Keep your project on the free tier.

### 5. Postgres Connection Details

**Where to get it:** Go to https://console.neon.tech → Your Project → **Connection Details**

You will need:
- **Host:** `ep-xxx.ap-southeast-1.aws.neon.tech`
- **Database:** `neondb`
- **User:** your username
- **Password:** your password

### 6. Error Workflow ID (Optional)

If you have a separate error-monitoring workflow (e.g., a supervisor/alerting system), note its workflow ID. You can set it as the Error Workflow for all 3 Shiny Gmail workflows so catastrophic failures are caught externally.

If you don't have one, you can skip this — the system still works without it.

> [!TIP]
> Save all these values in a temporary notepad file. You will paste them into n8n during setup. **Delete the notepad file after deployment for security.**

---

## STEP 1: CREATE GMAIL LABELS

Shiny Gmail uses 6 custom labels to organize your inbox. These must exist in Gmail **before** the first run.

**How to create a label in Gmail:**
1. Open https://mail.google.com
2. In the left sidebar, scroll down and click **+ Create new label**
3. Type the label name **exactly as shown below** (lowercase!)
4. Click **Create**

**Create these 6 labels:**

| # | Label Name | Purpose |
|---|-----------|---------|
| 1 | `unimportant` | Spam, newsletters, promotions, marketing |
| 2 | `receipts` | Purchase receipts, invoices, shipping notices |
| 3 | `priority` | Personal, work, bank alerts, security, OTP |
| 4 | `email-subs` | Services you subscribed to and may want to manage |
| 5 | `please-review` | Ambiguous emails or AI failures — needs your eyes |
| 6 | `processed` | Source of truth — marks emails as "done" |

> [!WARNING]
> **Do not try to create a label called `important`.** Gmail reserves that name as a system label (the yellow arrow/marker). You will get the error: *"Sorry, you can't create a label named 'important'."* This project uses `priority` instead.

> [!WARNING]
> Label names must be **exactly** as shown — all lowercase, with the hyphen in `email-subs` and `please-review`. Gmail labels are case-sensitive!

> [!TIP]
> The `processed` label is the most important one. It's how the system knows an email has been handled. **Never delete this label** while the automation is active.

---

## STEP 2: SET n8n ENVIRONMENT VARIABLES

These environment variables must be set **before** starting n8n.

### PowerShell (Windows)

```powershell
# Required for Shiny Gmail
$env:GMAIL_ACCOUNT = "primary"
$env:GEMINI_API_KEY = "YOUR_GEMINI_API_KEY_HERE"
$env:GEMINI_MODEL = "gemini-3.1-flash-lite"
$env:TELEGRAM_CHAT_ID = "YOUR_CHAT_ID_HERE"
$env:TELEGRAM_ALLOW_GROUP_RULES = "false"

# Required for n8n Code nodes
$env:NODE_FUNCTION_ALLOW_BUILTIN = "crypto"
$env:N8N_BLOCK_ENV_ACCESS_IN_NODE = "false"
```

### Linux / macOS

```bash
export GMAIL_ACCOUNT="primary"
export GEMINI_API_KEY="YOUR_GEMINI_API_KEY_HERE"
export GEMINI_MODEL="gemini-3.1-flash-lite"
export TELEGRAM_CHAT_ID="YOUR_CHAT_ID_HERE"
export TELEGRAM_ALLOW_GROUP_RULES="false"

export NODE_FUNCTION_ALLOW_BUILTIN="crypto"
export N8N_BLOCK_ENV_ACCESS_IN_NODE="false"
```

### Optional tuning variables

```powershell
# Limit emails fetched per run (default: 50)
$env:GMAIL_BATCH_SIZE = "50"

# Seconds between processing each email (default: 1, recommended: 5)
$env:WAIT_BETWEEN_EMAILS = "5"
```

> [!WARNING]
> `WAIT_BETWEEN_EMAILS` of 5 seconds is recommended to avoid Gemini API rate limits (429 errors), especially on free tier accounts with limited RPM (requests per minute).

> [!NOTE]
> After setting these, restart n8n completely. Environment variables are only read at startup.

> [!WARNING]
> **Windows users:** Do NOT use `export VAR=value` — that's Linux/Mac syntax. Always use `$env:VAR = "value"` in PowerShell.

> [!TIP]
> Add all environment variables to a startup script so they're set automatically every time you launch n8n. See the [Sample Startup Script](#sample-startup-script) section at the end of this guide.

### Choosing a Gemini Model

Gemini free tier quotas vary by region, account, and model. If one model returns `429 Too Many Requests`, try a different one.

**How to test a model quickly (PowerShell):**

```powershell
$key = "YOUR_API_KEY"
$url = "https://generativelanguage.googleapis.com/v1beta/models/gemini-3.1-flash-lite:generateContent?key=$key"
$body = '{"contents":[{"parts":[{"text":"Say hello"}]}]}'

Invoke-RestMethod -Uri $url -Method POST -Body $body -ContentType "application/json"
```

If you get a response → the model works for your account. If you get 429 → try another model.

**Models to try (in order of recommendation):**

| Model | Notes |
|-------|-------|
| `gemini-3.1-flash-lite` | Recommended — highest free tier RPD in many regions |
| `gemini-2.5-flash-lite` | Good alternative |
| `gemini-2.5-flash` | Capable but may have lower free tier limits |
| `gemini-2.0-flash` | Original default — may have very low quota in some regions |

Set the working model in your environment:

```powershell
$env:GEMINI_MODEL = "gemini-3.1-flash-lite"
```

---

## STEP 3: SET UP NGROK FOR TELEGRAM

Telegram requires HTTPS webhook URLs to deliver messages to your bot. Since self-hosted n8n runs on `http://localhost:5678`, Telegram cannot reach it directly. You need an HTTPS tunnel.

### Why ngrok?

| Feature | ngrok (free) |
|---------|-------------|
| HTTPS tunnel | ✅ |
| Free static domain | ✅ (1 permanent domain) |
| Works on Windows | ✅ |
| No configuration files | ✅ |

### Setup Instructions

#### 1. Create a free account

Go to https://ngrok.com and sign up.

#### 2. Download and install ngrok

Download the Windows version and unzip it to a convenient location (e.g., `C:\ngrok\`).

#### 3. Authenticate ngrok

Open PowerShell and run the auth command from your ngrok dashboard:

```powershell
ngrok authtoken YOUR_AUTH_TOKEN_HERE
```

This is a one-time setup.

#### 4. Claim your free static domain

1. Go to your ngrok Dashboard → **Domains**
2. Claim your free static domain (e.g., `something-random.ngrok-free.dev`)
3. Note this domain — you'll use it in your startup script

#### 5. Add to your environment variables

```powershell
$env:WEBHOOK_URL = "https://your-static-domain.ngrok-free.dev"
```

#### 6. Start ngrok before n8n

```powershell
Start-Process -FilePath "ngrok" -ArgumentList "http", "5678", "--domain", "your-static-domain.ngrok-free.dev" -WindowStyle Minimized
Start-Sleep -Seconds 3
npx n8n
```

> [!IMPORTANT]
> Both n8n **and** ngrok must stay running. If either is closed, the Telegram webhook will stop working. You can minimize both windows, but do not close them.

> [!NOTE]
> ngrok is only needed for Telegram webhooks (Workflow C). Workflows A and B send outgoing Telegram messages — those work fine on localhost without ngrok.

> [!NOTE]
> Your laptop must be on and awake for everything to work. ngrok is a tunnel, not a hosting service. If your laptop is off or asleep, n8n won't run and the tunnel disconnects.

---

## STEP 4: RUN THE DATABASE SCHEMA

The Shiny Gmail tables need to be created in your Postgres database.

### Instructions:

1. Go to https://console.neon.tech and open your project
2. Click **"SQL Editor"** in the left sidebar
3. Open the schema file: `database/schema.sql`
4. Copy the **entire** contents and paste it into the SQL Editor
5. Click **"Run"** — you should see "Query executed successfully"

### Verify the schema:

After running the schema, you should see:

```
✅ 3 tables: gmail_message_ledger, gmail_sender_rules, gmail_workflow_runs
✅ 3 unique constraints (1 PK + 2 UQ)
✅ 3 performance indexes
✅ All row counts = 0
```

> [!TIP]
> The schema uses `IF NOT EXISTS` on all statements, so it's safe to run multiple times.

---

## STEP 5: CREATE CREDENTIALS IN n8n

Before importing any workflow, create these credentials in n8n.

### How to create a credential:
1. In n8n, click the hamburger menu (☰) → **Credentials**
2. Click **"+ Add Credential"**
3. Search for the credential type
4. Fill in the fields
5. Click **"Save"**

### CREDENTIAL 1: Gmail OAuth2

| Field | Value |
|-------|-------|
| **Credential type** | Gmail OAuth2 |
| **Suggested name** | Gmail OAuth2 |
| **Client ID** | Your Google OAuth Client ID |
| **Client Secret** | Your Google OAuth Client Secret |

After filling in the fields:
1. Click **"Sign in with Google"**
2. Select your Gmail account
3. Grant the requested permissions (read, modify, labels)
4. You should see a green "Connected" message

> [!TIP]
> If the OAuth flow fails, make sure your redirect URI in Google Cloud Console matches your n8n URL exactly: `http://localhost:5678/rest/oauth2-credential/callback`

### CREDENTIAL 2: Neon Postgres

| Field | Value |
|-------|-------|
| **Credential type** | Postgres |
| **Suggested name** | Neon Postgres |
| **Host** | Your Neon host (ep-xxx.aws.neon.tech) |
| **Database** | neondb |
| **User** | Your Neon username |
| **Password** | Your Neon password |
| **SSL** | Toggle **ON** (required for Neon) |

After saving, click **"Test Connection"** to verify.

### CREDENTIAL 3: Telegram Bot API

| Field | Value |
|-------|-------|
| **Credential type** | Telegram |
| **Suggested name** | Telegram Bot API |
| **Bot Token** | Your bot token from BotFather |

---

## FILE-BY-FILE IMPORT INSTRUCTIONS

Import each workflow one at a time in the order shown below.

### How to import a workflow:
1. In n8n, go to an empty canvas (click **+ New Workflow**)
2. Click the menu: **···** → **Import from File** → select your `.json` file
3. Or: open the JSON file, copy all content, and press **Ctrl+V** on the empty canvas
4. **DO NOT activate yet** — wait until ALL 3 are imported and configured

---

### FILE 1: WORKFLOW B — Single Email Processor Child (18 nodes)

⭐ **Import this FIRST** — Workflow A needs its workflow ID.

**File:** `workflows/workflow_b_email_processor.json`

After import, connect these credentials:

| Node | Credential |
|------|-----------|
| Check Ledger Completed | Neon Postgres |
| Gemini Classify Email | No credential needed (uses API key via query parameter) |
| Apply Category Label | Gmail OAuth2 |
| Apply Processed Label | Gmail OAuth2 |
| Upsert Ledger Completed | Neon Postgres |
| Upsert Ledger Failed | Neon Postgres |

Then **note the workflow ID** from the URL bar: `http://localhost:5678/workflow/________`

Write it down — you need this for Workflow A!

> [!NOTE]
> The Gemini API key is passed as a query parameter using `$env.GEMINI_API_KEY`. No n8n credential is needed for the Gemini node.

---

### FILE 2: WORKFLOW C — Telegram Rules Manager (11 nodes)

**File:** `workflows/workflow_c_rules_manager.json`

After import, connect these credentials:

| Node | Credential |
|------|-----------|
| Telegram Rules Trigger | Telegram Bot API |
| List Sender Rules | Neon Postgres |
| Upsert Sender Rule | Neon Postgres |
| Disable Sender Rule | Neon Postgres |
| Send Telegram Reply | Telegram Bot API |

---

### FILE 3: WORKFLOW A — Daily Gmail Orchestrator (25 nodes)

⭐ **Import this LAST** — it references the child workflow ID.

**File:** `workflows/workflow_a_orchestrator.json`

After import, connect these credentials:

| Node | Credential |
|------|-----------|
| Simple Run Guard Query | Neon Postgres |
| Mark Run Started | Neon Postgres |
| Get Gmail Labels | Gmail OAuth2 |
| Load Sender Rules | Neon Postgres |
| Fetch Inbox Emails | Gmail OAuth2 |
| Mark Run Finished | Neon Postgres |
| Send Telegram Summary | Telegram Bot API |

Then find the **Call Single Email Processor** node and set the workflow ID:
- Open the node
- In the **"Workflow ID"** field, paste the **Workflow B ID** you noted from File 1

---

## POST-IMPORT: CONNECTING CREDENTIALS & REPLACING PLACEHOLDERS

After importing all 3 workflows, verify every placeholder is replaced.

### Workflow ID Replacements

| Placeholder | Replace With | Where |
|-------------|-------------|-------|
| Child workflow ID | Workflow B's actual ID | Workflow A → Call Single Email Processor node |
| Error Workflow (optional) | Your error monitoring workflow ID | All 3 workflows → Settings → Error Workflow |

**How to set the Error Workflow (optional):**
1. Open each workflow
2. Click **Settings** (gear icon)
3. Find **"Error Workflow"**
4. Paste your error monitoring workflow ID
5. Save

### Credential Verification

Double-click **every** Postgres, Gmail, and Telegram node across all 3 workflows. Make sure **none** show "Select Credential" — they should all have a named credential selected.

Quick count of credentials to connect:
- **Workflow A:** 4 Postgres + 2 Gmail + 1 Telegram = **7 nodes**
- **Workflow B:** 3 Postgres + 2 Gmail = **5 nodes**
- **Workflow C:** 3 Postgres + 2 Telegram = **5 nodes**
- **Total: 17 credential connections**

### Delete Placeholder Nodes

If you see a node called **"Replace Me"** after import, it's a harmless import artifact. Right-click and delete it.

---

## KNOWN IMPORT ISSUES

> [!IMPORTANT]
> This is critical to read before testing. n8n has a known behavior where imported JSON files can result in **internally corrupted wiring** on nodes with multiple outputs.

### What happens

When you import workflow JSON files, nodes with multiple outputs (IF, Switch, SplitInBatches) may have their output mappings scrambled internally. The connections **look correct in the visual editor** but actually route data to the wrong output during execution.

### Symptoms

You may see behavior like:

- Telegram `/rules` returns "Send /help for commands" instead of listing rules
- The daily workflow finds emails but processes 0
- The workflow jumps straight to the final summary, skipping processing entirely
- An IF node routes `true` to the `false` path or vice versa
- SplitInBatches sends items to the done branch instead of the loop branch
- A node throws: `compareOperationFunctions[compareData.operation] is not a function`

### How to detect it

1. Run a test execution
2. Open the execution trace in n8n
3. Follow the path the data actually took
4. Compare it with what the node should have done

**Trust the execution trace, not the visual wires.**

### How to fix it

For each affected node:

1. Note the node's settings and downstream connections
2. **Delete** the corrupted node
3. Add a **brand new** node of the same type
4. Reconfigure the settings exactly as before
5. **Re-draw all connections** manually
6. Save and retest

Recreating the node forces n8n to generate fresh internal output mappings.

### Nodes to verify after import

| Workflow | Node | Type |
|---|---|---|
| Workflow A | Env OK? | IF |
| Workflow A | Active Run? | IF |
| Workflow A | Labels OK? | IF |
| Workflow A | No Work? | IF |
| Workflow A | Split Emails | SplitInBatches |
| Workflow A | Should Skip Mark Finished? | IF |
| Workflow B | Input Failed? | IF |
| Workflow B | Ledger Completed? | IF |
| Workflow B | Needs Gemini? | IF |
| Workflow B | Label Ready? | IF |
| Workflow C | Route Command | Switch |

> [!TIP]
> Do not delete and recreate every node preemptively. Run your smoke tests first. If a test fails unexpectedly, check the execution trace to see if data routed to the wrong branch — then recreate only that specific node.

---

## ACTIVATION ORDER

After all workflows are imported, credentials connected, and placeholders replaced, activate (publish) the workflows in this **exact** order:

| Order | Workflow | Why This Order |
|-------|----------|----------------|
| 1️⃣ | **Workflow B** — Email Processor | Must be active before A calls it |
| 2️⃣ | **Workflow C** — Telegram Rules Manager | Independent, but needs ngrok running |
| 3️⃣ | **Workflow A** — Daily Orchestrator | **Activate last** — calls B for every email |

**How to activate:** Open each workflow → click the toggle in the top-right corner → OR click the **"Publish"** button.

> [!WARNING]
> If Workflow A is activated before Workflow B is published, every email will fail with "Workflow not found."

> [!NOTE]
> When activating Workflow C, n8n registers the Telegram webhook. If you see the error *"An HTTPS URL must be provided for webhook"*, your `$env:WEBHOOK_URL` is not set or ngrok is not running. See [Troubleshooting](#troubleshooting).

---

## DEPLOYMENT CHECKLIST

### PRE-IMPORT
- ☐ Gmail OAuth2 Client ID and Secret ready
- ☐ Telegram Bot Token ready
- ☐ Telegram Chat ID ready
- ☐ Gemini API Key ready and tested (from Google AI Studio, no billing linked)
- ☐ Gemini model tested and working (see [Choosing a Gemini Model](#choosing-a-gemini-model))
- ☐ Neon Postgres connection details ready
- ☐ All 6 Gmail labels created (`unimportant`, `receipts`, `priority`, `email-subs`, `please-review`, `processed`)
- ☐ Confirmed you did NOT try to create a label called `important`
- ☐ Environment variables set
- ☐ `NODE_FUNCTION_ALLOW_BUILTIN = "crypto"` set
- ☐ `N8N_BLOCK_ENV_ACCESS_IN_NODE = "false"` set
- ☐ ngrok installed, authenticated, and static domain claimed
- ☐ `WEBHOOK_URL` environment variable set with ngrok domain
- ☐ n8n restarted after setting environment variables
- ☐ Schema SQL executed on Neon
- ☐ Schema verification passed (3 tables, 3 constraints, 3 indexes)

### CREDENTIALS CREATED IN n8n
- ☐ Gmail OAuth2 credential created and authorized
- ☐ Neon Postgres credential created and tested
- ☐ Telegram Bot API credential created

### WORKFLOWS IMPORTED (in order)
- ☐ File 1: Workflow B (Child) — imported, all 5 credentials connected
- ☐ File 2: Workflow C (Rules Manager) — imported, all 5 credentials connected
- ☐ File 3: Workflow A (Orchestrator) — imported, all 7 credentials connected

### POST-IMPORT
- ☐ Workflow B's ID pasted into Workflow A → Call Single Email Processor node
- ☐ Error Workflow set in all 3 workflows (optional)
- ☐ All 17 credential connections verified (no "Select Credential" dropdowns)
- ☐ "Replace Me" placeholder node deleted (if present)
- ☐ n8n restarted

### SMOKE TESTS BEFORE ACTIVATION
- ☐ Telegram `/help` works
- ☐ Telegram `/rules` lists rules (not fallback message)
- ☐ All 9 Telegram commands work
- ☐ Small batch test with 3 emails passes
- ☐ Gmail labels applied correctly
- ☐ Postgres ledger entries created
- ☐ Sender rule match works (skips Gemini)
- ☐ Domain rule match works (subdomain matching)
- ☐ Emails show correct sender, subject, snippet (not empty)

### ACTIVATION
- ☐ Workflow B activated first
- ☐ Workflow C activated second
- ☐ Workflow A activated last

---

## VERIFICATION & SMOKE TESTS

After importing all workflows and connecting credentials, run these tests **before** activating for production.

> [!TIP]
> Set a low batch size during testing to avoid processing too many emails:
> ```powershell
> $env:GMAIL_BATCH_SIZE = "3"
> ```
> Increase it after all tests pass.

### TEST 1: TELEGRAM COMMANDS

This is the safest test — no emails are touched.

Send these commands to your bot one by one:

| # | Send to bot | Expected response |
|---|---|---|
| 1a | `/help` | Command list with 🧹 Shiny Gmail Manager header |
| 1b | `/rules` | "No active rules yet" or current rules |
| 1c | `/whitelist test@example.com` | ✅ Rule saved |
| 1d | `/rules` | Shows the rule you just created |
| 1e | `/unwhitelist test@example.com` | ✅ Rule disabled |
| 1f | `/blacklistdomain spam.com` | ✅ Rule saved |
| 1g | `/unblacklistdomain spam.com` | ✅ Rule disabled |
| 1h | `/unreceipt test@example.com` | "No matching active rule" |
| 1i | `/unreview test@example.com` | "No matching active rule" |

**If all commands return "Send /help for commands"** instead of the expected response, see [Troubleshooting: Telegram commands only return fallback](#telegram-commands-only-return-send-help-for-commands).

### TEST 2: SMALL BATCH — GEMINI CLASSIFICATION

1. Open **Workflow A** in n8n
2. Click **Test Workflow** (▶ button)
3. Wait for it to complete

**Pass if:**
- Telegram summary arrives showing emails found
- Each email gets a **category label** AND the **processed label** in Gmail
- Search Gmail for `label:processed` to confirm

### TEST 3: DATABASE VERIFICATION

Run these in your Neon SQL Editor:

```sql
-- Check workflow runs
SELECT run_id, status, total_discovered, total_completed, started_at
FROM gmail_workflow_runs ORDER BY started_at DESC LIMIT 5;

-- Check message ledger
SELECT gmail_message_id, category, source, status, processed_at
FROM gmail_message_ledger ORDER BY created_at DESC LIMIT 5;
```

**Pass if:** Runs show `status = 'completed'`, ledger shows `source = 'ai'` with real categories.

### TEST 4: SENDER RULE MATCH (Skips Gemini)

1. In Telegram: `/whitelist your-own-email@gmail.com`
2. Send yourself an email
3. Clear any stuck runs: `UPDATE gmail_workflow_runs SET status = 'failed', finished_at = NOW() WHERE status = 'running';`
4. Run Workflow A manually

**Pass if:** Email categorized as `priority` with `source = 'rule'` (not `ai`).

Clean up: `/unwhitelist your-own-email@gmail.com`

### TEST 5: DOMAIN RULE MATCH (Subdomain matching)

1. In Telegram: `/blacklistdomain` + a domain you have emails from
2. Run Workflow A manually

**Pass if:** Email categorized as `unimportant` with `source = 'domain_blacklist'`.

Clean up: `/unblacklistdomain the-domain.com`

### TEST 6: METADATA VERIFICATION

During any test run, open **Workflow A** → **Executions** → click on **Normalize Emails And Apply Rules** and verify that emails show:
- ✅ A real sender address (not empty)
- ✅ A real subject line (not "(no subject)" when there is one)
- ✅ A real snippet

If all fields are empty, see [Troubleshooting: All emails show empty sender](#all-emails-show-empty-sender-and-no-subject).

### TEST 7: GEMINI FAILURE FALLBACK (Optional)

This was likely already proven if you experienced 429 errors during model testing. If you want to test explicitly:

1. Temporarily set `$env:GEMINI_API_KEY = "invalid_key_for_testing"`
2. Restart n8n and run Workflow A

**Pass if:** Email routed to `please-review` with `source = 'ai_failed'`.

**Remember to restore your real API key after this test!**

---

## TROUBLESHOOTING

### Gmail says you cannot create `important`

Gmail reserves `important` as a system label. This project uses `priority` instead. Do not attempt to create a label called `important`.

---

### Telegram Rules Manager cannot activate

**Error:** `Bad Request: bad webhook: An HTTPS URL must be provided for webhook`

**Cause:** Telegram requires HTTPS webhook URLs. Local n8n URLs like `http://localhost:5678` are not accepted.

**Fix:**
1. Make sure ngrok is installed and running
2. Set the webhook URL environment variable:
   ```powershell
   $env:WEBHOOK_URL = "https://your-static-domain.ngrok-free.dev"
   ```
3. Restart n8n
4. Try activating Workflow C again

---

### 502 Bad Gateway when accessing ngrok URL

**Cause:** Either n8n or ngrok was closed. Both must be running simultaneously.

**Fix:** Restart your startup script. Both ngrok and n8n will start together. You can minimize both windows but do not close them.

---

### Telegram commands only return "Send /help for commands"

**Possible causes:**

**1. Wrong TELEGRAM_CHAT_ID:**
Your chat ID doesn't match what's in the environment variable. Verify it with @userinfobot.

**2. Switch node wiring corrupted from import:**
This is the most common cause. The Route Command Switch node looks correctly wired but internally routes all commands to the fallback output.

**How to confirm:**
1. Deactivate Workflow C
2. Send a command to your bot
3. Execute step-by-step through the nodes
4. Check which output of Route Command the data lands on
5. If everything lands on "Fallback" instead of the correct output → the Switch node is corrupted

**Fix:**
1. Delete the Route Command node
2. Add a brand new Switch node
3. Configure the rules and rename it
4. Re-draw all connections
5. Save and retest

See [Known Import Issues](#known-import-issues) for full details.

---

### Daily workflow finds emails but completes 0

**Example summary:**
```text
Emails found: 3
Completed: 0
Failures: 0
```

**Cause:** SplitInBatches node wiring is corrupted. Items go to the "done" output instead of the "loop" output.

**Fix:** Recreate the Split Emails node. Wire:
- Loop output → Call Single Email Processor
- Done output → Build Final Summary

---

### Workflow jumps to no-work summary even when emails exist

**Cause:** The No Work? IF node has swapped true/false outputs from import corruption.

**Fix:** Recreate the No Work? node. Wire:
- True (no work) → Build No Work Summary
- False (has work) → Split Emails

---

### Workflow B fails with `compareOperationFunctions` error

**Error:** `compareOperationFunctions[compareData.operation] is not a function`

**Cause:** An IF node's comparison operation was corrupted during import.

**Fix:** Delete and recreate the affected IF node (usually Input Failed?).

---

### All emails show empty sender and `(no subject)`

**Cause:** Gmail returns header fields with capital letters (`From`, `Subject`, `Date`) but the normalization code may expect lowercase (`from`, `subject`, `date`).

**Fix:** In Workflow A, open the **Normalize Emails And Apply Rules** node and update the field mapping to check both cases:

```javascript
const fromRaw = email.From || email.from || getHeader(email, 'from');
const sender = extractEmail(fromRaw);
const subject = email.Subject || email.subject || getHeader(email, 'subject') || '(no subject)';
const date = email.Date || email.date || getHeader(email, 'date') || '';
```

This ensures the code works regardless of which capitalization style Gmail returns.

---

### Gemini returns 429 Too Many Requests

**Cause:** Gemini free tier quotas vary by region, account, and model. Some models may have very low or zero quota.

**Fix:** Test different models:

```powershell
$key = "YOUR_API_KEY"
$body = '{"contents":[{"parts":[{"text":"Say hello"}]}]}'

# Try these one by one:
$url = "https://generativelanguage.googleapis.com/v1beta/models/gemini-3.1-flash-lite:generateContent?key=$key"
Invoke-RestMethod -Uri $url -Method POST -Body $body -ContentType "application/json"
```

Models to try: `gemini-3.1-flash-lite`, `gemini-2.5-flash-lite`, `gemini-2.5-flash`, `gemini-2.0-flash`.

Use the model with the best working quota for your account.

> [!CAUTION]
> Do **not** link a billing account to fix 429 errors. Enabling billing disables the free tier entirely. Create a new API key on a new project without billing instead.

---

### Run guard blocks new runs

**Error:** `Skipped because another run is active`

**Cause:** A previous run was interrupted during testing and its status is still `running` in Postgres.

**Fix:**

```sql
UPDATE gmail_workflow_runs
SET status = 'failed', finished_at = NOW()
WHERE status = 'running';
```

---

### `crypto.randomUUID is not a function`

**Cause:** n8n Code node can't access the `crypto` module.

**Fix:**
```powershell
$env:NODE_FUNCTION_ALLOW_BUILTIN = "crypto"
```
Restart n8n.

---

### `$env is not defined` in Code nodes

**Cause:** n8n is blocking environment variable access.

**Fix:**
```powershell
$env:N8N_BLOCK_ENV_ACCESS_IN_NODE = "false"
```
Restart n8n.

---

### Gmail OAuth token expired / 401 Unauthorized

**Fix:**
1. Go to n8n **Credentials** → **Gmail OAuth2**
2. Click **"Sign in with Google"** again
3. Re-authorize and save

---

### Postgres connection timeout

**Cause:** Neon serverless Postgres auto-suspends after inactivity and takes 1-3 seconds to wake.

**Fix:** All Postgres nodes already have `connectionTimeout: 10` configured. If timeouts persist, check that your Neon project is not paused and SSL is enabled.

---

### "Replace Me" placeholder node appears after import

**Cause:** Import artifact. Harmless but unnecessary.

**Fix:** Right-click and delete it.

---

### Duplicate email processing

This is extremely unlikely because:
- Gmail query `is:inbox -label:processed` excludes already-processed emails
- Same-run deduplication prevents duplicates within a single run
- Ledger `ON CONFLICT DO UPDATE` prevents duplicate records
- Gmail `addLabels` is idempotent

If it happens, check if the `processed` label was manually removed from an email in Gmail.

---

## MONTHLY COST ESTIMATE

| Component | Monthly Cost | Notes |
|-----------|-------------|-------|
| Gemini API | $0.00 | Free tier covers typical personal email volume |
| Neon PostgreSQL | $0.00 | Free tier: 0.5 GB storage |
| Gmail API | $0.00 | Free for personal use |
| Telegram Bot API | $0.00 | Free |
| n8n self-hosted | $0.00 | Running on your own machine |
| ngrok | $0.00 | Free tier with static domain |
| **TOTAL** | **$0.00** | Free tier everything |

> [!NOTE]
> Gemini free tier quotas vary by region. At typical personal email volumes (5–25 emails/day), you should stay well within free tier limits. Monitor your usage at https://aistudio.google.com if unsure.

---

## ARCHITECTURE SUMMARY

| Workflow | Nodes | Purpose | Runs When |
|----------|-------|---------|-----------|
| **A — Daily Orchestrator** | 25 (+2 notes) | Fetch inbox, apply rules, call child, summarize | Daily at 6:00 AM (configurable) |
| **B — Email Processor** | 18 (+2 notes) | Classify one email, apply labels, update ledger | Called by A per email |
| **C — Telegram Rules Manager** | 11 (+2 notes) | Manage sender/domain rules via Telegram | On Telegram message |

### System Totals

| Metric | Value |
|--------|-------|
| Total executable nodes | 54 |
| Total sticky notes | 6 |
| Grand total nodes | 60 |
| Database tables | 3 |
| External APIs | 3 (Gmail, Gemini, Telegram) |

### Source of Truth

```
Gmail `processed` label = operational source of truth
Postgres ledger = audit / idempotency helper only
```

### Label Application Order (never reversed)

```
1. Apply CATEGORY label (unimportant/receipts/priority/email-subs/please-review)
2. Apply PROCESSED label (visible commit point)
3. Upsert ledger completed (audit record)
```

### Classification Priority

```
1. Sender rules (exact_sender match)       → highest priority
2. Domain rules (domain + subdomain match)  → second priority
3. Gemini AI classification                 → fallback
4. please-review                            → safety net for any failure
```

---

## ENVIRONMENT VARIABLE QUICK REFERENCE

### Required

| Variable | Example Value | Purpose |
|----------|--------------|---------|
| `GMAIL_ACCOUNT` | `primary` | Gmail account identifier |
| `GEMINI_API_KEY` | `AIzaSy...` | Google AI Studio API key |
| `GEMINI_MODEL` | `gemini-3.1-flash-lite` | Gemini model name |
| `TELEGRAM_CHAT_ID` | `1234567890` | Your Telegram chat ID |
| `TELEGRAM_ALLOW_GROUP_RULES` | `false` | Allow rules from group chats |
| `NODE_FUNCTION_ALLOW_BUILTIN` | `crypto` | Allow crypto module in Code nodes |
| `N8N_BLOCK_ENV_ACCESS_IN_NODE` | `false` | Allow $env access in Code nodes |
| `WEBHOOK_URL` | `https://xxx.ngrok-free.dev` | Public HTTPS URL for Telegram webhooks |

### Optional

| Variable | Default | Purpose |
|----------|---------|---------|
| `GMAIL_BATCH_SIZE` | `50` | Max emails fetched per run |
| `WAIT_BETWEEN_EMAILS` | `1` | Seconds between processing each email (recommend `5`) |

---

## SAMPLE STARTUP SCRIPT

Here is a complete PowerShell startup script. Save it as `start-n8n.ps1` and run with `.\start-n8n.ps1`.

```powershell
# ═══════════════════════════════════════════════════════════════
# n8n Startup Script
# Usage: Open PowerShell and type: .\start-n8n.ps1
# ═══════════════════════════════════════════════════════════════

Write-Host ""
Write-Host "═══════════════════════════════════════════════════" -ForegroundColor Cyan
Write-Host "  n8n Startup Script — Starting n8n...            " -ForegroundColor Cyan
Write-Host "═══════════════════════════════════════════════════" -ForegroundColor Cyan
Write-Host ""

# ───────────────────────────────────────────────────────────────
# CODE NODE PERMISSIONS
# ───────────────────────────────────────────────────────────────
$env:NODE_FUNCTION_ALLOW_BUILTIN = "crypto"
$env:N8N_BLOCK_ENV_ACCESS_IN_NODE = "false"

Write-Host "[OK] Code node permissions loaded" -ForegroundColor Green

# ───────────────────────────────────────────────────────────────
# SYSTEM CONFIGURATION
# ───────────────────────────────────────────────────────────────
$env:WEBHOOK_URL = "https://your-static-domain.ngrok-free.dev"

Write-Host "[OK] System config loaded" -ForegroundColor Green

# ───────────────────────────────────────────────────────────────
# SHINY GMAIL AUTOMATION
# ───────────────────────────────────────────────────────────────
$env:GMAIL_ACCOUNT = "primary"
$env:GEMINI_API_KEY = "YOUR_GEMINI_API_KEY_HERE"
$env:GEMINI_MODEL = "gemini-3.1-flash-lite"
$env:TELEGRAM_CHAT_ID = "YOUR_CHAT_ID_HERE"
$env:TELEGRAM_ALLOW_GROUP_RULES = "false"

# Optional tuning
$env:GMAIL_BATCH_SIZE = "50"
$env:WAIT_BETWEEN_EMAILS = "5"

Write-Host "[OK] Shiny Gmail env vars loaded" -ForegroundColor Green

# ───────────────────────────────────────────────────────────────
# VERIFY
# ───────────────────────────────────────────────────────────────
Write-Host ""
Write-Host "--- Environment Variables ---" -ForegroundColor Yellow
Write-Host "  WEBHOOK_URL:                       $env:WEBHOOK_URL"
Write-Host "  GMAIL_ACCOUNT:                     $env:GMAIL_ACCOUNT"
Write-Host "  GEMINI_MODEL:                      $env:GEMINI_MODEL"
Write-Host "  TELEGRAM_CHAT_ID:                  $env:TELEGRAM_CHAT_ID"
Write-Host "  GMAIL_BATCH_SIZE:                  $env:GMAIL_BATCH_SIZE"
Write-Host "  WAIT_BETWEEN_EMAILS:               $env:WAIT_BETWEEN_EMAILS"
Write-Host ""

# ───────────────────────────────────────────────────────────────
# LAUNCH ngrok TUNNEL
# ───────────────────────────────────────────────────────────────
Write-Host "Starting ngrok tunnel..." -ForegroundColor Cyan
Start-Process -FilePath "ngrok" -ArgumentList "http", "5678", "--domain", "your-static-domain.ngrok-free.dev" -WindowStyle Minimized
Start-Sleep -Seconds 3
Write-Host "[OK] ngrok tunnel started" -ForegroundColor Green
Write-Host ""

# ───────────────────────────────────────────────────────────────
# LAUNCH n8n
# ───────────────────────────────────────────────────────────────
Write-Host "═══════════════════════════════════════════════════" -ForegroundColor Cyan
Write-Host "  Launching n8n on http://localhost:5678          " -ForegroundColor Cyan
Write-Host "  Press Ctrl+C to stop                            " -ForegroundColor Cyan
Write-Host "═══════════════════════════════════════════════════" -ForegroundColor Cyan
Write-Host ""

npx n8n
```

> [!WARNING]
> Replace `YOUR_GEMINI_API_KEY_HERE`, `YOUR_CHAT_ID_HERE`, and `your-static-domain.ngrok-free.dev` with your real values before first use.

---

## 🎉 DEPLOYMENT COMPLETE

If you've followed this guide and all smoke tests pass, your Shiny Gmail Automation is ready for production.

- Workflow A will run daily at 6:00 AM and send you a Telegram summary
- Manage rules anytime from Telegram
- Emails that fail classification go to `please-review` — nothing is lost
- Your Gmail will get cleaner every day

Enjoy your clean inbox! 🚀
