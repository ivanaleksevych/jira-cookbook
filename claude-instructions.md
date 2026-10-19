# Jira MCP Instructions – ATA → Stories Generator (Claude)

## Goal
Read an **ATA** ticket, take the **Suggested stories** table from its
**Technical Analysis** field, and create one Jira Story per table row under
the parent issue given by the user.

> Stories are created from the technical analysis inside the **ATA ticket**.
> No Epic is analyzed.

---

## Inputs (from the user)
| Input | Required | Description |
|-------|----------|-------------|
| `<ATA_KEY>` | yes | ATA ticket to read (e.g. `PROD-38907`). Never hardcode it. |
| `<PARENT_KEY>` | yes | Parent issue the stories go under (e.g. `PROD-37933`). This is NOT the ATA ticket. |
| `team` | no | `bravo` or `delta` (see Teams table). Default `delta`. |

Resolve `team` to `<TEAM_ID>` via the Teams table before creating anything.

---

## Step 1: Read the ATA ticket
Call `getJiraIssue`:
- `cloudId`: `"livescoregroup.atlassian.net"`
- `issueIdOrKey`: `<ATA_KEY>`
- `fields`: `["*all"]`
- `responseContentFormat`: `"adf"`

Save:
- `fields.priority.id` → `<PRIORITY_ID>` (fallback P2 = `10007`)
- `fields.customfield_18234` → **Technical Analysis** (ADF) – the single source of truth

---

## Step 2: Parse `Technical Analysis` (customfield_18234)

### 2.1 Stories list
1. Find the **`Suggested stories`** heading.
2. Read the table under it. Columns: **Component | Jira | Estimate**.
   - `Component` – component tag, used exactly as written (including the
     square brackets, e.g. `[media-aggregator-processor]`).
   - `Jira` – the story title.
   - `Estimate` – the story estimate.
3. Each table row = exactly one Story. Use the **table** as the list of
   stories, not the free-text bullets above it.
4. Do NOT skip, add, merge, reorder or reword rows.
5. Story `summary` = `"<Component> <Jira>"` – the two cell values joined by a
   single space, e.g.
   `"[media-aggregator-processor] Support header and blockquote blocks in Article HTML Content function"`.

### 2.2 Description source
Everything in `Technical Analysis` other than the `Suggested stories` table
(overview, per-service sections, processing flow, message/JSON examples,
potential problems) is the description source.

Map each story to its section:
- Match by the component-name heading (e.g. `media-aggregator-processor`,
  `media-aggregator-api`).
- If the component section has a sub-section dedicated to this specific
  story (matching the `Jira` text), use only that sub-section.
- If several stories share one component section with no per-story
  sub-sections, copy the whole component section into each of those stories.
  Do not try to split it.

### 2.3 STRICT rules for `description` (non-negotiable)
The description MUST contain **only text copied verbatim from the ATA
ticket's `Technical Analysis`**:
- **Do NOT invent, infer, assume, summarize, rephrase or expand** anything.
  Not a single sentence, field, value, example or detail may appear in the
  Story that is not literally present in the ATA.
- **Do NOT move content between sections.** Never attach one component's
  table or JSON example to a different component's story.
- If a story has no descriptive text beyond its table row, the description =
  only that row's `Jira` text. Never pad it.
- **Do NOT put the Estimate into the description.** It goes only into
  `timetracking.originalEstimate`. Remove any `Estimate: ...` line even if the
  ATA text contains it.

