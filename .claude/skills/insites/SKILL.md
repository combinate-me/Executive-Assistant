# Skill: Insites Intranet

Use this skill for any interaction with the Combinate Intranet built on the Insites platform (intranet.combinate.me). This covers CRM contacts, companies, tasks, activities, databases, and events.

## When to Use
- "Look up [contact/company] in the intranet"
- "What tasks are open in Insites?"
- "Add a contact to the CRM"
- "Check the database records for [item]"
- "What activity has there been on [contact]?"
- Any request involving the Combinate intranet, client CRM data, or Insites records
- Finding a client's Google Drive folder or TLA (these are stored in CRM custom fields)

## Client Company Custom Fields

Every company record has a `custom_field` object with two key fields used across all client workflows:

| Field | Description | Example |
|-------|-------------|---------|
| `client_tla` | Three-letter abbreviation for the client | `"MIG"`, `"IEC"` |
| `google_drive_url` | Direct link to the client's Google Drive folder | `"https://drive.google.com/drive/folders/..."` |

**To find a client's Drive folder:**
1. Look up the company by name (see Operation 3 - search, then filter client-side if needed)
2. Extract `custom_field.google_drive_url`
3. Parse the folder ID from the URL (the string after `/folders/`)
4. Use that ID with Google Drive MCP tools to navigate the folder

**Important:** Use the correct search parameter format: `?search_by=[field]&keyword=[value]`. For example, `?search_by=first_name&keyword=John` or `?search_by=email&keyword=domain.com`. You can also add `sort_by=[field]&sort_order=ASC`. Do NOT use `?search=` or `?name&keyword=` or `?company_name&keyword=` - these do not work correctly.

If `client_tla` or `google_drive_url` is missing from a company record, flag it to Shane and ask him to update the CRM.

## Authentication Setup

API key and site URL are stored in `.env`:
```
INSITES_API_KEY=your_key_here
INSITES_SITE=https://intranet.combinate.me
```

To get your API key:
1. Go to intranet.combinate.me > Admin > Integrations > Instance API Key
2. Copy the key and add it to `.env`
3. Regenerating the key will invalidate the previous one

The `.env` file is gitignored - the API key never gets committed.

## Base Request Pattern

Insites uses the API key directly in the `Authorization` header (not Basic Auth). Module paths vary by area - CRM endpoints use `/crm/api/v2/`, databases use `/databases/api/v2/`, events use `/events/api/v2/`.

```bash
source .env && curl -s \
  -H "Authorization: $INSITES_API_KEY" \
  -H "Accept: application/json" \
  "$INSITES_SITE/crm/api/v2/[endpoint]"
```

Rate limit: 300 requests per 60 seconds.

---

## Operations

### 1. List Contacts

```bash
source .env && curl -s \
  -H "Authorization: $INSITES_API_KEY" \
  -H "Accept: application/json" \
  "$INSITES_SITE/crm/api/v2/contacts?page=1&size=25" | python3 -c "
import sys, json
data = json.load(sys.stdin)
print(f\"Total: {data.get('total_entries', '?')} contacts\")
for c in data.get('results', []):
    name = c.get('name', f\"{c.get('first_name','')} {c.get('last_name','')}\".strip())
    email = c.get('email', '')
    company = (c.get('company') or {}).get('company_name', '')
    print(f\"[{c['id']}] {name}  {email}  {company}\")
"
```

---

### 2. Get a Single Contact

Replace `CONTACT_UUID` with the contact's UUID.

```bash
source .env && curl -s \
  -H "Authorization: $INSITES_API_KEY" \
  -H "Accept: application/json" \
  "$INSITES_SITE/crm/api/v2/contacts/CONTACT_UUID" | python3 -c "
import sys, json
c = json.load(sys.stdin)
print(f\"Name:      {c.get('name', '')}\")
print(f\"Email:     {c.get('email', '')}\")
print(f\"Phone:     {c.get('work_phone_number', '')}\")
print(f\"Job Title: {c.get('job_title', '')}\")
print(f\"Company:   {(c.get('company') or {}).get('company_name', '')}\")
print(f\"UUID:      {c.get('uuid', '')}\")
print(f\"Archived:  {c.get('is_archived', False)}\")
print(f\"Created:   {c.get('created_at', '')}\")
print(f\"Notes:     {c.get('notes', '')}\")
"
```

---

### 3. Search Contacts by Name or Email

Use `?search_by=[field]&keyword=[value]`. Supported fields: `first_name`, `last_name`, `email`. Replace spaces with `+`.

