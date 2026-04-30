# Jira MCP Instructions – Epic → Stories Generator

## Goal

Create Jira Stories from an Epic using MCP tools.

---

## Step 1: Read the Epic

Call MCP tool `getJiraIssue` with:
- `cloudId`: `"livescoregroup.atlassian.net"`
- `issueIdOrKey`: the Epic key (e.g., `"PROD-20494"`)


Save from the response:
- `key` → Epic key (e.g., `"PROD-20494"`)
- `id` → Epic internal ID (e.g., `"478482"`)
- `fields.priority.id` → Priority ID (e.g., `"10008"`)

---

## Step 2: Parse the Epic

Find in the Epic description or custom fields:
1. Section `LSM Technical analysis` → use as Story description source
2. Table `Proposed Stories` → each row = one Story

Example table:

| Summary | Estimate |
|---------|----------|
| Task 1  | 1d       |
| Task 2  | 4h       |

Rules:
- Do NOT skip rows
- Do NOT invent rows
- Do NOT modify summaries or estimates

## Critical Constraints
- NEVER generate new content that is not present in input

---

## Step 3: Preview

Before creating, show:

```
Preview:
1. Task 1 (1d)
2. Task 2 (4h)
```

Then proceed to create automatically. Do not ask for confirmation.

---

## Step 4: Create Stories

For EACH row in `Proposed Stories`, call MCP tool `createJiraIssue`.

### MCP Tool Parameters

The `createJiraIssue` tool has these parameters:

| Parameter | Value |
|-----------|-------|
| `cloudId` | `"livescoregroup.atlassian.net"` |
| `projectKey` | `"PROD"` |
| `issueTypeName` | `"Story"` |
| `summary` | From table row |
| `description` | From LSM Technical analysis (use markdown) |
| `contentFormat` | `"markdown"` |
| `additional_fields` | JSON object with all other Jira fields (see below) |

### The `additional_fields` Parameter (CRITICAL)

All Jira-specific fields MUST go inside `additional_fields` as a JSON object.

**IMPORTANT**: The `additional_fields` value must be a valid JSON object, not a string.

```json
{
  "priority": {"id": "<EPIC_PRIORITY_ID>"},
  "parent": {"key": "<EPIC_KEY>"},
  "components": [{"id": "11078"}],
  "customfield_10156": {"id": "70121:4acf64ef-fbea-4399-b874-411f0bc38345"},
  "customfield_10001": "ba3cca31-9b6c-4488-bc91-ea4af3f614fd",
  "customfield_10155": {"type": "doc", "version": 1, "content": [{"type": "paragraph", "content": [{"type": "text", "text": "-"}]}]},
  "assignee": {"id": "626a98c607b842006f154964"},
  "timetracking": {"originalEstimate": "<ESTIMATE>"},
  "fixVersions": [{"name": "1.234 Test Ivan"}]
}
```

---

## Complete Example

To create a Story "Extend Bds Market import" with estimate "1d" under Epic PROD-20494 (priority.id = "10008"):

```
Tool: createJiraIssue

Parameters:
- cloudId: "livescoregroup.atlassian.net"
- projectKey: "PROD"
- issueTypeName: "Story"
- summary: "Extend Bds Market import with selection probability"
- description: "Technical implementation details from LSM Technical analysis section..."
- contentFormat: "markdown"
- additional_fields: {
    "priority": {"id": "10008"},
    "parent": {"key": "PROD-20494"},
    "components": [{"id": "11078"}],
    "customfield_10156": {"id": "70121:4acf64ef-fbea-4399-b874-411f0bc38345"},
    "customfield_10001": "ba3cca31-9b6c-4488-bc91-ea4af3f614fd",
    "customfield_10155": {"type": "doc", "version": 1, "content": [{"type": "paragraph", "content": [{"type": "text", "text": "-"}]}]},
    "assignee": {"id": "626a98c607b842006f154964"},
    "timetracking": {"originalEstimate": "1d"},
    "fixVersions": [{"name": "1.234 Test Ivan"}]
  }
```

---

## Field Reference

### Mandatory Fields in `additional_fields`

These fields are required by Jira and MUST be included:

| Field | Description | Value |
|-------|-------------|-------|
| `priority` | Priority object | `{"id": "<from_epic>"}` |
| `parent` | Link to Epic | `{"key": "<EPIC_KEY>"}` |
| `components` | Component array | `[{"id": "11078"}]` |
| `customfield_10156` | Product Owner | `{"id": "70121:4acf64ef-fbea-4399-b874-411f0bc38345"}` |
| `customfield_10155` | Acceptance Criteria (ADF) | See below |

### Optional Fields in `additional_fields`

| Field | Description | Value |
|-------|-------------|-------|
| `customfield_10001` | Team | `"ba3cca31-9b6c-4488-bc91-ea4af3f614fd"` |
| `assignee` | Assignee | `{"id": "626a98c607b842006f154964"}` |
| `timetracking` | Estimate | `{"originalEstimate": "<from_table>"}` |
| `fixVersions` | Fix Version | `[{"name": "1.234 Test Ivan"}]` |

### Priority Fallback

If Epic has no priority, use P2:
```json
{"id": "10007"}
```

### Acceptance Criteria (ADF Format)

Always use this exact value:
```json
{
  "type": "doc",
  "version": 1,
  "content": [
    {
      "type": "paragraph",
      "content": [
        {"type": "text", "text": "-"}
      ]
    }
  ]
}
```

---

## Error Handling

### Error: "Description, Priority, Parent, Product Owner & Components are mandatory fields"

Missing required fields. Check that `additional_fields` contains:
- `priority` with `id`
- `parent` with `key`
- `components` array with `id`
- `customfield_10156` with `id`

Also ensure `description` parameter is not empty.

### Error: "parent: Could not find issue by id or key"

1. Re-read the Epic
2. Verify the Epic key
3. Try `parent` with `id` instead of `key`:
   ```json
   {"id": "<EPIC_INTERNAL_ID>"}
   ```

### Error: fixVersions rejected

Retry with empty array:
```json
"fixVersions": []
```

---

## Step 5: Output

After all Stories are created, return:
- List of created issue keys
- Any failed rows with the exact Jira error

---

## Quick Checklist Before Creating

Before calling `createJiraIssue`, verify:

- [ ] `cloudId` = `"livescoregroup.atlassian.net"`
- [ ] `projectKey` = `"PROD"`
- [ ] `issueTypeName` = `"Story"`
- [ ] `summary` = from table (not empty)
- [ ] `description` = from LSM Technical analysis (not empty)
- [ ] `contentFormat` = `"markdown"`
- [ ] `additional_fields` contains:
  - [ ] `priority.id` (from Epic or `"10007"`)
  - [ ] `parent.key` (Epic key)
  - [ ] `components[0].id` = `"11078"`
  - [ ] `customfield_10156.id` = Product Owner ID
  - [ ] `customfield_10155` = ADF object for Acceptance Criteria
  - [ ] `timetracking.originalEstimate` (from table)

---

## Constants

| Name | Value |
|------|-------|
| Cloud ID | `livescoregroup.atlassian.net` |
| Project Key | `PROD` |
| Issue Type | `Story` |
| Component ID (LSM BE) | `11078` |
| Product Owner ID | `70121:4acf64ef-fbea-4399-b874-411f0bc38345` |
| Team ID | `ba3cca31-9b6c-4488-bc91-ea4af3f614fd` |
| Assignee ID | `626a98c607b842006f154964` |
| Default Priority (P2) | `10007` |
| Fix Version | `1.234 Test Ivan` |