### 2.4 ADF → markdown conversion
`description` is sent as markdown, so convert the copied ADF nodes 1:1,
changing only the format, never the text:
| ADF node | Markdown |
|----------|----------|
| heading (level N) | `#` × N |
| paragraph | plain text line |
| bulletList / orderedList | `-` / `1.` items, nesting preserved |
| table | markdown table, same rows/columns/cell text |
| codeBlock (JSON / message examples) | ` ```json ` block (or the block's own language), content unchanged |
| inline code / strong / em / link | `` `x` `` / `**x**` / `*x*` / `[text](url)` |

### 2.5 Estimate normalization
Convert to Jira units for `timetracking.originalEstimate`:
- `<N>pd` → `<N>d` (`1pd` → `1d`, `2.5pd` → `2.5d`)
- `<N>ph` → `<N>h`
- Values already in `d` / `h` pass through unchanged. Keep the number exact.

---

## Step 3: Print the parsed list
Print the parsed list, then proceed to Step 4 immediately (no confirmation):
```
Preview:
1. [media-aggregator-processor] Support header and blockquote blocks in Article HTML Content function (1d)
2. [media-aggregator-processor] Support header and blockquote blocks in Article Web Export function (1d)
3. [media-aggregator-api] Support header and blockquote blocks in Article Ex and Combined Ex APIs (2d)
```

---

## Step 4: Create Stories
Create the stories one by one, in table order. For EACH row call
`createJiraIssue`:

| Parameter | Value |
|-----------|-------|
| `cloudId` | `"livescoregroup.atlassian.net"` |
| `projectKey` | `"PROD"` |
| `issueTypeName` | `"Story"` |
| `summary` | `"<Component> <Jira>"` |
| `description` | Per Step 2.2–2.4 (markdown, **no Estimate line**) |
| `contentFormat` | `"markdown"` |
| `additional_fields` | JSON object below |

```json
{
  "priority": {"id": "<PRIORITY_ID>"},
  "parent": {"key": "<PARENT_KEY>"},
  "components": [{"id": "11078"}],
  "customfield_10156": {"id": "70121:4acf64ef-fbea-4399-b874-411f0bc38345"},
  "customfield_10001": "<TEAM_ID>",
  "customfield_10155": {"type": "doc", "version": 1, "content": [{"type": "paragraph", "content": [{"type": "text", "text": "-"}]}]},
  "assignee": {"id": "626a98c607b842006f154964"},
  "timetracking": {"originalEstimate": "<ESTIMATE>"},
  "fixVersions": [{"name": "LSM - BE 1.69.0 - Platform"}]
}
```

Only `summary`, `description` and `timetracking.originalEstimate` change
between rows. If one row fails, continue with the remaining rows and report
the failure in Step 5.

---

## Worked Example (verified – ATA PROD-38907 → parent PROD-37933, team delta)
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

---

## Field Reference

### `additional_fields`
| Field | Meaning | Value |
|-------|---------|-------|
| `priority` | Priority (mandatory) | `{"id": "<PRIORITY_ID>"}` (from ATA, fallback `10007`) |
| `parent` | Parent link (mandatory) | `{"key": "<PARENT_KEY>"}` |
| `components` | Component (mandatory) | `[{"id": "11078"}]` |
| `customfield_10156` | Product Owner (mandatory) | `{"id": "70121:4acf64ef-fbea-4399-b874-411f0bc38345"}` (Natalia Klipailo) |
| `customfield_10155` | Acceptance Criteria, ADF (mandatory) | `{"type":"doc","version":1,"content":[{"type":"paragraph","content":[{"type":"text","text":"-"}]}]}` |
| `customfield_10001` | Team (always set) | `<TEAM_ID>` from the Teams table |
| `assignee` | Assignee (always set) | `{"id": "626a98c607b842006f154964"}` |
| `timetracking` | Estimate (always set) | `{"originalEstimate": "<ESTIMATE>"}` |
| `fixVersions` | Fix Version (always set) | `[{"name": "LSM - BE 1.69.0 - Platform"}]` |

### Teams (`customfield_10001`)
| Team | ID |
|------|-----|
| `bravo` | `8fbf2d82-c7a5-4079-a7e1-7f5299e53152-161` |
| `delta` (default) | `8fbf2d82-c7a5-4079-a7e1-7f5299e53152-163` |

---

## Error Handling
- **"Description, Priority, Parent, Product Owner & Components are mandatory"** →
  ensure `description` is non-empty and `additional_fields` has `priority.id`,
  `parent.key`, `components[0].id`, `customfield_10156.id`; retry the row.
- **"parent: Could not find issue"** → call `getJiraIssue` on `<PARENT_KEY>`,
  take its `id`, and retry the row with `"parent": {"id": "<PARENT_INTERNAL_ID>"}`.
- **`fixVersions` rejected** → retry the row with `"fixVersions": []` and
  mention in Step 5 that the story was created without a Fix Version.

---

## Step 5: Output
Return:
- created issue keys with their summaries, in table order;
- stories created with a fallback (e.g. without Fix Version);
- failed rows with the exact Jira error.

---

## Constants
| Name | Value |
|------|-------|
| Cloud ID | `livescoregroup.atlassian.net` |
| Project Key | `PROD` |
| Issue Type | `Story` |
| Technical Analysis field (ATA) | `customfield_18234` |
| Acceptance Criteria field | `customfield_10155` |
| Product Owner field | `customfield_10156` |
| Team field | `customfield_10001` |
| Component ID (LSM BE) | `11078` |
| Product Owner (Natalia Klipailo) | `70121:4acf64ef-fbea-4399-b874-411f0bc38345` |
| Team `bravo` | `8fbf2d82-c7a5-4079-a7e1-7f5299e53152-161` |
| Team `delta` (default) | `8fbf2d82-c7a5-4079-a7e1-7f5299e53152-163` |
| Assignee ID | `626a98c607b842006f154964` |
| Default Priority (P2) | `10007` |
| Fix Version | `LSM - BE 1.69.0 - Platform` |
