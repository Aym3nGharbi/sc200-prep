
## 1. Introduction

- Different data sources land in different tables:
  - `SecurityEvent` — Windows security event logs (logons, logoffs, process creation, etc.)
  - `SigninLogs` — Azure AD (Entra ID) sign-in logs (cloud logins)
  - Others: `DeviceProcessEvents`, `AzureActivity`, `OfficeActivity`, `ThreatIntelligenceIndicator`, etc.
- **Union** = stack tables **on top of each other** (append rows).
- **Join** = merge tables **side by side** based on a matching column (like Excel VLOOKUP).
- KQL is a **pipeline language** read queries top to bottom. Each stage's output becomes the next stage's input. **Order changes the result.**

---

## 2. The `union` Operator

### Definition
`union` takes two or more tables and returns **all rows from all of them, stacked**. It does not match/compare rows like `join` does. Columns that exist in one table but not the other become `null` for rows from the other table.

### The pipe `|`
- Takes the result **before** it and feeds it as input to the operator **after** it.
- Read top-to-bottom as sequential steps.
- Order matters: placing `summarize` before vs. after a `union` gives very different results.

### Query 1
```kql
SecurityEvent 
| union SigninLogs
```

| Line                  | What it does                                                               |
| --------------------- | -------------------------------------------------------------------------- |
| `SecurityEvent`       | Start with the entire `SecurityEvent` table (within the query time range). |
| `\| union SigninLogs` | Append every row from `SigninLogs` underneath.                             |

**Result:** All rows from `SecurityEvent` + all rows from `SigninLogs`, stacked. Unique columns from each table are `null` on the other table's rows.

**Input:**

`SecurityEvent`:

| TimeGenerated | EventID | Account |
| ------------- | ------- | ------- |
| 09:00         | 4624    | Bob     |
| 09:05         | 4634    | Bob     |

`SigninLogs`:

| TimeGenerated | UserPrincipalName | ResultType |
| ------------- | ----------------- | ---------- |
| 09:10         | bob@contoso.com   | 0          |
| 09:15         | alice@contoso.com | 50126      |

**Output:**

| TimeGenerated | EventID | Account | UserPrincipalName | ResultType |
| ------------- | ------- | ------- | ----------------- | ---------- |
| 09:00         | 4624    | Bob     | null              | null       |
| 09:05         | 4634    | Bob     | null              | null       |
| 09:10         | null    | null    | bob@contoso.com   | 0          |
| 09:15         | null    | null    | alice@contoso.com | 50126      |

### Query 2
```kql
SecurityEvent 
| union SigninLogs  
| summarize count() 
| project count_
```

| Line                   | What it does                                                                                            |
| ---------------------- | ------------------------------------------------------------------------------------------------------- |
| `SecurityEvent`        | Start with all `SecurityEvent` rows.                                                                    |
| `\| union SigninLogs`  | Stack all `SigninLogs` rows underneath.                                                                 |
| `\| summarize count()` | Aggregates ALL rows in the combined table into **one row** (no `by`, so no grouping) — total row count. |
| `\| project count_`    | Displays the `count_` column (auto-named by `count()`).                                                 |

**Result:** One row, one column — grand total of both tables combined.

**Output (using same sample data):**

| count_ |
| ------ |
| 4      |

### Query 3
```kql
SecurityEvent 
| union (SigninLogs | summarize count()| project count_)
```

| Line                                                             | What it does                                                                                                                                                                          |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `SecurityEvent`                                                  | Start with all `SecurityEvent` rows — full detail, untouched.                                                                                                                         |
| `\| union ( SigninLogs \| summarize count() \| project count_ )` | The parentheses create a **sub-query** that runs independently: counts `SigninLogs` rows into one row (`count_`), THEN that single row is stacked onto the full `SecurityEvent` rows. |

**Result:** All original `SecurityEvent` rows + ONE extra row containing the `SigninLogs` total count.

**Output:**

| TimeGenerated | EventID | Account | count_ |
| ------------- | ------- | ------- | ------ |
| 09:00         | 4624    | Bob     | null   |
| 09:05         | 4634    | Bob     | null   |
| null          | null    | null    | 2      |

**Key lesson:** Same tables, same operators — but the **placement** of `summarize` (outside vs inside the union) completely changes the output shape:
- Query 1 → full raw combined rows
- Query 2 → one combined total
- Query 3 → full raw `SecurityEvent` rows + one summarized total row from `SigninLogs`

### Wildcard union
```kql
union Security* 
| summarize count() by Type
```

