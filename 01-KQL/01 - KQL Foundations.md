
## What is KQL?

KQL (Kusto Query Language) is the query language used to search and analyze security logs inside Microsoft Sentinel and Microsoft Defender XDR. You use it to:

- Search for malicious activity
- Create analytics rules (automated detections)
- Build workbooks (dashboards/visualizations)
- Perform threat hunting (proactive searching)

KQL is **read-only** — it can never delete, modify, or add data. You can never break anything by running a query.

---

## The Data Hierarchy

```
Database (the whole Log Analytics Workspace)
  └── Table (one type of log, e.g. SecurityEvent)
        └── Columns (fields inside the table, e.g. Account, EventID)
              └── Rows (individual events/records)
```

- **Database** = the entire workspace, contains everything
- **Table** = one specific log type (SecurityEvent = Windows security logs, SigninLogs = Entra ID logins)
- **Column** = a specific field (Account = who, EventID = what happened, Computer = which machine)
- **Row** = one single event record

---

## KQL Query Structure

A KQL query is a sequence of statements. Results are shaped like a table (rows and columns — like Excel).

### Basic structure:

```
TableName
| operator1 something
| operator2 something
| operator3 something
```

### The pipe `|` = "then"

Data flows left to right through pipes. Output of each step becomes input of next step. Think of it as the word "then" in English.

### Example:

```
SecurityEvent | where EventID == "4626" | summarize count() by Account | limit 10
```

Reading in plain English:

> "Go to SecurityEvent table, **then** keep only rows where EventID is 4626, **then** count how many times each Account appears, **then** show only 10 results."

### Three phases of a query:

- **Data** = the raw full table (all rows, all columns untouched)
- **Condition** = where you filter, analyze, and prepare (the investigation)
- **Evidence** = the final clean result you use to make a decision or create an alert

---

## The Demo Environment

- **URL:** https://aka.ms/lademo
- Free, no cost, just needs your Microsoft/Azure account
- Data is live and constantly changing — if a query returns no results, increase the time range
- **Fix for no results:** change time range dropdown from "Last 24 hours" to "Last 7 days" or "Last 30 days"

### Three sections of the query window:

- **Left panel** = Tables list — all available tables and their columns
- **Middle top** = Query editor — where you type your KQL
- **Bottom** = Results — output after clicking Run
- **Columns button** = lets you choose which columns to display in results (hide irrelevant ones)

---

## Operators

### 1. `search` Operator — search everywhere

`search` looks across all tables and all columns for a word or phrase. Like Ctrl+F on your entire log warehouse.

**When to use:** When you don't know which table or column contains what you're looking for.

**Downside:** Inefficient and slow because it searches everything.

```kql
search "err"
```

> Search every table for any row containing "err" anywhere (finds "error", "kernel", "stderr", etc.)

```kql
search in (SecurityEvent, SecurityAlert, A*) "err"
```

> Search for "err" but only in SecurityEvent, SecurityAlert, and any table starting with A

**Note:** `A*` — the `*` is a wildcard meaning "anything". So `A*` = any table name starting with A.

**Tip:** For broad searches, set Time range to "Last hour" to avoid timeout errors.

---

### 2. `where` Operator — filter rows

`where` keeps only rows that match your condition. Everything else is thrown away. Like a bouncer — only matching rows get through.

**Most important and most used operator in KQL.**

#### Symbols used in `where`:

|Symbol|Meaning|Example|
|---|---|---|
|`>`|greater than (more recent in time)|`TimeGenerated > ago(1d)`|
|`<`|less than (older in time)|`TimeGenerated < ago(7d)`|
|`==`|exactly equals (case-sensitive)|`EventID == "4624"`|
|`=~`|equals (case-insensitive)|`AccountType =~ "user"`|
|`!=`|not equal to|`EventID != 4688`|
|`and`|both conditions must be true|`condition1 and condition2`|
|`in`|matches any value from a list|`EventID in (4624, 4625)`|
|`contains`|text contains a substring|`Account contains "SQL"`|

#### `ago()` function:

Built-in function that goes back in time from right now.

