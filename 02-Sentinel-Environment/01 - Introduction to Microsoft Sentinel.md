## 1. Introduction

- Scenario: you're a SOC (Security Operations Center) analyst. **SOC** = the team/function responsible for continuously monitoring, detecting, investigating, and responding to security threats — the "security nerve center" of an organization.
- Your organization is moving workloads to the **public cloud**, and now needs a security tool that works across **both on-premises and multicloud** environments (multiple cloud providers, not just one).
- You're evaluating **SIEM** solutions to fill this need, and looking specifically at Microsoft Sentinel.
- Three things you ideally want from a SIEM:
  1. The features/functionality you actually need.
  2. **Minimal administration** — spend time on security work, not on managing/patching/scaling infrastructure.
  3. **A flexible pricing model** — pay for what you use, scale cost with need.
- Microsoft Sentinel is presented as checking all three boxes.

### Learning Objectives

- Identify the various components and functionality of Microsoft Sentinel.
- Identify use cases where Microsoft Sentinel would be a good solution.

### Prerequisites

- Familiarity with security operations in an organization (this module assumes you already roughly understand SOC work).
- Basic experience with Microsoft Defender and Azure services (Defender data often feeds into Sentinel).

---

## 2. What Is Microsoft Sentinel?

### What is a SIEM?

- **SIEM = Security Information and Event Management.** A category of software (not one specific product) — like "database" or "web browser" is a category, with many vendors (Splunk, QRadar, Sentinel, etc.) making their own version.
- The systems a SIEM watches over can be **hardware appliances, applications, or both**.

**In its simplest form, a SIEM lets you:**

1. **Collect and query logs** — logs are timestamped records of events (e.g. "User X logged in at 9:05 AM from IP Y"). A SIEM gathers these from many sources into one place and lets you search/filter them.
2. **Do correlation or anomaly detection** — *correlation* = connecting related events across different sources (e.g. a failed login + a new admin account creation together look suspicious, even if neither alone does). *Anomaly detection* = flagging things that deviate from a normal baseline (e.g. a user who normally logs in from one country suddenly logging in from another).
3. **Create alerts and incidents based on findings** — turn the analysis into actionable notifications a human can act on.

### SIEM Functionality Breakdown

