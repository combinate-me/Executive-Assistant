---
name: pre-meeting-presentation
model: claude-sonnet-4-6
description: Builds a branded HTML presentation for any client meeting before it happens - prospect meetings, existing client meetings, project updates, or any other. Triggers on phrases like "help me prepare for my [client] meeting", "build a presentation for [meeting name]", "create slides for the [client] call", "pre-meeting prep for [client]", "I have a meeting with [client]". Gathers context from Teamwork task, Google Calendar, Gmail, Insites CRM, Drive, and Slack. Outputs a full-screen HTML slideshow (dark brand theme, large fonts, Phosphor icons, Mermaid diagrams) and exports a PDF. Requires Teamwork task ID, meeting name, and client/contact name.
---

# Skill: Pre-Meeting Presentation

Use this skill to build a polished, on-brand HTML presentation before any client meeting. The presentation runs in the browser for screen sharing and exports to a PDF for post-meeting delivery.

## When to Use

- "Help me prepare for my [client] meeting"
- "Build a presentation for the [meeting name]"
- "Create slides for the [client] call"
- "I have a meeting with [client] — help me prep"
- Any request to prepare, build, or create a presentation before a meeting

---

## Inputs Required

Before starting, collect:

- **Teamwork task ID or link** — task with details and any supporting articles
- **Meeting name** — exactly as it appears in Google Calendar
- **Client / contact name** — for Gmail search and CRM lookup

Ask for any that are not provided. Do not proceed on assumptions.

---

## Step 1 — Gather Context

Pull all of the following in parallel:

1. **Teamwork** — Read the task description, all comments, and any linked articles or documents
2. **Google Calendar** — Find the meeting by name. Read the event for attendees, agenda, attached notes, and video call link
3. **Gmail** — Search emails to/from the client contact and company domain. Capture recent threads, outstanding questions, and key context
4. **Insites CRM** — Look up the company record for contacts, notes, and activity history (use `.claude/skills/insites/crm/SKILL.md`)
5. **Google Drive** — For existing clients: find the client folder via the Teamwork custom item `Google Drive` record. Look for existing docs, briefs, or previous presentations
6. **Slack** — Search for recent internal conversations about the client or project

Synthesise all sources before building slides. Ask Shane if anything critical is missing or ambiguous.

---

## Step 2 — Confirm Meeting Context

Ask Shane: **"Is this a prospect (new lead) or an existing client?"**

### If prospect:
- Files go in the Prospects Google Drive folder: `https://drive.google.com/drive/folders/0ABg202coLgYuUk9PVA`
- Folder ID: `0ABg202coLgYuUk9PVA`
- Create subfolder: `[TEAMWORK_TASK_ID] - [Prospect Name] - Pre-Meeting Presentation`
- No TLA required — CRM lookup may return nothing if they are not yet a client

### If existing client:
- Ask: "What is the client TLA and project TLA?" (e.g. `IEC` / `WEB`)
- Use the Teamwork custom item resolution workflow (see `.claude/skills/combinate/SKILL.md`) to retrieve the `Google Drive` folder URL
- Navigate to that client folder → `Tasks` subfolder
- Create subfolder: `[TEAMWORK_TASK_ID] - Pre-Meeting Presentation - [Short Description]`

---

## Step 3 — Build HTML Presentation

Generate a single self-contained `.html` file. All CSS and JS must be inline or loaded from CDN. No local file dependencies.

### CDN dependencies

Always include Phosphor Icons. Include Mermaid only when a diagram slide is needed.

```html
<link rel="stylesheet" href="https://unpkg.com/@phosphor-icons/web@2.1.1/src/regular/style.css" />
<script src="https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.min.js"></script>
```

### Design specification

| Property | Value |
|----------|-------|
| Viewport | 100vw × 100vh, overflow hidden, no scroll |
| Background | `#181818` |
| Text | `#ffffff` |
| Accent | `#1E43FF` |
| Font | `'Helvetica Neue', Helvetica, Arial, sans-serif` |
| Cover H1 | 80px, bold |
| Slide title | 52px, bold |
| Body text | 28px |
| Bullets | 26px |
| Caption / small | 20px |
| Padding | ~40px on content areas — maximise usable screen space |
| Logo placement | Top-left on all inner slides; centred on cover |

