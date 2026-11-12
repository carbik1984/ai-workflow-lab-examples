# n8n Course #2 – Webhook to Google Sheets: Save Orders Automatically

Video: https://youtu.be/mzVNK8-E-mU

A new order comes in through a **webhook**, and n8n saves it to **Google Sheets** as a new row.
Our example company is **Hilltop Pets** – a made-up online shop for dogs and cats.

You need n8n running on your computer – see [#1 Install n8n with Docker](../01-install-n8n/).

## Files

| File | What it is |
|---|---|
| [02-Orders-to-Google-Sheets.json](02-Orders-to-Google-Sheets.json) | the finished workflow – import it into n8n |
| [02-order-1.json](02-order-1.json) | test order 1 (Emma Johnson, dog food) |
| [02-order-2.json](02-order-2.json) | test order 2 (Lucas Brown, cat scratching post) |

**Download the files:** open a file → **Download raw file** (icon top right),
or download the whole repository: **Code → Download ZIP** on the main page.

**Import the workflow:** in n8n, create a new workflow → menu **⋯** next to the workflow name → **Import from File**.
Then open the Google Sheets node, choose your credential and paste the URL of **your** sheet
(the file contains `PASTE-YOUR-GOOGLE-SHEET-URL-HERE`). Save, then **Publish** (top right) – otherwise the production URL won't work.

## 1. The spreadsheet

1. Type `sheets.new` in your browser – it creates a new, empty Google Sheet.
2. Name it `Hilltop Pets - Orders`.
3. First row (the header), exactly like this – n8n uses these names to find the columns:

   | Order ID | Date | Customer | Email | Product | Quantity | Price |
   |---|---|---|---|---|---|---|

## 2. Connect n8n to Google (once, about 10 minutes)

Self-hosted n8n needs its own "app" in Google Cloud. It's free.

1. Open https://console.cloud.google.com → **New project** (e.g. `n8n-ai-workflow-lab`) → switch to it.
2. Search for **Google Sheets API** → **Enable**. Do the same for **Google Drive API**.
3. **APIs & Services → OAuth consent screen → Get started:** app name, your e-mail,
   audience **External** → Create.
4. **Audience → Test users → Add users:** add your own Google e-mail.
5. In n8n: **Overview → + → Credential → Google Sheets OAuth2 API** and copy the
   **OAuth Redirect URL** – on your computer it is `http://localhost:5678/rest/oauth2-credential/callback`.
6. In Google Cloud: **Clients → Create client → Web application** → paste the redirect URL into
   **Authorized redirect URIs** → Create.
7. Copy the **Client ID** and **Client Secret** into the n8n credential.
   ⚠ The secret is shown only once – save it in your password manager.
8. **Sign in with Google** → choose your account → "Google hasn't verified this app" → **Continue**
   → allow access → **Save**.

**Never show the Client Secret on screen and never commit it to git.**

### Problems?

| Problem | Fix |
|---|---|
| `Client authentication failed` after sign-in | paste the Client Secret again (no spaces), Save, wait ~5 minutes, sign in again |
| It stops working after a week | the app is in **Testing** mode → login expires after 7 days → open the credential and click **Reconnect** |
| `redirect_uri_mismatch` | the redirect URI in Google Cloud must be exactly the URL from n8n |

## 3. The workflow

1. **Webhook** node: HTTP Method **POST**, Path `new-order`, Authentication None
   (OK on your own computer – on a real server, add authentication), Respond **Immediately**.
2. **Google Sheets → Append row in sheet:** your credential, Document **By URL** (paste your sheet's URL),
   Sheet `Sheet1`, Mapping Column Mode **Map Each Column Manually**.
3. Drag the values from `body` in the input to the columns. Each field gets an expression, for example:
   ```
   {{ $json.body.orderId }}
   ```
   `$json` = the data that comes into this step. So this takes `orderId` from the `body`.
4. **Execute step** – a new row appears in your sheet.
5. Save (**Ctrl+S**) → **Publish** → **Publish**. Now the production URL works.

### Test URL vs. Production URL

| | Test URL | Production URL |
|---|---|---|
| address | `http://localhost:5678/webhook-test/new-order` | `http://localhost:5678/webhook/new-order` |
| works | only after **Listen for test event**, for one request | all the time, after **Publish** |
| use it for | building the workflow – you see the data in the editor | real use |

`404 The requested webhook "new-order" is not registered` = you used the test URL without clicking **Listen for test event**
(or the production URL before **Publish**).

## 4. Send a test order with curl

Open a terminal **in the folder with the order files**. `curl` is already installed on Windows 10/11, Mac and Linux.

- **Windows:** in File Explorer, open the folder, click the address bar, type `cmd` and press Enter.
- **Mac:** right-click the folder → **New Terminal at Folder**.

In the video, my files are in a subfolder, so I type `@workflows\02-order-1.json`.
In the folder with the files, just use `@02-order-1.json`.

Test URL (click **Listen for test event** in n8n first):
```
curl -X POST http://localhost:5678/webhook-test/new-order -H "Content-Type: application/json" -d "@02-order-1.json"
```

Production URL (after **Publish** – no clicking in n8n):
```
curl -X POST http://localhost:5678/webhook/new-order -H "Content-Type: application/json" -d "@02-order-2.json"
```

| Part | Meaning |
|---|---|
| `-X POST` | the method – "I'm sending you data" |
| `-H "Content-Type: application/json"` | the data is JSON |
| `-d "@02-order-1.json"` | send the content of this file (`@` = from a file) |

The answer `{"message":"Workflow was started"}` means n8n received the order.

Tips:
- The video uses **Command Prompt (cmd)**. In **PowerShell**, type `curl.exe` instead of `curl` and keep the quotes around `"@02-order-1.json"`.
- Show a file in the terminal: `type 02-order-1.json` (Windows) or `cat 02-order-1.json` (Mac/Linux).

## 5. Check the runs

In n8n, the **Executions** tab shows every run. Runs from the editor (test) have a small **flask** icon,
production runs have no icon.

---

Next: [**#3 HTTP Request & Expressions**](../03-expressions-http-request/) – clean the data, calculate the total price and get the exchange rate from a free API.
