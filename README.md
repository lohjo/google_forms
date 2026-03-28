# google_forms
use google api in node.js

# data pipeline for google forms card

Let me walk you through how it works end-to-end.Yes — and it's simpler than it sounds. The full pipeline has three components: Google Forms writes to Sheets automatically, the **Google Sheets API** lets your JS program read new rows, and a trigger (either a Google Apps Script webhook or a polling interval) fires the card generation the moment a new response lands. Here's how it all connects:

![growth_card_pipeline](uploads\growth_card_pipeline.svg)

Click any box above to drill deeper. 

Here's the breakdown of each layer:

---

## The API you need: Google Sheets API v4

Yes — there is a direct, well-documented API. In Node.js you use the official `googleapis` package:

```bash
npm install googleapis
```

```javascript
const { google } = require('googleapis');

// Authenticate with a Service Account (a bot account you create
// in Google Cloud Console and share the sheet with)
const auth = new google.auth.GoogleAuth({
  keyFile: 'service-account-key.json', // downloaded from GCP
  scopes: ['https://www.googleapis.com/auth/spreadsheets.readonly'],
});

const sheets = google.sheets({ version: 'v4', auth });

async function getResponses() {
  const res = await sheets.spreadsheets.values.get({
    spreadsheetId: 'YOUR_SHEET_ID', // from the sheet URL
    range: 'Sheet1!A:AZ',           // all columns
  });
  return res.data.values; // returns array of rows
}
```

---

## Trigger option A — Google Apps Script (instant, recommended)

Inside your Google Sheet, go to **Extensions → Apps Script** and add this. It fires the moment a form is submitted and POSTs the new row straight to your Node.js server:

```javascript
// In Apps Script (runs server-side inside Google)
function onFormSubmit(e) {
  const row = e.values;          // array of all answers in order
  const timestamp = row[0];
  const participantCode = row[1];

  UrlFetchApp.fetch('https://your-server.com/new-response', {
    method: 'post',
    contentType: 'application/json',
    payload: JSON.stringify({ timestamp, participantCode, row }),
  });
}
```

Then register it: go to **Triggers** in Apps Script → add trigger → `onFormSubmit` → From spreadsheet → On form submit.

Your Node.js server receives it:

```javascript
const express = require('express');
const app = express();
app.use(express.json());

app.post('/new-response', async (req, res) => {
  const { participantCode, row } = req.body;

  // 1. Fetch the matching PRE-survey row from Sheets
  const preRow = await findPreSurveyByCode(participantCode);

  // 2. Compute change scores
  const scores = computeChangeScores(preRow, row);

  // 3. Render and send growth card
  await generateAndSendCard(participantCode, scores);

  res.sendStatus(200);
});
```

---

## Trigger option B — Polling (no server required)

If you don't want to host a server, a simple Node.js script can run locally (or on a free service like Railway/Render) and check the sheet on a timer:

```javascript
const LAST_ROW_SEEN = { value: 1 };

setInterval(async () => {
  const rows = await getResponses();
  const newRows = rows.slice(LAST_ROW_SEEN.value);

  for (const row of newRows) {
    const code = row[1];
    const preRow = await findPreSurveyByCode(code);
    const scores = computeChangeScores(preRow, row);
    await generateAndSendCard(code, scores);
  }

  LAST_ROW_SEEN.value = rows.length;
}, 30000); // check every 30 seconds
```

---

## Generating the growth card

The cleanest approach is `puppeteer` — it renders an HTML template to PDF or PNG, which means you can style the card exactly like a webpage:

```bash
npm install puppeteer
```

```javascript
const puppeteer = require('puppeteer');

async function generateAndSendCard(code, scores) {
  const html = buildCardHTML(scores); // your card template

  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  await page.setContent(html);

  // PNG (good for WhatsApp/Telegram sharing)
  await page.screenshot({ path: `cards/${code}.png`, fullPage: true });

  // Or PDF (good for email)
  await page.pdf({ path: `cards/${code}.pdf`, format: 'A5' });

  await browser.close();
}
```

The `buildCardHTML` function just returns a string of HTML — you can make it as visual as you want (radar chart via Chart.js, participant name, scores, a quote from Section E). Puppeteer renders it pixel-perfectly.

---

## Sending the card

```javascript
const nodemailer = require('nodemailer');

const transporter = nodemailer.createTransport({
  service: 'gmail',
  auth: { user: 'edu@impart.sg', pass: process.env.EMAIL_PASS }
});

await transporter.sendMail({
  from: 'platform by Impart <edu@impart.sg>',
  to: participantEmail,
  subject: 'Your platform growth card',
  text: 'Here is your personal growth card from platform.',
  attachments: [{ filename: 'growth-card.pdf', path: `cards/${code}.pdf` }]
});
```

---

## Summary of what you need

| Piece | Tool | Cost |
|---|---|---|
| Survey → Sheets | Google Forms (built-in) | Free |
| Sheets → Node.js | Google Sheets API v4 | Free |
| Trigger | Apps Script (instant) or polling | Free |
| Card generation | Puppeteer (HTML → PDF/PNG) | Free |
| Sending card | Nodemailer + Gmail | Free |
| Hosting the server | Railway or Render | Free tier |

The only tricky step is the **Service Account setup** in Google Cloud Console — that's what gives your Node.js script permission to read the sheet.