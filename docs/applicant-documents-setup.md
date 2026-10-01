# Applicant Closing Documents — Setup Guide

This guide covers everything outside this repo that `amko-applicant-documents.html`
needs: the SharePoint lists, the Power Automate intake flow, the two emails it
sends, and the one-line change to the "Documents Needed to Close" email so its
button opens the new page.

The page posts one JSON body per submission to a Power Automate
"When an HTTP request is received" trigger. Files travel inside that JSON as
base64. Each file is limited to 15 MB in the browser; a submission of all five
slots at that size is about 100 MB encoded, which is the trigger's ceiling, so
do not raise the per-file limit without splitting submissions.

---

## 1. SharePoint

### LeaseFiles (document library, already exists)

No changes. The flow creates one folder per request ID inside it and saves files as
`REQ-xxx_<DocType>_<original-name>`.

### LeaseDocuments (new list)

One row per file received. Create it on the AMKO Advisors Team Site with these columns
(internal names in code font; create them with exactly these names so the flow's
expressions match):

| Column | Type | Notes |
|---|---|---|
| Title | Single line (default) | Request ID |
| `EntityName` | Single line | |
| `DocumentType` | Choice | Invoice, Resolution, Insurance, W9, Other |
| `DocumentLabel` | Single line | Friendly label shown on the page |
| `FileName` | Single line | Name as saved in LeaseFiles |
| `OriginalFileName` | Single line | Name the entity uploaded |
| `FileSize` | Number | Bytes |
| `FileLink` | Hyperlink | Link to the file in LeaseFiles |
| `UploadedBy` | Single line | |
| `UploadedByTitle` | Single line | |
| `UploadedByEmail` | Single line | |
| `UploaderNotes` | Multiple lines (plain) | |

### Lease Requests (existing list, two new columns)

| Column | Type | Notes |
|---|---|---|
| `DocumentsStatus` | Choice | Not Started, In Progress, Complete (default Not Started) |
| `DocumentsCompleteDate` | Date and time | Set when the fourth required document lands |

---

## 2. Flow: "Lease Documents - Process Upload"

Create a new instant cloud flow with the trigger **When an HTTP request is received**.
Set "Who can trigger the flow" to **Anyone** (same as the decision-intake flow).

### Trigger — Request Body JSON Schema

```json
{
  "type": "object",
  "properties": {
    "requestID": { "type": "string" },
    "entityName": { "type": "string" },
    "leaseAmount": { "type": "string" },
    "leasePurpose": { "type": "string" },
    "winningBank": { "type": "string" },
    "uploaderName": { "type": "string" },
    "uploaderTitle": { "type": "string" },
    "uploaderEmail": { "type": "string" },
    "notes": { "type": "string" },
    "uploadTimestamp": { "type": "string" },
    "files": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "docType": { "type": "string" },
          "docLabel": { "type": "string" },
          "fileName": { "type": "string" },
          "contentType": { "type": "string" },
          "size": { "type": "integer" },
          "contentBase64": { "type": "string" }
        }
      }
    }
  }
}
```

After the first save, copy the trigger's **HTTP URL** and paste it into
`amko-applicant-documents.html` in place of `PASTE_DOCUMENT_UPLOAD_FLOW_URL_HERE`.

### Steps

Names matter: expressions below reference steps by these exact names
(spaces become underscores in expressions, e.g. `Compose_-_RequestID`).

**1. Compose - RequestID**
```
triggerBody()?['requestID']
```

**2. Get items - Req** (SharePoint) — List: Lease Requests, Filter Query:
```
Title eq '@{outputs('Compose_-_RequestID')}'
```
Top Count 1.

**3. Compose - Req**
```
first(outputs('Get_items_-_Req')?['body/value'])
```

**4. Apply to each - Files** — Select an output:
```
triggerBody()?['files']
```
Inside the loop:

- **Compose - SafeName** — strips characters SharePoint rejects in file names:
  ```
  concat(
    outputs('Compose_-_RequestID'), '_', items('Apply_to_each_-_Files')?['docType'], '_',
    replace(replace(replace(replace(replace(replace(replace(replace(replace(replace(
      items('Apply_to_each_-_Files')?['fileName'],
      '#',''),'%',''),'&','and'),'*',''),':',''),'<',''),'>',''),'?',''),'/','-'),'\','-')
  )
  ```