The Combinate logo mark can be rendered as SVG inline or as a text wordmark `combinate™` in the brand blue `#1E43FF` when SVG is not available.

### Slide types

Build all of the following as reusable templates within the HTML:

| Type | Description |
|------|-------------|
| `cover` | Full dark slide: meeting title (large), client name, date, Shane's name, Combinate logo centred |
| `agenda` | Numbered list of agenda items. Blue numbers, white text |
| `section-divider` | Full blue (`#1E43FF`) background, large white section title only. Used to separate major topics |
| `content-text` | Title + up to 6 bullet points. Each bullet can optionally have a Phosphor icon (outline style) as a prefix |
| `content-two-col` | Title + left column + right column. Each column can hold text, bullets, or an icon list |
| `content-diagram` | Title + Mermaid.js diagram (preferred) or custom CSS flow diagram. Use dark Mermaid theme with brand colours |
| `content-image` | Title + full-width image or screenshot. Image fills remaining space. Alt text always set |
| `next-steps` | Three-column table: Action / Owner / Deadline. Blue header row, alternating row backgrounds |

### Navigation and controls

- **Keyboard**: Left/Right arrow keys, Space bar (advance)
- **Click**: Click anywhere on the slide body to advance
- **Buttons**: Always-visible Prev (`CaretLeft`) and Next (`CaretRight`) icon buttons, bottom-right

### Controls bar (always visible at bottom)

```
[====================================-------]  4 / 8   ← →
```

- **Progress bar**: Full width, 6px tall, blue fill (`#1E43FF`), animates on slide change
- **Slide counter**: `4 / 8` format, white, 20px, bottom-right
- **Prev/Next buttons**: Phosphor `CaretLeft` / `CaretRight` icons, 28px, white, clickable, beside counter

The controls bar sits in a fixed 48px strip at the bottom. Content slides must not overlap it.

### Transitions

Use CSS `transform: translateX()` + `opacity` transitions — 350ms ease.

- Outgoing slide: slides left and fades out
- Incoming slide: slides in from the right and fades in
- Going backwards: reverse direction (slide right in, left out)

### Print / PDF CSS

```css
@media print {
  .controls-bar, .progress-bar { display: none !important; }
  .slide {
    position: relative !important;
    display: block !important;
    page-break-after: always;
    width: 297mm;
    height: 210mm;
    overflow: hidden;
  }
  body { margin: 0; }
}
```

### Mermaid diagram configuration

```js
mermaid.initialize({
  theme: 'base',
  themeVariables: {
    background: '#181818',
    primaryColor: '#1E43FF',
    primaryTextColor: '#ffffff',
    primaryBorderColor: '#1E43FF',
    lineColor: '#ffffff',
    secondaryColor: '#2a2a2a',
    tertiaryColor: '#333333',
    fontFamily: 'Helvetica Neue, Helvetica, Arial, sans-serif',
    fontSize: '22px'
  }
});
```

For custom CSS diagrams (when Mermaid is not suitable): build boxes as `div` elements with `border: 2px solid #1E43FF`, `background: #2a2a2a`, `color: #ffffff`. Use CSS `flexbox` for layout and absolutely-positioned arrow lines with `border-right: 2px solid #1E43FF`.

### Slide content

Populate slides from the context gathered in Step 1. Standard structure:

1. **Cover** — Meeting title, client name, date, "Presented by Shane McGeorge · Combinate"
2. **Agenda** — Items drawn from the calendar event agenda or Teamwork task description
3. **[Content slides]** — Generated from context: background, key discussion points, proposal details, analysis, diagrams as appropriate
4. **Next Steps / Action Items** — Table populated from outstanding items identified in context

Use section dividers to separate major topics. Use diagrams to explain processes, flows, or system architecture. Use two-column slides for comparisons.

---

## Step 4 — Save HTML File

Save to:

```
~/Downloads/[TASK_ID]-[client-slug]-[YYYY-MM-DD].html
```

