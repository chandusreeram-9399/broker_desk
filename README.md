---
name: Broker Desk plan
overview: First draft in 20 hours. Copy the existing React screens into a new frontend folder and add a new FastAPI folder. Wire the forms that already exist. The new work is Postgres, roles, policy-to-provider matching, stored PDFs, and email.
todos:
  - id: task-1
    content: "4h. New backend and frontend folders. Login, roles, provider, policy, client."
    status: pending
  - id: task-2
    content: "3.5h. Wire the existing debit form. Policy type filters providers. Save the draft."
    status: pending
  - id: task-3
    content: "4h. Store the preview PDF, send the approval email, broker alert."
    status: pending
  - id: task-4
    content: "2.5h. Issue the debit PDF, email the client, Finance records the client payment."
    status: pending
  - id: task-5
    content: "3h. Credit form already exists. Prefill it, reuse approval, email each provider."
    status: pending
  - id: task-6
    content: "3h. Finance settlement alert, record the payment, one full walk-through."
    status: pending
isProject: false
---

# Plan of action — 20 hours, first draft

Two new folders. `backend` is FastAPI and PostgreSQL. `frontend` is a copy of `ngbroker/client`. The forms, the PDF builder, and the approval page are already written, so this draft wires them to the new API. Cursor can generate a dropdown or a CRUD endpoint in minutes. The hours below are for that wiring, plus the few rules that are not in the current code.

Total: **20 hours**. A later pass can tighten layout and edge cases. That pass is not in these 20 hours.

Three logins. The role is a column on the user, checked on every request.

- **Broker** creates providers, policies, and clients. Drafts, edits, previews, sends for approval, issues, and sends the issued file.
- **Approver** only opens the email and approves or rejects.
- **Finance** records the client payment, then the settlement to each provider.

## Models

- `staff_user` — name, email, password hash, role (`broker`, `approver`, `finance`)
- `insurance_provider` — provider name, email, bank account name, account number, account code
- `policy` — policy type, `provider_id`. One provider can sell many policies. The same policy type can sit on more than one provider.
- `client` — company name, primary contact, industry, email, phone, address
- `debit_note` — same id for the life of the note. `client_id`, policy type, nickname, asset value, premium type, premium amount, particulars, status, issued time
- `debit_note_share` — `debit_note_id`, `provider_id`, `policy_id`, share percent, that provider’s premium
- `credit_note` — `client_id`, `debit_note_id`, period start, period end, show client name, commission type, commission amount, tax amount, net amount, status
- `credit_note_share` — `credit_note_id`, `provider_id`, share, gross, commission, tax, net
- `note_approval` — debit or credit, note id, snapshot, status (`pending`, `approved`, `rejected`), token hash, comment, decided time
- `note_pdf` — debit or credit, note id, kind (`preview` or `issued`), file path
- `finance_report` — `client_id`, `debit_note_id`, `credit_note_id` and `provider_id` on a settlement, kind (`client_payment` or `provider_settlement`), amount, payment date, reference, receipt file
- `alert` — for Broker or Finance, kind (`approved`, `rejected`, `settle_provider`), note id, read time

Status on both notes: `draft`, `approval_waiting`, `approved`, `issued`, `rejected`.

## 1. Login, providers, policies, clients — 4h

Models: `staff_user`, `insurance_provider`, `policy`, `client`.

- 1.1 Create `backend` and `frontend`. Copy `ngbroker/client` into `frontend`. Connect FastAPI to PostgreSQL. **1.5h**
- 1.2 Login, password check, token, three role menus. **1h**
- 1.3 Provider API and form. Name, email, bank account name, account number, account code. **0.5h**
- 1.4 Policy API and form. Pick a provider, save the policy type on `policy.provider_id`. **0.5h**
- 1.5 Client API. The form is already `Clients.tsx`. Name, contact, and email are required. **0.5h**

