# HubSpot Deal Update — Inbound Email to CRM

Processes the user's Outlook inbox and auto-creates HubSpot deals from qualifying inbound enquiries, or logs engagement notes against existing open deals. All output lands directly in HubSpot.

## Usage

```
/hubspot-deal-update
```

Before running, confirm the following with the user if not already known:

- **ownerEmail** — their Outlook/Microsoft 365 email address (e.g. `michael.barrett@tropicalmarinecentre.co.uk`)
- **hubspotOwnerId** — their HubSpot owner ID (find via `mcp__HubSpot__search_owners` if unknown)
- **internalDomain** — the company's email domain to suppress (e.g. `tropicalmarinecentre.co.uk`)

---

## STEP 0 — Lookback window

Run `date +%u` via Bash.
- Monday (1): lookback = 72 hours
- Otherwise: 24 hours

---

## STEP 1 — Pull recent email

Call `mcp__Microsoft-365__outlook_email_search` with:
- `afterDateTime`: lookback window (e.g. "72 hours ago" or "24 hours ago")
- `mailboxOwnerEmail`: **ownerEmail**
- `limit`: 50

**Noise filter — DROP outright:**
- Sender domain matches **internalDomain**
- Sender contains `notifications@tasks.clickup.com`, `noreply@`, `no-reply@`, `@email.amazonses.com`, `mailer-daemon@`
- Subject is null, empty, or starts with: `Call Logs`, `Undeliverable`, `Automatic reply`, `Out of Office`, `Delivery Status Notification`

---

## STEP 2 — Qualify each remaining email (MEDIUM filter)

**Two mandatory pre-conditions — both must be true:**
1. **ownerEmail appears in the To or CC field** of the email (read full email via `mcp__Microsoft-365__read_resource` if To/CC not visible in search results)
2. The body/summary expresses concrete commercial intent:
   - References a product, stock, pricing, quote, availability, enquiry, sample, MOQ, lead time, shipping, RFQ
   - Or phrases like "interested in", "looking for", "can you supply", "do you stock"
   - Or an external sender introducing themselves as a buyer, retailer, wholesaler, or distributor

**Drop** purely conversational email, marketing newsletters, vendor pitches selling TO the company, shared-inbox emails not addressed to the user, and emails where the user did not personally send a reply.

For each qualifying email capture:
- `senderEmail` (lowercase), `senderName` (display name or email local-part), `senderDomain`
- `subject`, `summaryBody`, `webLink`, `receivedDateTime` (as epoch ms for `hs_timestamp`)
- `topicKeyword` (inferred product/subject noun, e.g. "UV sterilisers", "LED panels")
- `explicitAmount` (number only if clearly stated)

---

## STEP 3 — Dedup by contact

For each qualifying sender, call `mcp__HubSpot__search_crm_objects`:
- `objectType`: "contacts"
- filter: `email EQ <senderEmail>`
- `properties`: ["email","firstname","lastname","company","hs_object_id"]
- `limit`: 1

**Branch A — Contact EXISTS:**
Search for open deals associated with this contact:
- `objectType`: "deals"
- filter: `dealstage IN ["77461203","appointmentscheduled","qualifiedtobuy","354572019"]`
- `associatedWith`: contact ID
- `properties`: ["dealname","dealstage","hs_lastmodifieddate","hubspot_owner_id"]
- `limit`: 5

- **A1** One or more open deals → **LOG NOTE** on the most recently modified open deal. Do NOT create a new deal. Go to STEP 5.
- **A2** No open deals → **CREATE** a new deal associated with the existing contact. Go to STEP 4 (skip contact creation).

**Branch B — Contact does NOT exist:**
→ **CREATE** contact AND deal. Go to STEP 4.

---

## STEP 4 — Create company (optional), contact (if Branch B), deal

### 4a. Company resolution by domain

Free-mail domains — **skip** company association:
`gmail.com, googlemail.com, hotmail.com, outlook.com, live.com, yahoo.com, aol.com, icloud.com, me.com, mac.com, protonmail.com`

Otherwise:
- Search `companies` where `domain EQ <senderDomain>`, limit 1
- If found → capture `company_id`
- If not → create company via `mcp__HubSpot__manage_crm_objects` (CONFIRMATION_WAIVED_FOR_SESSION):
  - `name`: Title-case from domain (e.g. `cascopet.com` → `Cascopet`)
  - `domain`: senderDomain
  - `website`: `https://<senderDomain>`
  - Capture new `company_id`

### 4b. Contact creation (Branch B only)

Create via `mcp__HubSpot__manage_crm_objects` (CONFIRMATION_WAIVED_FOR_SESSION):
- `email`, `firstname`, `lastname`
- `hubspot_owner_id`: **hubspotOwnerId**
- Associate with `company_id` if captured
- Capture `contact_id`

### 4c. Deal creation

Construct `dealname`:
- With company: `<Company name> - <topicKeyword or subject short>`
- Without company: `<senderName> enquiry - <topicKeyword or subject short>`
- Max 80 chars

Create via `mcp__HubSpot__manage_crm_objects` (CONFIRMATION_WAIVED_FOR_SESSION):
- `pipeline`: "default"
- `dealstage`: "77461203" (Planning)
- `hubspot_owner_id`: **hubspotOwnerId**
- `amount`: only if `explicitAmount` present
- `description`: "Auto-created from inbound email. Subject: <subject>. Source: <webLink>. Summary: <first 400 chars of body>."
- Associate with `contact_id` and `company_id` (if captured)
- Capture `deal_id`

---

## STEP 5 — Log source email as a note

Target `deal_id` = new deal (Branches A2/B) or most-recently-modified open deal (Branch A1).

Create note via `mcp__HubSpot__manage_crm_objects` (CONFIRMATION_WAIVED_FOR_SESSION):
- `hs_note_body`:
  ```
  <b>Email activity (auto-logged)</b><br>
  From: <senderName> &lt;<senderEmail>&gt;<br>
  Subject: <subject><br>
  Received: <receivedDateTime ISO>.<br><br>
  <summaryBody first 500 chars><br><br>
  <a href='<webLink>'>Open in Outlook</a>
  ```
- `hs_timestamp`: receivedDateTime epoch ms
- Associate with `deal_id`

---

## STEP 6 — Run summary

Write a one-paragraph summary (no artefacts, no UI — text output only):
- Emails scanned / qualified
- New contacts created (count + names)
- New companies created (count + names)
- New deals created (count + deal names + HubSpot URLs: `https://app.hubspot.com/contacts/25521889/record/0-3/<dealId>`)
- Notes logged on existing deals (count + deal names)
- Errors encountered (if any)

---

## Safety rails

- Never create a deal from an **internalDomain** sender
- Never create a deal without at least one contact association
- Never create duplicate deals — dedup by contact email always runs first
- Hard cap: **20 new deals per run**; create the 20 most recent and flag excess in summary
- Only qualify emails where **the user is in To or CC** — shared-inbox or CC'd-colleague-only emails are skipped
- If `manage_crm_objects` rejects a confirmation-waived call, retry with `confirmationStatus: "CONFIRMED"`
- If any HubSpot call fails, log the error and continue processing remaining emails
- UK English in all note bodies, deal descriptions, and summaries
- Do NOT create, update, or touch any ClickUp artefact
