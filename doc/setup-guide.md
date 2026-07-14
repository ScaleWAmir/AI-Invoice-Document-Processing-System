# Setup Guide

Step-by-step instructions to get the AI Invoice & Document Processing system running.

---

## 1. Prerequisites

- [n8n](https://n8n.io) instance — cloud or self-hosted, reachable by a public URL (needed for the manual upload webhook)
- A Gmail account to monitor for incoming invoices (or a shared/finance inbox)
- [OpenAI](https://platform.openai.com) API key with access to `gpt-4o` (vision-capable)
- A Google Sheet to act as the invoice database and review queue

---

## 2. Set up the Google Sheet

Create one spreadsheet with two tabs:

**`Invoice_Log`** — columns: `Doc_ID, Received_At, Processed_At, Source, Sender_Email, Sender_Name, Vendor_Name, Invoice_Number, Invoice_Date, Due_Date, Total_Amount, Currency, Document_Type, Confidence_Score, Approval_Tier, Status, Is_Overdue, Errors, Warnings, PO_Reference, Payment_Terms, File_Name`

**`Pending_Review`** — columns: `Doc_ID, Invoice_Number, Vendor, Total, Due_Date, Review_Type, Received_At, Reviewer_Action, Notes`

Copy the Sheet ID from the URL — you'll paste it into every Google Sheets node.

---

## 3. Set up Gmail

1. In n8n, create a Gmail OAuth2 credential with read + send scopes
2. Decide which inbox will receive invoices — a dedicated `invoices@yourcompany.com` alias is cleaner than a personal inbox
3. The trigger polls every minute for unread mail; it does not filter by sender or subject by default, so point it at an inbox that only receives invoices, or add a filter in the trigger node

---

## 4. Set up OpenAI

1. Get an API key with `gpt-4o` access — vision input is required, standard chat-only keys are not enough
2. Add it directly into the `GPT-4o Vision: Extract Document Data` node's Authorization header (this workflow uses a raw HTTP Request node rather than n8n's OpenAI credential, so the key is set inline — for production, move this into an n8n credential or environment variable instead of hardcoding it)

---

## 5. Import and configure the workflow

1. **Workflows → Import from File** → `AI_Invoice_Processing.json`
2. Replace every placeholder:

| Placeholder | Node | Replace with |
|---|---|---|
| `REPLACE_WITH_YOUR_CREDENTIAL_ID` | Email Trigger, Send Finance Team Notification, Send Vendor Acknowledgement | Your Gmail OAuth2 credential |
| `REPLACE_WITH_YOUR_CREDENTIAL_ID` | Log: No Attachment, Log to Invoice_Log Sheet, Add to Pending Review Queue | Your Google Sheets OAuth2 credential |
| `REPLACE_WITH_YOUR_GOOGLE_SHEET_ID` | all Google Sheets nodes | Your Sheet ID from step 2 |
| `REPLACE_WITH_OPENAI_API_KEY` | GPT-4o Vision node | Your OpenAI API key |
| `REPLACE_WITH_FINANCE_TEAM_EMAIL` | Send Finance Team Notification | The inbox/distribution list that should get every processed invoice |
| `REPLACE_WITH_VENDOR_EMAIL_OR_USE_$json.vendor.email` | Send Vendor Acknowledgement | Either a fixed address for testing, or leave as `{{ $json.vendor?.email || $json.sender_email }}` to reply to whoever sent it |

3. If you want the manual upload path too, deploy the workflow and note the `Manual Upload Webhook` URL — this accepts `{ sender_email, sender_name, subject, file_data (base64), file_name, file_type }` as JSON POST body

---

## 6. Tune the business rules

Approval thresholds live in the **Validation Engine** code node:

```js
const AUTO_APPROVE_LIMIT = 5000;   // auto-approve below this, if confidence ≥ 85
const REVIEW_LIMIT = 50000;        // manager review below this, executive above
```

Adjust these two numbers to match your company's actual approval policy. The `confidence_score >= 85` check on auto-approval is also worth revisiting once you've seen how GPT-4o's extraction confidence tracks with real accuracy on your document types.

---

## 7. Test before going live

1. Send a test PDF invoice to the monitored inbox — confirm it gets extracted, validated, routed, and logged
2. Test a low-value, clean invoice → should hit **Auto-Approved**
3. Test a high-value invoice → should hit **Executive Review** and land in `Pending_Review`
4. Test an invoice missing key fields (e.g. no total) → should hit **Rejected**
5. Test the manual upload webhook directly with `curl`:
   ```bash
   curl -X POST https://your-n8n-instance/webhook/invoice-upload \
     -H "Content-Type: application/json" \
     -d '{"sender_email": "vendor@test.com", "file_name": "test.pdf", "file_type": "application/pdf", "file_data": "<base64 PDF>"}'
   ```
6. Confirm both the finance team notification and (where applicable) the vendor acknowledgement actually arrive

---

## 8. Activate and monitor

- Activate the workflow
- Check `Pending_Review` regularly — this is the working queue for anything that isn't auto-approved
- Watch the `Errors` and `Warnings` columns in `Invoice_Log` for the first couple weeks to see if GPT-4o is consistently missing any particular field, and adjust the extraction prompt if so

---

## 9. Troubleshooting

- **Extraction returns empty/malformed JSON** → check `Parse Extraction Result` — it strips markdown fences before parsing, but very long or unusual invoices can still break this; check the raw GPT-4o response in n8n's execution log
- **Every invoice gets rejected** → check the vision node is actually receiving base64 data correctly (`attachment_data` must not be null coming out of `Normalize Input`)
- **Vendor acknowledgement not sending** → confirm `vendor.email` was extracted; if null, it falls back to `sender_email`, so check that field wasn't empty either
- **Math discrepancy warnings on everything** → GPT-4o may be extracting subtotal/tax/total inconsistently for your invoice format; check a few examples manually and adjust the prompt if needed