| Part                           | Meaning                                                                                                                                   |
| ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `union Security*`              | `*` = wildcard. Unions every table whose name starts with "Security" (e.g., `SecurityEvent`, `SecurityAlert`, `SecurityIncident`...).     |
| `\| summarize count() by Type` | `Type` = a **built-in hidden column** every table has, storing the source table's name. Groups/counts rows by which table they came from. |

**Output example:**

| Type          | count_ |
| ------------- | ------ |
| SecurityEvent | 1000   |
| SecurityAlert | 25     |

### `withsource` parameter
```kql
union withsource=SourceTable Security*
| summarize count() by SourceTable
```

| Part                                  | Meaning                                                                                                                                                                                                     |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `withsource=SourceTable`              | Creates a **new column you name yourself** (here called `SourceTable`, but the name is entirely arbitrary — you could call it `Origin`, `WhichTable`, anything) that labels which table each row came from. |
| `Security*`                           | Same wildcard as before.                                                                                                                                                                                    |
| `\| summarize count() by SourceTable` | Group/count by that custom column.                                                                                                                                                                          |

**IMPORTANT CLARIFICATION:** `SourceTable` is **not** a reserved/built-in KQL keyword — it has no special meaning to Kusto. You are naming a new column, exactly like naming `LogOnCount = count()`. Under the hood, `withsource=X` tells `union` to create a real, visible column named `X` and populate it with the same info the hidden `Type` column already holds.

**Why use `withsource` instead of just `Type`?**
1. **Clarity** — self-documenting name vs. the generic `Type`.
2. **Safety in complex/nested queries** — avoids ambiguity or collisions with other columns.
3. `withsource` columns behave reliably as normal columns everywhere (`project`, `summarize by`, `where`, etc.), whereas `Type` can behave inconsistently in some nested contexts.

**Output:**

| SourceTable   | count_ |
| ------------- | ------ |
| SecurityEvent | 1000   |
| SecurityAlert | 25     |

---

## 3. The `join` Operator

### Definition
`join` merges rows of two tables **side by side**, matching on specified column(s) — like a SQL JOIN or Excel VLOOKUP/INDEX-MATCH. Unlike `union`, it doesn't stack rows — it combines columns into wider rows where keys match.

### Syntax
```
LeftTable | join [JoinParameters] ( RightTable ) on Attributes
```

| Term               | Meaning                                                                                       |
| ------------------ | --------------------------------------------------------------------------------------------- |
| `LeftTable`        | The table/result **before** the `join` keyword — the "left" side.                             |
| `join`             | The operator.                                                                                 |
| `[JoinParameters]` | Optional settings, most importantly `kind=...` (determines what happens with unmatched rows). |
| `( RightTable )`   | The second table, in parentheses — the "right" side. Can be a raw table or a full sub-query.  |
| `on Attributes`    | The column(s) used to match rows between left and right.                                      |

### Worked example
```kql
SecurityEvent 
| where EventID == "4624" 
| summarize LogOnCount=count() by EventID, Account 
| project LogOnCount, Account 
| join kind = inner (
     SecurityEvent 
     | where EventID == "4634" 
     | summarize LogOffCount=count() by EventID, Account 
     | project LogOffCount, Account 
) on Account
```
**Goal:** For each account, compare logon count (Event ID 4624) vs logoff count (Event ID 4634) on one combined row.

**Left side, line by line:**

| Line                                                  | What it does                                                                                           |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `SecurityEvent`                                       | Start with the full table.                                                                             |
| `\| where EventID == "4624"`                          | Filter — keep only rows where `EventID == "4624"`. **4624 = "An account was successfully logged on."** |
| `\| summarize LogOnCount=count() by EventID, Account` | Group by `EventID` and `Account`, count rows per group, store result in a **new column** `LogOnCount`. |
| `\| project LogOnCount, Account`                      | Trim output to just these two columns.                                                                 |

Left table result:

| LogOnCount | Account |
| ---------- | ------- |
| 5          | Bob     |
| 2          | Alice   |

**Right side (sub-query), same pattern for logoffs:**

| Line                                                   | What it does                                               |
| ------------------------------------------------------ | ---------------------------------------------------------- |
| `SecurityEvent`                                        | Start over, independent sub-query.                         |
| `\| where EventID == "4634"`                           | Filter to logoffs. **4634 = "An account was logged off."** |
| `\| summarize LogOffCount=count() by EventID, Account` | Count logoffs per account → `LogOffCount`.                 |
| `\| project LogOffCount, Account`                      | Keep just these two columns.                               |