```bash
source .env && curl -s \
  -H "Authorization: $INSITES_API_KEY" \
  -H "Accept: application/json" \
  "$INSITES_SITE/crm/api/v2/contacts?page=1&size=25&search_by=email&keyword=SEARCH+TERM" | python3 -c "
import sys, json
data = json.load(sys.stdin)
results = data.get('results', [])
print(f\"Total: {data.get('total_entries', '?')}\")
if not results:
    print('No matches found.')
for c in results:
    name = c.get('name', '')
    email = c.get('email', '')
    company = (c.get('company') or {}).get('company_name', '')
    print(f\"[{c['id']}] {name}  {email}  {company}\")
"
```

**Do NOT use `?search=`, `?name&keyword=`, or `?company_name&keyword=` - these do not filter correctly.**

---

### 4. Add a Contact

```bash
source .env && curl -s -X POST \
  -H "Authorization: $INSITES_API_KEY" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "first_name": "FIRST NAME",
    "last_name": "LAST NAME",
    "email": "EMAIL",
    "work_phone_number": "PHONE",
    "job_title": "JOB TITLE"
  }' \
  "$INSITES_SITE/crm/api/v2/contacts" | python3 -c "
import sys, json
c = json.load(sys.stdin)
print(f\"Created: {c.get('name', '')}  UUID: {c.get('uuid', '')}\")
"
```

---

### 5. List or Search Companies

To list all companies (paginated):

```bash
source .env && curl -s \
  -H "Authorization: $INSITES_API_KEY" \
  -H "Accept: application/json" \
  "$INSITES_SITE/crm/api/v2/companies?page=1&size=25" | python3 -c "
import sys, json
data = json.load(sys.stdin)
print(f\"Total: {data.get('total_entries', '?')} companies\")
for c in data.get('results', []):
    print(f\"[{c['id']}] {c.get('company_name', '')}  {c.get('email_1', '')}  {c.get('website', '')}\")
"
```

To search by company name, use `?search_by=company_name&keyword=SEARCH+TERM`:

```bash
source .env && curl -s \
  -H "Authorization: $INSITES_API_KEY" \
  -H "Accept: application/json" \
  "$INSITES_SITE/crm/api/v2/companies?page=1&size=25&search_by=company_name&keyword=SEARCH+TERM" | python3 -c "
import sys, json
data = json.load(sys.stdin)
results = data.get('results', [])
print(f\"Total: {data.get('total_entries', '?')}\")
if not results:
    print('No matches found.')
for c in results:
    cf = c.get('custom_field') or {}
    print(f\"[{c['id']}] {c.get('company_name','')}  {c.get('email_1','')}  {c.get('website','')}\")
    print(f\"  TLA: {cf.get('client_tla','')}  Drive: {cf.get('google_drive_url','')}\")
"
```

**Do NOT use `?search=` or `?company_name&keyword=` - these do not filter correctly.**

---

### 6. Get a Single Company

Replace `COMPANY_UUID` with the company's UUID.

```bash
source .env && curl -s \
  -H "Authorization: $INSITES_API_KEY" \
  -H "Accept: application/json" \
  "$INSITES_SITE/crm/api/v2/companies/COMPANY_UUID" | python3 -c "
import sys, json
c = json.load(sys.stdin)
print(f\"Name:    {c.get('company_name', '')}\")
print(f\"Email:   {c.get('email_1', '')}\")
print(f\"Phone:   {c.get('phone_1_number', '')}\")
print(f\"Website: {c.get('website', '')}\")
print(f\"UUID:    {c.get('uuid', '')}\")
print(f\"Created: {c.get('created_at', '')}\")
"
```

---

### 7. List Tasks

```bash
source .env && curl -s \
  -H "Authorization: $INSITES_API_KEY" \
  -H "Accept: application/json" \
  "$INSITES_SITE/crm/api/v2/tasks?page=1&size=25" | python3 -c "
import sys, json
data = json.load(sys.stdin)
print(f\"Total: {data.get('total_entries', '?')} tasks\")
for t in data.get('results', []):
    status = 'Done' if t.get('completed_at') else 'Open'
    due = t.get('due_date', 'No due date')
    assignee = (t.get('assigned_to') or {}).get('name', 'Unassigned')
    print(f\"[{t.get('id','')}] [{status}] {t.get('name', '')}  Assignee: {assignee}  Due: {due}\")
"
```

---

### 8. Create a Task

```bash
source .env && curl -s -X POST \
  -H "Authorization: $INSITES_API_KEY" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "TASK TITLE",
    "description": "TASK DESCRIPTION",
    "due_date": "YYYY-MM-DD"
  }' \
  "$INSITES_SITE/crm/api/v2/tasks" | python3 -c "
import sys, json
t = json.load(sys.stdin)
print(f\"Created task: {t.get('name', '')}  ID: {t.get('id', '')}\")
"
```

---

### 9. Get Task Comments

Replace `TASK_ID` with the task's ID.

