# 

## 1. Introduction

- Understanding how to work with **structured** and **unstructured** string data in KQL is the foundation for extracting usable data when building detections.
- **Structured string data** = data with predictable internal format inside a text field — e.g., JSON, key-value pairs. Looks like one long string, but has internal organization.
- **Unstructured string data** = free-form text with no fixed format — e.g., a log line like `"Event: NotifySliceRelease (resourceName=PipelineScheduler, ...)"`. Human-readable, but no schema — must be parsed manually via pattern matching.
- Many raw logs (Syslog, custom app logs, Azure Activity logs) dump everything into one big string column instead of separate columns. To build a detection, you must first **extract specific values** out of that string into their own usable columns.

---

## 2. Extract Data from Structured String Data

### Dynamic Fields

- In Log Analytics tables, columns have data types: `string`, `int`, `datetime`, `bool`, etc. One special type is **`dynamic`**.
- A `dynamic` column stores **structured data** (usually a JSON object or array) inside a single cell. KQL understands its internal structure natively (keys/values) — it's not just plain text you have to search through.

**Example raw dynamic field content:**
```json
{"eventCategory":"Autoscale","eventName":"GetOperationStatusResult","operationId":"xxxxxxxx-6a53-4aed-bab4-575642a10226","eventProperties":"{\"OldInstancesCount\":6,\"NewInstancesCount\":5}","eventDataId":"xxxxxxxx-efe3-43c2-8c86-cd84f70039d3","eventSubmissionTimestamp":"2020-11-30T04:06:17.0503722Z","resource":"ch-appfevmss-pri","resourceGroup":"CH-RETAILRG-PRI","resourceProviderValue":"MICROSOFT.COMPUTE","subscriptionId":"xxxxxxxx-7fde-4caf-8629-41dc15e3b352","activityStatusValue":"Succeeded"}
```
This whole block lives in ONE cell, in ONE column.

### Dot Notation

```kql
SigninLogs 
| extend OS = DeviceDetail.operatingSystem
```

| Part                                          | Meaning                                                                                                                                                                                                                                                                              |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `SigninLogs`                                  | Start with the sign-in logs table.                                                                                                                                                                                                                                                   |
| `\| extend OS = DeviceDetail.operatingSystem` | `extend` adds a **new computed column** (`OS`) without removing existing columns (unlike `project`, which replaces the whole column list). `DeviceDetail` is a `dynamic` column; `.operatingSystem` reaches into that JSON object and pulls the value under key `"operatingSystem"`. |

**Input** (`DeviceDetail` column content, one row):
```json
{"deviceId":"abc123","operatingSystem":"Windows 10","browser":"Chrome 118"}
```

**Output:**

| DeviceDetail (raw)                                                            | OS         |
| ----------------------------------------------------------------------------- | ---------- |
| `{"deviceId":"abc123","operatingSystem":"Windows 10","browser":"Chrome 118"}` | Windows 10 |

Dot notation `Column.Key` works like accessing a JSON property in JS/Python. KQL also supports bracket notation: `DeviceDetail["operatingSystem"]`.

### Full Example Query — Line by Line

```kql
SigninLogs 
| extend OS = DeviceDetail.operatingSystem, Browser = DeviceDetail.browser 
| extend StatusCode = tostring(Status.errorCode), StatusDetails = tostring(Status.additionalDetails) 
| extend Date = startofday(TimeGenerated) 
| summarize count() by Date, Identity, UserDisplayName, UserPrincipalName, IPAddress, ResultType, ResultDescription, StatusCode, StatusDetails 
| sort by Date
```