| Feature | Explanation |
|---|---|
| Log management | Core plumbing: collecting logs from all resources, storing them, and being able to query/search them. Foundation everything else builds on. |
| Alerting | Proactively scanning log data (usually continuously/automatically) to surface potential incidents/anomalies without a human manually looking. |
| Visualization | Turning raw log data into graphs, charts, dashboards so patterns/trends are visible at a glance instead of buried in text rows. |
| Incident management | Workflow tooling around a detected problem: creating a formal record ("incident"), updating status (New → Active → Closed), assigning to an analyst, tracking investigation. |
| Querying data | A rich query language (KQL, in Sentinel's case) to dig into data yourself, beyond pre-built alerts/dashboards. |

Note: KQL fits directly into "Querying data," and underlies "Alerting," "Visualization," and "Incident management" too — this is why `union`, `join`, `extract`, `parse`, JSON handling from earlier modules matter; those are the actual mechanics behind Sentinel's SIEM capabilities.

### What Is Microsoft Sentinel Specifically?

- **Cloud-native** = built to run *in* the cloud (Azure) from the ground up — not old on-prem software adapted later. Means no physical hardware, no server installation, automatic scaling.

**Three core capabilities:**

1. **Get security insights across the enterprise by collecting data from virtually any source.** Not limited to Microsoft products — ingests logs from firewalls, other clouds (AWS, GCP), on-prem servers, SaaS apps, etc.
2. **Detect and investigate threats quickly using built-in machine learning and Microsoft threat intelligence.**
   - **Built-in ML** = some detection rules use ML models (built by Microsoft) to spot unusual patterns automatically, without you writing the detection logic.
   - **Microsoft threat intelligence** = Microsoft continuously tracks known malicious IPs, domains, malware signatures, attacker techniques globally (from visibility across Windows, Azure, Office 365, etc.), feeding that knowledge into Sentinel so it recognizes known-bad indicators automatically.
3. **Automate threat responses using playbooks and Azure Logic Apps integration.**
   - **Playbooks** = automated response workflows.
   - **Azure Logic Apps** = a separate Azure service for building automated "if this happens, then do that" workflows (similar in spirit to Zapier/Power Automate, but enterprise-grade, deeply integrated with Azure). Sentinel's playbooks are built ON TOP of Logic Apps — Sentinel doesn't reinvent automation.

### "No Servers" Distinction

- Traditional/legacy SIEMs often required standing up your OWN infrastructure — dedicated servers, storage, sometimes complex clustering — either physically on-prem or as self-managed cloud VMs. You'd be responsible for patching, scaling, maintaining it.
- Sentinel is **fully managed by Microsoft** — no servers to provision at all. You just "turn it on" in the Azure portal; Microsoft handles the underlying infrastructure, scaling, maintenance.
- You can get up and running with Sentinel in just a few minutes — a direct contrast to legacy SIEM deployments taking weeks/months.

### Tight Integration With Other Cloud Services

- Because Sentinel lives inside Azure, it naturally plugs into other Azure capabilities:
  - **Authorization** — Azure's identity/access system (Azure AD / Entra ID, role-based access control) governs who can do what inside Sentinel, without building a separate permissions system.
  - **Automation** — Logic Apps for playbooks (as above).
- This contrasts with a SIEM bolted onto the cloud as an afterthought — Sentinel was designed to be a native citizen of Azure.

### End-to-End Security Operations

- Microsoft Sentinel helps enable end-to-end security operations: **collection, detection, investigation, and response** — this is the **Collect → Detect → Investigate → Respond** cycle. Every Sentinel capability maps onto one of these four stages.

---

## 3. When to Use Microsoft Sentinel

- Use Sentinel if you want to:
  1. Collect event data from various sources.
  2. Perform security operations on that data to identify suspicious activity.

**"Security operations" includes** (each maps 1:1 to a Sentinel feature):

- Visualization of log data → **Workbooks**
- Anomaly detection → **Analytics alerts (ML-based)**
- Threat hunting → **Hunting**
- Security incident investigation → **Incidents & investigation graph**
- Automated response to alerts and incidents → **Automation playbooks**

### Additional Deciding Factors

| Factor | Explanation |
|---|---|
| Cloud-native SIEM: no servers to provision, effortless scaling | No infrastructure management overhead; automatically scales as data volume grows. |
| Integration with Azure Logic Apps and its hundreds of connectors | Automated playbooks can trigger actions across a huge range of third-party tools, not just Microsoft products. |
| Benefits of Microsoft research and machine learning | You inherit Microsoft's own security research/ML models (trained on Microsoft's global threat visibility) without building that expertise yourself. |
| Key log sources provided for free | Some common/important data sources don't cost extra to ingest (specifics can change — check current docs for a real deployment). |
| Support for hybrid cloud and on-premises environments | Explicitly supports mixed environments — some infrastructure in Azure, some on-prem, some in other clouds. |
| SIEM and a data lake all in one | A **data lake** = large-scale storage repository holding vast amounts of raw data cheaply, for long-term retention/analysis. Sentinel isn't just an alerting tool — the underlying Log Analytics workspace can serve as a long-term data repository too, not just security alerting. |

### The Scenario Resolution

- Original requirements: support for data from multiple cloud environments + SOC features/functionality without too much administrative overhead. Sentinel delivers both: multicloud via wide data connector support (including AWS, GCP, syslog); low overhead via cloud-native, serverless nature.
- **New realization**: automation (playbooks/SOAR) should be part of the SOC strategy going forward, even though it wasn't originally on the requirements list.

### Important Scope Clarification — When NOT to Use Sentinel Alone

