## What This Module Is About

Two new powerful operators:

- **`summarize`** — calculating and grouping data (counting, averaging, finding max/min, detecting patterns)
- **`render`** — turning results into a visual chart

Together these are how a SOC analyst detects patterns like "this account had 500 failed logins in the last hour" and visualizes them as graphs instead of raw rows.

---

## Why `summarize` Matters for Security

You cannot detect patterns like "Account with over 10 failed logons in the past hour" using just `where`. You need to:

1. Count how many failed logons each account had
2. Check if that count exceeds 10
3. If yes — fire an alert

That counting and grouping is exactly what `summarize` does. It is the engine behind almost every detection rule in Sentinel.

---

## The `summarize` Operator

### What is it?

Takes a big table of rows and collapses them into grouped calculations. Instead of seeing 10,000 individual rows, you see a summary like "Account A had 500 events, Account B had 23 events."

Think of it like a **pivot table in Excel** — take raw data and calculate totals, counts, averages grouped by category.

### Basic structure:

```
| summarize calculation by groupingColumn
```

- `calculation` = what to calculate (count, average, max, etc.)
- `by groupingColumn` = group the calculation by this column

---

### Query 1: `summarize by Activity` — unique list

```kql
SecurityEvent | summarize by Activity
```

**Input:**

|TimeGenerated|Account|EventID|Activity|Computer|
|---|---|---|---|---|
|2024-01-15 09:00|john|4624|Logon|SERVER01|
|2024-01-15 09:05|mary|4625|Logon Failed|SERVER01|
|2024-01-15 09:10|john|4624|Logon|SERVER02|
|2024-01-15 09:15|admin|4688|Process Created|SERVER01|
|2024-01-15 09:20|mary|4624|Logon|SERVER03|
|2024-01-15 09:25|john|4625|Logon Failed|SERVER02|
|2024-01-15 09:30|admin|4634|Logoff|SERVER01|

**Output:**

|Activity|
|---|
|Logon|
|Logon Failed|
|Process Created|
|Logoff|

**What happened:** All rows collapsed into just a unique list of activity names. Duplicates removed automatically. Useful for exploring what values exist in a column before writing more specific queries.

---

### Query 2: `summarize count() by Process, Computer`

```kql
SecurityEvent
| where EventID == "4688"
| summarize count() by Process, Computer
```

**Input** (after where filter — only EventID 4688 rows):

|TimeGenerated|Account|EventID|Process|Computer|
|---|---|---|---|---|
|2024-01-15 09:00|john|4688|cmd.exe|SERVER01|
|2024-01-15 09:05|admin|4688|cmd.exe|SERVER01|
|2024-01-15 09:10|mary|4688|powershell.exe|SERVER01|
|2024-01-15 09:15|john|4688|cmd.exe|SERVER02|
|2024-01-15 09:20|admin|4688|powershell.exe|SERVER02|
|2024-01-15 09:25|admin|4688|powershell.exe|SERVER02|
|2024-01-15 09:30|john|4688|cmd.exe|SERVER01|

**Output:**

|Process|Computer|count_|
|---|---|---|
|cmd.exe|SERVER01|3|
|powershell.exe|SERVER01|1|
|cmd.exe|SERVER02|1|
|powershell.exe|SERVER02|2|

**What happened:** Every unique combination of Process + Computer gets its own row with a count. Immediately tells you how many times each process ran on each computer. Useful for spotting unusual process activity.

---

## Naming Your Calculated Column

By default `count()` names the column `count_`. You can rename it by writing `yourname=function()` before the function.

### Query: `cnt=count()`

```kql
SecurityEvent
| where TimeGenerated > ago(1h)
| where EventID == 4624
| summarize cnt=count() by AccountType, Computer
```

**Input** (after both where filters):

|TimeGenerated|Account|AccountType|EventID|Computer|
|---|---|---|---|---|
|2024-01-15 09:00|john|User|4624|SERVER01|
|2024-01-15 09:10|admin|Administrator|4624|SERVER01|
|2024-01-15 09:15|mary|User|4624|SERVER01|
|2024-01-15 09:20|john|User|4624|SERVER02|
|2024-01-15 09:25|admin|Administrator|4624|SERVER02|
|2024-01-15 09:30|mary|User|4624|SERVER01|