- **Create file** (SharePoint) — Site: AMKO Advisors Team Site. Folder Path:
  ```
  /LeaseFiles/@{outputs('Compose_-_RequestID')}
  ```
  File Name: `@{outputs('Compose_-_SafeName')}`. File Content:
  ```
  base64ToBinary(items('Apply_to_each_-_Files')?['contentBase64'])
  ```
  Create file creates the request-ID folder if it does not exist yet.
- **FileLink note:** Create file does not output a "Link to item" token. Build the
  link from the path instead (used in Create item below):
  ```
  concat('https://amkoadvisors240.sharepoint.com', if(startsWith(outputs('Create_file')?['body/Path'], '/'), '', '/'), replace(outputs('Create_file')?['body/Path'], ' ', '%20'))
  ```
- **Create item** (SharePoint) — List: LeaseDocuments. Map:
  - Title: `@{outputs('Compose_-_RequestID')}`
  - EntityName: `@{triggerBody()?['entityName']}`
  - DocumentType: `@{items('Apply_to_each_-_Files')?['docType']}`
  - DocumentLabel: `@{items('Apply_to_each_-_Files')?['docLabel']}`
  - FileName: `@{outputs('Compose_-_SafeName')}`
  - OriginalFileName: `@{items('Apply_to_each_-_Files')?['fileName']}`
  - FileSize: `@{items('Apply_to_each_-_Files')?['size']}`
  - FileLink: the path-based link expression from the note above (fx tab)
  - UploadedBy: `@{triggerBody()?['uploaderName']}`
  - UploadedByTitle: `@{triggerBody()?['uploaderTitle']}`
  - UploadedByEmail: `@{triggerBody()?['uploaderEmail']}`
  - UploaderNotes: `@{triggerBody()?['notes']}`

Set the loop's **Concurrency Control** to off (sequential) so files save in order.

**5. Get items - AllDocs** (SharePoint) — List: LeaseDocuments, Filter Query:
```
Title eq '@{outputs('Compose_-_RequestID')}'
```
Top Count 500. This returns everything ever received for the request, including
earlier submissions.

**6. Select - DocTypes** — From: value of Get items - AllDocs. Switch the Map to
text mode (the small icon on the right) and enter:
```
item()?['DocumentType']?['Value']
```
DocumentType is a Choice column, so SharePoint returns it as an object with a
`Value` property; the `?['Value']` is required or the checklist will never match.
This yields a plain array of type names, e.g. `["Invoice","Resolution"]`.

**7. Compose - AllReceived**
```
and(
  contains(body('Select_-_DocTypes'), 'Invoice'),
  contains(body('Select_-_DocTypes'), 'Resolution'),
  contains(body('Select_-_DocTypes'), 'Insurance'),
  contains(body('Select_-_DocTypes'), 'W9')
)
```

**8. Compose - ChecklistRows** — the HTML rows both emails share:
```
concat(
  '<tr><td style="padding:6px 12px;border-bottom:1px solid #E2E8F0;">Invoice, quote, or purchase agreement</td><td style="padding:6px 12px;border-bottom:1px solid #E2E8F0;font-weight:bold;color:', if(contains(body('Select_-_DocTypes'),'Invoice'),'#1A6B3A;">&#10003; Received','#8B1A1A;">Outstanding'), '</td></tr>',
  '<tr><td style="padding:6px 12px;border-bottom:1px solid #E2E8F0;">Authorizing resolution or meeting minutes</td><td style="padding:6px 12px;border-bottom:1px solid #E2E8F0;font-weight:bold;color:', if(contains(body('Select_-_DocTypes'),'Resolution'),'#1A6B3A;">&#10003; Received','#8B1A1A;">Outstanding'), '</td></tr>',
  '<tr><td style="padding:6px 12px;border-bottom:1px solid #E2E8F0;">Certificate of insurance</td><td style="padding:6px 12px;border-bottom:1px solid #E2E8F0;font-weight:bold;color:', if(contains(body('Select_-_DocTypes'),'Insurance'),'#1A6B3A;">&#10003; Received','#8B1A1A;">Outstanding'), '</td></tr>',
  '<tr><td style="padding:6px 12px;">Completed IRS Form W-9</td><td style="padding:6px 12px;font-weight:bold;color:', if(contains(body('Select_-_DocTypes'),'W9'),'#1A6B3A;">&#10003; Received','#8B1A1A;">Outstanding'), '</td></tr>'
)
```

