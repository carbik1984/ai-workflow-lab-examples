# n8n Course #4 – AI Sorts Customer Emails: Claude + Structured Output

Video: https://youtu.be/DdVqJ7SmrNY

New customer emails arrive in Gmail. n8n sends each one to **Claude** (Anthropic), the AI returns a
**category, urgency, the customer's name and a one-sentence summary** – always in the same shape –
and n8n adds a new row to Google Sheets.
Our example company is **Hilltop Pets** – a made-up online shop for dogs and cats.

```
Gmail Trigger → Basic LLM Chain (Anthropic Chat Model + Structured Output Parser) → Append row in sheet
```

You need n8n on your computer ([#1](../01-install-n8n/)) and the Google Cloud app from
[#2 – Connect n8n to Google](../02-webhook-google-sheets/#2-connect-n8n-to-google-once-about-10-minutes).

## Files

| File | What it is |
|---|---|
| [04-Sort-customer-emails-with-AI.json](04-Sort-customer-emails-with-AI.json) | the finished workflow – import it into n8n |
| [04-prompt.txt](04-prompt.txt) | the prompt for the Basic LLM Chain |
| [04-output-example.json](04-output-example.json) | the JSON example for the Structured Output Parser |
| [04-test-emails.md](04-test-emails.md) | 7 made-up customer emails to test with |

**Download the files:** open a file → **Download raw file** (icon top right),
or download the whole repository: **Code → Download ZIP** on the main page.

**Import the workflow:** in n8n, create a new workflow → menu **⋯** next to the workflow name → **Import from File**.
Then choose your credentials in the Gmail Trigger, the Anthropic Chat Model and the Google Sheets node,
paste the URL of **your** sheet (the file contains `PASTE-YOUR-GOOGLE-SHEET-URL-HERE`) and choose the sheet again
from the list. Save, then **Publish**.

## 1. Anthropic API key

A Claude chat subscription (Pro, Max) **doesn't include the API**. n8n needs an API key, and the API is paid per use.
Sorting one email with Claude Haiku costs a fraction of a cent.

1. Open https://platform.claude.com (the Claude Console) and sign in.
2. **Settings → Billing:** buy prepaid credits and turn **auto-reload off** – then you can never spend more than you put in.
3. Optional: **Workspaces → Create workspace** (e.g. `n8n-course`). A workspace has its own keys, spending and limits.
4. **API keys → Create key** → a name, the workspace (or **Default**), when it expires (e.g. 30 days) → **Create** → **Copy key**.
   ⚠ The key is shown **only once**. Paste it straight into n8n (next step), never show it on screen, never commit it to git.
5. In n8n: **Overview → + → Credential → Anthropic** → paste the key → **Save** → "Connection tested successfully".

## 2. A test Gmail inbox

**Never use your real email for experiments.** Create a new Gmail account just for testing.
Tip: in Gmail, **⚙ → See all settings → General → Smart features** – turn them off. Then all emails land in one Inbox (no Promotions tab).

## 3. Connect n8n to Gmail (once)

This uses the Google Cloud project and OAuth client from [#2](../02-webhook-google-sheets/#2-connect-n8n-to-google-once-about-10-minutes).
Gmail needs **its own credential** – the Google Sheets credential from #2 doesn't work for Gmail.
Google Sheets stays on your main account, Gmail uses the **test account**.

1. https://console.cloud.google.com → your project → search bar at the top: **Gmail API** → **Enable**.
2. **Google Auth Platform → Audience → Test users → Add users:** add the **test Gmail address**.
   Without this, the sign-in fails.
3. In n8n: **Overview → + → Credential → Gmail OAuth2 API**. Paste the **Client ID** and **Client Secret**
   of your OAuth client (Google Cloud → **Clients**).
   ⚠ Google shows the Client Secret only once. Lost it? Open the client → **Add secret** – the old secret keeps working.
4. **Sign in with Google** → choose the **test account** → "Google hasn't verified this app" → **Continue**
   → **Select all** → **Continue** → **Save**.

### Problems?

| Problem | Fix |
|---|---|
| `The provided authorization grant is invalid, expired, revoked…` | the app is in **Testing** mode → the sign-in expires after **7 days** → open the credential → **Reconnect**. This also happens in **Append row in sheet** (your Sheets credential from #2) – reconnect that one too |
| `Access blocked` / `403 access_denied` during sign-in | the test Gmail address is missing in **Test users** |
| `Client authentication failed` | paste the Client Secret again (no spaces), Save, wait ~5 minutes, sign in again |

## 4. The spreadsheet

Open your `Hilltop Pets - Orders` sheet → **File → New → Spreadsheet** (so the new sheet is under the same Google account
– `sheets.new` uses your browser's default account). Name it `Hilltop Pets - support`. First row:

| Date | Customer | Subject | Category | Urgency | Summary |
|---|---|---|---|---|---|

Copy this line (the columns are separated by tabs), click cell **A1 once** (no double-click) and press **Ctrl+V**:
```
Date	Customer	Subject	Category	Urgency	Summary
```
Make the header bold and **View → Freeze → 1 row**.

## 5. The workflow

**Create workflow** → name it `04 - Sort customer emails with AI`.

### Gmail Trigger
- Trigger: **On message received**, your Gmail credential, Poll Times **Every Minute**.
- ⚠ Turn **Simplify off.** The simplified output has only a short snippet – the AI needs the whole email.
- Send yourself one email from [04-test-emails.md](04-test-emails.md) (e.g. #5 "Wrong item") **first** –
  **Fetch Test Event** takes the newest email in the inbox (a new account also has a welcome email from Google).

### Basic LLM Chain
- **Source for Prompt:** `Define below` → paste [04-prompt.txt](04-prompt.txt) into **Prompt (User Message)**.
  Make sure the field is **Expression** (not Fixed) – the `{{ }}` parts must turn green.
  The subject and the text come from the Gmail Trigger with expressions:
  ```
  Subject: {{ $json.subject }}
  Email:
  {{ $json.text }}
  ```
- Turn on **Require Specific Output Format** → n8n adds an **Output Parser** slot.
- Click **+** under **Model** → **Anthropic Chat Model** → your Anthropic credential → **Claude Haiku** (fast and cheap, enough for sorting).
- Click **+** under **Output Parser** → **Structured Output Parser** → Schema Type **Generate From JSON Example** →
  paste [04-output-example.json](04-output-example.json):
  ```json
  {
    "category": "shipping",
    "urgency": "medium",
    "customer": "Noah Wilson",
    "summary": "Order HP-1004 has not arrived after 5 days."
  }
  ```
- **Execute step** → the answer is in `output` and has exactly these four fields:
  `output.category` = `complaint` · `output.urgency` = `high` · `output.customer` = `Emma Clarke` · `output.summary` = one sentence.
  The **Logs** tab shows what n8n sent to Claude and what came back.

**Why structured output?** Without it, the AI answers with free text – a bit different every time.
n8n can't map free text to columns. The parser forces one exact shape, so every field can go into its own column.

### Google Sheets – Append row in sheet
- Your Google Sheets credential, Document **By URL** (paste your sheet's URL), Sheet: your first sheet from the list
  (`Sheet1`, or the name in your language),
  Mapping Column Mode **Map Each Column Manually**.
- Category, Urgency, Customer, Summary – drag them from the AI output in the **INPUT** panel on the left:
  ```
  {{ $json.output.category }}
  {{ $json.output.urgency }}
  {{ $json.output.customer }}
  {{ $json.output.summary }}
  ```
- ⚠ **Subject and Date are not in `$json`** – after the AI step, `$json` is the AI's answer.
  Take them from the Gmail Trigger by its name (like `$('Clean order')` in [#3](../03-expressions-http-request/#5-google-sheets--take-the-data-from-clean-order)):
  ```
  {{ $('Gmail Trigger').item.json.subject }}
  {{ $('Gmail Trigger').item.json.date }}
  ```
  ⚠ Use `subject` at the **top level**, not `headers.subject` – that one is encoded (`=?UTF-8?Q?Wrong_item_=E2=80=93…`).
- **Execute step** – a new row appears in your sheet.

## 6. Publish and test

Save → **Publish** → **Publish** (close the Production Checklist). Now n8n checks the inbox every minute and processes **only new** emails.
Send #6 (Thank you!) and #7 (spam) – after about a minute, **Executions** shows two successful runs.
Your sheet now has three rows – Emma from the test step, plus the two new ones:

| Customer | Subject | Category | Urgency |
|---|---|---|---|
| Emma Clarke | Wrong item – need it before Friday | complaint | high |
| Sophie Turner | Thank you! | praise | low |
| unknown | Grow your pet shop traffic 10x | spam | low |

The spam email has no name, so the AI writes `unknown` – exactly what the prompt says. It doesn't make one up.
(The Date column is in UTC.) Try #1–#4 too – each should get a different category.

**Cost:** each email = one call to Claude Haiku, a fraction of a cent. While the workflow is published,
every new email in the test inbox is sent to the AI – when you're done testing: **Published ▾ → Unpublish**.

---

Next: **#5** – errors, retries and logs. And our mini project is complete.