**Output:**

|AccountType|Computer|cnt|
|---|---|---|
|User|SERVER01|3|
|Administrator|SERVER01|1|
|User|SERVER02|1|
|Administrator|SERVER02|1|

**What happened:** Counted successful logins per account type per computer. Column named `cnt` instead of default `count_`.

---

## All Aggregate Functions

### 1. `count()` and `countif()`

**`count()`** = count ALL rows in the group

```kql
SecurityEvent
| summarize count() by Account
```

**Input:**

|TimeGenerated|Account|EventID|
|---|---|---|
|2024-01-15 09:00|john|4624|
|2024-01-15 09:05|john|4625|
|2024-01-15 09:10|mary|4624|
|2024-01-15 09:15|john|4688|
|2024-01-15 09:20|mary|4625|
|2024-01-15 09:25|admin|4624|

**Output:**

|Account|count_|
|---|---|
|john|3|
|mary|2|
|admin|1|

---

**`countif(condition)`** = count only rows matching the condition

```kql
SecurityEvent
| summarize failedLogins=countif(EventID == 4625) by Account
```

**Same input.**

**Output:**

|Account|failedLogins|
|---|---|
|john|1|
|mary|1|
|admin|0|

**Security use case:** Detect accounts with many failed logins → brute force detection.

---

### 2. `dcount()` and `dcountif()`

**`dcount(column)`** = count only UNIQUE values — ignores duplicates. "d" stands for "distinct."

```kql
SecurityEvent
| summarize dcount(Computer) by Account
```

**Input:**

|TimeGenerated|Account|EventID|Computer|
|---|---|---|---|
|2024-01-15 09:00|john|4624|SERVER01|
|2024-01-15 09:05|john|4624|SERVER02|
|2024-01-15 09:10|john|4624|SERVER01|
|2024-01-15 09:15|mary|4624|SERVER01|
|2024-01-15 09:20|mary|4624|SERVER01|
|2024-01-15 09:25|admin|4624|SERVER03|

**Output:**

|Account|dcount_Computer|
|---|---|
|john|2|
|mary|1|
|admin|1|

**What happened:** john logged into SERVER01 twice and SERVER02 once — only 2 UNIQUE computers. mary logged into SERVER01 twice — only 1 unique computer.

**Security use case:** Account logging into many different computers in short time = possible lateral movement attack.

---

**`dcountif(column, condition)`** = count unique values only where condition is true

```kql
SecurityEvent
| summarize uniqueFailedComputers=dcountif(Computer, EventID == 4625) by Account
```

**Input:**

|TimeGenerated|Account|EventID|Computer|
|---|---|---|---|
|2024-01-15 09:00|john|4624|SERVER01|
|2024-01-15 09:05|john|4625|SERVER02|
|2024-01-15 09:10|john|4625|SERVER03|
|2024-01-15 09:15|john|4625|SERVER02|
|2024-01-15 09:20|mary|4625|SERVER01|
|2024-01-15 09:25|mary|4624|SERVER02|

**Output:**

|Account|uniqueFailedComputers|
|---|---|
|john|2|
|mary|1|

**What happened:** Only counted unique computers where a FAILED login (4625) happened. john's successful login on SERVER01 was completely ignored. SERVER02 counted once despite appearing twice for john.

**Security use case:** Account failing to log in across multiple machines = possible credential stuffing attack.

---

### 3. `avg()` and `avgif()`

**`avg(column)`** = mathematical average of a column's values

```kql
SecurityEvent
| summarize avg(Duration) by Computer
```

**Input:**

|TimeGenerated|Account|Computer|Duration|
|---|---|---|---|
|2024-01-15 09:00|john|SERVER01|30|
|2024-01-15 09:30|mary|SERVER01|60|
|2024-01-15 10:00|admin|SERVER01|90|
|2024-01-15 09:00|john|SERVER02|20|
|2024-01-15 10:00|mary|SERVER02|40|

**Output:**

