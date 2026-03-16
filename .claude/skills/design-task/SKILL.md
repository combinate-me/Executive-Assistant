---
name: design-task
description: Full design task kickoff and wrap-up workflow for Combinate. Use this skill whenever Shane has been assigned a design task and needs to get started on it. Covers client research, Figma direction suggestions, Teamwork status updates, Slack team comms, and time logging. Trigger on any mention of starting a design task, being assigned a design job, "what should I design for X", "kick off the design for task Y", "I've been assigned a design task", "help me start this design", or any combination of a Teamwork task ID with design or Figma. This is the primary workflow for any design work at Combinate.
---

# Skill: Design Task

Full workflow for when Shane is assigned a design task. Gathers client context, generates Figma design direction, updates Teamwork, notifies Lee, and logs time when done.

**Auth:** Always source env with the full path:
```bash
source "/Users/shanemcgeorge/Claude/Combinate EA/.env"
```

---

## Inputs

- **Teamwork task ID** (required) - the task number, e.g. `25429514`, or a full URL like `https://pm.cbo.me/tasks/25429514`
- **Client name or TLA** (optional) - if not provided, infer it from the task's project name

---

## Step 1: Read the Teamwork Task

Fetch the task to understand what needs to be designed. Extract the task title, description, project name, and assigned team member.

```bash
source "/Users/shanemcgeorge/Claude/Combinate EA/.env" && curl -s \
  -u "$TEAMWORK_API_KEY:x" \
  "$TEAMWORK_SITE/tasks/TASK_ID.json" | python3 -c "
import sys, json
data = json.load(sys.stdin)
t = data.get('todo-item', {})
print(f'Title:       {t.get(\"content\")}')
print(f'ID:          {t.get(\"id\")}')
print(f'Project:     {t.get(\"project-name\", \"Unknown\")}')
print(f'Assignee:    {t.get(\"responsible-party-names\", \"Unassigned\")}')
print(f'Due:         {t.get(\"due-date\", \"None\")}')
print(f'Description: {t.get(\"description\", \"(none)\")}')
"
```

The project name usually contains the client name. Extract it to use in the CRM lookup.

---

## Step 2: Look Up the Client in the CRM

Search Insites CRM for the company record using the client name inferred from the task. This gives context about what the client does, their industry, contacts, and any notes.

```bash
source "/Users/shanemcgeorge/Claude/Combinate EA/.env" && curl -s \
  -H "Authorization: $INSITES_API_KEY" \
  -H "Accept: application/json" \
  "$INSITES_INSTANCE_URL/crm/api/v2/companies?page=1&size=10&search_by=company_name&keyword=CLIENT+NAME" | python3 -c "
import sys, json
data = json.load(sys.stdin)
for c in data.get('results', []):
    cf = c.get('custom_field') or {}
    print(f\"[{c.get('uuid','')}] {c.get('company_name','')}\")
    print(f\"  Email:   {c.get('email_1','')}\")
    print(f\"  Website: {c.get('website','')}\")
    print(f\"  TLA:     {cf.get('client_tla','(missing)')}\")
    print(f\"  Drive:   {cf.get('google_drive_url','(missing)')}\")
"
```

Then fetch the full company record for richer context (notes, contacts):

```bash
source "/Users/shanemcgeorge/Claude/Combinate EA/.env" && curl -s \
  -H "Authorization: $INSITES_API_KEY" \
  -H "Accept: application/json" \
  "$INSITES_INSTANCE_URL/crm/api/v2/companies/COMPANY_UUID" | python3 -c "
import sys, json
c = json.load(sys.stdin)
print(f\"Name:    {c.get('company_name', '')}\")
print(f\"Email:   {c.get('email_1', '')}\")
print(f\"Website: {c.get('website', '')}\")
print(f\"Notes:   {c.get('notes', '')}\")
print(f\"Custom:  {c.get('custom_field', {})}\")
"
```

---

## Step 3: Generate Figma Design Suggestions

Using everything gathered from the task description and CRM record, generate **3-5 tailored Figma design suggestions** for Shane.

Good suggestions are specific to this client and task. They should cover:

- **Visual direction** - colour palette, typography style, tone (e.g. corporate and minimal, warm and approachable, bold and energetic)
- **Layout approach** - section structure, grid, key components to include
- **Content priorities** - what to lead with based on the client's industry and goals
- **UX considerations** - interaction patterns, user flow, hierarchy
- **Reference styles** - if the client has an existing brand or website, reference that