Right table result:

| LogOffCount | Account |
| ----------- | ------- |
| 4           | Bob     |
| 2           | Alice   |
| 1           | Charlie |

**The join:**

| Part                           | Meaning                                                      |
| ------------------------------ | ------------------------------------------------------------ |
| `\| join kind = inner ( ... )` | `kind=inner` = only keep accounts present on **both** sides. |
| `on Account`                   | Match rows where `Account` is equal in both tables.          |

**Output:**

| LogOnCount | Account | LogOffCount |
| ---------- | ------- | ----------- |
| 5          | Bob     | 4           |
| 2          | Alice   | 2           |

- Charlie is dropped — no matching logon record, and `kind=inner` only keeps matches from both sides.
- Output column order: left columns → join key → right columns.

---

## 4. `$left.` and `$right.` — Explained in Depth

### Core rule
`$left` / `$right` are just **labels meaning "which table this column comes from."** You need them in the `on` clause when:
1. Column names are **spelled differently** in each table, OR
2. You want to be explicit/safe even when names match.

Outside the `on` clause, if both tables have a same-named column that isn't the join key, Kusto auto-renames the right-side one with a `1` suffix (e.g., `TimeGenerated` → `TimeGenerated1`). It does **not** use `$left`/`$right` prefixes in the output — that prefix only applies inside the `on` clause itself.

### Example 1 — matching column names (shorthand, no `$left`/`$right` needed)
```kql
SecurityEvent
| join kind=inner (SigninLogs) on Account
```
Works only if **both tables** literally have a column called `Account`. Kusto assumes `on Account` means `$left.Account == $right.Account`.

### Example 2 — different column names (mandatory use of `$left`/`$right`)
```kql
SecurityEvent
| join kind=inner (SigninLogs) on $left.Account == $right.UserPrincipalName
```

| Part                       | Meaning                                                          |
| -------------------------- | ---------------------------------------------------------------- |
| `$left.Account`            | Column `Account` in the **left table** (`SecurityEvent`)         |
| `$right.UserPrincipalName` | Column `UserPrincipalName` in the **right table** (`SigninLogs`) |
| `==`                       | Match rows where these values are equal                          |

**Input:**

`SecurityEvent` (left):

| TimeGenerated | Account | EventID |
| ------------- | ------- | ------- |
| 09:00         | Bob     | 4624    |
| 09:10         | Alice   | 4624    |

`SigninLogs` (right):

| TimeGenerated | UserPrincipalName | ResultType |
| ------------- | ----------------- | ---------- |
| 09:05         | Bob               | 0          |
| 09:15         | Alice             | 50126      |

**Output:**

| TimeGenerated | Account | EventID | TimeGenerated1 | UserPrincipalName | ResultType |
|---|---|---|---|---|---|
| 09:00 | Bob | 4624 | 09:05 | Bob | 0 |
| 09:10 | Alice | 4624 | 09:15 | Alice | 50126 |

Note: `Account` and `UserPrincipalName` both remain as **separate columns** in the output — join uses them only to *match* rows, it doesn't merge them into one column. Also note `TimeGenerated` auto-renamed to `TimeGenerated1` for the right side since both tables shared that name.

### Example 3 — matching on multiple columns with different names
```kql
DeviceLogonEvents
| join kind=inner (
    SigninLogs
) on $left.AccountName == $right.UserPrincipalName, $left.DeviceName == $right.DeviceDetail
```
Chaining conditions with a comma — matches only if **both** conditions are true simultaneously. Reduces false-positive matches vs. matching on a single loose column.

### Example 4 — explicit form even when names match (equivalent, for clarity)
```kql
SecurityEvent
| join kind=inner (SigninLogs) on $left.Account == $right.Account
```
This is 100% identical to `on Account`. Some prefer always writing it out explicitly in detection rules for readability.

### Example 5 — realistic Sentinel detection use case
```kql
SigninLogs
| join kind=inner (
    ThreatIntelligenceIndicator
) on $left.IPAddress == $right.NetworkIP
```

| Part               | Meaning                                             |
| ------------------ | --------------------------------------------------- |
| `$left.IPAddress`  | IP column as named in `SigninLogs`                  |
| `$right.NetworkIP` | IP column as named in `ThreatIntelligenceIndicator` |

**Input:**

`SigninLogs` (left):

| UserPrincipalName | IPAddress |
|---|---|
| bob@contoso.com | 203.0.113.5 |
| alice@contoso.com | 198.51.100.9 |