Example: `~/Downloads/25429514-IEC-2026-03-19.html`

---

## Step 5 — Approval Before PDF Export

After saving the HTML file, **do not immediately generate the PDF**. Ask Shane:

> "The presentation is ready — [N] slides saved to `~/Downloads/[filename].html`. Please open it in your browser and let me know:
>
> 1. Yes, it's okay
> 2. Yes, it's okay — download PDF
> 3. No, I need edits

Wait for Shane's response before proceeding.

- **Option 1** — Stop here. No PDF needed.
- **Option 2** — Proceed to export PDF (Step 6).
- **Option 3** — Ask what changes are needed, make the edits, re-save the HTML, and ask again.

---

## Step 6 — Export PDF

Only proceed here after explicit approval (option 2 from Step 5).

The print CSS must include `@page { size: A4 landscape; }` — verify this is present in the HTML before exporting. If it's missing, add it via a Python patch before running Chrome.

Try each method in order until one succeeds:

```bash
# Option 1: Google Chrome (macOS default)
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --headless --disable-gpu \
  --print-to-pdf="$HOME/Downloads/[filename].pdf" \
  --no-pdf-header-footer \
  --print-to-pdf-no-header \
  "file://$HOME/Downloads/[filename].html"

# Option 2: Chromium
chromium --headless \
  --print-to-pdf="$HOME/Downloads/[filename].pdf" \
  --no-pdf-header-footer \
  "file://$HOME/Downloads/[filename].html"

# Option 3: Playwright
python3 -m playwright pdf "file://$HOME/Downloads/[filename].html" "$HOME/Downloads/[filename].pdf"
```

If all methods fail, tell Shane the PDF could not be auto-generated and show the exact command to run manually in Terminal.

---

## Step 7 — Upload Files to Google Drive

### Primary method — Drive desktop sync

```bash
# Find the mounted Google Drive sync folder
DRIVE_BASE=$(ls ~/Library/CloudStorage/ | grep -i GoogleDrive | head -1)
DRIVE_PATH="$HOME/Library/CloudStorage/$DRIVE_BASE/My Drive"
```

If found: navigate the Drive path to the task subfolder created in Step 2 and copy both the HTML and PDF there.

If Drive desktop sync is not found: note the local file paths in the Teamwork comment and tell Shane to upload manually.

---

## Step 8 — Leave Teamwork Comment

Post a comment on the Teamwork task. Include a status table covering every step of the skill execution. Use ✓ for completed, ✗ for not found or failed, and – for not applicable.

Format:

```
#### Pre-Meeting Presentation — [Meeting Name] · [Date]

##### Context Gathered

| # | Source | Status | Detail |
|---|--------|--------|--------|
| 1 | Teamwork task | ✓ / ✗ | Description and [N] comments reviewed |
| 2 | Google Calendar | ✓ / ✗ | "[Meeting Name]" found on [Date] — [N] attendees |
| 3 | Gmail | ✓ / ✗ | [N] email threads from [contact / domain] |
| 4 | Insites CRM | ✓ / ✗ | [Company name] — [N] contacts, [N] notes |
| 5 | Google Drive | ✓ / ✗ | [N] existing documents reviewed |
| 6 | Slack | ✓ / ✗ | [N] relevant messages found |

##### Output

| # | Item | Status | Link |
|---|------|--------|------|
| 7 | HTML presentation ([N] slides) | ✓ / ✗ | [Link or local path] |
| 8 | PDF exported | ✓ / ✗ | [Link or local path] |
| 9 | Files saved to Drive | ✓ / ✗ | [Drive folder link] |

##### Slide Structure

Cover → Agenda → [list each content slide title] → Next Steps
```

Every row must reflect what actually happened — not assumed outcomes.

---

## Combinate Logo (inline SVG fallback)

When the SVG file is not accessible, use this wordmark treatment in the HTML:

```html
<span class="logo">combinate<sup>™</sup></span>
```

```css
.logo {
  font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif;
  font-weight: bold;
  font-size: 24px;
  color: #1E43FF;
  letter-spacing: -0.5px;
}
.logo sup { font-size: 12px; }
```
