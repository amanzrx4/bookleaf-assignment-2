# BookLeaf – Author Royalty Dashboard (Assignment 2)

Bubble + n8n + DeepSeek.

## Links and logins

- App: https://amansagar262.bubbleapps.io/version-test/
- Recording: link shared in the email
- Admin: admin@bookleaf.test / password - mentioned in email
- Authors: priya.sharma@email.com, farhan.sheikh@email.com, kavita.deshmukh@email.com / password - mentioned in email
- n8n workflow: n8n/royalty-summary-workflow.json
- loom video: shared in email

Good authors to test with: Priya (one book paid, one pending), Farhan (all paid), Kavita (one book in production, ₹850 pending, below the ₹1,000 minimum).

## ⚠️ Free plan limitations (please read first)

> **The Bubble → n8n send can't complete on the live app.** On my free plan, the backend workflow that calls n8n stopped running, even though Settings still showed workload remaining. Bubble's logs stayed empty throughout the project, so I couldn't confirm the cause; my assumption is a free-plan restriction. I tried to buy the Starter plan, but the payment kept failing.
>
> **How to test the flow instead:** in the admin portal, open any author (**View**) and click **View curl**. It shows the exact payload Bubble sends, built from that author's real data, as a curl command. Replace `<SECRET>` with the `x-webhook-secret` value shared on the email thread and run it. n8n generates the email and sends it. The notification log won't update, because there's no callback on the free plan (see below).
>
> **Emails go to my inbox for now.** The dataset's author emails are dummy addresses, so n8n sends in test mode: "[TEST]" in the subject, and the body names the real recipient.

Other free-plan limits that shaped the design:

- **Data API blocked (HTTP 401)** → Bubble pushes the full payload to n8n instead of n8n reading from Bubble.
- **No Workflow API** → n8n can't call Bubble back, so Bubble waits for n8n's reply, with a 2-minute timeout.
- **CSV upload blocked** → the dataset was entered by hand.
- **One editor only** → no collaborator invite; the app URL, logins and recording are shared instead.
- The link above is the development version, because publishing a live version needs a paid plan.

## Database

```
User (built-in)   role: Role, author: Author

Author            external_id, name, email, phone, city, joined_date, user: User,
                  books_count, total_earned, total_paid, total_pending,
                  payout_status: RoyaltyStatus

Book              external_id, author: Author, title, isbn, genre, status: BookStatus,
                  publication_date, mrp, royalty_per_copy, copies_sold,
                  royalty_earned, royalty_paid, royalty_pending, last_payout_date,
                  print_partner, available_on: list of text,
                  royalty_status: RoyaltyStatus

Payout            book: Book, author: Author, amount, payout_date, note

NotificationLog   author: Author, triggered_by: User, trigger_type: TriggerType,
                  batch_id, status: NotifStatus, subject, body, error_message,
                  completed_at, reference_date_used, n8n_execution_id

AppConfig         reference_date, overdue_days (90), min_payout (1000),
                  payout_window_days (45)            -- single record

Option sets       Role (Author, Admin)
                  BookStatus (9 production stages; attrs is_published, stage_order)
                  RoyaltyStatus (Paid up, Pending, Overdue, Not applicable; attrs color_hex, severity)
                  NotifStatus (Queued, Processing, Success, Failed)
                  TriggerType (Single, Bulk)
```

- isbn and phone are text, not numbers (hyphens, masked digits).
- Author totals, payout_status and Book.royalty_status are stored, because Bubble can only filter on stored fields. The backend workflows `recalc_book` and `recalc_author` keep them up to date, run for every record with "Schedule API workflow on a list".
- Payout.author is duplicated from the book, so privacy rules and searches stay one level deep.
- There is one Payout row per paid book, because the dataset only has the last payout date.

## Badges and the reference date

- Green: everything paid. Yellow: pending, within the cycle. Red: pending, and the last payout was over 90 days ago, or a published book older than 90 days was never paid. Grey: in production.
- **The data is a snapshot from about December 2025.** Checked against today's date, everything pending is red and yellow never appears. So statuses are calculated "as of" a reference date. The default is 5 Jan 2026, and the admin can switch between "Use snapshot date" and "Use today".
- The same date is sent to n8n, so the email and the dashboard always match.

## Privacy

- **Authors can only see their own Author record, books and payouts. Admins see everything.** The notification log is admin-only.
- Every type is private by default, and pages redirect the wrong role back to login, but the privacy rules are what actually protect the data.
- One thing I ran into: a rule written as `This Book's author's user is Current User` hides books from searches, so the book list came back empty. Rewriting it as `This Book's author is Current User's author` fixed it.