|Computer|avg_Duration|
|---|---|
|SERVER01|60|
|SERVER02|30|

**What happened:** SERVER01 = (30+60+90)/3 = 60 minutes average. SERVER02 = (20+40)/2 = 30 minutes average.

**Security use case:** Session lasting 4 hours when average is 60 minutes = anomaly worth investigating.

---

**`avgif(column, condition)`** = average only where condition is true

```kql
SecurityEvent
| summarize avgNormalUserDuration=avgif(Duration, AccountType == "User") by Computer
```

**Input:**

|TimeGenerated|Account|AccountType|Computer|Duration|
|---|---|---|---|---|
|2024-01-15 09:00|john|User|SERVER01|30|
|2024-01-15 09:30|mary|User|SERVER01|60|
|2024-01-15 10:00|admin|Administrator|SERVER01|90|
|2024-01-15 09:00|john|User|SERVER02|20|
|2024-01-15 10:00|mary|User|SERVER02|40|

**Output:**

|Computer|avgNormalUserDuration|
|---|---|
|SERVER01|45|
|SERVER02|30|

**What happened:** SERVER01 calculated only for User accounts (john=30, mary=60). Admin's 90 minutes excluded. (30+60)/2 = 45.

**Security use case:** Compare normal user vs admin session durations separately to detect unusual admin behavior.

---

### 4. `max()` and `maxif()`

**`max(column)`** = find the single highest value in the group

```kql
SecurityEvent
| summarize max(Duration) by Account
```

**Input:**

|TimeGenerated|Account|Computer|Duration|
|---|---|---|---|
|2024-01-15 09:00|john|SERVER01|30|
|2024-01-15 10:00|john|SERVER02|120|
|2024-01-15 11:00|john|SERVER03|45|
|2024-01-15 09:00|mary|SERVER01|60|
|2024-01-15 10:00|mary|SERVER02|25|

**Output:**

|Account|max_Duration|
|---|---|
|john|120|
|mary|60|

**What happened:** john's longest session = 120 minutes. mary's longest = 60 minutes.

**Security use case:** Session lasting 8 hours when max is normally 2 hours = suspicious.

---

**`maxif(column, condition)`** = maximum only where condition is true

```kql
SecurityEvent
| summarize maxFailedAttempts=maxif(FailedAttempts, EventID == 4625) by Account
```

**Input:**

|TimeGenerated|Account|EventID|FailedAttempts|
|---|---|---|---|
|2024-01-15 09:00|john|4625|3|
|2024-01-15 09:05|john|4624|0|
|2024-01-15 09:10|john|4625|7|
|2024-01-15 09:15|mary|4625|2|
|2024-01-15 09:20|mary|4624|0|

**Output:**

|Account|maxFailedAttempts|
|---|---|
|john|7|
|mary|2|

**What happened:** Maximum FailedAttempts only for failed login rows (4625). john's successful login row with 0 attempts was ignored.

---

### 5. `min()` and `minif()`

**`min(column)`** = find the single lowest value in the group

```kql
SecurityEvent
| summarize firstSeen=min(TimeGenerated) by Account
```

**Input:**

|TimeGenerated|Account|EventID|
|---|---|---|
|2024-01-10 08:00|john|4624|
|2024-01-12 14:00|john|4688|
|2024-01-15 09:00|john|4625|
|2024-01-11 10:00|mary|4624|
|2024-01-14 16:00|mary|4625|

**Output:**

|Account|firstSeen|
|---|---|
|john|2024-01-10 08:00|
|mary|2024-01-11 10:00|

**What happened:** Earliest TimeGenerated per account. john first appeared January 10th, mary January 11th.

**Security use case:** "When did this account first appear in logs?" — detecting new accounts that shouldn't exist.

---

**`minif(column, condition)`** = minimum only where condition is true

```kql
SecurityEvent
| summarize firstFailedLogin=minif(TimeGenerated, EventID == 4625) by Account
```

**Same input.**

**Output:**

|Account|firstFailedLogin|
|---|---|
|john|2024-01-15 09:00|
|mary|2024-01-14 16:00|

