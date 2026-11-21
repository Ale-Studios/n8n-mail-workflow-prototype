# N8N Mail WORKFLOW Prototype: Setup guide

## The rule that never changes
This workflow **never sends an email by itself**. Every reply is saved as a Gmail draft. You read it and press Send. Only drafts, log rows and alerts are automatic. Nothing irreversible happens without a person.

## What you need before importing

### 1. A Gmail credential (OAuth2)
It is used to read your inbox, create drafts and search the emails you sent.

**This is not a one-click connection.** On self-hosted n8n (n8n Cloud offers a managed Google login), Gmail needs your own OAuth project in Google Cloud Console. That is a Google requirement for any third-party app that reaches Gmail, not only n8n. It costs nothing, but you have to do it once:

1. Go to **console.cloud.google.com** and create a new project (or use an existing one).
2. **APIs & Services → Library**, search for "Gmail API", click **Enable**.
3. Set up the OAuth consent screen (in the newer console: **Google Auth Platform**):
   - User type: **External**.
   - Fill in an app name and a support email.
   - Under **Audience**, add your own Gmail address as a **test user**.
4. Create an OAuth client: **Clients → Create client**:
   - Application type: **Web application**.
   - **Authorized redirect URIs**: add `http://localhost:5678/rest/oauth2-credential/callback` (replace `localhost:5678` with your own address and port if n8n does not run locally).
   - Create, then copy the **Client ID** and the **Client secret**.
5. In n8n, open the Gmail Trigger node, click **Connect to Gmail**, paste the Client ID and Client secret, save, and complete the Google login.

**Publish the Google app, or the workflow stops after 7 days.** While the app stays in "Testing" mode, Google expires the access every 7 days: the workflow stops reading your mail and, without the error workflow described below, nobody notices. When your tests are done, go to Google Auth Platform → **Audience** and click **Publish app** ("In production"). For personal or company use with your own mailbox the "app not verified" warning will remain: it is normal, and you pass it with Advanced → Continue. After publishing, sign in again once in the credential.

### 2. An Anthropic API key
The workflow calls Claude directly over HTTP. It works on self-hosted n8n and on n8n Cloud, with no environment variables.

- Go to console.anthropic.com, create an API key, and add some credit.
- In n8n: **Credentials → Create credential → Header Auth**
  - **Name**: `x-api-key`
  - **Value**: your API key
  - give the credential a recognisable name, for example "Anthropic".
- Open the 5 nodes with "(Claude)" in their name (Classify Email, Draft Reply - Lead, Draft Reply - Client Question, Extract Invoice Details, Draft Follow-up Reminder) and choose that credential under **Credential for Header Auth**.
- Cost: each incoming email uses one fast call (classification) and, when a reply is needed, a second one. For 50 to 200 emails a month this is usually a few euros, billed by Anthropic. There is no automatic spending cap: set a monthly limit in the Anthropic console.

### 3. A Google Sheet for invoices
- Create a Google Sheet with a tab named "Invoices" and the columns: `date_logged, from, subject, amount, currency, due_date, status`.
- Connect a Google Sheets OAuth2 credential in n8n.
- In the nodes "Log Invoice to Sheet" and "Read Invoice Tracker", replace `REPLACE_ME_SHEET_ID` with the sheet ID (it is in the sheet URL).
- `status` is `pending` (to pay) or `needs_review` (the amount was not recognised, mostly because the invoice is only a PDF attachment, which the workflow does not read). The weekly summary lists both. When an invoice is paid, change its status by hand to `paid` (or any other word) and it will no longer appear.

### 4. A Telegram bot (urgent alerts, weekly summary, errors)
- Write to @BotFather on Telegram, create a free bot, and copy the token.
- Write to your bot once, then get your `chat_id` (for example with @userinfobot).
- In n8n: **Credentials → Create credential → Telegram API**, paste the token.
- Replace `REPLACE_ME_TELEGRAM_CHAT_ID` in the nodes "Send Urgent Alert" and "Send Digest" (and in the error workflow, see below).
- "Send Urgent Alert" sends two kinds of message: a red one for urgent emails, and a yellow one for emails that Claude could not classify ("to review").
- If you prefer Slack or email, replace the Telegram nodes with the matching node.

### 5. The error workflow (recommended: it avoids silent failures)
If the Anthropic key is wrong or out of credit, if the Google login expires or a model is retired, n8n does not warn anyone by default. `error-alert.json` fixes that:

1. Import `error-alert.json`, open the node "Send Error Alert", choose the Telegram credential and enter your Chat ID.
2. **Publish** the error workflow. If it stays off, n8n will not run it.
3. Open "N8N Mail WORKFLOW Prototype" → **⋯ → Settings → Error workflow** and pick "N8N Mail WORKFLOW Prototype — Error Alert".
4. Note: n8n triggers it only for automatic executions, not for runs you start by hand with "Execute workflow".

From then on, every failure reaches you on Telegram with the name of the step that broke.