- **If collecting infrastructure/application logs for performance monitoring** → use **Azure Monitor and Log Analytics** for that purpose instead. Azure Monitor (with Log Analytics) is the general-purpose monitoring platform for performance/uptime/app health — not specifically security. Even though Sentinel is built on Log Analytics, if the goal is "is my app running slow?" rather than "is my environment under attack?", that's really just plain Azure Monitor, not Sentinel's security-specific features.
- **If you want to understand security posture, ensure policy compliance, and check for security misconfigurations** → use **Microsoft Defender for Cloud**. This is a separate but related product focused on **security posture management** — e.g. "is this storage account publicly exposed?", "am I compliant with a regulatory framework?", "are VMs missing security patches?" More about prevention/configuration hygiene than detecting/responding to an active attack.
- These tools aren't mutually exclusive: **Defender for Cloud alerts can be ingested as a data connector into Sentinel**, so you get posture management (Defender for Cloud) AND full SIEM/SOAR capability (Sentinel) working together.
- **Exam distinction to remember: Defender for Cloud = posture/prevention; Sentinel = detection/response (SIEM/SOAR).**

---

## 4. How Microsoft Sentinel Works — The 7 Stages

### The Core Model: Collect → Detect → Investigate → Respond

Every feature of Microsoft Sentinel fits into one of these four stages.

### Stage 1: Data Connectors (Collect)

- Sentinel starts empty — no data until you configure a source.
- A **data connector** is a bridge: brings logs from a specific source into Sentinel.
- Two-step setup:
  1. **Content hub** — an "app store" inside Sentinel; packaged "solutions" usually bundle a data connector + related analytics rules + workbooks + hunting queries, installed together in one click.
  2. **Data connectors page** — where you actually turn a connector ON and configure it.
