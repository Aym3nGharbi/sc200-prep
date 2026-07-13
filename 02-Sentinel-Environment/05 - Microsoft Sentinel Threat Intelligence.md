## 📌 Overview
- Threat indicators (IoCs) are stored in a dedicated table, manageable via the **Threat Intelligence** page in Sentinel.
- Sources: automatic **connectors** (threat intel providers) + **manual entry** (your own threat hunting team).
- Indicator types: IP addresses, domains, file hashes, URLs.

**Scenario:** SOC analyst receives indicators from external TI providers (auto-imported via connectors) AND from the internal threat hunting team (must be manually added via the Threat Intelligence page).

**Learning objectives:**
- Manage threat indicators
- Query threat indicators with KQL

**Prerequisite:** Basic knowledge of monitoring, logging, alerting.

---

## 🔄 Threat Intelligence Lifecycle (Diagram Breakdown)

**1. Ingestion (Data connectors):**

| Path | Description |
|---|---|
| TAXII Server → Sentinel | TAXII = standard protocol for sharing threat intel; Sentinel connector pulls data automatically |
| TIP/custom solution → Threat Intel Indicators API → Sentinel | TIP = Threat Intelligence Platform (3rd-party or custom); pushes data via Microsoft's upload API |

**2. Storage & Query (Sentinel Logs):**
- Data lands in `ThreatIntelligenceIndicator` table.
- Queryable via KQL:
```kql
ThreatIntelligenceIndicator
| where TimeGenerated > ago(24h)
| limit 10
```

**3. Usage — feeds into two things:**

**A) Sentinel Analytics (detection rules)**
Example rule: *"TI map IP entity to AzureActivity"*
- Severity: Medium | Type: Scheduled
- Purpose: matches IPs in `AzureActivity` against IP-type IOCs from TI
- Data sources: `ThreatIntelligenceIndicator` + `AzureActivity`
- Tactic: Impact (MITRE ATT&CK category)
- Sample query logic:
```kql
let dt_lookBack = 1h;      // lookback window for activity logs
let ioc_lookBack = 14d;    // lookback window for indicators
ThreatIntelligenceIndicator
| where TimeGenerated >= ago(ioc_lookBack) and Exp... // not-expired filter
| where Active == true     // only active indicators
// Picking up only IOC's that contain the entities...
```
- Triggered rules → generate **Incidents** → can trigger **Playbooks** (automated response)

**B) Sentinel Workbooks (visualization)**
Example: bar chart of indicator matches by category — WatchList (375K), Malware (126K), CryptoMining (11.1K), MaliciousUrl (1.58K), C2 (849).
- C2 = Command and Control (attacker infrastructure controlling infected machines)

**Full flow:** Connectors → `ThreatIntelligenceIndicator` table → Analytics rules (→ Incidents → Playbooks) + Workbooks (dashboards)

---

## 🧠 What is Cyber Threat Intelligence (CTI)?

**Sources of CTI:**
- Open-source data feeds
- Threat intelligence-sharing communities
- Paid intelligence feeds
- Internal security investigations (e.g. your own threat hunting team)

**Forms of CTI:**
- **Strategic** — written reports on threat actor motivations, infrastructure, techniques
- **Tactical** — specific IOCs: IPs, domains, file hashes

**Why CTI matters:** gives context to raw log data — e.g. "connection to 1.2.3.4" means nothing alone, but becomes urgent if TI says that IP is a known C2 server.

**Key terms:**
- **SIEM** = Security Information and Event Management (what Sentinel is)
- **IoC (Indicator of Compromise)** = specific evidence of malicious activity (bad IP/domain/hash)
- **Tactical threat intelligence** = machine-readable IOC data, usable at scale by automated systems (as opposed to narrative reports)

---

## 🎯 5 Ways to Integrate TI into Sentinel

1. **Data connectors** — auto-import from TI platforms (TAXII, TIP/API)
2. **View/manage in Logs + Threat Intelligence area** — via KQL or UI
3. **Built-in Analytics rule templates** — pre-made detection rules using your TI data
4. **Threat Intelligence workbook** — pre-built visualization dashboard
5. **Threat hunting** — manually use TI data in hunting queries

---

## 🛠️ Managing Threat Indicators

- **Threat Intelligence area** (in Sentinel menu) lets you view/sort/filter/search indicators **without writing KQL**.
- Also lets you **create new indicators** manually and do admin tasks like **tagging**.

### Create a new threat indicator (Defender portal)
1. Open Defender portal (security.microsoft.com) → Microsoft Sentinel
2. `Threat management > Threat intelligence`
3. If prompted "This page has a new home" → click **Open Intel management**
4. Redirected to **Intel management** page (under Defender portal's own Threat Intelligence section)
   > 💡 Tip: Since TI features are merging into Defender portal, you can go directly to Intel management from there going forward.
5. Click **Add new**
6. Choose indicator type → fill required fields (marked with red asterisk *) → click **Apply**

### Tagging indicators
- **Tag** = label attached to indicators for easy grouping/searching.
- Use cases: group indicators by **incident** (e.g. "INC-2025-114") or by **threat actor/campaign** (e.g. "APT29")
- Can tag **individually** or **multi-select** (bulk tagging)
- Tags are **free-form** → use a **standard naming convention** to avoid inconsistency (e.g. "Phishing2025" vs "phishing_2025")
- **Multiple tags allowed per indicator**

---

## 🔍 Querying Threat Indicators with KQL

- Table: `ThreatIntelligenceIndicator`
- This table is also the base source for Analytics rules and Workbooks.
- Access via: `Logs` (General section of Sentinel menu)

```kql
ThreatIntelligenceIndicator
```
(basic query — returns all rows; add `where`, `project`, `limit` etc. to refine)

---

## ⚠️ Critical Update — Legacy Table Being Replaced

- **April 3, 2025:** Two new tables previewed to support **STIX** schemas (STIX = Structured Threat Information eXpression, an industry-standard format for threat intel):
  - **`ThreatIntelIndicators`** — replaces `ThreatIntelligenceIndicator` for indicator data
  - **`ThreatIntelObjects`** — new table for broader STIX "objects" (threat actors, campaigns, relationships, etc.)

- **Transition period:** data ingested into BOTH new tables AND legacy table until **July 31, 2025**.
- **After July 31, 2025:** legacy `ThreatIntelligenceIndicator` table **stops receiving new data**.
  - Must update custom queries, analytics/detection rules, workbooks, and automation to use the new tables.
- Microsoft updated all out-of-the-box Content hub solutions to use the new tables automatically.

> ⚠️ Since this deadline has already passed (today's context is mid-2026), any real Sentinel environment should now be using `ThreatIntelIndicators` / `ThreatIntelObjects` — NOT the legacy table.

---

## 💡 Quick Mental Model
> TI = known-bad data (IPs/domains/hashes) → comes in via connectors (auto) or manual entry (Intel management) → stored in `ThreatIntelIndicators`/`ThreatIntelObjects` (new) — `ThreatIntelligenceIndicator` (legacy, deprecated post July 2025) → powers Analytics rules (alerts/incidents), Workbooks (dashboards), and Threat Hunting.