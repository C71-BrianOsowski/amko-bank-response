# Bank Bid Access — Build Plan & Daily Status

_Open this first each day. It's the running record of where we are, what's next,_
_and the exact steps to do in Power Automate (Claude can't reach your tenant, so_
_this doc is the source of truth)._

Last updated: 2026-07-30

---

## The approach we chose: link token (not a typed code)

Each bank's invitation link carries a long random **token** in the URL. The bank
just clicks the link — nothing to type. The bid page silently checks the token
against SharePoint and opens the form only if it matches. It's unguessable, it's
zero-friction, and it scales: every invite auto-generates its own token, whether
that's 5 banks or 500.

> A typed access code is the fallback if you ever want banks to enter something
> by hand — but for growth, the token is the better fit.

---

## Where we are — check off as you go

- [x] Bid page gates the form until access is verified _(on branch `claude/bank-bid-access-code-rgntzc`)_
- [ ] **Piece 1 — Invitation loop: generate token, store it, append `&token=` to the link**  ← YOU ARE HERE
- [ ] Piece 2 — Validate flow (`AMKO - Validate Access Code`): match on token, return valid/invalid
- [ ] Piece 3 — Switch bid page from typed-code entry to silent token check _(Claude will do on your go)_
- [ ] End-to-end test: send a real invite, click the link, confirm the form opens
- [ ] Decide: expire tokens after the submission deadline? (optional)

---

## Piece 1 — Invitation email loop  ← today's work
**Flow:** `AMKO Lease - Application Processing v2` → **Loop – Send Bank Opportunity Emails**
**Token list:** `Platform Access Codes` (rename to `Bank Access Codes` if you like — safe,
cosmetic; Power Automate references it by internal ID). Applicant codes live separately in
`Approved Entities`, so this list is free to be bank-specific.

Inside the loop, for each bank, add these before the email is composed:

1. **Generate the token** — add a **Compose** action named `BankToken`
   - Expression: `guid()`   _(want it longer? `concat(guid(), guid())`)_

2. **Store the token** — add **Create item** (SharePoint) in **Platform Access Codes**,
   mapping onto its EXISTING columns:

   | Column | Value |
   |---|---|
   | `AccessCode` | `outputs('Compose_-_BankToken')`  ← the token the link checks against |
   | `Organization` | current bank's **Bank Name** |
   | `PersonName` | current bank's **ContactName** |
   | `Email` | current bank's **ContactEmail** |
   | `Active` | `Yes` |
   | `DateIssued` | expression `utcNow()` |
   | `ExpiresOn` | *(optional)* submission deadline, or blank |
   | `Notes` | *(optional)* `Bank bid access – <RequestID>` for traceability |

   No RequestID column is needed — the token is unique on its own. (Add one later only
   if you want a link to work for just one specific opportunity.)

3. **Put the token in the link** — in **Compose – BankOpportunityEmailBody**, build the URL as:
   ```
   https://<your-page-host>/amko-bank-response.html?requestID=<id>&entityName=<name>&token=@{outputs('Compose_-_BankToken')}
   ```
   (keep the other params you already pass; just add `&token=…`)

---

## Piece 2 — Validate flow
**Flow:** `AMKO - Validate Access Code` (the one with manual → Get items → Condition → Response)

- **Get items** from **Platform Access Codes**
  - Filter Query: `AccessCode eq '@{triggerBody()?['token']}' and Active eq 1`
  - Top Count: 1
  - (optional, tighter) also check `ExpiresOn` is in the future, or match the RequestID
- **Create item** in **Platform Access Log** — record token, bank, requestID, result, timestamp
- **Condition:** `length(body('Get_items')?['value'])` is greater than `0`
  - **True** → **Response**, body: `{ "valid": true }`
  - **False** → **Response**, body: `{ "valid": false }`
- The Response action MUST return JSON (a bare 202 with no body = the page can't
  read it and stays locked).

---

## Piece 3 — Bid page (this repo)
- Reads `token` from the URL and silently POSTs it to the validate flow on load.
- Valid → the bid form shows. Invalid/missing → a locked message.
- Right now the page shows a **typed-code** gate — Claude swaps it to the silent
  token check once Pieces 1 & 2 exist. Just say the word.

---

## Page ↔ flow contract (so both sides agree)
- **Validate request** the page sends: `{ action: "verifyAccessCode", token, requestID, entityName }`
- **Validate response** the page expects: `{ "valid": true }` or `{ "valid": false }`
- **`ACCESS_FLOW_URL`** near the top of the page's `<script>` = the Validate flow's trigger URL.

---

## Open decisions
- Which list holds tokens: **Platform Access Codes** (recommended) or Bank Lease Responses?
- Expire/mark-used tokens after the deadline? (nice-to-have, not required for v1)