```bash
source .env && curl -s \
  -H "Authorization: $INSITES_API_KEY" \
  -H "Accept: application/json" \
  "$INSITES_SITE/crm/api/v2/tasks/comments?task_id=TASK_ID" | python3 -c "
import sys, json
data = json.load(sys.stdin)
results = data.get('results', [])
if not results:
    print('No comments.')
for c in results:
    author = (c.get('author') or {}).get('name', 'Unknown')
    created = c.get('created_at', '')
    body = c.get('body', '')
    print(f\"--- {author} ({created})\")
    print(body)
    print()
"
```

---

### 10. Add a Task Comment

```bash
source .env && curl -s -X POST \
  -H "Authorization: $INSITES_API_KEY" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "task_id": "TASK_ID",
    "body": "COMMENT TEXT"
  }' \
  "$INSITES_SITE/crm/api/v2/tasks/comments"
```

---

### 11. List Activities

```bash
source .env && curl -s \
  -H "Authorization: $INSITES_API_KEY" \
  -H "Accept: application/json" \
  "$INSITES_SITE/crm/api/v2/activities?page=1&size=25" | python3 -c "
import sys, json
data = json.load(sys.stdin)
for a in data.get('results', []):
    created = a.get('created_at', '')
    desc = a.get('description', a.get('name', ''))
    author = (a.get('author') or a.get('created_by') or {}).get('name', '')
    print(f\"[{created}] {desc}  {author}\")
"
```

---

### 12. Create an Email Activity on a Company

Logs an outbound email against a company record. All fields are required.

Shane's Insites admin UUID (do not change): `18319c9b-62df-412b-a464-ce0cf32df7d0`

```bash
source .env && curl -s -X POST \
  -H "Authorization: $INSITES_API_KEY" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "email",
    "subject": "EMAIL SUBJECT",
    "message": "EMAIL BODY (plain text)",
    "feature_type": "company",
    "company_uuid": "COMPANY_UUID",
    "related_uuid": "COMPANY_UUID",
    "last_updated_by_administrator_uuid": "18319c9b-62df-412b-a464-ce0cf32df7d0",
    "start_date_time": "YYYY-MM-DDTHH:MM:SS.000Z"
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
"
```

Note: Both `company_uuid` and `related_uuid` must be set to the same company UUID or the API will reject the request.

For the full log-email-to-crm workflow, see `.claude/skills/log-email-to-crm/SKILL.md`.

---

### 12. List Databases

Note: Databases use the `/databases/api/v2/` prefix, not `/crm/api/v2/`. Response is wrapped under `items`.

```bash
source .env && curl -s \
  -H "Authorization: $INSITES_API_KEY" \
  -H "Accept: application/json" \
  "$INSITES_SITE/databases/api/v2/databases?page=1&size=25" | python3 -c "
import sys, json
data = json.load(sys.stdin)
items = data.get('items', data)
print(f\"Total: {items.get('total_entries', '?')} databases\")
for d in items.get('results', []):
    label = (d.get('metadata') or {}).get('label', d.get('path', ''))
    print(f\"[{d['id']}] {label}  path: {d.get('path', '')}\")
"
```

---

### 13. List Events

Note: Events use the `/events/api/v2/` prefix.

```bash
source .env && curl -s \
  -H "Authorization: $INSITES_API_KEY" \
  -H "Accept: application/json" \
  "$INSITES_SITE/events/api/v2/events?page=1&size=25" | python3 -c "
import sys, json
data = json.load(sys.stdin)
print(f\"Total: {data.get('total_entries', '?')} events\")
for e in data.get('results', []):
    print(f\"[{e.get('id','')}] {e.get('name', '')}  {e.get('start_date', '')} - {e.get('end_date', '')}\")
"
```

---

## URL Reference

| Module | Base Path |
|--------|-----------|
| Contacts | `$INSITES_SITE/crm/api/v2/contacts` |
| Companies | `$INSITES_SITE/crm/api/v2/companies` |
| Tasks | `$INSITES_SITE/crm/api/v2/tasks` |
| Task Comments | `$INSITES_SITE/crm/api/v2/tasks/comments` |
| Activities | `$INSITES_SITE/crm/api/v2/activities` |
| Databases | `$INSITES_SITE/databases/api/v2/databases` |
| Events | `$INSITES_SITE/events/api/v2/events` |

## Presenting Results

- Use tables when listing multiple contacts, companies, or tasks
- For contacts: show name, email, company
- For companies: show name, email, website
- For tasks: show status (Open/Done), title, assignee, due date - flag overdue items
- For databases: show label and path
- When creating records, confirm success and return the new ID/UUID

## Error Handling

- **401 Unauthorized** - API key is wrong or missing. Check `.env` for `INSITES_API_KEY`
- **404 Not Found** - Wrong path or ID. Refer to the URL Reference table above
- **429 Too Many Requests** - Rate limit hit (300 req/60s). Wait and retry
- **HTML response instead of JSON** - The `Authorization` header is not being sent correctly. Ensure `.env` is sourced before the curl command