**What happened:** Earliest FAILED login only. Successful logins and other events ignored.

**Security use case:** "When did this account start failing logins?" — identifies the start of an attack pattern.

---

### 6. `percentile()`

**`percentile(column, N)`** = value below which N% of all values fall. Used to define "normal" dynamically.

```kql
SecurityEvent
| summarize percentile(Duration, 95) by Computer
```

**Input:**

|TimeGenerated|Account|Computer|Duration|
|---|---|---|---|
|2024-01-15 09:00|john|SERVER01|10|
|2024-01-15 09:30|mary|SERVER01|20|
|2024-01-15 10:00|admin|SERVER01|25|
|2024-01-15 10:30|john|SERVER01|30|
|2024-01-15 11:00|mary|SERVER01|35|
|2024-01-15 11:30|admin|SERVER01|40|
|2024-01-15 12:00|john|SERVER01|45|
|2024-01-15 12:30|mary|SERVER01|50|
|2024-01-15 13:00|admin|SERVER01|55|
|2024-01-15 13:30|john|SERVER01|200|

**Output:**

|Computer|percentile_Duration_95|
|---|---|
|SERVER01|200|

**What happened:** 95% of sessions were shorter than 200 minutes. The extreme outlier (200) sits at the 95th percentile. 9 out of 10 sessions were between 10-55 minutes.

**Security use case:** If a new session's duration exceeds the 95th percentile it is abnormally long. Much smarter than a fixed threshold — the threshold adapts to your actual environment's behavior.

---

### 7. `stdev()` and `stdevif()`

**`stdev(column)`** = measures how spread out / inconsistent values are from the average.

- **Low stdev** = values are consistent and predictable (normal behavior)
- **High stdev** = values vary wildly (inconsistent / anomalous behavior)

```kql
SecurityEvent
| summarize stdev(Duration) by Computer
```

**Input:**

|TimeGenerated|Account|Computer|Duration|
|---|---|---|---|
|2024-01-15 09:00|john|SERVER01|58|
|2024-01-15 09:30|mary|SERVER01|60|
|2024-01-15 10:00|admin|SERVER01|62|
|2024-01-15 09:00|john|SERVER02|5|
|2024-01-15 09:30|mary|SERVER02|60|
|2024-01-15 10:00|admin|SERVER02|180|

**Output:**

|Computer|stdev_Duration|
|---|---|
|SERVER01|2|
|SERVER02|88.5|

**What happened:**

- SERVER01: sessions were 58, 60, 62 — barely different from each other. Low stdev (2) = stable predictable behavior
- SERVER02: sessions were 5, 60, 180 — wildly different. High stdev (88.5) = unpredictable behavior

**Security use case:** A legitimate user has predictable login patterns. An attacker using a stolen account behaves differently from the real owner — high stdev on login times or session durations signals possible account compromise.

---

**`stdevif(column, condition)`** = standard deviation only where condition is true

```kql
SecurityEvent
| summarize stdevFailedDuration=stdevif(Duration, EventID == 4625) by Computer
```

**Security use case:** Very consistent timing between failed login attempts (low stdev) = automated attack tool, not a human. Humans are inconsistent — tools are precise.

---

### 8. `sum()` and `sumif()`

**`sum(column)`** = add up all values in the group

```kql
SecurityEvent
| summarize totalBytes=sum(BytesTransferred) by Account
```

**Input:**

|TimeGenerated|Account|EventID|BytesTransferred|
|---|---|---|---|
|2024-01-15 09:00|john|4624|500|
|2024-01-15 09:30|john|4624|1200|
|2024-01-15 10:00|john|4624|300|
|2024-01-15 09:00|mary|4624|800|
|2024-01-15 10:00|mary|4624|400|
|2024-01-15 09:00|admin|4624|50000|

**Output:**

|Account|totalBytes|
|---|---|
|john|2000|
|mary|1200|
|admin|50000|

**What happened:** john = 500+1200+300 = 2000. mary = 800+400 = 1200. admin = 50000.

**Security use case:** admin transferred 50,000 bytes — 25x more than anyone else. Classic data exfiltration detection. Summing data transfer per account is one of the most common exfiltration detection techniques.