| Line                                                                                                                                              | What it does                                                                                                                                                                                                                                            |
| ------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `SigninLogs`                                                                                                                                      | Start with the table.                                                                                                                                                                                                                                   |
| `\| extend OS = DeviceDetail.operatingSystem, Browser = DeviceDetail.browser`                                                                     | Adds TWO new columns in one `extend` (comma-separated) — `OS` and `Browser`, both from the `DeviceDetail` dynamic field.                                                                                                                                |
| `\| extend StatusCode = tostring(Status.errorCode), StatusDetails = tostring(Status.additionalDetails)`                                           | `Status` is another dynamic field. `tostring()` explicitly converts the extracted value to plain text — dynamic sub-values can come out as a different internal type, so wrapping in `tostring()` guarantees clean text instead of a raw JSON fragment. |
| `\| extend Date = startofday(TimeGenerated)`                                                                                                      | `startofday()` rounds a `datetime` down to midnight of that day. Example: `2024-03-15 14:32:07` → `2024-03-15 00:00:00`. Used for grouping/counting events per day instead of per exact second.                                                         |
| `\| summarize count() by Date, Identity, UserDisplayName, UserPrincipalName, IPAddress, ResultType, ResultDescription, StatusCode, StatusDetails` | Groups rows by all these columns together, counts how many sign-in events match each unique combination.                                                                                                                                                |
| `\| sort by Date`                                                                                                                                 | Orders output rows by `Date` (ascending by default).                                                                                                                                                                                                    |

**Input** (2 raw `SigninLogs` rows, simplified):

| TimeGenerated    | DeviceDetail                                          | Status                                   | Identity | UserPrincipalName | IPAddress   | ResultType |
| ---------------- | ----------------------------------------------------- | ---------------------------------------- | -------- | ----------------- | ----------- | ---------- |
| 2024-03-15 09:00 | `{"operatingSystem":"Windows 10","browser":"Chrome"}` | `{"errorCode":0,"additionalDetails":""}` | Bob      | bob@contoso.com   | 203.0.113.5 | 0          |
| 2024-03-15 18:00 | `{"operatingSystem":"Windows 10","browser":"Chrome"}` | `{"errorCode":0,"additionalDetails":""}` | Bob      | bob@contoso.com   | 203.0.113.5 | 0          |

**Output:**

| Date       | Identity | UserPrincipalName | IPAddress   | ResultType | StatusCode | StatusDetails | count_ |
| ---------- | -------- | ----------------- | ----------- | ---------- | ---------- | ------------- | ------ |
| 2024-03-15 | Bob      | bob@contoso.com   | 203.0.113.5 | 0          | 0          | *(empty)*     | 2      |

Both rows collapse into ONE summary row because every grouping column matched, and `count_` = 2.

### JSON Functions

| Function | Description |
|---|---|
| `parse_json()` / `todynamic()` | Takes a **plain string** column that contains JSON text and converts it into an actual `dynamic` type, so you can use dot/bracket notation. These two functions are **synonyms** — identical behavior, different names. |
| `mv-expand` | Applied on a `dynamic` array or property bag column — **splits it into multiple rows**, one row per array element. Every other column in the original row gets **duplicated** across all new rows. `mv` = "multi-value." Easiest way to process JSON arrays. |
| `mv-apply` | Similar to `mv-expand`, but runs a **sub-query** against each element of the array and returns the combined results — useful for filtering/transforming each item before expanding. |

**Why `parse_json()` is needed even for "Dynamic"-looking data:** sometimes a column is *typed* as `string` even though its *content* is JSON text — e.g., `AuthenticationDetails` in `SigninLogs` is stored as plain string that *looks like* JSON. Must explicitly convert with `parse_json()` before using dot notation.

### JSON Query 1

```kql
SigninLogs 
| extend AuthDetails =  parse_json(AuthenticationDetails) 
| extend AuthMethod =  AuthDetails[0].authenticationMethod 
| extend AuthResult = AuthDetails[0].["authenticationStepResultDetail"] 
| project AuthMethod, AuthResult, AuthDetails 
```

