## Core Foundational Fact

Microsoft Sentinel is **installed inside a Log Analytics Workspace** — not a standalone product. The workspace is the container/database; Sentinel is the security layer of features built on top.

---

## Part 1 — Planning the Workspace

### Most important setting: Region

- Determines the physical location where log data is stored
- **Cannot be changed after creation** — must delete and recreate the workspace to change it
- Critical for data governance/legal compliance (some countries require data to stay physically within their borders due to jurisdiction — controls which government/legal system has authority over that data, not about who can remotely view it)

### Three Implementation Options

**1. Single-Tenant, Single Workspace** One company, one central Sentinel workspace collecting logs from all regions.

|Pros|Cons|
|---|---|
|Central pane of glass (one screen for everything)|May not meet data governance requirements|
|All security data consolidated|Cross-region bandwidth costs|
|Easier to query everything at once||
|Azure RBAC + Sentinel RBAC for access control||

**2. Single-Tenant, Regional Workspaces** One company, multiple separate Sentinel workspaces (one per region).

|Pros|Cons|
|---|---|
|No cross-region bandwidth costs|No central pane of glass|
|Meets data governance requirements|Analytics rules/workbooks must be deployed multiple times|
|Granular access control per region||
|Granular retention settings per region||
|Split billing||

Query across regional workspaces using:

```kql
TableName
| union workspace("WorkspaceName").TableName
```

**3. Multiple Tenant Workspaces** For managing a Sentinel workspace belonging to a **different company** (different tenant) — used via **Azure Lighthouse**, common for security consulting companies managing multiple clients.

### Sharing workspace with Microsoft Defender for Cloud

- Recommended for centralization — all Defender for Cloud logs become usable by Sentinel
- **Catch:** The default auto-created Defender for Cloud workspace CANNOT be reused for Sentinel — must manually create a workspace first, then point Defender for Cloud to it

---

## Part 2 — Creating the Workspace

### Prerequisites (permissions needed)

- **Contributor** on the subscription → to enable Sentinel
- **Contributor or Reader** on the resource group → to use Sentinel

### Basics tab fields

|Field|Meaning|
|---|---|
|Subscription|Billing account|
|Resource Group|Container/folder for the resource|
|Name|Shared name for both Log Analytics Workspace and Sentinel|
|Region|Storage location — cannot be changed later|

### Sentinel's four navigation areas

- **General** — overview/dashboard
- **Threat management** — incidents, workbooks, hunting
- **Content management** — content hub, pre-built packs
- **Configuration** — settings, connectors, analytics rules

---

## Part 3 — Managing Multiple Workspaces

**Sentinel Workspace Manager** — one central workspace pushes content (rules, workbooks) automatically to multiple "member" workspaces. Enabled in Configuration settings.

**Azure Lighthouse** — grants access into another company's tenant without separate logins. Relevant for consulting companies (like Consultim-IT) managing multiple client environments from their own tenant.

---

## Part 4 — Roles and Permissions (RBAC)

**RBAC** = Role-Based Access Control — bundle permissions into named roles instead of assigning individually.

### Built-in Sentinel Roles (each builds on the previous)

|Role|Can do|
|---|---|
|Sentinel Reader|View data, incidents, workbooks only|
|Sentinel Responder|Reader + manage incidents (assign/dismiss)|
|Sentinel Contributor|Responder + create/edit workbooks, analytics rules|
|Sentinel Automation Contributor|Internal use only (adds playbooks to automation rules) — not for real users|

**Best practice:** assign at the resource group level so it applies to all related resources automatically.

### Additional roles for specific tasks