---

**`sumif(column, condition)`** = sum only where condition is true

```kql
SecurityEvent
| summarize suspiciousBytes=sumif(BytesTransferred, TimeGenerated > ago(1h)) by Account
```

**Security use case:** admin transferring 50,000 bytes in just the last hour is far more alarming than 50,000 over a whole month.

---

### 9. `variance()` and `varianceif()`

**`variance(column)`** = mathematically similar to stdev but squared. Less commonly used directly in daily SOC work.

```kql
SecurityEvent
| summarize variance(Duration) by Computer
```

**Input (same as stdev example):**

|Computer|Duration values|
|---|---|
|SERVER01|58, 60, 62|
|SERVER02|5, 60, 180|

**Output:**

|Computer|variance_Duration|
|---|---|
|SERVER01|4|
|SERVER02|7832.3|

**What happened:** Variance = stdev squared. SERVER01 stdev was 2, so variance = 2² = 4. SERVER02 stdev was ~88.5, so variance = 88.5² ≈ 7832.

**Difference between stdev and variance:**

- Both measure how spread out values are
- stdev is in the same units as your data (minutes, bytes) — easier to interpret
- variance is in squared units — harder to interpret directly
- For daily security analysis, `stdev` is more useful and readable than `variance`

---

**`varianceif(column, condition)`** = variance only where condition is true. Rarely used directly in day-to-day security queries.

---

## `dcount` Real-World Example — Password Spray Detection

```kql
let timeframe = 30d;
let threshold = 1;
SigninLogs
| where TimeGenerated >= ago(timeframe)
| where ResultDescription has "Invalid password"
| summarize applicationCount = dcount(AppDisplayName) by UserPrincipalName, IPAddress
| where applicationCount >= threshold
```

### Every line explained:

**`let timeframe = 30d;`** = variable: look back 30 days

**`let threshold = 1;`** = variable: minimum number of apps that triggers suspicion. In real life set to 3 or 5.

**`SigninLogs`** = Entra ID table — contains every cloud login attempt to Microsoft 365, Azure, or any connected app. Different from SecurityEvent (which is Windows on-prem logs).

**`| where TimeGenerated >= ago(timeframe)`** = last 30 days only. `>=` means greater than OR equal to — includes events from exactly 30 days ago.

**`| where ResultDescription has "Invalid password"`**

- `has` = checks if text contains a whole word (faster than `contains` for whole words)
- Keeps only failed login attempts due to wrong password

**`| summarize applicationCount = dcount(AppDisplayName) by UserPrincipalName, IPAddress`**

- `dcount(AppDisplayName)` = count unique application names
- `applicationCount` = name for the result column
- `by UserPrincipalName, IPAddress` = group by user AND IP together
- Result: for each user+IP combination, how many different apps had wrong password failures?

**`| where applicationCount >= threshold`** = keep only results at or above threshold

### Input/Output Example:

**Input** (after both where filters):

|TimeGenerated|UserPrincipalName|IPAddress|AppDisplayName|ResultDescription|
|---|---|---|---|---|
|2024-01-15 09:00|john@company.com|5.5.5.5|Office 365|Invalid password|
|2024-01-15 09:01|john@company.com|5.5.5.5|SharePoint|Invalid password|
|2024-01-15 09:02|john@company.com|5.5.5.5|Teams|Invalid password|
|2024-01-15 09:03|mary@company.com|5.5.5.5|Office 365|Invalid password|
|2024-01-15 09:04|mary@company.com|9.9.9.9|Office 365|Invalid password|
|2024-01-15 09:05|john@company.com|5.5.5.5|Office 365|Invalid password|

**After summarize:**

|UserPrincipalName|IPAddress|applicationCount|
|---|---|---|
|john@company.com|5.5.5.5|3|
|mary@company.com|5.5.5.5|1|
|mary@company.com|9.9.9.9|1|

Note: john tried Office 365 twice but dcount counts it once — 3 unique apps total.

**Final output** (threshold = 1, all pass):