## Company Profile (fill it in BEFORE you publish the workflow)
The workflow does not know your business until you tell it. Open the **Company Profile** node (on the left, under the "COMPANY PROFILE" note) and replace every `REPLACE_ME` with your real details:

- **company_name / business_description**: who you are and what you do
- **services_and_prices**: the services and prices you can quote
- **working_hours / booking_and_contact**: opening hours and how to book or reach you
- **faq**: the questions you get most, with your standard answers
- **tone_of_voice**: the tone (formal, friendly, first-name basis…)
- **never_promise**: what must never be promised (delivery dates, discounts, refunds…)
- **reply_language**: leave it as is to answer in the sender's language, or write for example "Italian"
- **signature**: the signature added at the end of drafts
- **follow_up_after_days**: after how many days an unanswered email gets a reminder draft (default 3)
- **ai_model_fast / ai_model_writer**: the Claude model names. If Anthropic retires a model, change only these two fields.

How it works: Claude uses ONLY this information for facts about the business. A field left as `REPLACE_ME` is ignored, and where a fact is missing Claude leaves a [PLACEHOLDER] instead of inventing it. You do not need to touch the other two nodes there (Build Profile Text, Which Trigger?).

Honest limit: the workflow only knows your business as well as you fill in the profile, and every draft must be reviewed before you press Send.

## Importing the workflow
1. In n8n create an empty workflow (**Create workflow**), then in the **⋯** menu at the top choose **Import → From file** (or **From URL**).
2. Select `n8n-mail-workflow.json` (then repeat with `error-alert.json`, see section 5). The workflow appears with 33 nodes. **Save it** with Ctrl+S: until it is saved, the Publish button stays grey.
3. Replace every `REPLACE_ME` (Gmail, Anthropic, Sheets and Telegram credentials, the sheet ID, the chat ID and the **Company Profile**). Nodes that wait for a credential show a red triangle: there are 15 in total (6 Gmail, 5 Claude, 2 Google Sheets, 2 Telegram). When you choose a credential in one node, n8n usually applies it to the other nodes of the same type: check anyway that no red triangle is left.
4. Before publishing, try **Execute workflow** with an unread email in your inbox: you should see a draft in Gmail.
5. Publish the workflow (**Publish** button; it only turns on when the nodes are in order).

## Requirements and limits (read before you install)
- **First start**: with `npx n8n start` the first start downloads n8n and can take several minutes. In the meantime the page may show "Cannot GET /" or "n8n is starting up": wait and reload. Then n8n asks you to create an owner account (it stays on your computer).
- **n8n must stay on**: if the computer is off, emails wait until it starts again. For serious use you need a server or a Docker container that is always on.
- **n8n version**: tested on 2.40 to 2.43. Much older versions may be missing some nodes.
- **Time zone**: the file sets `Europe/Rome` (the 9:00 follow-up and the Monday summary). To change it: Workflow → Settings → Timezone.
- **Which emails are read**: unread mail in the inbox, excluding the Promotions, Social and Forums tabs and messages over 10 MB. Newsletters and automated senders (no-reply) never get a drafted reply; invoices and urgent alerts are still handled.
- **Attachments**: emails are read as text only. An invoice that exists only as a PDF is logged with the status `needs_review`.
- **Drafts do not remember the conversation**: Claude sees the incoming message (and the quoted text, if any), not the whole history.
- **Never sends**: the workflow only creates drafts. A person must always read and press Send.
- **Privacy**: the text of your emails is sent to Anthropic to be processed. Check it against your own privacy and data protection obligations.
- **Quality**: classification is done by a language model and can be wrong. That is why, when Claude's answer is unclear, the workflow sends a yellow alert instead of ignoring the email.

## Customising
- **Claude models**: set in the Company Profile node (`ai_model_fast` to classify, `ai_model_writer` to write drafts).
- **Tone of the drafts**: edit the `system` text in the "Draft Reply..." nodes.
- **Follow-up delay**: change `follow_up_after_days` in the Company Profile node (the search adapts by itself).
- **Categories**: to add one (for example "PARTNERSHIP"), edit both the prompt in "Classify Email (Claude)" and the branches of the switch "Route by Category".

## How the follow-up works (and its limits)
Every day at 9 the workflow looks for the emails YOU sent at least N days ago (N is `follow_up_after_days`, default 3) and checks the whole conversation. It creates a reminder draft only if:

- the last message in the conversation is yours (nobody answered after it),
- the set number of days has passed,
- there is no draft already waiting in the conversation,
- the conversation has not already received a reminder from this workflow (it remembers, even if you delete the draft).

The draft's recipient is already filled in. Drafts, trash and spam do not count when deciding who wrote last.
Limits: the memory of reminders already made only works while the workflow is ACTIVE (Publish); the workflow cannot tell an important email from a courtesy one, so the reminder is always a draft to review; automatic replies ("out of office") count as an answer.
