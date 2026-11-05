# ADR 0003: Store GreenChit data in Azure SQL Database with ledger tables

## Status
Accepted (date: 2026-11-04)

## Context
- **The data is relational.** A claim belongs to a claimant, has a line manager, has one or more receipts, and moves through statuses. Finance needs questions like "all approved claims for October, by department, not yet exported". These are joins and filters across several related tables, which is what a relational database does best.
- **Several things must change together or not at all.** When a claim is approved, three writes happen: the claim's status, an audit row, and an outbox event for the notification (see the sequence diagram, step 23). If one succeeds and another fails, the audit trail or the notifications become wrong. We need transactions across these tables.
- **The audit log must be tamper-evident for 7 years.** Finance policy requires every status change to be kept for 7 years, and it must be possible to prove nobody changed or deleted past entries.
- **The team knows SQL.** The developers use Entity Framework Core and SQL Server daily. They have no production experience with document databases, partition keys or request-unit cost planning.
- **The volume is small.** Even with every BISTEC employee submitting several claims a month, GreenChit stores tens of thousands of rows per year, not millions. Throughput and global distribution are not concerns.
- **Two options are allowed:** Azure SQL Database or Cosmos DB.
- **What we don't yet know:** whether finance will later need reporting directly on GreenChit data, and how Sri Lanka's Personal Data Protection Act will apply to staff data kept for 7 years.

## Decision
- **We choose Azure SQL Database.**
- **Configuration:**
  - Tier: **General Purpose, provisioned, 2 vCores**, with zone redundancy enabled.
  - Access: Microsoft Entra authentication with the App Service's **managed identity**; no SQL passwords exist.
  - Schema managed by **Entity Framework Core migrations**, applied in the deployment pipeline.
- **Tamper-evident audit log:** the `AuditLog` table is an **append-only ledger table**. Rows can be inserted but never updated or deleted, and Azure SQL keeps a cryptographic hash chain of all changes. The `Claims` table is an **updatable ledger table**, so its full change history is also verifiable. Ledger **digests are stored automatically in immutable Azure Blob Storage**, so even a database administrator cannot rewrite history without detection.
- **7-year retention:** **long-term backup retention** keeps a weekly backup for 52 weeks and a yearly backup for 7 years, in addition to normal point-in-time restore.
- **One transaction per status change:** the claim update, its audit row and its outbox event are always written in one SQL transaction.

## Consequences
- **Easier:** the team works with tools it already knows; transactions keep claims, audit rows and notifications consistent; finance queries and the payroll export are simple SQL; tamper evidence comes built in rather than being something we design ourselves.
- **Harder:** **ledger tables are permanent.** A table cannot be turned back into a normal table, and rows in the append-only audit ledger can never be deleted. If a staff member leaves and asks BISTEC to erase their personal data, we **cannot remove their name from 7 years of audit history**. We will have to store only employee IDs in audit rows, keep names in a separate normal table that can be anonymised, and get legal confirmation that this is acceptable. This is genuinely uncomfortable, and it is the main risk of this decision.
- **Harder:** schema changes need more care. Some changes to ledger tables (such as dropping columns) behave differently, so migrations must be reviewed with that in mind.
- **Different:** we pay for 2 vCores around the clock, although GreenChit is busy only in business hours. We accept this, because the cheaper "serverless with auto-pause" option would add a slow first request each morning (see below).

## Alternatives considered
- **Azure Cosmos DB:** rejected. Its strengths are global distribution, very high throughput and flexible document shapes, and GreenChit needs none of these. Its transactions only work within one partition, so keeping a claim, its audit row and its outbox event consistent would need careful partition design or extra code. It has no built-in tamper-evident ledger, so we would have to build hash chaining ourselves. The team would also need to learn partition keys and request-unit costs on a project with a deadline.
- **Azure SQL serverless with auto-pause:** rejected. It would cost less outside business hours, but after a pause the first request waits while the database resumes, which can take up to a minute. That would break the 1.5-second submission target for whoever submits first each morning.
- **Azure SQL with a normal audit table plus our own hash chain:** rejected, because writing and verifying our own tamper-evidence is easy to get subtly wrong, and the ledger feature gives the same guarantee, tested and supported by Microsoft.