| Line                                                                       | What it does                                                                                                                                                                                                                                 |
| -------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `SigninLogs`                                                               | Start with the table.                                                                                                                                                                                                                        |
| `\| extend AuthDetails = parse_json(AuthenticationDetails)`                | `AuthenticationDetails` is a plain string column containing JSON **array** text (e.g. `"[{...},{...}]"`). `parse_json()` converts that string into a real `dynamic` array so you can index into it.                                          |
| `\| extend AuthMethod = AuthDetails[0].authenticationMethod`               | `AuthDetails[0]` = the **first element** (index 0) of the array (zero-indexed). `.authenticationMethod` pulls that key out of that first element.                                                                                            |
| `\| extend AuthResult = AuthDetails[0].["authenticationStepResultDetail"]` | Same idea, using **bracket notation** `.["keyName"]` instead of dot notation. Required when the key name has characters dot notation can't handle (spaces, special chars). Functionally identical to `.authenticationStepResultDetail` here. |
| `\| project AuthMethod, AuthResult, AuthDetails`                           | Keep only these three columns.                                                                                                                                                                                                               |

**Input** (`AuthenticationDetails` raw content, one row):
```json
[{"authenticationMethod":"Password","authenticationStepResultDetail":"Success","succeeded":true}]
```

**Output:**

| AuthMethod | AuthResult | AuthDetails                                                                                         |
| ---------- | ---------- | --------------------------------------------------------------------------------------------------- |
| Password   | Success    | `[{"authenticationMethod":"Password","authenticationStepResultDetail":"Success","succeeded":true}]` |

### JSON Query 2 — `mv-expand`

```kql
SigninLogs 
| mv-expand AuthDetails = parse_json(AuthenticationDetails) 
| project AuthDetails
```

| Line                                                           | What it does                                                                                                                                                                                        |
| -------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `SigninLogs`                                                   | Start with the table.                                                                                                                                                                               |
| `\| mv-expand AuthDetails = parse_json(AuthenticationDetails)` | Parses the string into a JSON array, then **explodes** it so each array element becomes its own separate row, stored in new column `AuthDetails`. A row with 2 authentication steps becomes 2 rows. |
| `\| project AuthDetails`                                       | Keep just that column.                                                                                                                                                                              |

**Input** (one row, `AuthenticationDetails` = 2-element array):
```json
[
  {"authenticationMethod":"Password","succeeded":true},
  {"authenticationMethod":"MFA","succeeded":true}
]
```

**Output** (1 input row → 2 output rows):

| AuthDetails                                            |
| ------------------------------------------------------ |
| `{"authenticationMethod":"Password","succeeded":true}` |
| `{"authenticationMethod":"MFA","succeeded":true}`      |

Key behavior: **rows multiply** — one row becomes many, one per array item. Other original columns (like `TimeGenerated`, `Identity`) would be **duplicated** onto every new row if kept via `project`.

### JSON Query 3 — `mv-apply`

```kql
SigninLogs 
| mv-apply AuthDetails = parse_json(AuthenticationDetails) on
(where AuthDetails.authenticationMethod == "Password")
```

| Line                                                                     | What it does                                                                                                                                        |
| ------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `SigninLogs`                                                             | Start with the table.                                                                                                                               |
| `\| mv-apply AuthDetails = parse_json(AuthenticationDetails) on ( ... )` | Parses the string into a JSON array, then for **each element**, runs the sub-query in parentheses. Only elements satisfying the condition are kept. |
| `where AuthDetails.authenticationMethod == "Password"`                   | Sub-query: filter array elements to only those where `authenticationMethod == "Password"`.                                                          |

**Input** (same 2-element array):
```json
[
  {"authenticationMethod":"Password","succeeded":true},
  {"authenticationMethod":"MFA","succeeded":true}
]
```

**Output** (only matching element survives):

| AuthDetails                                            |
| ------------------------------------------------------ |
| `{"authenticationMethod":"Password","succeeded":true}` |

**`mv-expand` vs `mv-apply`:** `mv-expand` always gives every array element (needs a separate `where` after to filter). `mv-apply` filters/transforms each element **as part of the expansion itself**, in one step — more efficient/flexible (can also do aggregations inside the `on (...)` sub-query, not just `where`).