**Format the suggestions clearly** so Shane can skim them and decide which direction to take into Figma. Number them. Keep each one to 3-5 bullet points.

Example structure:
```
Option 1: [Direction Name]
- Visual: ...
- Layout: ...
- Tone: ...
- Key component: ...

Option 2: [Direction Name]
...
```

---

## Step 4: Post Design Suggestions as a Teamwork Comment

Post the generated suggestions as a comment on the task so the team can see the design direction being considered.

```bash
source "/Users/shanemcgeorge/Claude/Combinate EA/.env" && curl -s -X POST \
  -u "$TEAMWORK_API_KEY:x" \
  -H "Content-Type: application/json" \
  -d '{
    "comment": {
      "body": "DESIGN SUGGESTIONS HERE (plain text or markdown)",
      "notify": ""
    }
  }' \
  "$TEAMWORK_SITE/tasks/TASK_ID/comments.json"
```

The comment body should start with a clear header, e.g.:
> **Design direction options for review:**
> [the suggestions]

---

## Step 5: Send Slack Update to Lee

Notify Lee Agosila (Head of Design) via Slack with a brief update. Keep it short - what the task is, who it's for, and that design is now starting.

Use the Slack MCP tool `slack_send_message`. Search for Lee's Slack user first if needed using `slack_search_users` with "Lee Agosila".

Message format:
> Hey Lee, just picked up a design task: [Task Title] for [Client Name]. I've noted some Figma direction options in the Teamwork task comments. Will keep you posted as it progresses.

Send as a direct message to Lee.

---

## Step 6: Update Task Status to In Progress

Mark the Teamwork task as in progress so the team knows it's being actively worked on.

```bash
source "/Users/shanemcgeorge/Claude/Combinate EA/.env" && curl -s -X PUT \
  -u "$TEAMWORK_API_KEY:x" \
  -H "Content-Type: application/json" \
  -d '{
    "todo-item": {
      "status": "inprogress"
    }
  }' \
  "$TEAMWORK_SITE/tasks/TASK_ID.json"
```

Confirm the update succeeded before moving on.

---

## Step 7: Log Time (on completion)

Once the design work is done, log 1 hour of time against the task. Run this step only when Shane confirms the work is finished.

```bash
source "/Users/shanemcgeorge/Claude/Combinate EA/.env" && curl -s -X POST \
  -u "$TEAMWORK_API_KEY:x" \
  -H "Content-Type: application/json" \
  -d '{
    "time-entry": {
      "description": "Design work",
      "person-id": "SHANE_PERSON_ID",
      "date": "YYYYMMDD",
      "time": "09:00",
      "hours": "1",
      "minutes": "0",
      "isbillable": true
    }
  }' \
  "$TEAMWORK_SITE/tasks/TASK_ID/time_entries.json"
```

To get Shane's person ID, use the people endpoint:
```bash
source "/Users/shanemcgeorge/Claude/Combinate EA/.env" && curl -s \
  -u "$TEAMWORK_API_KEY:x" \
  "$TEAMWORK_SITE/people.json" | python3 -c "
import sys, json
data = json.load(sys.stdin)
for p in data.get('people', []):
    if 'shane' in p.get('first-name','').lower() or 'mcgeorge' in p.get('last-name','').lower():
        print(f\"{p['id']}  {p['first-name']} {p['last-name']}  ({p.get('email-address', '')})\")
"
```

Use today's date in `YYYYMMDD` format. Set `time` to a reasonable time of day.

---

## Summary of the Full Workflow

| Step | Action | Tool |
|------|--------|------|
| 1 | Read task details | Teamwork API |
| 2 | Look up client in CRM | Insites CRM API |
| 3 | Generate Figma design suggestions | (Claude generates based on context) |
| 4 | Post suggestions as Teamwork comment | Teamwork API |
| 5 | Notify Lee via Slack DM | Slack MCP |
| 6 | Update task status to in progress | Teamwork API |
| 7 | Log 1 hour when done | Teamwork API |

Steps 1-6 run on kickoff. Step 7 runs when Shane says the work is complete.

---

## Error Handling

- **Client not found in CRM** - Try searching by a shortened version of the project name. If still not found, flag to Shane and proceed with the task description alone for generating suggestions.
- **Task status update fails** - Confirm the task ID is correct. Some tasks may be in a completed state already.
- **Slack user not found** - Search for "Lee" in `slack_search_users`. If still not found, ask Shane for Lee's Slack handle.
- **Time entry fails** - Double-check Shane's person ID and that the date format is `YYYYMMDD` with no separators.
