# GreenChit — Architecture Design Pack

Day 4 — Deliverable 1

**Design choices assumed in these diagrams:** hosting on Azure App Service (ADR-0002) and data in Azure SQL Database (ADR-0003). Both are explained and defended in the ADR set.

## 1\. System Context

GreenChit is an internal BISTEC tool for reimbursement claims. A **claimant** (any staff member) creates a claim with a category, date, amount in LKR, description and up to 5 receipt images, then submits it. Their **line manager** is notified in Microsoft Teams (with email as a fallback) and approves it, rejects it, or asks for more information. A **finance officer** exports approved claims as a CSV into a SharePoint folder, where BISTEC's **payroll automation** picks it up and pays the claim. An **auditor** can review the full history of any claim, because every status change is recorded with who, when and, for rejections, why. Everyone signs in with their BISTEC account through **Microsoft Entra ID**, and only the claimant, their line manager, finance and the auditor can see a given claim. The system must answer a submission in under 1.5 seconds (p95), accept receipts of up to 10 MB even on unreliable connections, be available 99.9% of business hours (08:00–19:00 Sri Lanka time, Monday to Friday), and keep a tamper-evident audit log for 7 years.

## 2\. Containers (C4 Level 2)



### Containers inside GreenChit

|Container|Technology|Responsibility|
|-|-|-|
|Web App|React single-page app, served by App Service, runs in the browser|Claim forms, claim status, approval screens for managers, export screen for finance, audit view. Uploads receipts straight to Blob Storage with signed URLs.|
|Claims API|ASP.NET Core 8 on Azure App Service (Premium v3)|The only entry point for business logic: claim workflow, access rules, signed upload URLs, audit records and export requests.|
|Claims Database|Azure SQL Database|Claims, receipt metadata and statuses. The audit log is a ledger table, so any change to past rows can be detected. Long-term backups are kept for 7 years.|
|Receipt Storage|Azure Blob Storage|Receipt images. The browser uploads directly using short-lived signed URLs (SAS), in resumable blocks, so a dropped connection only retries the missing part.|
|Message Queue|Azure Service Bus|Holds claim events and export requests, so slow or failing Teams, email or SharePoint calls never slow down a submission.|
|Notification Worker|WebJob running on the same App Service plan|Reads queued events, posts Teams Adaptive Cards, sends fallback email, and writes the payroll CSV to SharePoint.|

### People and external systems

|Element|Type|Role|
|-|-|-|
|Claimant (staff)|Person|Submits claims and tracks their status|
|Line manager|Person|Approves, rejects or asks for more information|
|Finance officer|Person|Exports approved claims for payroll|
|Auditor|Person|Reviews the status history of any claim|
|Microsoft Entra ID|External system (BISTEC tenant)|Single sign-on and access tokens|
|Defender for Storage|External system (Microsoft)|Scans every uploaded receipt for malware|
|Microsoft Teams|External system (Adaptive Card webhook)|Shows approval requests in the manager's chat|
|Exchange Online|External system (Microsoft Graph)|Sends fallback email if Teams delivery fails|
|SharePoint|External system|Folder that receives the payroll CSV|
|Payroll Automation|External system|Reads the CSV and pays staff|

**Platform services not drawn**, to keep the diagram readable: the Claims API and Notification Worker read secrets from **Azure Key Vault** using a managed identity \[HTTPS], and send logs, metrics and traces to **Application Insights** \[HTTPS]. This is stated in a note on the diagram.

### Relationships

Every arrow has a verb-led label and a protocol.

|From|To|Label|Protocol|
|-|-|-|-|
|Claimant|Web App|Submits claims, tracks status|HTTPS|
|Finance officer|Web App|Exports approved claims|HTTPS|
|Auditor|Web App|Reviews audit trail|HTTPS|
|Line manager|Web App|Approves, rejects, asks for info|HTTPS|
|Microsoft Teams|Line manager|Notifies with approval card|Teams chat|
|Web App|Microsoft Entra ID|Signs users in (SSO)|OIDC over HTTPS|
|Web App|Claims API|Calls REST API with JWT|HTTPS/JSON|
|Web App|Receipt Storage|Uploads receipts|HTTPS, signed URL|
|Claims API|Microsoft Entra ID|Fetches signing keys to validate tokens|HTTPS/OIDC|
|Claims API|Receipt Storage|Issues signed URLs, checks uploads|HTTPS|
|Defender for Storage|Receipt Storage|Scans new receipts|Azure platform|
|Claims API|Claims Database|Reads/writes claims, appends audit|SQL over TLS|
|Claims API|Message Queue|Publishes claim events|AMQP over TLS|
|Message Queue|Notification Worker|Delivers queued events|AMQP over TLS|
|Notification Worker|Claims Database|Reads claim details, records delivery|SQL over TLS|
|Notification Worker|Microsoft Teams|Posts Adaptive Card|HTTPS webhook|
|Notification Worker|Exchange Online|Sends fallback email|HTTPS, Graph API|
|Notification Worker|SharePoint|Writes payroll CSV|HTTPS, Graph API|
|Payroll Automation|SharePoint|Reads payroll CSV|HTTPS|

### Notation

Both diagrams use the same notation, shown in the legend on each diagram:

* **Dark blue:** people. **Blue:** containers inside GreenChit. **Light blue:** components inside the Claims API. **Grey:** external systems.
* Each box shows its **name**, its **type and technology** in square brackets, and its **responsibility**.
* **Dashed rectangle:** a boundary (the GreenChit system on Level 2, the Claims API on Level 3).
* **Arrows:** solid, pointing in the direction of the call or message, labelled "verb phrase \[protocol]".
* Container names are identical on both diagrams (Web App, Claims Database, Receipt Storage, Message Queue), so they can be matched one to one.

## 3\. Components (C4 Level 3) for the API service

### 

### Components inside the Claims API

|Component|Technology|Responsibility|
|-|-|-|
|API Endpoints|ASP.NET Core minimal APIs|Receives HTTP requests, validates input, and maps errors to HTTP status codes (for example `422` for invalid input, `403` for no access).|
|Auth \& Access Policy|ASP.NET Core authorization policies|Validates Entra ID tokens and decides who may see or act on a claim: only the claimant, their line manager, finance and the auditor.|
|Claims Workflow|C# service|The claim state machine: Draft → Submitted → Approved or Rejected → Paid, plus "request more information", which returns a claim to Draft. Rejects invalid transitions, such as approving your own claim.|
|Receipts|Azure Storage SDK|Issues 10-minute signed upload URLs (10 MB per file, 5 files per claim), confirms each upload, and reads Defender's malware scan result before a claim can be submitted.|
|Audit Log|SQL ledger table|Appends one row per status change: who, when, old status, new status, and the reason for a rejection. Rows can be added but not changed without detection.|
|Notifications Publisher|Service Bus SDK with a transactional outbox|Publishes `claim.submitted`, `claim.decided` and `export.requested` events. The event is saved in the same database transaction as the status change, so no notification is lost.|
|Payroll Export|C# service|Selects approved claims that have not been exported, requests the CSV export, and marks those claims as exported.|

### Relationships inside the Claims API

|From|To|Label|Protocol|
|-|-|-|-|
|Web App|API Endpoints|Calls REST API with JWT|HTTPS/JSON|
|API Endpoints|Auth \& Access Policy|Checks caller and claim access|in-process call|
|Auth \& Access Policy|Microsoft Entra ID|Fetches token signing keys|HTTPS/OIDC|
|API Endpoints|Claims Workflow|Passes claim commands|in-process call|
|API Endpoints|Payroll Export|Starts payroll export|in-process call|
|Payroll Export|Claims Workflow|Gets approved claims|in-process call|
|Claims Workflow|Receipts|Confirms receipts uploaded and clean|in-process call|
|Claims Workflow|Audit Log|Records each status change|in-process call|
|Claims Workflow|Notifications Publisher|Raises claim events|in-process call|
|Payroll Export|Notifications Publisher|Raises export.requested|in-process call|
|Claims Workflow|Claims Database|Reads and writes claims|SQL over TLS|
|Audit Log|Claims Database|Appends ledger rows|SQL over TLS|
|Receipts|Receipt Storage|Creates signed URLs, reads scan result|HTTPS|
|Notifications Publisher|Message Queue|Publishes events|AMQP over TLS|

### How the components meet the requirements

|Requirement|Component that meets it|
|-|-|
|Only the claimant, line manager, finance and auditor can see a claim|Auth \& Access Policy|
|Every status change recorded with who, when and why|Claims Workflow calling Audit Log on every transition|
|Audit log tamper-evident, kept for 7 years|Audit Log (SQL ledger table) and 7-year long-term backups|
|Receipts up to 10 MB, 5 per claim, on unreliable connections|Receipts (signed URLs, resumable block uploads)|
|Submission under 1.5 s p95|Claims Workflow saves and returns; Teams, email and SharePoint work happens later through the Notifications Publisher and the queue|
|Manager notified in Teams, with email fallback|Notifications Publisher → Message Queue → Notification Worker|
|Approved claims exported for payroll|Payroll Export → Notification Worker → SharePoint|

## 4\. Reading order

Read the pack in this order. Each step follows the same claim, so the views connect.

1. **Section 1 (context).** Understand who uses GreenChit and the key numbers: 1.5 s submissions, 10 MB receipts, 99.9% business-hours availability, 7-year audit log.
2. **Container diagram, top to bottom.** Start with the four people at the top. Follow the claimant's arrow into the **Web App**, and note the two paths out of it: receipts go straight to **Receipt Storage** with a signed URL, and the claim goes to the **Claims API** with a JWT.
3. **Container diagram, the synchronous core.** From the Claims API, follow the arrows to **Claims Database** (the claim and its audit row) and to **Microsoft Entra ID** (token checks). This is everything that happens before the claimant sees "Submitted".
4. **Container diagram, the asynchronous path.** Follow Claims API → **Message Queue** → **Notification Worker**, then out to **Teams**, **Exchange Online** and **SharePoint**. Then go up the right-hand side: Teams notifies the **line manager**, who approves through the Web App. Finally, payroll reads the CSV from SharePoint.
5. **Component diagram.** Zoom into the Claims API and follow the same claim: **API Endpoints** → **Auth \& Access Policy** (may this person act?) → **Claims Workflow** (is this transition allowed?) → **Receipts** (are the files uploaded and clean?) → **Audit Log** (record it) → **Notifications Publisher** (tell the manager). The containers at the edges are the same ones as in step 3, with the same names.
6. **Next:** the sequence diagram (`diagrams/sequence-submit-approve.md`) shows this same journey over time, including what happens when a receipt upload fails. Then read the ADRs to see why App Service and Azure SQL were chosen.

