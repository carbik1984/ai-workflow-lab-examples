# n8n Course #3 – HTTP Request & Expressions: Call a Free API, Clean Data

Video: https://youtu.be/8RzOocJUJ9g

A messy order comes in through a webhook. n8n **cleans** it (extra spaces, capital letters),
**calculates** the total price, gets today's **exchange rate** from a free API and saves everything
to Google Sheets – with the total in USD and in EUR.
Our example company is **Hilltop Pets** – a made-up online shop for dogs and cats.

```
Webhook → Clean order (Edit Fields) → Get exchange rate (HTTP Request) → Append row in sheet (Google Sheets)
```

You need the workflow and the Google connection from [#2 Webhook to Google Sheets](../02-webhook-google-sheets/).

## Files

| File | What it is |
|---|---|
| [03-Orders-with-total-and-EUR.json](03-Orders-with-total-and-EUR.json) | the finished workflow – import it into n8n |
| [03-order-1.json](03-order-1.json) | test order 1 – messy data (Mia Davis, 3 × 4.35) |
| [03-order-2.json](03-order-2.json) | test order 2 – messy data (Noah Wilson, 2 × 12.99) |

**Download the files:** open a file → **Download raw file** (icon top right),
or download the whole repository: **Code → Download ZIP** on the main page.

**Import the workflow:** in n8n, create a new workflow → menu **⋯** next to the workflow name → **Import from File**.
Then open the Google Sheets node, choose your credential and paste the URL of **your** sheet
(the file contains `PASTE-YOUR-GOOGLE-SHEET-URL-HERE`). Save, then **Publish**.

## 1. The spreadsheet

Add two columns to the header of your `Hilltop Pets - Orders` sheet from #2:

| Order ID | Date | Customer | Email | Product | Quantity | Price | **Total** | **Total EUR** |
|---|---|---|---|---|---|---|---|---|

Select the columns Price, Total and Total EUR → **Format → Number → Number** (two decimal places).

## 2. Copy the workflow from #2

Menu **⋯** next to the workflow name → **Duplicate** → name it `03 - Orders with total and EUR`.

⚠ **Duplicate changes the webhook path to random text** (so it doesn't clash with the original).
Open the Webhook node and change the path back to `new-order`.
Two published workflows can't use the same path – unpublish the old one before you publish the copy (step 6).

## 3. Clean order (Edit Fields)

Click **+** on the arrow between Webhook and Google Sheets → **Edit Fields (Set)** → rename it to `Clean order`
(press **F2**). Mode **Manual Mapping**.

1. Send a test order (see step 7, test URL) so you see `body` in the input.
2. Drag all 7 fields from `body` into the list.
3. Rename the fields without `body.` (`body.customer` → `customer`).
4. Change the expressions:

| Field | Type | Value |
|---|---|---|
| customer | String | `{{ $json.body.customer.trim() }}` – removes spaces at the start and the end |
| email | String | `{{ $json.body.email.toLowerCase() }}` – all small letters |
| total (new field) | Number | `{{ Math.round($json.body.quantity * $json.body.price * 100) / 100 }}` |

**Why `Math.round`?** `3 * 4.35` gives `13.049999999999999`, not `13.05`.
Computers store decimal numbers in binary, and some of them can't be stored exactly.
Multiply by 100, round, divide by 100 → `13.05`.

Everything inside `{{ }}` is JavaScript. A red error while you type is normal – it disappears when the expression is complete.

## 4. Get exchange rate (HTTP Request)

**+** after Clean order → **HTTP Request** → rename it to `Get exchange rate`.

| Setting | Value |
|---|---|
| Method | `GET` (we only read data) |
| URL | `https://api.frankfurter.dev/v1/latest` |
| Send Query Parameters | on: `base` = `USD`, `symbols` = `EUR` |

The [Frankfurter API](https://frankfurter.dev) is free, needs no API key, and the rates come from the European Central Bank.
Don't paste `?base=USD&symbols=EUR` into the URL if you also turn on the query parameters.

**Execute step** – the answer looks like this:
```json
{ "amount": 1, "base": "USD", "date": "2026-11-09", "rates": { "EUR": 0.89342 } }
```

## 5. Google Sheets – take the data from "Clean order"

⚠ **HTTP Request replaces the data.** After it, `$json` is the exchange rate, not the order –
the old expressions `{{ $json.body.orderId }}` now show `undefined`.
`$json` is always the data from the step right before.

Take the order from an earlier step **by its name**:
```
{{ $('Clean order').item.json.orderId }}
```

1. Click the **refresh** icon next to **Values to Send** – n8n loads the new columns Total and Total EUR.
2. In each field, delete the old expression (**Ctrl + A**, **Delete**) and drag the field from **Clean order** in the input.
3. Total EUR:
   ```
   {{ Math.round($('Clean order').item.json.total * $json.rates.EUR * 100) / 100 }}
   ```
   ⚠ No space or Enter after `}}` – otherwise the number becomes text (`"11.66 "`).
4. ⚠ **Execute step on Google Sheets really writes a row.** Check the preview under each field first – no `undefined`.

## 6. Save and publish

Save (**Ctrl + S**). Open the workflow from #2 → **Published ▾ → Unpublish**
(same path `new-order`). Then open `03 - Orders with total and EUR` → **Publish** → **Publish**.

## 7. Send the orders with curl

Open a terminal in the folder with the order files (see [#2](../02-webhook-google-sheets/#4-send-a-test-order-with-curl)).

Test URL (click **Listen for test event** in n8n first):
```
curl -X POST http://localhost:5678/webhook-test/new-order -H "Content-Type: application/json" -d "@03-order-1.json"
```

Production URL (after **Publish**):
```
curl -X POST http://localhost:5678/webhook/new-order -H "Content-Type: application/json" -d "@03-order-2.json"
```

In **PowerShell**, type `curl.exe` instead of `curl`. Show an order: `type 03-order-1.json` (Windows) or `cat 03-order-1.json` (Mac/Linux).

Result in the sheet – clean name, small letters in the email, total and total in euros:

| Order ID | Customer | Email | Quantity | Price | Total | Total EUR |
|---|---|---|---|---|---|---|
| HP-1003 | Mia Davis | mia.davis@example.com | 3 | 4.35 | 13.05 | 11.66 |
| HP-1004 | Noah Wilson | noah.wilson@example.com | 2 | 12.99 | 25.98 | 23.21 |

(Total EUR depends on the exchange rate of the day – in the video it was 0.89342.)

---

Next: **#4** – AI reads customer emails and sorts them automatically.