Done when one provider has two policies, a second provider shares one of those policy names, and one client exists.

## 2. Debit draft — 3.5h

Models: `debit_note`, `debit_note_share`. The form is already `NoteDraftForm.tsx`.

- 2.1 Debit or Credit choice on the existing Notes page. Credit stays closed until task 5. **15m**
- 2.2 Point the client dropdown at `GET /clients`. The select is already on the form. **15m**
- 2.3 Policy type loads only providers that have a `policy` row of that type. This rule is new. **1h**
- 2.4 Save nickname, asset value, premium, and particulars. Show the provider bank fields from `insurance_provider`. **45m**
- 2.5 Shares must total 100. That check already exists in `debitNote.ts`. Connect it to the save call. **30m**
- 2.6 Insert the note and the share rows. Status `draft`. Edit only while `draft` or `rejected`. **45m**

Done when the client page shows the draft and the providers match that policy type.

## 3. Preview, approval, broker alert — 4h

Models: `note_pdf`, `note_approval`, `alert`. PDF layout is already `pdf.ts`. The approver page is already `Reviewer.tsx`.

- 3.1 Preview calls the existing PDF builder, stores the file as `preview`, and opens it. **1h**
- 3.2 Send for approval. Status `approval_waiting`. Store the snapshot. Email the Approver. **1.5h**
- 3.3 Wire Approve and Reject on the existing review page. Reject needs a comment. **45m**
- 3.4 Write the Broker alert and show the new status on the same debit note id. **45m**

Done when the Broker sees the alert, and a rejected note can be edited and sent again.

## 4. Issue the debit note, then the client payment — 2.5h

Model: `finance_report` for `client_payment`. The payment form is already `RecordNotePaymentModal.tsx`.

- 4.1 Issue and download, only from `approved`. Store the `issued` PDF. Status `issued`. **45m**
- 4.2 Email that stored PDF to `client.email`. **30m**
- 4.3 Finance saves amount, date, reference, and receipt with `client_id` and `debit_note_id`. **45m**
- 4.4 Dialogue: client payment is recorded, and it is time to send the credit note. **30m**

Done when the receipt is stored on that debit note and the dialogue opens.

## 5. Credit note to the providers — 3h

Models: `credit_note`, `credit_note_share`. The form is already `SourceCreditNoteDraftForm.tsx`. Approval from task 3 is reused.

- 5.1 Open that form filled from the debit note: client, policy, address, providers, shares, gross premium. **1h**
- 5.2 Save the period and the commission. Net to each provider is gross minus commission and tax. **45m**
- 5.3 Call the same approval endpoint. Broker alert and status change. **30m**
- 5.4 Issue the credit PDF, then email it to each `insurance_provider.email`. **45m**

Done when every provider on that note has the email and the credit note is `issued`.

## 6. Finance settles with the providers — 3h

- 6.1 On credit issue, one Finance alert per provider still unpaid. **30m**
- 6.2 Finance records amount, date, and reference on `finance_report`, kind `provider_settlement`. **45m**
- 6.3 That provider’s alert closes. The others stay. **15m**
- 6.4 One walk-through and the fixes it turns up: provider, policy, client, debit, approval, client payment, credit, provider email, settlement. **1.5h**

Done when both finance rows exist and both alerts are gone.

## Email on the real app

The app sits on one fixed HTTPS address, `APP_PUBLIC_ORIGIN`. SMTP user and password stay on the server.

1. **Approver.** Send for approval emails the note and two links, Approve and Reject. The link is that HTTPS address plus a one-time token. The token is stored only as a hash. The Approver must press the button. The server updates the note and writes the Broker alert.
2. **Client.** After issue, the stored debit PDF goes to `client.email`.
3. **Providers.** After the credit note is issued, that PDF goes to each provider email on the note.

Localhost cannot be the address in those links. On the real server the site, the database, and the SMTP password are together, so the link and the status change use the same database.
