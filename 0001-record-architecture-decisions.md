# ADR 0001: Record architecture decisions as ADRs in the repository

## Status
Accepted (date: 2026-11-04)

## Context
- **Decisions get lost.** GreenChit's design involves choices that are expensive to reverse: the hosting platform, the database, how receipts are stored and scanned. Six months from now, a new developer will ask "why App Service and not Container Apps?", and the person who knows the answer may have moved to another project.
- **Finance and audit need evidence.** GreenChit handles money and has a 7-year audit requirement. When an auditor or the finance team asks why the audit log is built a certain way, we need a written, dated answer, not a recollection.
- **The team is small and busy.** A 10-person team will not maintain heavy architecture documents. Whatever we choose must take minutes per decision, not days.
- **Decisions are reviewed in pull requests already.** The design pack lives in a Git repository, and every change is reviewed. A record that lives next to the code gets reviewed the same way, for free.
- **What we don't yet know:** how many decisions GreenChit will need recorded, and whether BISTEC will want ADRs from all projects collected in one central place later.

## Decision
- We record every significant architecture decision as an **Architecture Decision Record (ADR)**, using Michael Nygard's template: Status, Context, Decision, Consequences, Alternatives considered.
- ADRs live in the `adrs/` folder of the `greenchit-design` repository, as Markdown files named `NNNN-short-title.md`, numbered in order from `0001`.
- A decision is **significant** if it is hard to reverse, affects more than one container, or affects cost, security or compliance.
- We write the ADR **when the decision is made**, in the same pull request as the change it explains. At least one other engineer reviews it.
- Each ADR has a status: **Proposed** (under review), **Accepted**, or **Superseded by ADR-NNNN**. Once accepted, an ADR is never edited to change its decision. If we change our minds, we write a new ADR that supersedes it, and update only the old one's status line.
- The tech lead owns the ADR set and checks it during design reviews.

## Consequences
- **Easier:** onboarding, because a new engineer can read why GreenChit is built this way in under an hour. Audits, because decisions are dated and attributed. Revisiting decisions, because the original assumptions ("expected load under 200 requests per second") are written down and can be checked.
- **Harder:** every significant change now takes extra time to write up, typically 20–40 minutes, and reviewers must read it. Under deadline pressure, the team will be tempted to skip ADRs or write them afterwards, when the real reasons are already forgotten. We accept this cost, and the tech lead rejects pull requests that change architecture without an ADR.
- **Different:** decisions become visible and open to challenge. Someone can point at an ADR and say "this assumption is no longer true", which is uncomfortable but intended.

## Alternatives considered
- **Pages in Confluence or SharePoint:** rejected because they are separate from the code, are not reviewed in pull requests, and drift out of date without anyone noticing.
- **No written records (decisions in meetings and Teams chats):** rejected because the reasons disappear when people leave or forget, and finance and audit cannot rely on chat history as evidence.
- **A full architecture document (for example arc42):** rejected for now because it is too heavy for a 10-person team to keep current. ADRs give most of the value for a fraction of the effort, and we can add a fuller document later if GreenChit grows.