- `ago(1h)` = 1 hour ago
- `ago(1d)` = 1 day ago (24 hours)
- `ago(7d)` = 7 days ago
- `ago(30d)` = 30 days ago
- `ago(1m)` = 1 minute ago

#### Examples:

```kql
SecurityEvent
| where TimeGenerated > ago(1d)
```

> All SecurityEvent rows from the last 24 hours

```kql
SecurityEvent
| where TimeGenerated > ago(1h) and EventID == "4624"
```

> Successful logins (EventID 4624) from the last hour. Both conditions must be true.

```kql
SecurityEvent
| where TimeGenerated > ago(1h)
| where EventID == 4624
| where AccountType =~ "user"
```

> Same as above but split across three where operators. Both approaches are valid. `=~` means case-insensitive — matches "user", "User", "USER".

```kql
SecurityEvent | where EventID in (4624, 4625)
```

> All login attempts — both successful (4624) and failed (4625). Useful for detecting brute force attacks.

#### Important EventIDs to remember:

|EventID|Meaning|
|---|---|
|4624|Successful logon|
|4625|Failed logon|
|4688|New process created (very noisy, often excluded)|

---

### 3. `let` Statement — name and reuse values

`let` creates a named variable or named query result you can reuse. Always ends with a semicolon `;`.

**Why use it:**

- Easier to read (names are descriptive)
- Easier to change (update value in one place)
- Breaks complex queries into manageable named pieces

#### Use 1: Simple variables

```kql
let timeOffset = 7d;
let discardEventId = 4688;
SecurityEvent
| where TimeGenerated > ago(timeOffset*2) and TimeGenerated < ago(timeOffset)
| where EventID != discardEventId
```

- `timeOffset = 7d` → variable holding 7 days
- `discardEventId = 4688` → variable holding EventID to exclude
- `timeOffset*2` = 14 days
- `TimeGenerated > ago(14d) and TimeGenerated < ago(7d)` = events between 14 days ago and 7 days ago (a historical window, not recent data)
- `EventID != discardEventId` = exclude process creation events (too noisy)

#### Use 2: Dynamic lists with `datatable`

```kql
let suspiciousAccounts = datatable(account: string) [
    @"\administrator", 
    @"NT AUTHORITY\SYSTEM"
];
SecurityEvent | where Account in (suspiciousAccounts)
```

- `datatable` = creates a mini custom table inside your query (not from real logs)
- `(account: string)` = the mini table has one column named `account`, data type is text
- `@"\administrator"` = the `@` symbol means treat `\` as a literal character, not an escape character
- Result: `suspiciousAccounts` is a list of two account names
- Then search SecurityEvent for any event where Account matches one of them

#### Use 3: Dynamic query results as a named table

```kql
let LowActivityAccounts =
    SecurityEvent 
    | summarize cnt = count() by Account 
    | where cnt < 1000;