**9. Compose - ThisUploadRows** — what arrived in this submission. Use a
**Select - ThisUpload** first (From: `triggerBody()?['files']`, text-mode Map):
```
concat('<li>', item()?['docLabel'], ' &mdash; ', item()?['fileName'], '</li>')
```
then Compose - ThisUploadRows:
```
join(body('Select_-_ThisUpload'), '')
```

**10. Update item - Lease Requests** — Id: `@{outputs('Compose_-_Req')?['ID']}`.
- DocumentsStatus: `@{if(outputs('Compose_-_AllReceived'), 'Complete', 'In Progress')}`
- DocumentsCompleteDate: `@{if(outputs('Compose_-_AllReceived'), utcNow(), null)}`

(If the Update item action rejects a null date, wrap this step in a Condition on
AllReceived and only set the date on the true side.)

**11. Compose - ReceiptBody** — see template A below.

**12. Send an email (V2) - Entity Receipt** — To: `@{triggerBody()?['uploaderEmail']}`.
Subject:
```
AMKO Lease Advantage — Documents Received: @{triggerBody()?['entityName']} (@{outputs('Compose_-_RequestID')})
```
Body: `@{outputs('Compose_-_ReceiptBody')}`. Attachments: none.

**13. Compose - InternalBody** — see template B below.

**14. Send an email (V2) - Internal Alert** — To: AMKO Lease Advantage mailbox and
bond counsel's address, separated by a semicolon. Subject:
```
@{if(outputs('Compose_-_AllReceived'), 'ALL DOCUMENTS RECEIVED', 'Documents Received')} — @{triggerBody()?['entityName']} (@{outputs('Compose_-_RequestID')})
```
Body: `@{outputs('Compose_-_InternalBody')}`.

No Response action is needed. The trigger returns 202 Accepted on its own, which
the page treats as success.

---

## 3. Email templates

Both use the hosted logo and the 620px table wrapper that renders correctly in Outlook.

### A. Compose - ReceiptBody (to the entity)

```html
<table width="620" cellpadding="0" cellspacing="0" border="0" align="center" style="width:620px;border-collapse:collapse;font-family:Arial,Helvetica,sans-serif;background-color:#ffffff;border:1px solid #E2E8F0;"><tr><td style="font-family:Arial,Helvetica,sans-serif;background-color:#1A6B3A;padding:14px 28px;color:#ffffff;font-size:15px;font-weight:bold;letter-spacing:0.5px;">AMKO LEASE ADVANTAGE &mdash; DOCUMENTS RECEIVED</td></tr><tr><td style="font-family:Arial,Helvetica,sans-serif;padding:28px 28px 20px 28px;"><p style="font-family:Arial,Helvetica,sans-serif;margin:0 0 18px;font-size:15px;color:#1A1A2E;font-weight:bold;">Dear @{triggerBody()?['entityName']},</p><p style="font-family:Arial,Helvetica,sans-serif;margin:0 0 14px;font-size:15px;color:#4A5568;line-height:1.6;">Thank you. AMKO Capital has received the following documents for request <strong>@{outputs('Compose_-_RequestID')}</strong>, submitted by @{triggerBody()?['uploaderName']} on @{convertTimeZone(utcNow(),'UTC','Central Standard Time','MMMM d, yyyy h:mm tt')} CT:</p><ul style="font-family:Arial,Helvetica,sans-serif;margin:0 0 20px 20px;padding:0;font-size:14px;color:#1A1A2E;line-height:1.7;">@{outputs('Compose_-_ThisUploadRows')}</ul><table width="100%" cellpadding="0" cellspacing="0" border="0" style="border-collapse:collapse;margin:0 0 20px;border:1px solid #E2E8F0;font-family:Arial,Helvetica,sans-serif;font-size:14px;"><tr><td colspan="2" style="font-family:Arial,Helvetica,sans-serif;background-color:#1B3A6B;padding:10px 16px;color:#ffffff;font-weight:bold;font-size:15px;">Closing Document Checklist</td></tr>@{outputs('Compose_-_ChecklistRows')}</table>@{if(outputs('Compose_-_AllReceived'), '<p style="font-family:Arial,Helvetica,sans-serif;margin:0 0 16px;font-size:15px;color:#1A6B3A;font-weight:bold;">All required documents have been received. Bond counsel will now begin preparing your lease documents, and AMKO Capital will be in touch regarding closing.</p>', '<p style="font-family:Arial,Helvetica,sans-serif;margin:0 0 16px;font-size:15px;color:#4A5568;line-height:1.6;">Items marked <strong style="color:#8B1A1A;">Outstanding</strong> are still needed before closing can proceed. You can upload them at any time using the same link in your Documents Needed to Close email.</p>')}<p style="font-family:Arial,Helvetica,sans-serif;margin:0 0 28px;font-size:13px;color:#4A5568;line-height:1.6;">If you have any questions, reply to this email or contact us at <a href="mailto:info@amkocap.com" style="color:#1B3A6B;">info@amkocap.com</a> or (701) 540-6821.</p><img src="https://www.dropbox.com/scl/fi/s8devubl7bm9mlvzby9gr/AMKO-Capital-Tagline.jpg?rlkey=00ia3pmw8ki3qq7ta9vktyblu&st=r3cgyips&raw=1" width="260" style="display:block;border:0;outline:none;margin-bottom:8px;" alt="AMKO Capital" /><p style="font-family:Arial,Helvetica,sans-serif;margin:0;font-size:13px;color:#4A5568;">AMKO Lease Advantage &nbsp;&middot;&nbsp; Municipal Lease Program</p></td></tr></table>
```