|UserPrincipalName|IPAddress|applicationCount|
|---|---|---|
|john@company.com|5.5.5.5|3|
|mary@company.com|5.5.5.5|1|
|mary@company.com|9.9.9.9|1|

**Why this detects password spraying:** One IP failing authentication across many apps for the same user = not a human forgetting their password = automated attack. With threshold set to 3, only john would be flagged.

---

## `arg_max()` and `arg_min()` — Get Entire Row at Max/Min Value

These don't return a calculated number — they return an **entire row** that has the highest or lowest value of a specific column.

### `arg_max()` — most recent row

```kql
SecurityEvent 
| where Computer == "SQL10.na.contosohotels.com"
| summarize arg_max(TimeGenerated, *) by Computer
```

- `arg_max(TimeGenerated, *)` = find the row with the MAXIMUM (most recent) TimeGenerated
- `*` = return ALL columns from that winning row
- `by Computer` = do this per computer

**Input:**

|TimeGenerated|Account|EventID|Computer|Activity|
|---|---|---|---|---|
|2024-01-10 08:00|admin|4624|SQL10.na.contosohotels.com|Logon|
|2024-01-12 14:30|john|4688|SQL10.na.contosohotels.com|Process Created|
|2024-01-15 09:47|mary|4625|SQL10.na.contosohotels.com|Logon Failed|
|2024-01-13 11:00|admin|4634|SQL10.na.contosohotels.com|Logoff|

**Output:**

|TimeGenerated|Account|EventID|Computer|Activity|
|---|---|---|---|---|
|2024-01-15 09:47|mary|4625|SQL10.na.contosohotels.com|Logon Failed|

**Security use case:** "What was the last thing that happened on this machine?"

---

### `arg_min()` — oldest row

```kql
SecurityEvent 
| where Computer == "SQL10.na.contosohotels.com"
| summarize arg_min(TimeGenerated, *) by Computer
```

**Same input. Output:**

|TimeGenerated|Account|EventID|Computer|Activity|
|---|---|---|---|---|
|2024-01-10 08:00|admin|4624|SQL10.na.contosohotels.com|Logon|

**Security use case:** "When did this machine first appear in our logs?"

---

## Pipe Order Matters — Critical Lesson

The same operators in a different order give completely different results.

### Statement 1 — summarize THEN filter:

```kql
SecurityEvent
| summarize arg_max(TimeGenerated, *) by Account
| where EventID == "4624"
```

**Input:**

|TimeGenerated|Account|EventID|Activity|
|---|---|---|---|
|2024-01-10 08:00|john|4624|Logon|
|2024-01-12 14:30|john|4688|Process Created|
|2024-01-15 09:47|john|4634|Logoff|
|2024-01-10 09:00|mary|4625|Logon Failed|
|2024-01-14 11:00|mary|4624|Logon|
|2024-01-11 10:00|admin|4624|Logon|
|2024-01-13 15:00|admin|4624|Logon|

**After summarize** (most recent event per account regardless of type):

|TimeGenerated|Account|EventID|Activity|
|---|---|---|---|
|2024-01-15 09:47|john|4634|Logoff|
|2024-01-14 11:00|mary|4624|Logon|
|2024-01-13 15:00|admin|4624|Logon|

**After where EventID == 4624:**

|TimeGenerated|Account|EventID|Activity|
|---|---|---|---|
|2024-01-14 11:00|mary|4624|Logon|
|2024-01-13 15:00|admin|4624|Logon|

**john is missing** — his most recent event was a Logoff not a Login so he got filtered out.

**Answer to:** "Which accounts had a login as their LAST activity?"

---

### Statement 2 — filter THEN summarize:

```kql
SecurityEvent
| where EventID == "4624"
| summarize arg_max(TimeGenerated, *) by Account
```

**After where EventID == 4624** (only login events):

|TimeGenerated|Account|EventID|Activity|
|---|---|---|---|
|2024-01-10 08:00|john|4624|Logon|
|2024-01-14 11:00|mary|4624|Logon|
|2024-01-11 10:00|admin|4624|Logon|
|2024-01-13 15:00|admin|4624|Logon|

**After summarize** (most recent login per account):

