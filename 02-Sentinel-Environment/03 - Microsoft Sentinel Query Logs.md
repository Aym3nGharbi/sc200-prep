## 📌 Overview
- Microsoft Sentinel stores collected log data in **tables**.
- **KQL (Kusto Query Language)** = language used to query these tables.
- Used for: analytics rules, workbooks, hunting.
- Data Connectors → pull data from sources → write it into specific tables (e.g. Windows Security Events → `SecurityEvent` table).

### Portals
| Portal | URL | Query Page |
|---|---|---|
| Azure portal | portal.azure.com | **Logs** page (under General) |
| Defender portal | security.microsoft.com | **Advanced hunting** OR **Data lake exploration → KQL queries** |

> ⚠️ Azure portal Sentinel is being retired — starting July 2026 all users redirected to Defender portal.

---

## 🖥️ Where to Run KQL (Defender Portal)

### 1. Advanced Hunting
`Investigation & response → Hunting → Advanced hunting`
- Run/save queries, create alert/detection rules, link results to incidents.
- Left panel = Schema (tables/fields), Functions, Queries.
- Top right = **workspace selector** (choose which Log Analytics workspace to query).

### 2. Data Lake Exploration
`Microsoft Sentinel → Data lake exploration → KQL queries`
- For ad-hoc queries on **long-term/archived data**.
- ⚠️ Usage-based billing: querying the data lake or copying data to analytics tier costs extra.
- Sub-pages: **KQL queries**, **Notebooks** (Jupyter-style), **Search & restore** (find/restore archived data).

### Workspace Scope popup
- Lets you pick which workspace(s) a query runs against (tenant can have multiple workspaces).
- Shows checkboxes per workspace + Apply/Cancel.

---

## 📋 Microsoft Sentinel Core Tables
*(Manage alerts/incidents themselves)*

| Table | Purpose |
|---|---|
| **SecurityAlert** | Alerts from Sentinel Analytic Rules, or directly from a connector |
| **SecurityIncident** | Incidents (grouped alerts) |
| **ThreatIntelligenceIndicator** | IOCs — file hashes, IPs, domains (manual or feed-ingested) |
| **Watchlist** | Manually imported reference data (e.g. VIP users, bad IPs) |

---

## 📋 Common Ingested Tables
*(Raw data from connected sources)*

| Table | Purpose |
|---|---|
| **AzureActivity** | Subscription/mgmt-group level Azure actions |
| **AzureDiagnostics** | Internal resource logs (Azure Diagnostics mode) |
| **AuditLogs** | Microsoft Entra ID admin/directory activity |
| **CommonSecurityLog** | Syslog in CEF format (firewalls, network devices) |
| **McasShadowItReporting** | Defender for Cloud Apps — shadow IT detection |
| **OfficeActivity** | O365 audit logs (Exchange, SharePoint, Teams) |
| **SecurityEvent** | Windows Security Event logs |
| **SigninLogs** | Azure AD sign-in logs |
| **Syslog** | Linux syslog events |
| **Event** | Sysmon events (Windows) |
| **WindowsFirewall** | Windows Firewall events |

---

## 📋 Microsoft Defender XDR Tables
*(Populated via Defender XDR connector — used heavily in Advanced Hunting)*

**Alerts**

| Table             | Purpose                                       |
| ----------------- | --------------------------------------------- |
| **AlertEvidence** | Files/IPs/URLs/users/devices tied to an alert |

**Cloud Apps**

| Table | Purpose |
|---|---|
| **CloudAppEvents** | Activity in O365 & other cloud apps |

**Device/Endpoint**

| Table | Purpose |
|---|---|
| **DeviceEvents** | Security control events (AV, exploit protection, etc.) |
| **DeviceFileCertificateInfo** | Cert info from signed file verification |
| **DeviceFileEvents** | File create/modify/delete |
| **DeviceImageLoadEvents** | DLL load events |
| **DeviceInfo** | Machine/OS info |
| **DeviceLogonEvents** | Local device sign-ins |
| **DeviceNetworkEvents** | Network connections |
| **DeviceNetworkInfo** | Device network config (adapters, IP/MAC, domains) |
| **DeviceProcessEvents** | Process creation (key for malware hunting) |
| **DeviceRegistryEvents** | Registry create/modify (persistence detection) |

**Email**

| Table                       | Purpose                                               |
| --------------------------- | ----------------------------------------------------- |
| **EmailEvents**             | Delivery/blocking events                              |
| **EmailPostDeliveryEvents** | Post-delivery security actions (e.g. zero-hour purge) |
| **EmailUrlInfo**            | URLs found in emails                                  |
| **EmailAttachmentInfo**     | Email attachment details                              |

**Identity**

| Table | Purpose |
|---|---|
| **IdentityDirectoryEvents** | On-prem AD domain controller events |
| **IdentityLogonEvents** | AD + Microsoft online service auth events |
| **IdentityQueryEvents** | AD object queries (recon detection) |