---

## 3. Integrate External Data

### The `externaldata` Operator

- Returns a table whose **schema is defined in the query itself**, and whose **data is read from an external storage artifact** (e.g., a blob in Azure Blob Storage or Azure Data Lake Storage file).
- Lets you pull in data that lives **outside** Sentinel entirely — e.g., a text/CSV file in blob storage — and treat it like a normal KQL table you can query, filter, or join against.
- **Real-world use case:** maintaining a text file of known-bad IPs or watchlisted usernames in blob storage (updated by a separate process), and always checking Sentinel queries against the *latest* version without manually pasting the list into KQL.

### Syntax
```
externaldata ( ColumnName : ColumnType [, ...] )
  [ StorageConnectionString [, ...] ]
  [with ( PropertyName = PropertyValue [, ...] )]
```

| Part                                               | Meaning                                                                                                                                                                                                                            |
| -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `externaldata ( ColumnName : ColumnType [, ...] )` | Defines what columns/schema the external data should be interpreted as — the external file has no built-in schema KQL understands automatically. E.g. `(UserID:string)` = "treat this as one column called `UserID`, type string." |
| `[ StorageConnectionString [, ...] ]`              | URL(s)/connection strings pointing to where the data lives (e.g. a blob URL + SAS token = Shared Access Signature, a temporary secure access key).                                                                                 |
| `[with ( PropertyName = PropertyValue [, ...] )]`  | Optional extra settings on how to interpret the file.                                                                                                                                                                              |

### Properties table

| Property | Type | Description |
|---|---|---|
| `format` | string | Data format (CSV, JSON, TSV, etc). If omitted, KQL guesses from file extension, defaults to CSV. |
| `ignoreFirstRecord` | bool | If `true`, skips the first line — useful for CSV files with header rows. |
| `ingestionMapping` | string | Advanced mapping of source file columns to result columns. |

### Example (with detailed execution order)

```kql
Users
| where UserID in ((externaldata (UserID:string) [
    @"https://storageaccount.blob.core.windows.net/storagecontainer/users.txt" 
      h@"?...SAS..." // Secret token needed to access the blob
    ]))
| ...
```

**IMPORTANT CLARIFICATION:** `Users` here is **NOT** a real built-in Microsoft Sentinel table (real built-in tables are things like `SecurityEvent`, `SigninLogs`, `AzureActivity`). It's a **generic placeholder name** — like `foo` in a programming tutorial. Mentally substitute it with whatever real table you're working with (e.g. `IdentityInfo`, or a custom table).

**Execution order — KQL evaluates the innermost parentheses FIRST:**

**Step A (innermost, runs first):** `externaldata` reaches out to the blob URL, downloads `users.txt`, reads it per the schema `(UserID:string)` — treats every line of the file as one value in a column called `UserID`.

`users.txt` content:
```
alice@contoso.com
bob@contoso.com
```
Produces this virtual table:

| UserID            |
| ----------------- |
| alice@contoso.com |
| bob@contoso.com   |

**Step B:** That virtual table feeds into `in (...)` — KQL flattens the single column into a simple value list: `("alice@contoso.com", "bob@contoso.com")`.

**Step C (outer, runs last):** `Users | where UserID in (that list)` — keeps only rows from your real table whose `UserID` matches one of the values pulled from the file.

**Conceptually equivalent to:**
```kql
Users
| where UserID in ("alice@contoso.com", "bob@contoso.com")
```
— except the list is pulled live from the file instead of hardcoded.

**Full worked example using a real table name (`IdentityInfo`):**

`IdentityInfo`:

| AccountUPN          | Department |
| ------------------- | ---------- |
| alice@contoso.com   | Finance    |
| charlie@contoso.com | IT         |

```kql
IdentityInfo
| where AccountUPN in ((externaldata (UserID:string) [
    @"https://storageaccount.blob.core.windows.net/storagecontainer/users.txt" 
      h@"?...SAS..."
    ]))
```

