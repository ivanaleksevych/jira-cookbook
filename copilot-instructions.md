# Jira MCP Instructions – ATA → Stories Generator

## Goal
Read an **ATA** ticket, take the **Suggested stories** table from its
**Technical Analysis** field, and create one Jira Story per row.

> You no longer analyze an Epic. Stories are created from the technical
> analysis inside the **ATA ticket** the user provides.

---

## Inputs (from the user)
- `<ATA_KEY>` – the ATA ticket to read (e.g. `PROD-38907`). Never hardcode it.
- `<PARENT_KEY>` – the parent issue the stories must be linked under
  (e.g. `PROD-37933`). **Required user input.** This is NOT the ATA ticket.
- `team` – which team to set (see Teams table). Default `delta` if not given.

---

## Step 1: Read the ATA ticket
Call `getJiraIssue`:
- `cloudId`: `"livescoregroup.atlassian.net"`
- `issueIdOrKey`: `<ATA_KEY>`
- `fields`: `["*all"]`
- `responseContentFormat`: `"adf"`

Save:
- `key` → `<ATA_KEY>`
- `id` → `<ATA_INTERNAL_ID>`
- `fields.priority.id` → `<PRIORITY_ID>` (fallback P2 = `10007`)
- `fields.customfield_18234` → **Technical Analysis** (ADF) – the single source of truth

---

## Step 2: Parse `Technical Analysis` (customfield_18234)
1. Find the **`Suggested stories`** heading.
2. Read the table under it. Columns: **Component | Jira | Estimate**.
   Each row = one Story.
3. Everything else in `Technical Analysis` (overview, per-service sections,
   processing flow, message/JSON examples, potential problems) is the Story
   **description** source.

Rules:
- Use the **table** as the list of stories (not the free-text bullets above it).
- Do NOT skip, add, merge or reword rows.
- Story `summary` = `"<Component> <Story>"` (e.g. `"[ai-service] Add embed_iframe ..."`).

### STRICT rules for `description` (non-negotiable)
The description MUST contain **only text copied verbatim from the ATA ticket's
`Technical Analysis`**. These rules cannot be relaxed:
- **Do NOT invent, infer, assume, summarize, rephrase or expand** anything.
  Not a single sentence, field, value, example or detail may appear in the
  Story that is not literally present in the ATA.
- **Do NOT move content between sections.** Copy each story's own matching
  section as-is. Do not attach one component's table or JSON example to a
  different component's story.
- Map each story to its own section of `Technical Analysis` (match by the
  component name heading, e.g. `media-aggregator-processor`,
  `media-aggregator-api`, etc.).
- If a story has no descriptive text in the ATA beyond its table row, the
  description = only that row's text. Never pad it.
- Copy JSON / message examples exactly as written, wrapped in ` ```json ` blocks.
- **Do NOT put the Estimate into the description.** The estimate goes only into
  `timetracking.originalEstimate`. Remove any `Estimate: ...` line even if the
  ATA text contains it.
- Never invent content that is not in the ATA.

### Estimate normalization
Convert to Jira units for `timetracking.originalEstimate`:
- `<N>pd` → `<N>d` (`1pd` → `1d`, `2.5pd` → `2.5d`)
- `<N>ph` → `<N>h`
- Values already in `d` / `h` pass through unchanged. Keep the number exact.

---

## Step 3: Preview
Show the parsed list, then create automatically (no confirmation):
```
Preview:
1. [media-aggregator-processor] Support header and blockquote blocks in Article HTML Content function (1d)
2. [media-aggregator-processor] Support header and blockquote blocks in Article Web Export function (1d)
3. [media-aggregator-api] Support header and blockquote blocks in Article Ex and Combined Ex APIs (2d)
```

---

## Step 4: Create Stories
For EACH row call `createJiraIssue`:

| Parameter | Value |
|-----------|-------|
| `cloudId` | `"livescoregroup.atlassian.net"` |
| `projectKey` | `"PROD"` |
| `issueTypeName` | `"Story"` |
| `summary` | `"<Component> <Story>"` |
| `description` | From `Technical Analysis` (markdown, **no Estimate line**) |
| `contentFormat` | `"markdown"` |
| `additional_fields` | JSON object below |

```json
{
  "priority": {"id": "<PRIORITY_ID>"},
  "parent": {"key": "<PARENT_KEY>"},
  "components": [{"id": "11078"}],
  "customfield_10156": {"id": "70121:4acf64ef-fbea-4399-b874-411f0bc38345"},
  "customfield_10001": "8fbf2d82-c7a5-4079-a7e1-7f5299e53152-163",
  "customfield_10155": {"type": "doc", "version": 1, "content": [{"type": "paragraph", "content": [{"type": "text", "text": "-"}]}]},
  "assignee": {"id": "626a98c607b842006f154964"},
  "timetracking": {"originalEstimate": "<ESTIMATE>"},
  "fixVersions": [{"name": "LSM - BE 1.69.0 - Platform"}]
}
```

---

## Worked Example (verified — ATA PROD-38907 → parent PROD-37933)
Row 1 of the `Suggested stories` table
(`[media-aggregator-processor]` | `Support header and blockquote blocks in Article HTML Content function` | `1d`)
was created successfully with this exact call:

```
Tool: createJiraIssue
- cloudId: "livescoregroup.atlassian.net"
- projectKey: "PROD"
- issueTypeName: "Story"
- summary: "[media-aggregator-processor] Support header and blockquote blocks in Article HTML Content function"
- description: <Technical Analysis text for this story only, WITHOUT the Estimate line>
- contentFormat: "markdown"
- additional_fields: {
    "priority": {"id": "10007"},
    "parent": {"key": "PROD-37933"},
    "components": [{"id": "11078"}],
    "customfield_10156": {"id": "70121:4acf64ef-fbea-4399-b874-411f0bc38345"},
    "customfield_10001": "8fbf2d82-c7a5-4079-a7e1-7f5299e53152-163",
    "customfield_10155": {"type": "doc", "version": 1, "content": [{"type": "paragraph", "content": [{"type": "text", "text": "-"}]}]},
    "assignee": {"id": "626a98c607b842006f154964"},
    "timetracking": {"originalEstimate": "1d"},
    "fixVersions": [{"name": "LSM - BE 1.69.0 - Platform"}]
  }