## How Bubble talks to n8n

1. The admin clicks "Generate & Send" (or the bulk button, which does the same for every author with a pending balance).
2. Bubble creates a NotificationLog (Queued) and runs the backend workflow `send_summary`.
3. `send_summary` marks the log Processing and POSTs the author's data to the n8n webhook, with a secret header.
4. n8n generates the email and replies right away with Success or Failed.
5. Bubble saves the reply on the log, and the admin's log list updates by itself.

Payload Bubble sends:

```
log_id, author_name, author_email, reference_date (yyyy-mm-dd), total_pending,
books: [ { title, status, copies_sold, royalty_earned, royalty_paid,
           royalty_pending, last_payout_date, publication_date } ]
```

Test it directly (replace `<SECRET>`), or use the **View curl** button in the app:

```
curl -X POST "https://amanzrx4.app.n8n.cloud/webhook/royalty-summary" \
  -H "Content-Type: application/json" \
  -H "x-webhook-secret: <SECRET>" \
  -d '{"log_id":"test-sneha","author_name":"Sneha Kulkarni","author_email":"sneha.kulkarni@email.com","reference_date":"2026-01-05","total_pending":7865,"books":[{"title":"The Algorithm of Love","status":"Published & Live","copies_sold":1567,"royalty_earned":39175,"royalty_paid":35000,"royalty_pending":4175,"last_payout_date":"2025-11-15","publication_date":"2022-12-25"},{"title":"Ctrl+Alt+Delete My Ex","status":"Published & Live","copies_sold":723,"royalty_earned":21690,"royalty_paid":18000,"royalty_pending":3690,"last_payout_date":"2025-10-15","publication_date":"2024-02-14"},{"title":"Midnight in Mysore","status":"Cover Design","copies_sold":0,"royalty_earned":0,"royalty_paid":0,"royalty_pending":0,"last_payout_date":"","publication_date":""}]}'
```

Success reply: `{"status":"Success","log_id":...,"subject":...,"body":...,"email_status":...,"execution_id":...}`
Failure reply: `{"status":"Failed","log_id":...,"error":"...","execution_id":...}`
Wrong or missing secret: HTTP 403.

## n8n workflow

Webhook (header auth) → Validate & Compute → DeepSeek → Validate AI output → Send email → Respond to webhook. **Each step that can fail has a branch that returns a Failed reply.**

- Validate & Compute checks the payload, recalculates the totals from the books, and works out everything the email needs: next payout date (quarter end + 45 days), all paid, never paid, below ₹1,000, overdue books, books in production.
- The email is sent in test mode (see the free plan section above).

## AI prompt

- **The code computes all the numbers and dates; the AI only writes the email.** It's told to copy figures exactly and never invent any.
- The system prompt contains the knowledge base (royalty policy, payout cycle, production stages) and the tone guidelines.
- Flags tell the AI which case it's writing for:
  - all paid → thank the author, no date
  - never paid or overdue → apologise, ask them to check bank details, promise resolution within 48 hours
  - below ₹1,000 → explain the rollover
  - book in production → short status update
- DeepSeek returns JSON with a subject and body. n8n rejects it if it's empty or missing the correct pending total.
- Only the author's name and book figures are sent to the AI. No phone numbers or bank details.

## Errors, idempotency, timeouts

**How it works:** send_summary calls n8n and **waits for the reply**. A callback from n8n into Bubble would need Bubble's Workflow API, which the free plan doesn't offer, so the result comes back as the webhook response instead. Because this runs in a backend workflow, the admin's screen isn't blocked while it waits.

- n8n always replies, whether it succeeded or failed. A bad payload, an AI timeout (30 s) or a bad AI answer all come back as Failed with the reason, and the log shows Failed.
- If n8n is down or rejects the secret, the API call errors. "Include errors in response" lets the workflow continue, so the log is marked Failed instead of stopping silently.
- **If no reply arrives, the log stays Processing. After 2 minutes the admin list shows it as "Timed out" in red**, re-checked every 10 seconds, so nobody is left watching a spinner.
- Double clicks: the Send button is disabled while that author has a summary in progress, and the workflow checks the same condition before it runs.
- Trade-off of waiting instead of a callback: if n8n finishes after Bubble has stopped waiting, the email can still be sent while the log shows Timed out.

## Assumptions

- The ₹1,000 minimum applies to the author's total pending amount.
- Payout history is one combined row per book, because the dataset only gives the last payout date.