- Two speeds of setup: **easy/one-click** (e.g. Azure Activity, since it's native to Azure) vs. **harder/more configuration** (e.g. Syslog, needing an agent installed on the source machine).
- Supported source types include: Syslog, CEF (Common Event Format — standard format used by security appliances like firewalls), TAXII (protocol for sharing threat intelligence feeds), Azure Activity, Microsoft Defender services, AWS, GCP, and third-party products (e.g. CrowdStrike).
- **AMA (Azure Monitor Agent)** — modern software used to collect/forward logs to Sentinel (replacing older "Legacy Agent" methods).

**Simple summary:** Data connectors = trucks bringing data into the warehouse. No connectors = no data = nothing else in Sentinel can work.

### Stage 2: Log Retention (Collect — storage)

- Data collected via connectors is stored in a **Log Analytics workspace**.
- Sentinel does NOT have its own separate storage system — it sits on top of Log Analytics, a more general-purpose Azure storage/database service (used for many things, not just security). Sentinel adds security features on top.
- Once stored, **KQL** is used to search/analyze the data — same language from the earlier modules (`where`, `union`, `join`, `extract`, `parse_json`, etc.).
- Each connected source creates its own table (e.g. `AzureActivity`, `SecurityEvent`, `SigninLogs`) with different columns — exactly why `union` (combining tables) and `join` (matching rows across tables) are needed in real KQL work.

**Simple summary:** Data connectors bring data IN. Log Analytics workspace stores it. KQL is the tool used to search/analyze it.

### Stage 3: Workbooks (Detect — visual side)

- A **workbook** is a dashboard, turning raw log data into charts/graphs/tables instead of plain text rows.
- Every chart/graph inside a workbook is powered by a **KQL query** running behind the scenes — a query is written, and the workbook displays its result as a picture instead of a text table.
- Can use Microsoft's **built-in workbooks** and edit them, or build custom ones from scratch.
- Workbooks are Sentinel's implementation of the broader **Azure Monitor Workbooks** feature, not something exclusive to Sentinel.

**Simple summary:** Workbooks = dashboards. Every piece of a dashboard = a KQL query result, displayed visually.

### Stage 4: Analytics Alerts (Detect — automatic side)

- **Analytics rules** run automatically, in the background, on a schedule, creating alerts by themselves when suspicious activity is detected. No human needs to be actively watching.
- Three types:
  1. **Built-in rules you can edit** — Microsoft wrote the underlying KQL logic; you can tweak it.
  2. **Built-in ML-based rules** — use pattern-detection models built by Microsoft. Internal logic usually can't be edited directly, but settings/scope can often be adjusted. **Fusion** = Microsoft's name for their ML correlation engine — connects multiple separate, low-confidence signals together to detect complex, multi-stage attacks a single simple rule would miss.
  3. **Custom rules you write yourself** — write your own KQL query, define a schedule, specify what triggers an alert.
- Rules have a **severity** rating (High, Medium, Low, Informational) indicating how serious a match is.

**Simple summary:** Analytics rules = automatic watchers. Run constantly in the background, create alerts on their own.

### Stage 5: Threat Hunting (Detect — manual/human side)

- Analytics rules can only catch things someone already thought to write a rule for.
- **Threat hunting** = a human analyst actively searches for suspicious activity using KQL, testing ideas/hunches — even without a specific alert triggering first.
- Sentinel provides **pre-made hunting query templates** (often via Content hub solutions), so analysts don't have to invent search ideas from scratch every time. Analysts can also write custom hunting queries.
- Hunting queries are often organized around known **attack stages/techniques** (Reconnaissance, Initial access, Execution, Persistence, Privilege escalation, Defense evasion, Credential access, Discovery, Lateral movement) — mapping to the **MITRE ATT&CK** framework (a well-known catalog of attacker techniques).
- **Azure Notebooks integration** — lets advanced analysts combine KQL with a full programming language (typically Python) for more sophisticated, code-driven analysis (statistical modeling, custom visualizations, external libraries) than KQL alone easily supports.

**Simple summary:** Threat hunting = manual, proactive searching using KQL, often from a ready-made template, instead of waiting for an automatic alert.

### Stage 6: Incidents and Investigations (Investigate)

- Whether from an automatic analytics rule firing or a human hunter escalating a finding, the result is the same: an **incident** gets created — a formal case record that "something needs attention."
- Standard incident management tasks: changing status (New → Active → Closed), assigning to a specific analyst.
- **Investigation graph** — a visual map (not a text list) connecting **entities** (a user, computer, IP address, website, file — anything relevant) and how they relate to each other across log data, along a timeline.
- A **Timeline** panel lists the same connected events in the order they happened, with timestamps — building the full story of an attack from start to finish.
- Saves an analyst from manually running several separate KQL queries and mentally connecting the dots — Sentinel builds the connected picture automatically.

**Simple summary:** An incident is the case file. The investigation graph is a visual map connecting related people/computers/alerts, with a timeline, for quickly understanding "what really happened here."

### Stage 7: Automation Playbooks (Respond)

- Investigating tells you what happened. **Playbooks** take **action** — automatically, without manual steps every time.
- A **playbook** is a workflow: "when X happens, automatically do Y, then Z."
- Built using **Azure Logic Apps** — an existing Azure automation service. Sentinel reuses Logic Apps rather than building its own engine, gaining access to hundreds of pre-built connectors to third-party tools (email, Teams, ServiceNow, Slack, etc.).
- Types of automation:
  - **Automation rule** — simple, non-code automation (e.g. auto-assign High severity incidents to a specific analyst).
  - **Playbook with incident trigger** — runs automatically when a new incident is created.
  - **Playbook with alert trigger** — runs automatically when an alert fires (before becoming a full incident).
  - **Playbook with entity trigger** — runs based on a specific entity (e.g. a specific user or IP) being involved.
  - **Blank playbook** — build one from scratch.
- Use cases: incident management (auto-assigning), enrichment (auto-gathering extra context, like IP reputation), investigation (automated data-gathering), **remediation** (actually fixing the problem automatically — e.g. disabling a compromised account, blocking a malicious IP).
- This category of capability is called **SOAR** — Security Orchestration, Automation, and Response.
- Real example: "Block-AADUser-Alert" / "Block-AADUser-Incident" — playbooks that automatically block/disable a user's Azure AD (Entra ID) account when triggered.

**Simple summary:** Playbooks = "if this happens, then automatically do that" workflows. Example: a "Suspicious sign-in" alert can trigger a playbook that automatically disables that user's account within seconds — much faster than a human noticing and acting manually.

---

## Putting It All Together

| Stage | What it does | Maps to |
|---|---|---|
| 1. Data connectors | Bring logs in from many sources (Azure, AWS, firewalls, Office 365...) | Collect |
| 2. Log retention (Log Analytics + KQL) | Store the data; searchable with KQL | Collect (storage) |
| 3. Workbooks | Turn data into dashboards for visual insight | Detect (visual side) |
| 4. Analytics alerts | Watch data automatically, create alerts on their own | Detect (automatic side) |
| 5. Threat hunting | Manually search data with KQL for what automatic rules missed | Detect (manual/human side) |
| 6. Incidents & investigation | Formal case record + visual map/timeline of the full attack story | Investigate |
| 7. Automation playbooks | Automatically respond/fix problems without manual action | Respond |

---

## Quick-Reference Cheat Sheet

| Term | Meaning |
|---|---|
| SOC | Security Operations Center — the team monitoring/responding to threats |
| SIEM | Security Information and Event Management — category of tool for collecting/analyzing security logs |
| Cloud-native | Built for the cloud from the ground up; no servers to provision |
| Correlation | Connecting related events across different sources to find a meaningful pattern |
| Anomaly detection | Flagging activity that deviates from a normal baseline |
| Data connector | Mechanism to ingest logs from a specific source into Sentinel |
| Content hub | Marketplace of bundled solutions (connector + rules + workbooks + hunting queries) |
| AMA (Azure Monitor Agent) | Modern software used to collect/forward logs to Sentinel |
| Log Analytics workspace | The underlying data store for all ingested logs; queried via KQL |
| Workbook | A KQL-powered dashboard/visualization |
| Analytics rule/alert | Automated detection logic (built-in, ML-based/Fusion, or custom scheduled KQL) |
| Fusion | Microsoft's ML correlation engine connecting low-confidence signals into multi-stage attack detections |
| Threat hunting | Proactive, human-driven search for threats not yet caught by alerts |
| MITRE ATT&CK | A known catalog/framework of attacker techniques, organized by attack stage |
| Incident | The case record created when an alert fires; central unit of investigation |
| Entity | A specific thing involved in an incident (user, computer, IP, file, website) |
| Investigation graph | Visual map of related entities across log data over time |
| Playbook | Automated response workflow, built on Azure Logic Apps |
| Azure Logic Apps | Separate Azure service for building automated workflows; playbooks are built on top of it |
| SOAR | Security Orchestration, Automation, and Response — the category playbooks belong to |
| Remediation | Automatically fixing/stopping a problem (e.g. disabling an account, blocking an IP) |
| Data lake | Large-scale storage repository for holding vast amounts of raw data cheaply, long-term |
| Defender for Cloud | Separate product for security posture/compliance/misconfiguration — complements, doesn't replace, Sentinel |

### Key Takeaway

Microsoft Sentinel is a **cloud-native SIEM + SOAR platform** built around the **Collect → Detect → Investigate → Respond** lifecycle: data connectors ingest logs from virtually any source into a Log Analytics workspace, queried via **KQL**; workbooks visualize that data; analytics rules (built-in, ML/Fusion, or custom) and threat hunting detect suspicious activity automatically and manually; incidents and the investigation graph help analysts understand the full attack story; and automation playbooks (built on Azure Logic Apps, i.e. SOAR) can respond and remediate automatically — all without managing any servers. Use Sentinel for detection/response; use Azure Monitor for general performance monitoring; use Defender for Cloud for security posture/compliance (though its alerts can also feed into Sentinel).