```
Result: created `PROD-39815` (plus `PROD-39816`, `PROD-39817` for the other rows).
Use this as the template for the remaining rows, changing only `summary`,
`description` and `timetracking.originalEstimate`.

---

## Field Reference

### Mandatory (`additional_fields`)
| Field | Meaning | Value |
|-------|---------|-------|
| `priority` | Priority | `{"id": "<from_ata>"}` (fallback `10007`) |
| `parent` | Parent link | `{"key": "<PARENT_KEY>"}` (user input) |
| `components` | Component | `[{"id": "11078"}]` |
| `customfield_10156` | Product Owner | `{"id": "70121:4acf64ef-fbea-4399-b874-411f0bc38345"}` (Natalia Klipailo) |
| `customfield_10155` | Acceptance Criteria (ADF) | `{"type":"doc","version":1,"content":[{"type":"paragraph","content":[{"type":"text","text":"-"}]}]}` |

### Optional (`additional_fields`)
| Field | Meaning | Value |
|-------|---------|-------|
| `customfield_10001` | Team | `<TEAM_ID>` — pick per user request (see Teams table) |
| `assignee` | Assignee | `{"id": "626a98c607b842006f154964"}` |
| `timetracking` | Estimate | `{"originalEstimate": "<from_table>"}` |
| `fixVersions` | Fix Version | `[{"name": "LSM - BE 1.69.0 - Platform"}]` |

### Teams (`customfield_10001`)
Select the team the user asks for. Default to `delta` if none specified.

| Team | ID |
|------|-----|
| `bravo` | `8fbf2d82-c7a5-4079-a7e1-7f5299e53152-161` |
| `delta` | `8fbf2d82-c7a5-4079-a7e1-7f5299e53152-163` |

---

## Error Handling
- **"Description, Priority, Parent, Product Owner & Components are mandatory"** →
  ensure `description` is set and `additional_fields` has `priority.id`,
  `parent.key`, `components[0].id`, `customfield_10156.id`.
- **"parent: Could not find issue"** → re-read Epic; try `parent` with
  `{"id": "<EPIC_INTERNAL_ID>"}`.
- **`fixVersions` rejected** → retry with `"fixVersions": []`.

---

## Step 5: Output
Return created issue keys, plus any failed rows with the exact Jira error.

---

## Constants
| Name | Value                                        |
|------|----------------------------------------------|
| Cloud ID | `livescoregroup.atlassian.net`               |
| Project Key | `PROD`                                       |
| Issue Type | `Story`                                      |
| Technical Analysis field (ATA) | `customfield_18234`                          |
| Product Owner field | `customfield_10156`                          |
| Team field | `customfield_10001`                          |
| Component ID (LSM BE) | `11078`                                      |
| Product Owner (Natalia Klipailo) | `70121:4acf64ef-fbea-4399-b874-411f0bc38345` |
| Team `bravo` | `8fbf2d82-c7a5-4079-a7e1-7f5299e53152-161`   |
| Team `delta` (default) | `8fbf2d82-c7a5-4079-a7e1-7f5299e53152-163`   |
| Assignee ID | `626a98c607b842006f154964`                   |
| Default Priority (P2) | `10007`                                      |
| Fix Version | `LSM - BE 1.69.0 - Platform`                 |

