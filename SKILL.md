---
name: deal-registration-check
description: >
  Process incoming deal registration requests from external partners and contractors.
  Checks Gmail for new Formspree form submissions, looks up each prospective company
  in the Attio pipeline, and posts a full briefing to Slack for Joe to personally review
  and respond to. Does NOT send any determination to the partner automatically — Joe
  approves or declines every registration himself. Run this skill whenever a deal
  registration is submitted, or on a scheduled basis to process new submissions
  automatically. Trigger phrases: "process deal registrations", "check deal registration
  inbox", "run deal registration check", "any new deal registrations?", "review partner
  registrations".
---

# Deal Registration Check

This skill processes deal registration requests submitted by Surfaice's external partners
and contractors via the Surfaice Deal Registration web form.

**Important:** This skill never sends any communication to the partner and never writes a
final determination anywhere. Its sole job is to gather all relevant facts, surface the
pipeline context from Attio, and post a clear briefing to Slack so Joe can make the
determination personally and reply to the partner in his own words within 5 business days.

---

## Configuration

```
GMAIL_SEARCH_QUERY     = "subject:\"New Deal Registration Request\" from:formspree.io"
SLACK_CHANNEL          = "#deal-registrations"
TRACKING_FILE          = "processed_registrations.json"   # in outputs dir
ACTIVE_PIPELINE_LISTS  = ["US customer pipeline", "Sales"]
ACTIVE_STAGES          = ["Lead", "Cold Mapping/ Engaging", "Warm Mapping/Engaging",
                           "Exploration", "Discovery", "NDA - Proposal",
                           "Proposal Out", "Warm", "Contracted"]
EXCLUDED_STAGES        = ["Rejected"]
```

---

## Step-by-step workflow

### Step 1 — Load the processed-registrations tracker

Read `processed_registrations.json` from the outputs directory. This file prevents
re-processing emails that have already been briefed to Joe. Format:

```json
{ "processed_thread_ids": ["thread_abc123", "thread_xyz789"] }
```

If the file doesn't exist, treat it as `{"processed_thread_ids": []}` and create it
at the end of the run.

### Step 2 — Search Gmail for new deal registration submissions

Search Gmail using:
```
subject:"New Deal Registration Request" from:formspree.io
```

For each thread returned:
- Skip if `thread_id` is already in `processed_thread_ids`
- Otherwise fetch the full thread content

If no new threads are found, post a brief note to the Slack channel:
> "✅ Deal Registration Check complete — no new submissions today."

Then stop.

### Step 3 — Parse the submission

Extract these fields from the Formspree email body:

| Field | Form name |
|-------|-----------|
| Partner name | `partner_name` |
| Partner email | `partner_email` |
| Partner company | `partner_company` |
| Partner phone | `partner_phone` (optional) |
| Prospective client | `client_name` |
| Department / buying center | `department` |
| Engagement summary | `engagement_summary` |
| Estimated deal value | `est_deal_value` (optional) |
| Key contacts | `contact_name[]` + `contact_title[]` pairs |

If a field can't be parsed, note "not found" and continue — don't block the run.

### Step 4 — Research the prospective company in Attio

Search the Attio `companies` object for `client_name` (try exact match first, then
partial). For each matching record:

1. Check if the company is in the "US customer pipeline" list (slug: `sales_3`) or
   "Sales" list (slug: `sales_32`).
2. Check the `stages` attribute. Active = anything except "Rejected".
3. Read `source` to understand which channel currently owns this relationship.
4. Read `icp_type` (Construction-Led / Real Estate-Led / Mixed) and
   `primary_buyer_role` for department context.
5. Pull any recent notes that describe the current state of engagement.

Compile everything into a factual summary — do not make a determination yet.

### Step 5 — Draft the pipeline status summary

Based on your Attio research, write a plain-English summary covering:

