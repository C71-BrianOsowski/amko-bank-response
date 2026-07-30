# Bank Bid Access Code — Power Automate + SharePoint Setup

This is the backend setup for the access-code gate on `amko-bank-response.html`.
It mirrors how an **applicant** gets into the application: they enter a code, a
Power Automate flow checks that code against a SharePoint list, and access is
granted only if it matches. The bank bid page works the same way.

## How it works end to end

1. A bank opens the bid link and sees only an **"Enter Your Access Code"** card —
   the opportunity details and bid form stay hidden.
2. They enter their code and click **Continue**. The page POSTs the code to the flow.
3. The flow looks the code up in SharePoint, writes the attempt to **Platform
   Access Log**, and responds `{ "valid": true }` or `{ "valid": false }`.
4. Valid → the bid form unlocks. The verified code is then included in the bid
   submission so it's recorded with the bid.

The webpage does **not** need to know how codes are scoped — it just sends the
code plus the opportunity context and trusts the flow's yes/no. No page changes
are needed to complete this; only the flow + SharePoint steps below.

## 1. SharePoint — where the bank code lives

The direct parallel to `AccessCode` on the **Approved Entities** list (used for
applicants) is an `AccessCode` column on the **Banks** list:

- Add a single-line text column **`AccessCode`** to the **Banks** list.
- Give each active bank a code, e.g. `AMKO-FNB-001`.

> Prefer one central list instead? Use **Platform Access Codes** — the flow steps
> are identical, just point "Get items" at that list. That's the better choice if
> a bank needs different codes per opportunity or codes that expire.

## 2. Power Automate — the verify flow

Easiest path: **Save As** a copy of your existing applicant verify flow and swap
the list. Otherwise, build it like this:

**Trigger:** *When an HTTP request is received* (the manual trigger the page
already calls). The page sends:

```json
{
  "action": "verifyAccessCode",
  "accessCode": "AMKO-FNB-001",
  "requestID": "REQ-1001",
  "entityName": "City of Fargo"
}
```

**Steps:**

1. **Condition** — `action` is equal to `verifyAccessCode`. (Only needed if this
   flow also handles bid submission; see section 3.)
2. **Get items** — Site: *AMKO Advisors Team Site*, List: **Banks**
   - *Filter Query:* `AccessCode eq '@{triggerBody()?['accessCode']}' and Active eq 1`
     (use `Status eq 'Active'` if that's your column)
   - *Top Count:* 1
3. **Create item** in **Platform Access Log** — record the code, the matched bank
   name (from Get items, if any), `requestID`, the result, and the timestamp.
4. **Condition** — `length(body('Get_items')?['value'])` is greater than 0
   - **If yes** → *Respond to a PowerApp or flow* / *Response*: body `{ "valid": true }`
   - **If no**  → *Respond to a PowerApp or flow* / *Response*: body `{ "valid": false }`

> **Important:** the reply MUST be a **Respond to a PowerApp or flow** (or HTTP
> **Response**) action returning JSON. Your current bid flow returns `202 Accepted`
> with no body, which is why the page treats any reply as success today. Without a
> real JSON response the gate can't tell valid from invalid and **fails closed**
> (nobody gets in) — so wire up the Response action before going live.

## 3. Bid submission (your existing flow)

The bid POST now also carries `action: "submitBid"` and `accessCode`:

- If **one flow** handles both verify and submit, branch on `action` at the top.
- If **separate flows**, the submit flow can ignore `action` and simply store
  `accessCode` alongside the rest of the bid.

## Page contract (already implemented — no code changes required)

- **Verify request:** `{ action: "verifyAccessCode", accessCode, requestID, entityName }`
- **Verify response expected:** `{ "valid": true }` or `{ "valid": false }`
  (the page also accepts `status: "valid"` or `access: "granted"`)
- **`ACCESS_FLOW_URL`** in the page defaults to the same URL as the bid
  submission flow (`FLOW_URL`), distinguished by the `action` field. If you build
  a dedicated verify flow, change `ACCESS_FLOW_URL` near the top of the page's
  `<script>` to that flow's URL.
