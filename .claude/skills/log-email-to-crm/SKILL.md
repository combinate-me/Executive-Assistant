---
name: log-email-to-crm
description: Log a sent client email as an activity in the Insites CRM against the contact record. Use this skill whenever Shane has sent an email to a client and wants to record it in the CRM. Trigger on phrases like "log that email to Insites", "add that email as a CRM activity", "record the email I just sent to [client]", or any time an outbound client email needs to be tracked in the CRM.
---

# Skill: Log Email to Insites CRM

Log a copy of an email sent to a client as an activity against their **contact** record in the Insites CRM.

## When to Use

- "Log that email to Insites"
- "Add that email as a CRM activity"
- "Record the email I just sent to [client]"
- Any time an outbound client email is sent and needs to be tracked in the CRM

## What It Does

Creates an `email` activity on the client's **contact** record in Insites with:
- The email subject
- The full email body (plain text version)
- The date sent
- Logged under Shane's Insites account

---

## Step 1: Find the Contact UUID

Search for the contact by name or email using the correct search format:

```bash
source "/Users/shanemcgeorge/Claude/Combinate EA/.env" && curl -s \
  -H "Authorization: $INSITES_API_KEY" \
  -H "Accept: application/json" \
  "$INSITES_SITE/crm/api/v2/contacts?page=1&size=25&search_by=email&keyword=SEARCH+TERM" | python3 -c "
import sys, json
data = json.load(sys.stdin)
results = data.get('results', [])
if not results:
    print('No matches. Try search_by=first_name or search_by=last_name')
for c in results:
    company = (c.get('company') or {}).get('company_name', '')
    print(f\"[{c['uuid']}] {c.get('name','')}  {c.get('email','')}  {company}\")
"
```

Use `search_by=email` for email searches, `search_by=first_name` or `search_by=last_name` for name searches. Replace spaces with `+`.

Alternatively, Shane can navigate directly to the contact in the Insites admin UI and copy the UUID from the URL:
`https://intranet.combinate.me/admin/insites#/crm/contacts/CONTACT_UUID`

---

## Step 2: Create the Email Activity

Replace placeholders before running:
- `CONTACT_UUID` - from Step 1
- `EMAIL_SUBJECT` - subject line of the email
- `EMAIL_BODY` - plain text body of the email
- `DATE` - date sent in ISO format (e.g. `2026-03-10T00:00:00.000Z`)

Shane's Insites admin UUID (do not change): `18319c9b-62df-412b-a464-ce0cf32df7d0`

```bash
source "/Users/shanemcgeorge/Claude/Combinate EA/.env" && curl -s -X POST \
  -H "Authorization: $INSITES_API_KEY" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "email",
    "subject": "EMAIL_SUBJECT",
    "message": "EMAIL_BODY",
    "feature_type": "contact",
    "contact_uuid": "CONTACT_UUID",
    "related_uuid": "CONTACT_UUID",
    "last_updated_by_administrator_uuid": "18319c9b-62df-412b-a464-ce0cf32df7d0",
    "start_date_time": "DATE"
  }' \
  "$INSITES_SITE/crm/api/v2/activities" | python3 -c "
import sys, json
a = json.load(sys.stdin)
if 'errors' in a:
    print('ERROR:', a['errors'])
else:
    print(f\"Activity created: ID {a.get('id')}  UUID: {a.get('uuid')}\")
    print(f\"Contact: {(a.get('contact') or {}).get('name', '')}\")
    print(f\"Subject: {a.get('subject', '')}\")
    print(f\"Date:    {a.get('start_date_time', '')}\")
"
```

---

## Required Fields

| Field | Value |
|-------|-------|
| `type` | `"email"` |
| `subject` | Email subject line |
| `message` | Plain text email body |
| `feature_type` | `"contact"` |
| `contact_uuid` | UUID of the contact in Insites |
| `related_uuid` | Same as `contact_uuid` |
| `last_updated_by_administrator_uuid` | Shane's UUID: `18319c9b-62df-412b-a464-ce0cf32df7d0` |
| `start_date_time` | Date/time sent in ISO 8601 format |

---

## Error Handling

- **`last_updated_by_administrator_uuid` invalid** - Check that Shane's UUID is correct: `18319c9b-62df-412b-a464-ce0cf32df7d0`
- **`related_uuid` invalid** - Must match `contact_uuid`
- **401 Unauthorized** - Check `INSITES_API_KEY` in `.env`
- **No search results** - Try a different `search_by` field or search term

---

## Verification

After creation, verify the activity appears on the contact record:
`https://intranet.combinate.me/admin/insites#/crm/contacts/CONTACT_UUID/activities`