- **Is this company already in our pipeline?** (Yes / No / Multiple matches)
- **If yes — current stage and how long they've been there**
- **If yes — which channel or source owns the relationship**
- **If yes — what department/function is currently engaged**
- **Does the submitted department overlap with our existing engagement?**
  Assess this honestly. "Real Estate & Store Development" and "Store Development" 
  are the same. "Lease Administration" and "Construction" likely do not overlap.
  When it's genuinely ambiguous, say so plainly.
- **Any other relevant context from notes** (open proposals, past meetings, etc.)

Then offer a **suggested determination** — one of Approved, Conditionally Approved,
or Conflicted — with a brief reason. Make clear this is a suggestion for Joe's review,
not a decision.

### Step 6 — Post the briefing to Slack

Post one message per new registration to `#deal-registrations`. Use this format:

```
📋 *Deal Registration Briefing — Action Required*
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

*Partner:* [Name] — [Company] — [Email] [Phone if provided]
*Prospective Client:* [Client Name]
*Department / Buying Center:* [Department]
*Key Contacts:*
  • [Name], [Title]
  • [Name], [Title]
*Engagement Summary:* [Summary from partner]
*Estimated Value:* [Value or "Not provided"]
*Submitted:* [Date of email]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 *Attio Pipeline Research*

[Plain-English pipeline status summary from Step 5]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💡 *Suggested Determination (for Joe's review)*

[APPROVED / CONFLICTED / CONDITIONALLY APPROVED] — [1–2 sentence reasoning]

⚠️ _This is a suggested analysis only. Joe must personally review and reply to
[partner_email] with the official determination within 5 business days._
```

Formatting notes:
- Keep the pipeline research section factual and specific — cite stage names, source
  labels, and contact names from Attio where available
- If there are multiple Attio matches for the company name, list them all and ask Joe
  to confirm which record is relevant
- If the department overlap question is genuinely close, say so explicitly and ask Joe
  to weigh in rather than offering a leaning

### Step 7 — Log a pending note in Attio

**If the company exists in Attio:**
Create a note on the company record to record that a registration request was received.
Keep this note factual and status-neutral — do not record any determination.

Note title: `Deal Registration Received — [Partner Company] — [Date]`

Note body:
```
Deal registration request received from [Partner Name] ([Partner Company]).
Department: [Department]
Key contacts listed: [Names and titles]
Engagement summary: [Summary]
Estimated value: [Value]
Status: Pending Joe's review. No determination has been made.
```

**If the company does NOT exist in Attio:**
Do not create a new company record yet. Joe may want to do that himself after reviewing
the registration. Just include a note in the Slack post that the company isn't currently
in the CRM.

### Step 8 — Update the tracker and summarize

1. Add each processed `thread_id` to `processed_registrations.json` and save it.
2. Post a brief run summary to Slack:
   > "📬 Deal Registration Check complete — [N] new submission(s) found and briefed above. No automated actions were taken."

---

## Edge cases

- **No new registrations**: Report cleanly to Slack and exit.
- **Parse failure**: Post the raw email body to Slack with a note asking Joe to review manually. Mark as processed so it doesn't reappear.
- **Multiple Attio company matches**: List all matches in the Slack briefing. Ask Joe to confirm which record the note should be logged against.
- **Already-received registration (same partner + company)**: Flag this in the Slack post — it may be a re-submission or follow-up.
- **Company in "Rejected" stage**: Note this clearly. A previously rejected deal should be treated similarly to a company not in the pipeline, but flag it so Joe is aware.

---

## Key Attio field reference

| Field | Slug | Notes |
|-------|------|-------|
| Company name | `name` | Text |
| Pipeline stage | `stages` | Status; active = anything except "Rejected" |
| Source / channel | `source` | Multi-select |
| Dept authority | `icp_type` | Construction-Led / Real Estate-Led / Mixed |
| Primary buyer role | `primary_buyer_role` | Text |
| Next action | `next_action` | Text |

Active pipeline lists: **"US customer pipeline"** (slug: `sales_3`) and **"Sales"** (slug: `sales_32`).
