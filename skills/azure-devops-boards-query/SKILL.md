---
name: azure-devops-boards-query
description: "Query Azure DevOps Boards work items via az CLI (WIQL), summarize results, and generate a PowerPoint report. Use whenever the user mentions Azure DevOps, ADO boards, work items, WIQL queries, or creating a PPT from ADO data."
---

# Azure DevOps Boards Query & Report Skill

## Prerequisites

1. **Azure CLI** installed:
   ```bash
   curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
   ```
2. **Azure DevOps extension** installed:
   ```bash
   az extension add --name azure-devops
   ```
3. **Authenticated**:
   - Interactive: `az login`
   - Or set PAT: `export AZURE_DEVOPS_EXT_PAT=<your_pat>`

## Core Commands

### Query Work Items (WIQL)
```bash
az boards query \
  --wiql "SELECT [System.Id], [System.Title], [System.State], [System.AssignedTo], [System.ChangedDate] FROM workitems WHERE [System.TeamProject] = 'MyProject' AND [System.State] = 'Active'" \
  --organization https://dev.azure.com/<org> \
  --project <project> \
  --output json
```

### Show Single Work Item
```bash
az boards work-item show \
  --id <ID> \
  --organization https://dev.azure.com/<org> \
  --project <project> \
  --output json
```

### List Saved Queries
```bash
az boards query list \
  --organization https://dev.azure.com/<org> \
  --project <project> \
  --output table
```

### Run a Saved Query by ID
```bash
az boards query \
  --id <query-id> \
  --organization https://dev.azure.com/<org> \
  --project <project> \
  --output json
```

## Common WIQL Patterns

| Goal | WIQL Snippet |
|------|--------------|
| Active bugs | `SELECT [System.Id], [System.Title] FROM workitems WHERE [System.WorkItemType] = 'Bug' AND [System.State] = 'Active'` |
| Assigned to me | `... AND [System.AssignedTo] = @Me` |
| Changed this week | `... AND [System.ChangedDate] >= @Today - 7` |
| Sprint items | `... AND [System.IterationPath] UNDER 'MyProject\Sprint 1'` |
| High priority | `... AND [Microsoft.VSTS.Common.Priority] = 1` |

> **Tip**: Use the Azure DevOps web UI to design queries, then click **"Copy query as WIQL"**.

## Full Workflow: Query → Summarize → PPT

### Step 1: Execute Query & Save JSON
```bash
az boards query \
  --wiql "SELECT [System.Id], [System.Title], [System.State], [System.AssignedTo], [Microsoft.VSTS.Common.Priority] FROM workitems WHERE [System.TeamProject] = 'MyProject' AND [System.State] IN ('Active', 'Resolved')" \
  --organization https://dev.azure.com/<org> \
  --project MyProject \
  --output json > /tmp/ado_workitems.json
```

### Step 2: Summarize with Python
```python
import json
from collections import Counter

with open("/tmp/ado_workitems.json") as f:
    items = json.load(f)

# Flatten fields
records = []
for i in items:
    fields = i.get("fields", {})
    records.append({
        "id": i.get("id"),
        "title": fields.get("System.Title", "N/A"),
        "state": fields.get("System.State", "N/A"),
        "assigned": fields.get("System.AssignedTo", {}).get("displayName", "Unassigned"),
        "priority": fields.get("Microsoft.VSTS.Common.Priority", "-"),
    })

state_counts = Counter(r["state"] for r in records)
priority_counts = Counter(str(r["priority"]) for r in records)

print(f"Total items: {len(records)}")
print("By state:", dict(state_counts))
print("By priority:", dict(priority_counts))
```

### Step 3: Generate PPT Report
Load the **powerpoint** skill, then create a deck with:

1. **Title slide** — "Azure DevOps Work Items Report"
2. **Summary slide** — big stats (total count, active count, resolved count)
3. **Breakdown slide** — state distribution (chart or icon grid)
4. **Top Items slide** — table or cards of high-priority / recently changed items
5. **Conclusion slide** — key takeaways

Example Python+PptxGenJS snippet:
```bash
npm install -g pptxgenjs
```

```javascript
const PptxGenJS = require("pptxgenjs");
const fs = require("fs");

const pptx = new PptxGenJS();
const data = JSON.parse(fs.readFileSync("/tmp/ado_workitems.json", "utf8"));

// Title
let slide = pptx.addSlide();
slide.addText("Azure DevOps Work Items Report", { x: 0.5, y: 1.5, fontSize: 36, bold: true, color: "1E2761" });

// Stats
const states = {};
data.forEach(i => {
  const s = i.fields?.["System.State"] || "Unknown";
  states[s] = (states[s] || 0) + 1;
});

slide = pptx.addSlide();
slide.addText("Summary", { x: 0.5, y: 0.5, fontSize: 28, bold: true });
Object.entries(states).forEach(([state, count], idx) => {
  slide.addText(`${count}  ${state}`, { x: 0.5, y: 1.2 + idx * 0.8, fontSize: 20, color: "36454F" });
});

pptx.writeFile({ fileName: "ADO_Report.pptx" });
```

## Pitfalls

1. **Missing `--output json`** — default table output is hard to parse programmatically.
2. **Unescaped WIQL quotes** — wrap the whole `--wiql` value in single quotes if the query contains double quotes.
3. **Permission denied** — ensure the PAT has **Work Items (Read)** scope at minimum.
4. **Large result sets** — WIQL returns all matches; paginate with `TOP N` or filter by date.
5. **AssignedTo format** — it returns a JSON object `{displayName, uniqueName}`, not a plain string.

## Quick Checklist
- [ ] `az --version` works and `azure-devops` extension is installed
- [ ] `az login` completed or `AZURE_DEVOPS_EXT_PAT` is exported
- [ ] `--organization` uses `https://dev.azure.com/<org>` (not `dev.azure.com/<org>` without protocol)
- [ ] WIQL query validated in Azure DevOps web UI first
- [ ] `--output json` used for downstream parsing
- [ ] PowerPoint skill loaded for PPT generation
