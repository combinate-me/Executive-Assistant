# Skill: Log Email to Insites CRM

Log a copy of an email sent to a client as an activity against their company record in the Insites CRM.

## When to Use

- "Log that email to Insites"
- "Add that email as a CRM activity"
- "Record the email I just sent to [client]"
- Any time an outbound client email is sent and needs to be tracked in the CRM

## What It Does

Creates an `email` activity on the client's company record in Insites with:
- The email subject
- The full email body (plain text version)
- The date sent
- Logged under Shane's Insites account

---

## Step 1: Find the Company UUID

If you already have the company UUID, skip to Step 2.

Look up the company in the Insites CRM. The search API does not filter reliably - fetch all companies and filter client-side:

```bash
source .env && python3 << EOF
import subprocess, json, os

api_key = "$INSITES_API_KEY"
site = "$INSITES_SITE"

all_companies = []
page = 1

while True:
    result = subprocess.run(
        ['curl', '-s',
         '-H', f'Authorization: {api_key}',
         '-H', 'Accept: application/json',
         f'{site}/crm/api/v2/companies?page={page}&size=100'],
        capture_output=True, text=True
    )
    data = json.loads(result.stdout)
    results = data.get('results', [])
    all_companies.extend(results)
    if len(results) < 100:
        break
    page += 1

search_term = "CLIENT NAME HERE"
for c in all_companies:
    name = c.get('company_name', '')
    if search_term.lower() in name.lower():
        print(f"[{c.get('uuid', c.get('id'))}] {name}")
EOF
```

Alternatively, Shane can navigate directly to the company in the Insites admin UI and copy the UUID from the URL:
`https://intranet.combinate.me/admin/insites#/crm/companies/COMPANY_UUID`

---

## Step 2: Create the Email Activity

Replace placeholders before running:
- `COMPANY_UUID` - from Step 1
- `EMAIL_SUBJECT` - subject line of the email
- `EMAIL_BODY` - plain text body of the email
- `DATE` - date sent in ISO format (e.g. `2026-03-10T00:00:00.000Z`)

Shane's Insites admin UUID (do not change): `18319c9b-62df-412b-a464-ce0cf32df7d0`

```bash
source .env && curl -s -X POST \
  -H "Authorization: $INSITES_API_KEY" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "email",
    "subject": "EMAIL_SUBJECT",
    "message": "EMAIL_BODY",
    "feature_type": "company",
    "company_uuid": "COMPANY_UUID",
    "related_uuid": "COMPANY_UUID",
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
    print(f\"Company: {(a.get('company') or {}).get('company_name', '')}\")
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
| `feature_type` | `"company"` |
| `company_uuid` | UUID of the client's company in Insites |
| `related_uuid` | Same as `company_uuid` |
| `last_updated_by_administrator_uuid` | Shane's UUID: `18319c9b-62df-412b-a464-ce0cf32df7d0` |
| `start_date_time` | Date/time sent in ISO 8601 format |

---

## Optional: Link to a Contact

If you also want to link the activity to a specific contact (e.g. Olivia Scullard), add `related_contacts` to the payload:

```json
"related_contacts": [{ "uuid": "CONTACT_UUID" }]
```

To find a contact UUID:
```bash
source .env && curl -s \
  -H "Authorization: $INSITES_API_KEY" \
  -H "Accept: application/json" \
  "$INSITES_SITE/crm/api/v2/contacts?page=1&size=50" | python3 -c "
import sys, json
data = json.load(sys.stdin)
for c in data.get('results', []):
    if 'scullard' in c.get('name', '').lower() or 'scullard' in c.get('email', '').lower():
        print(f\"[{c['uuid']}] {c.get('name','')}  {c.get('email','')}\")
"
```

---

## Error Handling

- **`last_updated_by_administrator_uuid` invalid** - Check that Shane's UUID is correct: `18319c9b-62df-412b-a464-ce0cf32df7d0`
- **`related_uuid` invalid** - Must match `company_uuid`
- **401 Unauthorized** - Check `INSITES_API_KEY` in `.env`
- **Empty company search results** - The Insites search param doesn't filter; always filter client-side

---

## Verification

After creation, you can verify the activity appears on the company record by visiting:
`https://intranet.combinate.me/admin/insites#/crm/companies/COMPANY_UUID/activities`