**Output:**

| AccountUPN        | Department |
| ----------------- | ---------- |
| alice@contoso.com | Finance    |

Charlie is filtered out — his UPN isn't in the external file. Common real pattern: VIP accounts, watchlisted usernames, known-compromised accounts kept in an externally-updated file.

**Syntax note:** `h@"?...SAS..."` — the `h@""` prefix marks that string literal as a **secret** (the SAS token query string). KQL masks it in query history/logs so it doesn't leak in screenshots or shared queries. `?...SAS...` is a placeholder for a real token like `?sv=2022-11-02&ss=b&srt=o&sp=r&se=...&sig=...`.

*(Note: this specific example is not runnable in the MS Learn demo/training environment — it requires a real external blob URL/SAS token.)*

---

## 4. Create Parsers with Functions

### The Concept

- Parsers are functions that define a **virtual table** with already-parsed unstructured string fields (e.g. Syslog data).
- Instead of retyping a long/complex KQL query every time, **save a query as a reusable function**. Once saved, call it by name — just like referencing a table — and it runs the underlying query fresh each time.

### How to Create One (workflow)

1. Write your query in the Logs window.
2. Click **Save**.
3. Enter a **Name**.
4. From the dropdown, choose **Save As Function** (instead of saving as a plain query).
5. That name now behaves like a **virtual table** referenceable in any future query.

### Example

**Step 1 — save this as a function named `PrivLogins`:**
```kql
SecurityEvent
| where EventID == 4672 and AccountType == 'User'
```
- `EventID == 4672` = Windows Security Event ID for **"Special privileges assigned to new logon"** (someone logged in with elevated/admin rights).
- `AccountType == 'User'` filters to standard user accounts (excludes system/service accounts).

**Step 2 — from then on, anywhere in Sentinel:**
```kql
PrivLogins
```
Behaves exactly as if you typed the full underlying query — runs live using current data.

**Build on top of it like a normal table:**
```kql
PrivLogins
| summarize count() by Account
```
Reuses saved logic without rewriting it — shorter, more maintainable queries, and consistency across a team (everyone references the same definition of "privileged login").

### ASIM Parsers

- **ASIM = Advanced Security Information Model.** Pre-built functions (built by Microsoft) that normalize data from **different log sources** into a **common schema**.
- Follow the exact same saved-function mechanism as above, but:
  - Maintained by **Microsoft**, not built by you.
  - Deployed at the **workspace level** — automatically available across the whole Sentinel workspace, not just private to one person's saved queries.
- **The problem ASIM solves:** different vendors name columns differently (source IP might be `SrcIP`, `SourceAddress`, or `src_ip` depending on product). Without normalization, you'd need a separate KQL query per vendor to search for the same thing.
- **ASIM parsers** map each vendor's raw, differently-named data into a **standardized set of column names**. Write ONE query against the normalized view, and it works across all supported log sources.

### ASIM Naming Pattern

- **Unifying parser** — `imCategory` (e.g. `imAuthentication`, `imNetworkSession`, `imDns`, `imWebSession`) — automatically combines data from **every supported connected source**.
- **Source-specific parser** — `vimCategorySourceName` (e.g. `vimAuthenticationAzureAD`) — only that one specific source, in ASIM's normalized column format.

### ASIM Example — `imAuthentication`

Without ASIM: querying "all failed logins across Azure AD AND on-prem Windows AND VPN" requires **three separate queries** with three different column-naming schemes.

**With ASIM, ONE query:**
```kql
imAuthentication
| where EventResult == "Failure"
| summarize FailedAttempts = count() by TargetUsername, SrcIpAddr
| where FailedAttempts > 5
```