|TimeGenerated|Account|EventID|Activity|
|---|---|---|---|
|2024-01-10 08:00|john|4624|Logon|
|2024-01-14 11:00|mary|4624|Logon|
|2024-01-13 15:00|admin|4624|Logon|

**john IS included** — because we filtered to logins first so his logoff never appeared.

**Answer to:** "What was the most recent login time for each account?"

**Key lesson:** Always think about WHAT DATA is flowing into each step. Filter before summarize vs summarize before filter = completely different results. Statement 2 is almost always what analysts want.

---

## `make_list()` and `make_set()` — Collect Values into Arrays

These collect multiple values into a single cell as a JSON array instead of multiple rows.

**JSON array** = a standard list format that looks like: `["value1", "value2", "value3"]`

### `make_list()` — list WITH duplicates

```kql
SecurityEvent
| where EventID == "4624"
| summarize make_list(Account) by Computer
```

**Input:**

|TimeGenerated|Account|EventID|Computer|
|---|---|---|---|
|2024-01-15 09:00|john|4624|SERVER01|
|2024-01-15 09:10|admin|4624|SERVER01|
|2024-01-15 09:15|john|4624|SERVER01|
|2024-01-15 09:20|mary|4624|SERVER02|
|2024-01-15 09:25|john|4624|SERVER02|
|2024-01-15 09:30|admin|4624|SERVER01|

**Output:**

|Computer|list_Account|
|---|---|
|SERVER01|["john", "admin", "john", "admin"]|
|SERVER02|["mary", "john"]|

john appears twice in SERVER01 because he logged in twice. Full login history preserved.

---

### `make_set()` — list WITHOUT duplicates

```kql
SecurityEvent
| where EventID == "4624"
| summarize make_set(Account) by Computer
```

**Same input.**

**Output:**

|Computer|set_Account|
|---|---|
|SERVER01|["john", "admin"]|
|SERVER02|["mary", "john"]|

john appears once even though he logged in twice. Cleaner answer to "which unique accounts logged into each computer?"

**When to use which:**

- `make_list()` = when duplicates matter (you want full history)
- `make_set()` = when you just want unique values (duplicates are noise)

---

## The `render` Operator — Create Visualizations

**`render`** turns query results into a visual chart. Always goes at the **very end** of a query as the last step.

### Syntax:

```
| render charttype
```

### Supported chart types:

|Chart type|Best used for|
|---|---|
|`areachart`|Volume over time with filled area below the line|
|`barchart`|Comparing categories with horizontal bars|
|`columnchart`|Comparing categories with vertical bars|
|`piechart`|Showing proportions / percentages of a whole|
|`scatterchart`|Finding correlations between two values|
|`timechart`|Trends and patterns over time (most used in security)|

### Example:

```kql
SecurityEvent 
| summarize count() by Account
| render barchart
```

**After summarize:**

|Account|count_|
|---|---|
|john|3|
|admin|2|
|mary|1|

**After render barchart** (visual output):

```
john  |████████████████████████| 3
admin |████████████████| 2
mary  |████████| 1
```

Instantly see which account has the most activity without reading rows of numbers.

---

## The `bin()` Function — Group Time into Buckets

**`bin(column, bucketSize)`** = rounds time values down to the nearest bucket of the given size. Groups events by time period instead of exact millisecond timestamps.

Think of it like rounding time to the nearest hour: 9:47 AM → 9:00 AM. 9:15 AM → 9:00 AM. Both end up in the "9 AM bucket."

### Syntax:

```
bin(TimeGenerated, 1d)   // group by day
bin(TimeGenerated, 1h)   // group by hour
bin(TimeGenerated, 7d)   // group by week
```

### Example:

```kql
SecurityEvent 
| summarize count() by bin(TimeGenerated, 1d) 
| render timechart
```

**Input:**

|TimeGenerated|Account|EventID|
|---|---|---|
|2024-01-13 09:00|john|4624|
|2024-01-13 14:30|mary|4688|
|2024-01-13 22:00|admin|4625|
|2024-01-14 08:00|john|4624|
|2024-01-14 11:00|mary|4624|
|2024-01-15 09:00|john|4688|
|2024-01-15 10:00|admin|4624|
|2024-01-15 11:00|mary|4625|
|2024-01-15 14:00|john|4624|
|2024-01-15 22:00|admin|4688|