### B. Compose - InternalBody (to AMKO Lease Advantage and bond counsel)

```html
<div style="font-family:Arial,sans-serif;max-width:620px;margin:0;background-color:#ffffff;border:1px solid #E2E8F0;"><div style="background-color:@{if(outputs('Compose_-_AllReceived'),'#1A6B3A','#1B3A6B')};padding:14px 28px;"><span style="color:#ffffff;font-size:15px;font-weight:bold;letter-spacing:0.5px;">@{if(outputs('Compose_-_AllReceived'),'ALL CLOSING DOCUMENTS RECEIVED','CLOSING DOCUMENTS RECEIVED')} &mdash; INTERNAL ALERT</span></div><div style="padding:24px 28px 8px;"><p style="margin:0 0 14px;font-size:14px;color:#4A5568;line-height:1.7;">@{if(outputs('Compose_-_AllReceived'),'All four required closing documents are now on file for this request. Bond counsel may begin preparing lease documentation.','An entity has uploaded closing documents. Items still outstanding are listed below.')} Files are saved in <strong>LeaseFiles / @{outputs('Compose_-_RequestID')}</strong>.</p></div><table cellpadding="0" cellspacing="0" style="font-family:Arial,sans-serif;border-collapse:collapse;margin:0 28px 20px;border:1px solid #E2E8F0;width:calc(100% - 56px);"><tbody><tr><td colspan="2" style="background-color:#1B3A6B;padding:10px 16px;color:#ffffff;font-weight:bold;font-size:15px;">Upload Summary</td></tr><tr><td style="padding:6px 16px;font-weight:bold;color:#1B3A6B;font-size:13px;background-color:#F4F6F9;width:40%;">Request ID</td><td style="padding:6px 16px;font-size:13px;color:#1A1A2E;background-color:#F4F6F9;font-family:monospace;">@{outputs('Compose_-_RequestID')}</td></tr><tr><td style="padding:6px 16px;font-weight:bold;color:#1B3A6B;font-size:13px;">Entity</td><td style="padding:6px 16px;font-size:13px;color:#1A1A2E;">@{triggerBody()?['entityName']}</td></tr><tr><td style="padding:6px 16px;font-weight:bold;color:#1B3A6B;font-size:13px;background-color:#F4F6F9;">Winning Bank</td><td style="padding:6px 16px;font-size:13px;color:#1A1A2E;background-color:#F4F6F9;">@{triggerBody()?['winningBank']}</td></tr><tr><td style="padding:6px 16px;font-weight:bold;color:#1B3A6B;font-size:13px;">Uploaded By</td><td style="padding:6px 16px;font-size:13px;color:#1A1A2E;">@{triggerBody()?['uploaderName']} (@{triggerBody()?['uploaderTitle']}) &middot; <a href="mailto:@{triggerBody()?['uploaderEmail']}" style="color:#1B3A6B;">@{triggerBody()?['uploaderEmail']}</a></td></tr><tr><td style="padding:6px 16px;font-weight:bold;color:#1B3A6B;font-size:13px;background-color:#F4F6F9;">Timestamp</td><td style="padding:6px 16px;font-size:13px;color:#1A1A2E;background-color:#F4F6F9;">@{convertTimeZone(utcNow(),'UTC','Central Standard Time','MMMM d, yyyy h:mm tt')} CT</td></tr><tr><td style="padding:6px 16px;font-weight:bold;color:#1B3A6B;font-size:13px;vertical-align:top;">Files in This Upload</td><td style="padding:6px 16px;font-size:13px;color:#1A1A2E;"><ul style="margin:0 0 0 16px;padding:0;">@{outputs('Compose_-_ThisUploadRows')}</ul></td></tr><tr><td style="padding:6px 16px;font-weight:bold;color:#1B3A6B;font-size:13px;background-color:#F4F6F9;vertical-align:top;">Entity Notes</td><td style="padding:6px 16px;font-size:13px;color:#1A1A2E;background-color:#F4F6F9;">@{if(empty(triggerBody()?['notes']),'(none provided)',triggerBody()?['notes'])}</td></tr></tbody></table><table cellpadding="0" cellspacing="0" style="font-family:Arial,sans-serif;border-collapse:collapse;margin:0 28px 20px;border:1px solid #E2E8F0;width:calc(100% - 56px);font-size:13px;"><tbody><tr><td colspan="2" style="background-color:#1B3A6B;padding:10px 16px;color:#ffffff;font-weight:bold;font-size:15px;">Checklist Status</td></tr>@{outputs('Compose_-_ChecklistRows')}</tbody></table><p style="margin:0 28px 24px;font-size:12px;color:#6B6760;">This is an automated internal notification from AMKO Lease Advantage. The LeaseDocuments SharePoint list contains the full upload history for this request.</p></div>
```

