<div align="center">

# ✨ Shiny Gmail Automation

**Wake up to a cleaner Gmail inbox — every morning.**

[![Version](https://img.shields.io/badge/version-2.6_Lean-blue?style=for-the-badge)](https://github.com) [![Workflows](https://img.shields.io/badge/workflows-3-orange?style=for-the-badge)](https://github.com) [![License](https://img.shields.io/badge/license-MIT-yellow?style=for-the-badge)](LICENSE) [![Cost](https://img.shields.io/badge/cost-$0-brightgreen?style=for-the-badge)](https://github.com)

Shiny Gmail Automation is a personal inbox-cleaning system that sorts new Gmail messages, applies helpful labels, and sends you a short Telegram summary.

No monthly subscription.
No complicated dashboard.
No need to manually clean your inbox every day.

[Quick Start](#-quick-start) · [What It Does](#-what-it-does) · [Safety & Privacy](#-safety--privacy) · [Telegram Commands](#-telegram-commands)

</div>


---

## 🤔 What It Does

Shiny Gmail Automation checks your new Gmail messages once a day and organizes them for you.

Every morning, it can:

- Mark newsletters and promotions as **unimportant**
- Label order confirmations and invoices as **receipts**
- Highlight work, banking, OTP, and urgent messages as **important**
- Set unclear messages aside as **please-review**
- Send you a Telegram summary of what happened

Instead of opening Gmail to a messy inbox, you wake up to something already sorted.

---

## 👤 Who This Is For

This project is for you if:

- You receive a manageable number of personal emails each day
- You want Gmail to feel less messy
- You like the idea of controlling rules from Telegram
- You are comfortable following a setup guide once
- You want a free, self-hosted automation instead of another paid app

This project is probably **not** for you if:

- You want a one-click mobile app
- You do not want to set up n8n, Gmail API access, Telegram, and a database
- You need enterprise email compliance features
- You want the system to delete emails automatically

Shiny Gmail Automation is designed to **organize**, not destroy. It labels emails. It does not delete your inbox.

---

## ☕ The Simple Version

Imagine this:

You sleep.

At 6:00 AM, Shiny Gmail checks your new emails.

It sees:

- Amazon order confirmation
- A bank security alert
- A random newsletter
- A work email
- One confusing email it is not sure about

Then it labels them:

- Amazon → `receipts`
- Bank alert → `important`
- Newsletter → `unimportant`
- Work email → `important`
- Confusing email → `please-review`

Then it sends you a Telegram message:

```text
✅ Gmail cleanup finished

5 emails processed
⭐ 2 important
🧾 1 receipt
📋 1 unimportant
👀 1 needs review
```

You open Gmail only when something actually needs your attention.

---

## 🏷️ Gmail Labels Used

Shiny Gmail uses six Gmail labels.

| Label | Meaning |
|---|---|
| `important` | Emails you probably care about |
| `receipts` | Orders, invoices, deliveries, payments, and money records |
| `email-subs` | Services you intentionally signed up for |
| `unimportant` | Newsletters, marketing, spam-like messages, or low-priority mail |
| `please-review` | Messages the system is unsure about |
| `processed` | A "done" stamp showing the email was already handled |

The most important idea:

> `please-review` is the safety net.
> If the system is unsure, it asks you instead of guessing silently.

---

## 📱 Telegram Commands

You do not need to open n8n every time you want to change a rule.

Just message your Telegram bot.

### Examples

Always mark this sender as important:

```text
/whitelist boss@company.com
```

Always mark this sender as unimportant:

```text
/blacklist annoying@newsletter.com
```

Always mark this sender as receipts:

```text
/receipt orders@store.com
```

Always send this sender to please-review:

```text
/review weird@example.com
```

You can also manage whole domains:

```text
/whitelistdomain company.com
/blacklistdomain spammy-site.com
```

Domain rules also catch subdomains.

For example:

```text
/blacklistdomain spammy-site.com
```

Can also catch:

```text
news.spammy-site.com
mail.spammy-site.com
offers.spammy-site.com
```

### Add Rules

```text
/whitelist email@example.com
/blacklist email@example.com
/receipt email@example.com
/review email@example.com
```

### Add Domain Rules

```text
/whitelistdomain example.com
/blacklistdomain example.com
```

### Remove Rules

```text
/unwhitelist email@example.com
/unblacklist email@example.com
/unreceipt email@example.com
/unreview email@example.com
/unwhitelistdomain example.com
/unblacklistdomain example.com
```

### View Help

```text
/rules
/help
```

---

## 🔒 Safety & Privacy

Shiny Gmail Automation is designed to be careful.

### It does not delete emails

The system labels messages. It does not permanently delete your mail.

### It does not send emails

This automation is for sorting and labeling. It is not designed to reply to people or send messages from your Gmail account.

### Your rules beat AI

If you create a rule, that rule wins.

For example:

```text
/whitelist boss@company.com
```

means emails from that sender are treated as important even if AI would have guessed something else.

### If AI is unsure, the email goes to review

If Gemini fails, times out, returns something invalid, or cannot confidently classify the email, the message goes to:

```text
please-review
```

That means you stay in control.

### It only needs lightweight email information

The workflow is designed around email metadata such as sender, subject, date, and Gmail snippet.

The snippet is a short preview provided by Gmail, so avoid using this project if you are uncomfortable with an AI classifier seeing any preview text.

### Gmail labels are the source of truth

The `processed` label tells the system an email has already been handled.

If an email does not get processed because something failed, it stays available for the next run.

---

## ⚡ How It Works

```text
Every morning at 6:00 AM

Gmail inbox
   ↓
Find new unprocessed emails
   ↓
Check your Telegram rules
   ↓
If no rule exists, ask Gemini AI to classify
   ↓
Apply the right Gmail label
   ↓
Add the processed label
   ↓
Send you a Telegram summary
```

The system is made of three n8n workflows:

| Workflow | What It Does |
|---|---|
| Workflow A — Daily Orchestrator | Finds new emails and starts the cleanup |
| Workflow B — Email Processor | Classifies and labels one email at a time |
| Workflow C — Telegram Rules Manager | Lets you add, remove, and view rules from Telegram |

---

## 🧰 What You Need Before Setup

You will need accounts or access for:

- Gmail
- n8n
- Telegram
- Google AI Studio or Gemini API
- Neon Postgres or another Postgres database

Recommended tools:

- Latest stable n8n version
- Gmail API access through Google Cloud
- A Telegram bot created with BotFather
- A Gemini model suitable for structured JSON classification
- A Postgres database for rules and audit history

> [!NOTE]
> Gemini model availability changes over time. If Gemini 2.0 Flash is not available to your project, use Gemini 2.5 Flash, Gemini 2.5 Flash-Lite, or a newer compatible Gemini Flash model.

---

## 🚀 Quick Start

This is the short version.

For full setup instructions, read the [Deployment Guide](docs/deployment-guide.md).

### Step 1 — Create Gmail Labels

In Gmail, create these labels:

```text
important
receipts
email-subs
unimportant
please-review
processed
```

### Step 2 — Create Your Database

Use the SQL file in this repo:

```text
database/schema.sql
```

Run it in your Postgres database.

### Step 3 — Prepare Your API Access

You need credentials for:

- Gmail
- Telegram bot
- Gemini API
- Postgres database

Keep these private. Do not commit secrets to GitHub.

### Step 4 — Import the n8n Workflows

Import the three workflow files:

```text
workflows/workflow_a_orchestrator.json
workflows/workflow_b_email_processor.json
workflows/workflow_c_rules_manager.json
```

Recommended order:

1. Import Workflow B — Email Processor
2. Import Workflow C — Rules Manager
3. Import Workflow A — Orchestrator

### Step 5 — Connect Credentials

Inside n8n, connect your Gmail, Telegram, Gemini, and Postgres credentials.

Replace placeholder IDs as described in the [Deployment Guide](docs/deployment-guide.md).

### Step 6 — Activate

Turn on the workflows.

The system will run automatically on the next scheduled cleanup.

---

## 🧪 First Test

Before trusting it fully, do a small test.

1. Send yourself a few test emails.
2. Add one Telegram rule:

```text
/whitelist your-email@example.com
```

3. Manually run the daily workflow in n8n.
4. Check Gmail.
5. Confirm the correct labels were added.
6. Check Telegram for the summary.

Once the test looks good, leave it running.

---

## 🧠 Design Philosophy

This project used to be much bigger.

An earlier version had over 200 nodes, multiple safety systems, replay queues, scanners, digest reports, and enterprise-style patterns.

That was too much for a personal Gmail inbox.

So it was rebuilt with a simpler philosophy:

> Process emails simply.
> Label safely.
> If something fails, do not hide the email.
> Let the user review anything uncertain.

The current version keeps the important safety ideas while removing unnecessary complexity.

It went through multiple rounds of review by different AI agents to make sure it was both lean and safe before release.

---

## 🛠️ Tech Stack

[![n8n](https://img.shields.io/badge/n8n-workflow_automation-EA4B71?style=for-the-badge&logo=n8n&logoColor=white)](https://n8n.io) [![Gemini](https://img.shields.io/badge/Gemini-AI_classification-4285F4?style=for-the-badge&logo=google&logoColor=white)](https://aistudio.google.com) [![Gmail](https://img.shields.io/badge/Gmail-email_processing-D14836?style=for-the-badge&logo=gmail&logoColor=white)](https://gmail.com) [![PostgreSQL](https://img.shields.io/badge/Postgres-audit_&_rules-336791?style=for-the-badge&logo=postgresql&logoColor=white)](https://neon.tech) [![Telegram](https://img.shields.io/badge/Telegram-rule_management-2CA5E0?style=for-the-badge&logo=telegram&logoColor=white)](https://telegram.org)

| Tool | Purpose |
|---|---|
| **n8n** | Runs the automation workflows |
| **Gmail API** | Reads and labels Gmail messages |
| **Gemini** | Classifies emails when no personal rule exists |
| **Telegram Bot** | Lets you manage rules from your phone |
| **Postgres** | Stores rules and audit history |

---

## 📊 Project Stats

| Metric | Value |
|---|---|
| Executable nodes | 54 |
| Workflows | 3 |
| Gmail labels | 6 |
| Telegram commands | 16 |
| Database tables | 3 |
| Software cost | $0 (free tier everything) |
| Goal | Keep personal Gmail clean with minimal effort |

---

## 🏗️ Project Structure

```text
shiny-gmail-automation/
├── README.md
├── LICENSE
├── workflows/
│   ├── workflow_a_orchestrator.json
│   ├── workflow_b_email_processor.json
│   └── workflow_c_rules_manager.json
├── database/
│   └── schema.sql
└── docs/
    ├── deployment-guide.md
    └── master-briefing.md
```

---

## ❓ FAQ

### Can this delete my emails?

No. The system is designed to label emails, not delete them.

### Can I undo a label?

Yes. You can manually remove labels in Gmail like any normal Gmail label.

### What if AI makes a mistake?

Create a Telegram rule for that sender. Your rule will override AI next time.

Example:

```text
/whitelist sender@example.com
```

### What if the system crashes?

Unprocessed emails stay in Gmail and can be retried later.

### What if Gemini is unavailable?

The email is sent to `please-review` instead of being silently misclassified.

### Can I change the 6:00 AM schedule?

Yes. Change the schedule in n8n.

### Does this work without Telegram?

The cleanup can work without Telegram commands, but Telegram is the easiest way to manage rules and receive daily summaries.

### Is this beginner-friendly?

It is beginner-friendly after setup, but setup still requires following technical steps. If you can follow a guide and copy/paste credentials carefully, you can run it.

---

## 📄 License

This project is licensed under the MIT License.

You can use it, modify it, and share it. Just keep the copyright notice.

See [LICENSE](LICENSE) for details.

---

## 👋 Author

Built by Chan — a non-technical automation builder from Manila who wanted a calmer inbox without paying for another subscription.

If this project helps you, consider giving it a ⭐

<div align="center">

**Shiny Gmail Automation**
*Built with coffee, n8n, Gmail, Telegram, Postgres, and AI.*

</div>