| Line | What it does |
|---|---|
| `imAuthentication` | Calls the ASIM authentication parser — a Microsoft-built saved function that internally unions/normalizes authentication events from **every ASIM-supported connected source** (Azure AD sign-ins, Windows security logs, Okta, Duo, VPN logs, etc.). |
| `\| where EventResult == "Failure"` | `EventResult` is a **normalized column name** — regardless of the source's original naming (`ResultType`, `outcome`, `status`, etc.), ASIM maps it to one consistent name/value set (`"Success"`/`"Failure"`). |
| `\| summarize FailedAttempts = count() by TargetUsername, SrcIpAddr` | `TargetUsername` and `SrcIpAddr` are likewise normalized names — count failed attempts grouped by user and source IP. |
| `\| where FailedAttempts > 5` | Filter to user/IP combos with more than 5 failures — classic brute-force detection pattern. |

**Input** (two different underlying log sources, both feeding `imAuthentication`):

Source 1 — Azure AD (`SigninLogs` real columns): `UserPrincipalName`, `ResultType`, `IPAddress`
Source 2 — On-prem Windows (`SecurityEvent` real columns): `Account`, `EventID` (4625 = failed logon), `IpAddress`

ASIM internally normalizes both into the same shape:

| TargetUsername | EventResult | SrcIpAddr | EventProduct |
|---|---|---|---|
| bob@contoso.com | Failure | 203.0.113.5 | Azure AD |
| bob@contoso.com | Failure | 203.0.113.5 | Azure AD |
| BOB | Failure | 203.0.113.5 | Windows |
| BOB | Failure | 203.0.113.5 | Windows |
| BOB | Failure | 203.0.113.5 | Windows |
| BOB | Failure | 203.0.113.5 | Windows |
| BOB | Failure | 203.0.113.5 | Windows |

*(ASIM also normalizes username casing/format via identity mapping in reality — conceptually shown here as different raw sources producing the same output shape.)*

**Output of the full query:**

| TargetUsername | SrcIpAddr | FailedAttempts |
|---|---|---|
| bob@contoso.com | 203.0.113.5 | 7 |

One clean detection query catches failed logins regardless of which product generated them.

**Why this matters:** if the org adds a new log source later (e.g. a new VPN product) and that vendor has an ASIM parser available, **existing `imAuthentication`-based detection rules automatically start covering the new source too** — no rewriting needed.

---

## Quick-Reference Cheat Sheet

| Concept | Keyword/Function | What it does |
|---|---|---|
| Add a computed column (keep existing ones) | `extend` | Adds new column(s) without dropping the rest |
| Access a key inside a dynamic/JSON column | `.KeyName` or `.["KeyName"]` | Dot or bracket notation to reach into a JSON object |
| Convert a value to plain text | `tostring()` | Forces a value (often from dynamic data) into string type |
| Round a datetime down to midnight | `startofday()` | Truncates a timestamp to just its calendar day |
| Convert a JSON-looking string into real dynamic data | `parse_json()` / `todynamic()` | Synonyms — turn string into usable dynamic type |
| Split a JSON array into multiple rows | `mv-expand` | One row per array element; other columns duplicated |
| Filter/transform each array element during expansion | `mv-apply` | Runs a sub-query per array element |
| Pull in data from outside Sentinel | `externaldata` | Query a file (e.g. blob storage) as if it were a table |
| Reusable saved query | Save As Function | Turns a query into a callable virtual table by name |
| Microsoft's pre-built normalized parsers | ASIM parsers | Standardize column names across different vendor log sources |
| ASIM unifying parser naming | `imCategory` (e.g. `imAuthentication`) | Combines all supported connected sources for that category |
| ASIM source-specific parser naming | `vimCategorySourceName` | Normalized data from one specific source only |

### Key Takeaway
This module is about **getting usable columns out of messy string data** — JSON (`dynamic` fields, `parse_json`, `mv-expand`/`mv-apply`), free-form log text (`extract`/`parse`), or entirely external files (`externaldata`) — and then **saving that parsing logic as a reusable function**, so you (or ASIM, at Microsoft's scale) don't have to rewrite it every time. If "Save As Function" is greyed out, it's almost always a sandbox/permissions limitation, not a query error.