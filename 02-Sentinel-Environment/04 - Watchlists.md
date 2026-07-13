## 📌 Overview
- Watchlists = tables in Sentinel where you upload your own external data (CSV), queryable via KQL.
- Managed from: `Microsoft Sentinel > Configuration > Watchlist`.

**Scenario:** SOC team wants to prioritize alerts on high-value servers → upload server list as a watchlist → detection rules reference it to boost priority.

**Learning objectives:**
- Create a watchlist
- Query a watchlist with KQL

---

## 🧠 What a Watchlist Actually Is (Important Clarification)

A watchlist is **NOT** just a single column of names. It's a full table (like a CSV/Excel sheet) with multiple columns — one column acts as the **search key** (unique identifier), and you can add extra columns for context.

Example — "HighValueMachines" watchlist:

| ServerName | Owner | Department | Criticality |
|---|---|---|---|
| pca | John  | Finance    | High |
| pcb | Sarah | IT         | Critical |
| pcd | Mike  | HR         | Medium |

- Even a "simple" one-column watchlist (just server names) still has a search key — it's just that one column.
- **Search key field** = the column used to uniquely identify/match each row (chosen when the watchlist is created). Used to detect duplicates during updates.

---

## 🎯 Why Use Watchlists (4 Scenarios)

| Scenario | Purpose |
|---|---|
| Threat investigation | Quickly import IPs/hashes/IOCs from CSV → use in joins/filters across rules, hunting, workbooks |
| Business data import | Import privileged users / terminated employees → build allowlists/blocklists |
| Reduce alert fatigue | Allowlist known-safe users/IPs → suppress expected/benign alerts |
| Enrich event data | Join watchlist to logs to add extra context (e.g. department, owner) |

---

## ⚙️ Key Concepts
- Stored as **name-value pairs**, **cached** → low latency, fast query performance.
- Data comes from **external sources** (not Sentinel connectors) — e.g. a CSV you made.
- Usable in: search queries, detection (analytic) rules, threat hunting, response playbooks (Logic Apps).

---

## 🛠️ Creating a Watchlist (Azure Portal)

1. `Microsoft Sentinel > Configuration > Watchlist > Add new`
2. **General page:** Name, Description, **Alias** (used to reference it in KQL) → Next
3. **Source page:** choose dataset type, upload CSV file → Next
   - ⚠️ Max file size: **3.8 MB**
4. **Review page:** verify → Create. Notification confirms when ready.

### Query a watchlist in KQL
```kql
_GetWatchlist('HighValueMachines')
```
- `'HighValueMachines'` = the **alias**, not necessarily the display name.
- Returns the watchlist as a table — usable like any other table (filter, join, etc.)

**Example — join with logs:**
```kql
SecurityEvent
| join kind=inner (_GetWatchlist('HighValueMachines')) on $left.Computer == $right.ServerName
```
→ Only returns SecurityEvent rows where the computer matches a high-value server.

---

## 🔧 Managing Watchlists

### General rule: Edit, don't delete + recreate
- Log Analytics has a **5-minute SLA** for data ingestion.
- Delete + immediately recreate → both old (deleted) and new entries may briefly appear together during that window → confusing duplicates.
- Duplicates lasting longer than 5 min → submit a support ticket.

### Edit a single item
1. Sentinel → select workspace → `Configuration > Watchlist`
2. Select the watchlist
3. `Update watchlist > Edit watchlist items`
4. **Edit existing item:** check its box → edit → Save → confirm Yes
5. **Add new item:** Add new → fill fields → Add

### Bulk update (many items at once)

**How it behaves:**
- **Appends** new data — does NOT wipe/replace the whole watchlist.
- **Deduplicates** rows that are 100% identical (all columns match on the search key).
- **Cannot delete** — removing a row from your upload file does NOT delete it from the existing watchlist.
- To delete: remove items individually, or if many deletions needed → delete & recreate the whole watchlist.
- Upload file **must** include the search key column with **no blank values**.

**Example of bulk update behavior:**

Existing watchlist:

| ServerName |
|---|
| pca |
| pcb |
| pcd |

Uploaded bulk file:

| ServerName |
|---|
| pcb |
| pce |
| pcf |

Result after bulk update:

| ServerName |
|---|
| pca |
| pcb |
| pcd |
| pce |
| pcf |

- `pcb` → already existed → not duplicated
- `pce`, `pcf` → new → added
- `pca`, `pcd` → missing from upload file → **still kept** (bulk update never auto-deletes)

**Steps:**
1. Sentinel → select workspace → `Configuration > Watchlist`
2. Select the watchlist
3. `Update watchlist > Bulk update`
4. Upload file (drag-drop or browse)
5. If error → fix file → Reset → retry upload
6. `Next: Review and update > Update`

---

## 💡 Quick Mental Model
> Bulk update = "add anything new, keep anything old, skip exact duplicates, cannot delete." For deletions: remove manually, or delete & recreate the whole watchlist if there are many.