LowActivityAccounts | where Account contains "SQL"
```

- `summarize cnt = count() by Account` = count how many events each Account has, name that count column `cnt`
- `where cnt < 1000` = keep only accounts with fewer than 1000 events (low activity accounts)
- The `;` ends the let statement
- `LowActivityAccounts | where Account contains "SQL"` = use the result like a real table, filter to SQL accounts only
- **Why useful:** Quiet SQL accounts doing unusual things = suspicious behavior

---

### 4. `extend` Operator — add a calculated column

`extend` creates a new column calculated from existing columns. Like adding a formula column in Excel. The new column is appended to the results.

```kql
SecurityEvent
| where ProcessName != "" and Process != ""
| extend StartDir = substring(ProcessName, 0, string_size(ProcessName)-string_size(Process))
```

- `ProcessName` = full file path, e.g. `C:\Windows\System32\cmd.exe`
- `Process` = filename only, e.g. `cmd.exe`
- `where ProcessName != "" and Process != ""` = filter out empty values first (otherwise the calculation below fails)
- `extend StartDir =` = create a new column called `StartDir`
- `substring(text, start, length)` = extract part of a string
    - First argument = text to extract from → `ProcessName`
    - Second argument = where to start → `0` (the beginning)
    - Third argument = how many characters → `string_size(ProcessName) - string_size(Process)`
- `string_size()` = counts characters in a string

**Example calculation:**

- `ProcessName` = `C:\Windows\System32\cmd.exe` (27 chars)
- `Process` = `cmd.exe` (7 chars)
- `27 - 7 = 20`
- `substring(..., 0, 20)` = `C:\Windows\System32\`
- So `StartDir` = the directory where the process was launched from

**Why useful:** Detecting processes running from suspicious locations (temp folders, desktop, downloads) instead of normal system directories.

---

### 5. `order by` Operator — sort results

`order by` sorts your results by one or more columns. Like sorting a column in Excel.

- `desc` = descending (Z→A, big→small) — **default if not specified**
- `asc` = ascending (A→Z, small→big)
- Multiple columns separated by commas — first column is primary sort, second is tiebreaker

```kql
SecurityEvent
| where ProcessName != "" and Process != ""
| extend StartDir = substring(ProcessName, 0, string_size(ProcessName)-string_size(Process))
| order by StartDir desc, Process asc
```

> Sort by StartDir descending (Z→A), and for rows with the same StartDir, sort by Process ascending (A→Z)

---

### 6. `project` Operators — control columns in output

Project operators control which columns appear in your results. Default = all columns (30-40 sometimes). Project = trim to what you need.

**Performance benefit:** Fewer columns = less data = faster query.

#### All project variations:

|Operator|What it does|
|---|---|
|`project`|Show ONLY the columns you specify|
|`project-away`|Hide specific columns, show everything else|
|`project-keep`|Keep specific columns (similar to project)|
|`project-rename`|Rename a column in the output|
|`project-reorder`|Change the order columns appear|

#### `project` — pick exactly which columns to show:

```kql
SecurityEvent
| project Computer, Account
```

> Show ONLY Computer and Account columns. Everything else hidden.

```kql
SecurityEvent
| where ProcessName != "" and Process != ""
| extend StartDir = substring(ProcessName, 0, string_size(ProcessName)-string_size(Process))
| order by StartDir desc, Process asc
| project Process, StartDir
```

> After all the filtering and calculating, show ONLY Process and StartDir in final results.

#### `project-away` — hide specific columns, keep the rest:

```kql
SecurityEvent
| where ProcessName != "" and Process != ""
| extend StartDir = substring(ProcessName, 0, string_size(ProcessName)-string_size(Process))
| order by StartDir desc, Process asc
| project-away ProcessName
```

> Remove the ProcessName column from results. We already extracted StartDir from it so it's now redundant clutter.

**When to use `project` vs `project-away`:**

- Want most columns but remove 1-2 → use `project-away`
- Want only a few specific columns → use `project`

#### `project-rename` — rename a column:

```kql
| project-rename LoginAccount = Account
```

> Rename the Account column to LoginAccount in results.

#### `project-reorder` — reorder columns:

```kql
| project-reorder TimeGenerated, Computer, Account
```

> Put these three columns first. Remaining columns still appear after them.

---

## Full Operator Summary

|Operator|Job|Analogy|
|---|---|---|
|`search`|Find a word anywhere in all tables|Ctrl+F on everything|
|`where`|Filter rows by condition|Bouncer at the door|
|`let`|Name a value or query for reuse|Creating a variable|
|`extend`|Add a calculated column|Excel formula in new column|
|`order by`|Sort results|Sorting a column in Excel|
|`project`|Show only specific columns|Hiding most columns in Excel|
|`project-away`|Hide specific columns|Deleting specific columns|
|`project-keep`|Keep specific columns|Same as project|
|`project-rename`|Rename a column|Renaming a column header|
|`project-reorder`|Change column order|Dragging columns in Excel|

---

## Key Rules to Always Remember

1. **Pipe `|` = "then"** — data flows left to right, each step feeds the next
2. **`==` is case-sensitive, `=~` is case-insensitive**
3. **`let` statements always end with `;`**
4. **`ago()` goes back in time from right now**
5. **`search` is slow (searches everything), `where` is fast (knows exactly where to look)**
6. **No results? Increase the time range — the data might just be older**
7. **KQL is read-only — you can never break or delete anything**