---

## 4. Point the "Documents Needed to Close" email at the new page

In the Lease Decision flow, open **Compose - DocRequestBody** and replace the
`href` on the "Upload Your Documents" button (currently the shared SharePoint folder
link) with:

```
@{concat('https://c71-brianosowski.github.io/amko-bank-response/amko-applicant-documents.html?requestID=', outputs('Compose_-_RequestID'), '&entityName=', encodeUriComponent(outputs('Compose_-_Req')?['EntityName']), '&leaseAmount=', string(outputs('Compose_-_Req')?['LeaseAmount']), '&leasePurpose=', encodeUriComponent(outputs('Compose_-_Req')?['LeasePurpose']), '&winningBank=', encodeUriComponent(first(outputs('Get_items_-_WinningBid')?['body/value'])?['Title']))}
```

Then change the yellow note under the button from the "add your entity name to each
file name" instruction to:

> Your documents are matched to this request automatically. You may return to the
> upload page as many times as needed until all items are received.

---

## 5. Test plan

1. Save the flow, copy the trigger URL into the page, and push to `main`.
2. Open the documents email link for a throwaway request ID. Upload one file.
   Confirm: file appears in LeaseFiles under the request-ID folder, one row in
   LeaseDocuments, entity receipt lists the file and three Outstanding items,
   internal alert subject reads "Documents Received", Lease Requests shows
   In Progress.
3. Return to the same link and upload the remaining three. Confirm: internal alert
   subject reads "ALL DOCUMENTS RECEIVED", receipt shows four Received, Lease
   Requests shows Complete with a completion date.
4. Upload a replacement for one document. Confirm a second row is added rather
   than overwriting, and the checklist still shows Received.
5. Try a 16 MB file. Confirm the page blocks it with the contact-AMKO message
   before anything is sent.