**After bin — timestamps rounded to nearest day:**

|TimeGenerated (binned)|Account|EventID|
|---|---|---|
|2024-01-13 00:00|john|4624|
|2024-01-13 00:00|mary|4688|
|2024-01-13 00:00|admin|4625|
|2024-01-14 00:00|john|4624|
|2024-01-14 00:00|mary|4624|
|2024-01-15 00:00|john|4688|
|2024-01-15 00:00|admin|4624|
|2024-01-15 00:00|mary|4625|
|2024-01-15 00:00|john|4624|
|2024-01-15 00:00|admin|4688|

**After summarize count():**

|TimeGenerated|count_|
|---|---|
|2024-01-13|3|
|2024-01-14|2|
|2024-01-15|5|

**After render timechart** (visual line chart):

```
5 |                    ●
4 |                   /
3 |         ●        /
2 |          \      /
1 |           ●----
  |________________________
    Jan 13   Jan 14   Jan 15
```

January 15th spike is immediately visible. In a real investigation this prompts you to drill down on what happened that day.

**Why this is powerful for security:**

- Detect sudden spikes in activity = possible attack
- See attack timing patterns
- Compare baseline vs anomalies
- Power real-time dashboards in Sentinel

---

## Key Rules from This Module

1. **`summarize` = pivot table** — collapses rows into grouped calculations
2. **`count()` vs `dcount()`** — count() counts everything including duplicates, dcount() counts only unique values
3. **`make_list()` keeps duplicates, `make_set()` removes them**
4. **`render` always goes last** — visualization of whatever comes before it
5. **`bin()` + `summarize` + `render timechart`** = classic combination for time-series security analysis
6. **`has` is faster than `contains`** for whole-word matching
7. **`arg_max(TimeGenerated, *)`** = get most recent event — one of the most used patterns in Sentinel analytics rules
8. **Order in the pipe changes everything** — filter before summarize vs summarize before filter = completely different answers
9. **`if` versions of functions** = built-in filter inside the calculation (countif, avgif, maxif, etc.)
10. **`>=` means greater than OR equal to** — includes the boundary value, `>` excludes it

---

## Complete Function Reference

|Function|Returns|Security use case|
|---|---|---|
|`count()`|Total row count|Total events per account|
|`countif(condition)`|Count matching rows only|Count failed logins only|
|`dcount(column)`|Unique value count|How many unique computers touched|
|`dcountif(column, condition)`|Unique count where condition true|Unique computers with failed logins|
|`avg(column)`|Mathematical average|Session duration baseline|
|`avgif(column, condition)`|Average where condition true|Average duration for users only|
|`max(column)`|Highest value|Longest session duration|
|`maxif(column, condition)`|Highest value where condition true|Max failed attempts in failed logins only|
|`min(column)`|Lowest value|First time account appeared|
|`minif(column, condition)`|Lowest value where condition true|First failed login time|
|`percentile(column, N)`|Value at Nth percentile|Define normal threshold dynamically|
|`stdev(column)`|How spread out values are|Detect inconsistent anomalous behavior|
|`stdevif(column, condition)`|Spread where condition true|Consistent attack timing = automated tool|
|`sum(column)`|Total sum of values|Total bytes transferred = exfiltration|
|`sumif(column, condition)`|Sum where condition true|Bytes transferred in last hour only|
|`variance(column)`|Mathematical variance (stdev squared)|Statistical analysis, less common daily|
|`varianceif(column, condition)`|Variance where condition true|Rarely used directly|
|`arg_max(column, *)`|Entire row at maximum value|Most recent event for a machine/account|
|`arg_min(column, *)`|Entire row at minimum value|Oldest/first event for a machine/account|
|`make_list(column)`|JSON array with duplicates|Full login history per computer|
|`make_set(column)`|JSON array without duplicates|Unique accounts per computer|
|`bin(column, size)`|Time rounded to bucket|Group events by hour/day/week|