# GreenChit — Design Pack

Sajanikka — Intern Onboarding Track, Day 4: End-to-End Design

GreenChit is an internal BISTEC tool for reimbursement claims. Staff submit a claim with receipt photos, their line manager approves it from a Teams notification, finance exports approved claims to a SharePoint folder for payroll, and an auditor can review every status change for 7 years. This repository is the architecture design pack Finance asked for: diagrams at two C4 levels, a sequence diagram, four architecture decision records (ADRs), and a trade-off analysis with design review feedback.

## The design in one paragraph

GreenChit runs as **one ASP.NET Core 8 application on Azure App Service**, with a React Web App and a Notification Worker running as a WebJob on the same plan. Claims and a **tamper-evident audit ledger** live in **Azure SQL**. Receipt photos go **straight from the browser to Azure Blob Storage** with short-lived signed URLs, in resumable blocks, and are scanned by **Defender for Storage**. Notifications travel through **Azure Service Bus** to the worker, which posts **Teams Adaptive Cards** with email as a fallback and writes the payroll CSV to **SharePoint**. Everyone signs in with **Microsoft Entra ID**.

## Repository structure

```
greenchit-design/
├── README.md                                  ← you are here
├── sajanikka-day4-greenchit-design.md         Deliverable 1: architecture design pack
├── sajanikka-day4-trade-offs-and-review.md    Deliverable 4: trade-off table and design review
├── diagrams/
│   ├── container-diagram.png                  C4 Level 2 (image)
│   ├── container-diagram.drawio               C4 Level 2 (editable source)
│   ├── component-diagram.png                  C4 Level 3 for the Claims API (image)
│   ├── component-diagram.drawio               C4 Level 3 (editable source)
│   ├── sequence-submit-approve.md             Deliverable 2: sequence diagram (Mermaid)
│   ├── sequence-submit-approve.png            Deliverable 2: exported image, both paths
│   ├── sequence-happy-path.png                Happy path, exported from Mermaid Chart
│   └── sequence-error-path.png                Error path, exported from Mermaid Chart
├── adrs/                                      Deliverable 3: architecture decision records
│   ├── 0001-record-architecture-decisions.md
│   ├── 0002-hosting-platform.md
│   ├── 0003-database-choice.md
│   └── 0004-receipts-storage-and-virus-scan.md
└── trade-offs/
    └── hosting-options.md                     Scored comparison behind ADR-0002
```

## How to read this pack

Follow these steps in order. Each one follows the same claim, so the views connect.

1. **Context:** read Section 1 of `sajanikka-day4-greenchit-design.md` for who uses GreenChit and the key numbers: submissions under 1.5 s (p95), receipts up to 10 MB, 99.9% availability in business hours, and a 7-year tamper-evident audit log.
2. **Container diagram** (`diagrams/container-diagram.png`): start with the four people at the top, follow the claimant into the Web App, then the two paths out of it (receipts to Receipt Storage, the claim to the Claims API). Then follow the asynchronous path: Message Queue → Notification Worker → Teams, email and SharePoint.
3. **Component diagram** (`diagrams/component-diagram.png`): zoom into the Claims API and follow the same claim through API Endpoints → Auth & Access Policy → Claims Workflow → Receipts → Audit Log → Notifications Publisher.
4. **Sequence diagram** (`diagrams/sequence-submit-approve.md`): see the same journey over time, step by step. The happy path ends with the manager's approval; the error path shows what happens when a receipt upload fails after the claim was created.
5. **ADRs** (`adrs/`): read ADR-0001 first (why we record decisions), then 0002 (hosting), 0003 (database) and 0004 (receipts and malware scanning). Each one names what became harder, not just what became easier.
6. **Trade-offs and review** (`sajanikka-day4-trade-offs-and-review.md`): the scored comparison of App Service and Container Apps, the two attributes that decided it, and the feedback from the design review.

## Key decisions at a glance

| ADR | Decision | The honest cost |
|-----|----------|-----------------|
| [0001](adrs/0001-record-architecture-decisions.md) | Record significant decisions as ADRs in this repository | Each decision takes 20–40 minutes to write up and review |
| [0002](adrs/0002-hosting-platform.md) | One application on Azure App Service, not split services on Container Apps | Every change redeploys everything; the API and worker cannot scale separately |
| [0003](adrs/0003-database-choice.md) | Azure SQL with ledger tables for the audit log | Audit history can never be erased, so only employee IDs may be stored in it |
| [0004](adrs/0004-receipts-storage-and-virus-scan.md) | Direct uploads to Blob Storage with signed URLs, scanned by Defender for Storage | A wrongly flagged receipt can delay someone's reimbursement |

## What this design will NOT do in v1

GreenChit v1 does not reimburse anyone directly: it only produces a CSV that BISTEC's existing payroll automation picks up. It does not read receipts automatically (no OCR) or check amounts against company policy limits; managers and finance check those by hand. It handles only Sri Lankan rupees (LKR), with no currency conversion. It does not let a manager approve directly inside the Teams card: the card links to the Web App, so every approval passes through the same access checks. It runs in a single Azure region, so a full regional outage stops GreenChit until the region recovers. And it has no native mobile app: staff use the Web App in their phone's browser.

## Viewing and editing the diagrams

- **PNG files** open in any image viewer and display on GitHub.
- **`.drawio` files** open in draw.io (app.diagrams.net or the desktop app). After editing, export with **File → Export as → PNG** and replace the matching `.png`.
- **The sequence diagram** is Mermaid code in `diagrams/sequence-submit-approve.md`. GitHub renders it automatically; to edit, paste one diagram at a time into [mermaid.live](https://mermaid.live).
- Edit Markdown files in **VS Code**, not Notepad's formatted view, which adds backslashes that break Mermaid code.
