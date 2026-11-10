---
name: Broker Desk plan
overview: Add a new broker note system to the existing b2b admin. Clients are created in this system and reused on later notes. PDFs are stored for review and audit. Commission and tax rates stay open.
todos:
  - id: schema
    content: Add the broker client, note, payment, approval, and PDF tables. Do not change existing platform tables.
    status: pending
  - id: api
    content: Add the new /admin/broker endpoints, including create client, pick client on a note, and store review and issued PDFs.
    status: pending
  - id: sidebar
    content: Add Clients, Notes and PDFs, Approvals, Finance, and Alerts to the b2b admin sidebar and wire each page to the new endpoints.
    status: pending
isProject: false
---

# Plan of action

We add a new note system onto the admin people already use. It has its own endpoints and its own tables. The current Dashboard, Enterprise, General Request, Admin, and Settings screens stay as they are. We do not change their APIs.

Inside this new system the word is **client**. A client is someone we create here. It is not a row from another module.

The screens we copy are the ones already built in Broker Desk under `ngbroker/client/src`. The live desk is [Broker Desk](https://ngbroker-4fcxqnvv.manus.space/).

## Screens we take

We do not take Leads, Proposals, Policies, Claims, Work queue, or Settings.

- **Clients** — page `client/src/pages/Clients.tsx`, with a New client form. Opening one uses `client/src/pages/ClientDetail.tsx`. Staff can create a client here. The list is every client created in this system. Opening a client shows that client’s notes.
- **Notes & PDFs** — page `client/src/pages/Notes.tsx`. The debit form is `client/src/components/NoteDraftForm.tsx`. The client field on that form is a list of clients already created. If the client is not in the list yet, the same form can create the client first, then the note uses that new client. The credit form is `client/src/components/SourceCreditNoteDraftForm.tsx` (opened through `CreditNoteDraftForm.tsx`). Saved notes are shown by `DebitNoteReview.tsx` and `CreditNoteReview.tsx`.
- **Approvals** — the waiting list inside `client/src/pages/Notes.tsx`. The decision is the email page `client/src/pages/Reviewer.tsx` (Approve, or Send for review). The approver reviews the PDF that was saved when the note was submitted.
- **Finance** — page `client/src/pages/Finance.tsx`. Two totals and the payment list. Recording a payment uses `client/src/components/RecordNotePaymentModal.tsx`.
- **Alerts** — page `client/src/pages/Alerts.tsx`. The bell is `client/src/components/PaymentAlerts.tsx`. It counts the same rows. Recording a payment uses the same window as Finance.

## Where it plugs in

- The admin sidebar is `b2b_admin/src/components/Sidebar.jsx`. We add the five items above. The items already there stay.
- Staff still log in the way they do now. The role on that login is `broker`, `approver`, `finance`, or `admin`. People who are `admin` today stay `admin`. No new staff table.
- New APIs and new tables live in `flo_user_service`, in the same database. Existing endpoints are not changed.

## Clients, then notes

Two ways to get a client onto a note. Both write the same client table.

1. Create the client on the Clients page. Later, on Notes & PDFs, pick that client from the list.
2. Start a debit note and the client is not in the list. Create the client from the note form. That still inserts one client row, then the note stores that client’s id.

Picking an existing client does not create a second client. The note stores `client_id`, and also copies the client name and address onto the note at save time, so a later change to the client does not rewrite an old note.

## PDFs are saved

In the Broker Desk demo, `client/src/lib/pdf.ts` builds the PDF in the browser and only downloads it. Nothing is kept. That is not enough once this is real.

We save the file each time a PDF is produced for review or for issue. Old files stay. A new submit or a new issue adds a new file. We do not overwrite.

- **Review PDF.** Saved when someone presses Email for approval, before the mail goes out. The email page and the note page open this stored file. This is the PDF the approver reviews. If the note is corrected and submitted again, a new review PDF is saved and the previous one remains for audit.
- **Issued PDF.** Saved when Finance or the Approver presses Issue & download PDF. Download after that returns this stored file, including the approval stamp. If the stamp image cannot be loaded, the file is not saved and the note stays Approved.
- **Mark as issued** (Broker only) changes the status and does not create an issued PDF. The review PDF from submit is still there.

Payment proof files are separate. Those are the JPEG or PDF the staff attach when they record a payment. They are not the note PDF.

## Who can press what

- **Broker** can create a client, draft a note, send it for approval, mark an approved note as issued without an issued PDF, and record a payment without a proof file. Sees alerts for the first 30 days.
- **Approver** does not create clients or draft notes. Opens the stored review PDF from the email, then approves or sends it back. After approval, can issue and download the PDF, which stores the issued file.
- **Finance** can do what a Broker can do, and can also issue and store the PDF, attach a proof file, see alerts after 30 days, and start the credit note from a debit payment.
- **Admin** can create a client, draft, send for approval, record a payment with a proof file, and see alerts after 30 days. Keeps the current admin screens. Does not use “Issue & download PDF”.

The person who created a note cannot approve it.

## New endpoints

All of these are new.

Clients

- `POST /admin/broker/clients` — create a client
- `GET /admin/broker/clients` — clients created here
- `GET /admin/broker/clients/{id}` — one client, plus that client’s notes and payments

Notes

- `GET /admin/broker/notes` — all notes
- `GET /admin/broker/notes/{id}` — one note, its insurers, its payments, and its saved PDFs
- `POST /admin/broker/notes` — create a debit note or a credit note. Body includes `client_id`, or the fields for a new client.
- `PATCH /admin/broker/notes/{id}` — edit while it is a draft, or after it was sent back
- `POST /admin/broker/notes/{id}/submit` — save the review PDF, send the email, mark the note waiting
- `POST /admin/broker/notes/{id}/issue` — save the issued PDF and mark the note issued
- `GET /admin/broker/notes/{id}/pdfs/{pdfId}` — open a stored PDF

Approvals

- `GET /admin/broker/approvals` — notes that are waiting
- `GET /broker/review/{id}` — the email page, including the stored review PDF. The link token is the key.
- `POST /broker/review/{id}` — Approve, Send for review, or Cancel

Finance and alerts

- `GET /admin/broker/finance` — the two totals and the payment list
- `POST /admin/broker/notes/{id}/payments` — record a payment. A proof file is optional and only for Finance and Admin.
- `GET /admin/broker/alerts` — issued notes that still have money open, for the signed-in role

## How the screens pass work to each other

1. Clients creates a client, or lists clients already created. A note number on the client opens that note.
2. Notes creates the debit note with `client_id`. If the staff typed a new client on the form, the client row is created first, then the note.
3. Submit saves the review PDF, writes the approval copy, and emails the link. Approvals only lists notes that are waiting. The email page shows the stored review PDF and writes the decision back onto the same note.
4. Issue saves the issued PDF and sets the issue time. From that moment Finance and Alerts can see the note, if money is still open.
5. Finance and Alerts write a payment on that note. If the payment is on a debit note, the screen asks whether to draft the credit note, and opens Notes with that debit note and that payment already chosen.
6. The credit note is a second note. It uses the same client as the debit note. It points at the debit note and, in this case, at the payment. Submit saves its own review PDF. Issue saves its own issued PDF. After it is issued, Finance and Alerts show it on its own.

Money is saved in kobo so naira amounts do not get rounded badly. 1 naira = 100 kobo.

The debit form asks for the client, policy type, optional nickname, asset value, premium as a flat amount or a percentage, particulars, insurers whose shares must add up to 100%, and an optional bank line. The bank line is printed as a sample. It is not a live bank account. The debit total is the premium only.

The credit form asks for an issued debit note, the policy period, whether to show the client name, and a commission that is either a flat amount or a percentage. Tax on that commission is calculated with it. The rates are not chosen in this plan. We set the numbers when we build. The credit total is what is left to pay the insurer after commission and that tax.

An alert exists only for an issued note with money still open. The clock starts on the issue date. For 30 days Broker and Finance see it. After 30 days only Finance and Admin see it. Paying it in full removes it. Paying part of it leaves it, with the new balance.

## New tables

No finance table and no alert table. Eight new tables. Each has `id`, `created_at`, and `updated_at`. Clients, notes, payments, and files also have `deleted_at`.

### `broker_client`

One client created in this system.

- `name` — required
- `contact_name`
- `sector`
- `email`
- `phone`
- `address`
- `created_by_admin_id`

### `broker_note`

One debit note or one credit note.

- `client_id` — which client
- `note_type` — `debit` or `credit`
- `reference` — the note number, unique
- `status` — `draft`, `awaiting_approval`, `approved`, `issued`, `rejected`
- `policy_type`, `policy_nickname`, `particulars`
- `customer_name`, `customer_address` — copied from the client when the note is saved
- `created_by_admin_id`, `owner_admin_id`
- `created_by_role`
- `issue_date`, `due_date`, `issued_at`
- `approved_by`, `approved_at`, `rejection_reason`
- `total_kobo`, `paid_kobo`
- Debit only: `asset_value_kobo`, `premium_type`, `premium_rate_bps`, `premium_kobo`, `bank_name`, `bank_account_name`, `bank_account_number`
- Credit only: `source_debit_note_id`, `source_payment_id`, `source_payment_locked`, `policy_period_start`, `policy_period_end`, `show_client_name`, `display_client_name`, `commission_type`, `commission_rate_bps`, `source_premium_kobo`, `commission_kobo`, `vat_rate_bps`, `vat_kobo`, `net_kobo`

### `broker_note_provider`

One row per insurer on the note.

- `note_id`, `provider_name`, `share_bps`, `premium_kobo`, `sort_order`
- On a credit note the same row also keeps that insurer’s commission, tax, and net

### `broker_note_pdf`

One stored PDF. Old rows are kept.

- `note_id`
- `approval_id` — set on a review PDF, empty on an issued PDF
- `pdf_kind` — `review` or `issued`
- `file_name`
- `storage_key` — the file in the platform’s existing file store
- `created_by_admin_id`

### `broker_note_payment`

- `note_id`, `amount_kobo`, `payment_date`, `reference`, `recorded_by_admin_id`
- Allowed only when the note is `issued`
- Saving it adds the amount onto `broker_note.paid_kobo`

### `broker_note_payment_file`

- `payment_id`, `file_name`, `content_type`, `storage_key`, `uploaded_by_admin_id`
- JPEG or PDF proof of payment
- Broker cannot add one

### `broker_note_approval`

A new row every time someone submits. A correction does not overwrite the old row.

- `note_id`
- `review_pdf_id` — the review PDF saved for this submit
- `snapshot_json` — the note as it was at submit time
- `status` — `pending`, `approved`, `returned`
- `email_status` — `sending`, `sent`, `failed`
- `reviewer_token_hash`
- `expires_at` — 7 days
- `approver_email`, `approved_by`, `decided_at`, `review_comment`
- `resubmitted_from_id`

### `broker_note_approval_event`

- `approval_id`, `event_type`, `actor_admin_id`, `actor_name`, `comment`
- Event types: `submitted`, `email_sent`, `email_failed`, `approved`, `returned_for_review`, `approval_cancelled`

## Status, in order

- Draft, or sent back, can be edited and submitted.
- Submit saves the review PDF and makes the note Awaiting approval.
- Approve makes it Approved. Cancel is allowed for 15 minutes and puts it back to Awaiting approval. The review PDF stays.
- Send for review makes it Rejected and stores the comment. The review PDF stays.
- Issue saves the issued PDF and makes the note Issued. Finance and Alerts only look at Issued notes.

## Build order

1. Add the eight tables.
2. Add create and list for clients.
3. Add create, edit, and read for debit notes. The client is either an existing `client_id` or a new client created in the same request.
4. Add submit, which stores the review PDF, plus the email page and the approval tables.
5. Add issue, which stores the issued PDF.
6. Add finance, alerts, and record payment.
7. Add the credit note, including the path that starts from a debit payment. It uses the same client and the same PDF rules.
8. Add the five sidebar items and point each page at these endpoints.

# Example

Ada is a Broker. Raymond is the Approver. Chidi is Finance.

The client “Northbank Engineering Ltd” does not exist yet.

## 1. Ada creates the client

On Clients she presses New client and enters the name, contact, email, phone, and address.

- Calls `POST /admin/broker/clients`
- `broker_client`: new row for Northbank
- She can also skip this page and create Northbank from the debit form. That uses the same endpoint and the same table. Either way there is one client row.

Later notes pick this same row. They do not create Northbank again.

## 2. Ada drafts a debit note

On Notes & PDFs she chooses Debit note. The client list already contains Northbank, so she picks it. She fills policy type, asset value, premium, particulars, and two insurers whose shares add up to 100%. She saves.

- Calls `POST /admin/broker/notes` with `client_id` = Northbank
- `broker_client`: unchanged
- `broker_note`: new row, `note_type = debit`, `status = draft`, `client_id` = Northbank, name and address copied onto the note, `total_kobo` = the premium, `paid_kobo` = 0, `reference` = DN-2026-0007
- `broker_note_provider`: one row per insurer
- Northbank’s client page lists DN-2026-0007 because it reads notes for this `client_id`

If she had typed a client that was not in the list, this same save would insert `broker_client` first, then the note.

## 3. Ada clicks Email for approval

The server builds the review PDF and stores it, then sends the mail. The mail links to that stored file.

- Calls `POST /admin/broker/notes/{id}/submit`
- `broker_note_pdf`: new row, `pdf_kind = review`, file key saved
- `broker_note`: `draft` becomes `awaiting_approval`
- `broker_note_approval`: new row, `status = pending`, `review_pdf_id` points at that PDF, `snapshot_json` is the note as Ada saved it
- `broker_note_approval_event`: `submitted`, then `email_sent` or `email_failed`
- Approvals lists DN-2026-0007. Ada cannot edit it while it is waiting.

## 4. Raymond reviews the stored PDF

He opens the email and the review page. The page loads the stored review PDF. It does not build a new file. Opening the link does not decide the note. He presses Approve.

- Calls `GET` the stored PDF, then `POST /broker/review/{id}`
- `broker_note_pdf`: unchanged. The file he reviewed stays
- `broker_note_approval`: `pending` becomes `approved`
- `broker_note`: `awaiting_approval` becomes `approved`
- `broker_note_approval_event`: `approved`
- For 15 minutes he can press Cancel. That puts the note back to Awaiting approval and keeps the PDF.

If he pressed Send for review, he would type a comment. The approval becomes `returned`, the note becomes `rejected`, and the review PDF stays. Ada edits and submits again. The new submit stores a second review PDF and a new approval row. The first PDF remains for audit.

## 5. Chidi issues the note

He presses Issue & download PDF. The issued file, with the stamp, is stored, then downloaded.

- Calls `POST /admin/broker/notes/{id}/issue`
- `broker_note_pdf`: new row, `pdf_kind = issued`
- `broker_note`: `approved` becomes `issued`, `issued_at` is set
- The review PDF from step 3 is still there
- If the stamp image failed, no issued PDF is saved and the note stays `approved`

If Ada had pressed Mark as issued, the status would become `issued` and no issued PDF would be saved.

## 6. Finance and Alerts pick the note up

- `GET /admin/broker/finance` includes DN-2026-0007 in the amount still to collect (`total_kobo - paid_kobo`)
- `GET /admin/broker/alerts` includes it. On the issue date the age is 0. The bell goes up by one for Broker and Finance
- Tables changed: none

## 7. Chidi records the payment

He enters the date, amount, and reference, and attaches a proof PDF. That proof is not the note PDF.

- Calls `POST /admin/broker/notes/{id}/payments`
- `broker_note_payment`: new row
- `broker_note_payment_file`: new row for the proof
- `broker_note.paid_kobo` goes up
- A full payment removes the note from Alerts. A partial payment leaves it, with the new balance
- He chooses to draft the credit note. Notes opens with this debit note and this payment already filled in

## 8. Chidi saves the credit note

The client is already Northbank, taken from the debit note. He does not create the client again. He sets the policy period and the commission. The server works out the commission, the tax on it, and the net. The rates are set when we build.

- Calls `POST /admin/broker/notes`
- `broker_client`: unchanged
- `broker_note`: new row, `note_type = credit`, `status = draft`, `client_id` = Northbank, `source_debit_note_id` = DN-2026-0007, `source_payment_id` = the payment, `source_payment_locked` = true, `reference` = CN-2026-0003
- `broker_note_provider`: one row per insurer
- The debit note and its PDFs are not changed

## 9. The credit note takes the same path

Email for approval stores a review PDF for CN-2026-0003 and sets it to Awaiting approval.

Raymond opens that stored PDF and approves. The credit note becomes Approved.

Issue & download stores an issued PDF for CN-2026-0003 and sets it to Issued.

Each of those files is its own `broker_note_pdf` row. The debit note’s PDFs stay as they were.

## 10. What is on file at the end

- **Clients / Northbank** lists DN-2026-0007 and CN-2026-0003.
- **PDFs** on the debit note: the review file from submit, and the issued file. On the credit note: its own review file and its own issued file. None of these were overwritten.
- **Finance** no longer includes the debit note in the amount to collect if it was paid in full. The amount to pay includes CN-2026-0003 until that credit note is paid.
- **Alerts** shows CN-2026-0003 while it still has an open balance. The clock starts on the credit note’s own `issued_at`.
- A later payment on the credit note adds `broker_note_payment` and increases that note’s `paid_kobo`. It does not create a client or a note PDF.