`ThreatIntelligenceIndicator` (right):

| NetworkIP | ThreatType |
|---|---|
| 203.0.113.5 | Botnet |

**Output:**

| UserPrincipalName | IPAddress | NetworkIP | ThreatType |
|---|---|---|---|
| bob@contoso.com | 203.0.113.5 | 203.0.113.5 | Botnet |

Alice drops out (with `kind=inner`) — no threat intel match for her IP. This is exactly the filtering pattern used for real detections: "only show sign-ins from known-bad IPs."

### One-sentence summary
`$left.ColumnName` / `$right.ColumnName` = **"this specific column, from this specific table."** Required in the `on` clause only when the matching columns are named differently between the two tables (or when you want to be explicit even if names match).

---

## 5. Join Flavors (`kind=`)

Every `join` needs a flavor telling it what to do with rows that **don't** have a match on the other side.

| Flavor | Plain-English explanation | Output records |
|---|---|---|
| `kind=leftanti` / `kind=leftantisemi` | "Show only what's missing on the right." Useful for anomalies — e.g. accounts that logged on but never logged off. | All left rows with **no match** on the right (right columns excluded). |
| `kind=rightanti` / `kind=rightantisemi` | Flipped — "what's missing on the left?" e.g. logoffs with no matching logon. | All right rows with **no match** on the left. |
| *(unspecified)* / `kind=innerunique` | **Default** if `kind=` isn't typed. For each distinct left key value, only one left row is matched against the right. | One row per left key value, matched against all matching right rows. |
| `kind=leftsemi` | "Which left rows had a match?" A filter, not a merge — no right-side columns in output. | All left rows **that DO have** a match on the right. |
| `kind=rightsemi` | Same, flipped. | All right rows **that DO have** a match on the left. |
| `kind=inner` | Only rows existing on **both** sides, fully merged. | One row for every combination of matching left+right rows. |
| `kind=leftouter` | Keep everyone from the left, blanks for right if unmatched. | One row for every left row; unmatched → right columns = null. |
| `kind=rightouter` | Flipped — keep everyone from the right, blanks for left if unmatched. | One row for every right row; unmatched → left columns = null. |
| `kind=fullouter` | Keep everyone from both sides, no matter what. | One row for every row on either side; unmatched cells = null. |

### Practical security example (all flavors, same data)
Left (logons): Bob(5), Alice(2)
Right (logoffs): Bob(4), Alice(2), Charlie(1)

- **`kind=inner`** → Bob, Alice (Charlie dropped — no match either side)
- **`kind=leftouter`** → Bob, Alice (no extra left row to show, since only Bob & Alice exist on the left)
- **`kind=rightouter`** → Bob, Alice, **Charlie** (Charlie appears with `LogOnCount = null` → he logged off with no recorded logon — a hunting red flag)
- **`kind=fullouter`** → Bob, Alice, Charlie (union of everyone, nulls where no match)
- **`kind=leftanti`** → *(empty here — every left account has a right match)*
- **`kind=rightanti`** → **Charlie only** — accounts that logged off without ever logging on. Common pattern for detecting session hijacking, log gaps, or clock/timing anomalies.

**Exam gotcha:** The **default** join (no `kind=` specified) is `innerunique`, **not** `inner`.

---

## 6. Quick-Reference Cheat Sheet

| Concept | Keyword | What it does |
|---|---|---|
| Combine tables (stack rows) | `union` | Appends rows from multiple tables into one result |
| Show source table of unioned rows | `withsource=ColName` | Adds a custom-named column identifying origin table |
| Wildcard table matching | `TableName*` | Matches all tables whose name starts with that prefix |
| Combine tables (merge columns) | `join` | Matches rows by key column(s), merges their columns |
| Filter rows | `where` | Keeps only rows matching a condition |
| Aggregate/count rows | `summarize` | Collapses rows into summary stats, optionally grouped `by` a column |
| Select/rename columns | `project` | Chooses which columns appear in the output |
| Pipe | `\|` | Passes result of one step into the next — order matters! |
| Ambiguous column reference | `$left.Col` / `$right.Col` | Explicitly specifies which table's column to reference in `on` |

### Key takeaway
**`union` = stack rows. `join` = merge columns by matching key.** Where you place `summarize`, `project`, or a parenthesized sub-query relative to `union`/`join` changes your output. Always trace a KQL query top-to-bottom, one pipe at a time, asking "what does the table look like right now, at this line?"