|Task|Extra role needed|
|---|---|
|Create/edit playbooks (Logic Apps)|Logic App Contributor|
|Sentinel auto-running playbooks|Admin needs Owner on resource group to grant the special internal service account permission|
|Add data connectors|Write permission on Sentinel workspace|
|Guest user assigning incidents|Sentinel Responder + Directory Reader (Entra role, not Azure role — regular users get this by default, guests don't)|
|Create/delete workbooks|Sentinel Contributor OR lower role + Workbook Contributor (Azure Monitor role)|

### Broader Azure/Log Analytics roles (can override Sentinel roles)

- **Owner** — full control, including access management
- **Contributor** — create/edit/delete, no access management
- **Reader** — view only
- **Log Analytics Contributor / Reader** — same idea but scoped to Log Analytics

**Warning:** combining roles can accidentally grant more access than intended — e.g., Sentinel Reader + Azure Contributor = can actually edit Sentinel data, because Azure Contributor is broader and overrides.

### Permissions Table

|Role|Run playbooks|Create/edit rules & workbooks|Manage incidents|View data|
|---|---|---|---|---|
|Reader|No|No|No|Yes|
|Responder|No|No|Yes|Yes|
|Contributor|No|Yes|Yes|Yes|
|Contributor + Logic App Contributor|Yes|Yes|Yes|Yes|

**Custom roles** available if built-in roles don't fit — assignable at management group, subscription, or resource group level.

---

## Part 5 — Settings and Retention

Settings managed in **two places**: Sentinel's own Settings tab (Pricing, Settings, Workspace Settings) and the underlying Log Analytics Workspace (most detailed settings — e.g., data connector configs live here).

**Log retention range:** 30 to 730 days (2 years), except legacy Free tier. Adjusted via Workspace Settings → Log Analytics portal → "Usage and estimated costs" → Retention button.

---

## Part 6 — Data States, Table Plans, and Tiers (the core complex topic)

### Concept 1 — Two Data States (general, applies to all data)

|State|Meaning|
|---|---|
|**Analytics retention (hot)**|Fully usable now — real-time queries, alerting, hunting|
|**Long-term retention (cold)**|Cheap, archived — not instantly usable, accessed via Search Jobs or Restore|

**Search Job:** temporarily pulls cold data back into a special searchable table (suffix `_SRCH`) when you need to investigate it — like requesting a box back from storage.

### Concept 2 — Three Table Plans (assigned per table; Analytics is the default)

|Plan|KQL support|Interactive (hot) retention|Long-term retention|Used for real-time alerting?|
|---|---|---|---|---|
|**Analytics**|Full (summarize, join, union, everything)|30 days – 2 years (up to 730 days)|0–4,380 days (12 years)|**Yes**|
|**Basic**|Limited — where, extend, project, project-away, project-keep, project-rename, project-reorder, parse, parse-where only. No join/union/summarize|30 days|0–4,380 days (12 years)|No|
|**Auxiliary**|Same limited set as Basic, but unoptimized (slower)|30 days|0–4,380 days (12 years)|No|

**Critical distinction:** "interactive retention days" ≠ "real-time alerting capability." Basic/Auxiliary CAN be manually queried (with limited operators) for 30 days for troubleshooting/investigation — but Sentinel's analytics rules/detection engine only work on **Analytics plan** tables. Basic/Auxiliary are for occasional manual investigation only, never active threat detection.

### Data model — table origins (separate from plan/cost)

|Type|Description|
|---|---|
|Azure table|Built-in logs from Azure resources (predefined schema + custom columns)|
|Custom table|User-defined schema, any data source|
|Search results|Temporary table from a Search Job|
|Restored logs|Archived logs brought back, same schema as original|

### Concept 3 — Sentinel-Specific Tiers (where data physically/administratively lives)

This mostly **renames** Concept 1's hot/cold idea using official Sentinel terms, plus adds one genuinely new fact (the XDR tier).

**Analytics tier** (inside Sentinel workspace) = the hot zone. Sentinel + custom tables + supported XDR hunting tables. 30 days–2 years analytics retention; total retention 30 days–12 years.

**Data lake tier** (inside Sentinel workspace) = the cold zone. Same tables, NOT used for real-time analytics/hunting. Retention 30 days–12 years. Accessed via KQL jobs, scheduled Spark jobs, or summary rules. Can be re-ingested into Analytics tier if needed later.

**XDR default tier** (OUTSIDE Sentinel workspace, lives in Defender XDR separately) = XDR's own free 30-day bucket, included with XDR license. NOT part of Sentinel workspace or its billing. To bring this data into Sentinel: extend the retention beyond 30 days → this **ingests/moves** the data into Sentinel's Analytics tier (not a duplicate copy — the data actually moves in). Only after it's inside Sentinel's Analytics tier can you then also apply Data Lake tier options to it if long-term cheap storage is wanted.

### Two ways data becomes "long-term/cold" (clearing up Data Lake tier vs Total Retention confusion)

**Option A — Select "Data lake tier" directly for a table** Entire table is cold from day one. No real-time analytics, alerting, or hunting on it ever. Used for tables you never need to actively monitor, only keep for compliance/audit.

**Option B — Stay on "Analytics tier," but set Total Retention higher than Analytics Retention** Table stays fully hot/active for real-time monitoring for the Analytics Retention period (e.g., 30 days), AND a cheap backup copy automatically continues in the Data Lake for the remaining time up to Total Retention (e.g., 180 days total). This is the correct choice for almost every important security table — real-time detection now, cheap safety net for later investigation/audits.

**Key distinction:** "Data lake tier" (the button) = fully cold table, no exceptions. "Total retention > Analytics retention while staying on Analytics tier" = stays hot AND gets a long-term cheap backup — these are two different configurations, not the same thing.

---

## Part 7 — Managing Tables in the Defender Portal

### What is "the Microsoft Defender portal"?

Previously, Sentinel was managed in the **Azure Portal**, while Defender XDR products (Endpoint, Identity, Office 365) were managed in a separate **Microsoft 365 Defender Portal** (security.microsoft.com). Microsoft has now merged Sentinel management INTO that unified Defender portal — so both Sentinel and Defender products can be managed from one single website instead of two.

If your Sentinel workspace is "connected to the Defender portal," it means you've adopted this new unified management experience instead of the old Azure-Portal-only way.

### Table types manageable in the Defender portal

|Table type|Description|Examples|In Sentinel workspace?|
|---|---|---|---|
|**Microsoft Sentinel**|Built-in Azure tables + Sentinel tables + supported XDR hunting tables (only once retention extended beyond 30 days)|AzureDiagnostics, SigninLogs, AWSCloudTrail, SecurityAlert|Yes|
|**Custom**|User-created tables, or auto-created via summary rules/search jobs|Tables with `_CL` or `_SRCH` suffix|Yes|
|**XDR**|Default XDR tier tables — viewable but NOT manageable from Defender portal|IdentityInfo|No|

**Basic logs catch:** Basic-tier tables are visible in the Defender portal but can currently only be MANAGED from the old Log Analytics workspace interface. To manage them via the new Defender portal, you must first convert them from Basic to Analytics plan using the old interface — a one-time bridging step, not a permanent cost increase (you can move it back to a cheaper tier afterward once it's visible in the new interface). This is a feature-support gap in the newer tool, not a deliberate cost trap — and it's an edge case relevant only to old, pre-existing enterprise setups, not fresh homelabs.

### Manage Table workflow

1. Sentinel → Configuration → Tables (in Defender portal)
2. Table list shows: Table name, Tier, Table type, Analytics retention, Total retention, Workspace
3. Click a table → side panel with description, tier, retention details
4. Click "Manage table" → adjust settings:
    - **Analytics retention:** 30 days – 2 years
    - **Total retention:** up to 12 years (default = same as Analytics retention, meaning no extra long-term storage unless increased)
    - Example: 90 days Analytics retention + 180 days Total retention = 90 days hot + 90 more days cheap cold storage
    - **Data lake tier retention:** 30 days – 12 years (if choosing this tier directly)
5. Some tables (XDR tables, Sentinel solution tables) CANNOT change tier — must stay in Analytics tier because Microsoft's security services require near-real-time access to them
6. Warnings shown: increased retention = increased cost; switching Analytics → Data lake tier disables Alerting, Advanced hunting, Analytics rules, Custom detection rules for that table

---

## Part 8 — Summary Rules Tables

**Problem solved:** Running expensive `summarize` queries repeatedly on massive raw tables is costly and slow, especially once that raw data is sitting in cold Data Lake storage.

**Solution:** A **Summary Rule** automatically runs a scheduled aggregation query (like an hourly/daily `summarize`) and saves the result into a new small, custom table.

**Example:**

```kql
FirewallLogs
| summarize BlockedCount = count() by bin(TimeGenerated, 1h), SourceIP
```

This runs automatically on schedule, storing results in a new table (e.g., `FirewallLogs_Summary`).

**Benefit:** The massive raw table (`FirewallLogs`) can be moved to cheap Data Lake tier, while the small summary table stays fast, cheap, and instantly queryable — giving you trend visibility without paying to keep millions of raw rows hot.

**Analogy:** Instead of keeping every receipt (expensive, rarely needed individually), keep a monthly summary spreadsheet (cheap, always useful) while the actual receipts get boxed away in cold storage (Data Lake) for the rare occasion you need to dig one out.

---

## Part 9 — Basic Logs KQL Limitations (full detail)

All tables default to Analytics plan. Only specific table types can be configured as Basic:

- Tables created via Data Collection Rule (DCR)-based custom logs API
- `ContainerLogV2` (Container Insights verbose logs)
- `AppTraces` (Application Insights trace logs)

**Supported KQL on Basic logs:** `where`, `extend`, `project`, `project-away`, `project-keep`, `project-rename`, `project-reorder`, `parse`, `parse-where`

**NOT supported on Basic logs:** `join`, `union`, `summarize` (any aggregate functions)

---

## Key Rules to Remember From This Module

1. Region choice is permanent — decide carefully before creating a workspace
2. Data governance laws control legal jurisdiction over data, not remote viewing access
3. Table plan (Analytics/Basic/Auxiliary) is assigned **per table**, with Analytics as default
4. Only **Analytics plan** tables get real Sentinel alerting/detection — Basic/Auxiliary are investigation-only
5. "Data lake tier" (direct selection) = fully cold, no real-time features, ever
6. "Total retention > Analytics retention" (while staying on Analytics tier) = stays hot AND gets a cheap long-term backup — different from #5
7. XDR default tier data lives outside Sentinel entirely — must extend retention to move/ingest it into Sentinel's Analytics tier before any Sentinel-specific tier options apply to it
8. Defender portal = the new unified website merging Sentinel + Defender XDR management (replacing the old two-separate-websites approach)
9. Basic-plan tables need temporary conversion to Analytics before the new Defender portal interface can manage them — a one-time bridging step, not a permanent forced cost increase
10. "Public preview" (like Sentinel data lake) means usable now but still evolving — not broken, not finished either
11. Summary rules = scheduled auto-aggregation into small cheap tables, letting raw data go cold while trends stay fast and queryable