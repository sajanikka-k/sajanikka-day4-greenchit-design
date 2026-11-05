# ADR 0004: Upload receipts directly to Blob Storage with signed URLs, and scan them with Defender for Storage

## Status
Accepted (date: 2026-11-04)

## Context
- **Receipts are large and come over unreliable connections.** Each claim can have up to 5 receipt photos of up to 10 MB each, often taken on a phone and uploaded over office Wi-Fi or mobile data. The requirement says uploads must succeed even with intermittent connectivity.
- **The constraint is fixed:** receipts go into Azure Blob Storage using a signed-URL pattern.
- **Receipts are untrusted files.** They come from any staff device, and a finance officer or auditor will later open them. One infected PDF opened on a finance laptop could spread malware inside BISTEC.
- **Receipts are private and financial evidence.** Only the claimant, their line manager, finance and the auditor may see a receipt, and receipts are part of the claim record that finance keeps for 7 years.
- **The submission target is tight:** under 1.5 seconds (p95) from tapping Submit to confirmation. Moving 10 MB files through the API during submission would make that impossible on slow connections.
- **What we don't yet know:** which file types staff will actually send (for example HEIC photos from iPhones); how often a malware scan will flag a legitimate receipt; and whether finance policy requires receipts to be stored as unchangeable (immutable) records.

## Decision
- **We choose direct browser uploads to Azure Blob Storage with short-lived signed URLs**, and **malware scanning on upload with Microsoft Defender for Storage**.
- **Storage setup:**
  - A dedicated storage account with **zone-redundant storage (ZRS)** and a private container `receipts`. Public access and shared-key access are **disabled**.
  - Blob path: `receipts/{claimId}/{receiptId}`.
  - Blob soft delete for **14 days**, so an accidental deletion can be undone.
- **Uploads:**
  - The Claims API issues a **user delegation SAS** (signed with Entra ID, not an account key) for **one blob path only**, with **create and write permission only**, valid for **10 minutes**, HTTPS only.
  - The Web App uploads in **4 MB blocks** and retries a failed block up to **3 times** (after 1, 2 and 4 seconds), so a dropped connection only re-sends the missing block.
  - Limits: **10 MB per file, 5 files per claim**, and only **JPEG, PNG, HEIC and PDF**. The Web App checks these before uploading; the API checks them again on confirmation by reading the blob's size and its first bytes, and deletes any file that breaks the rules.
- **Malware scanning:**
  - **Defender for Storage malware scanning on upload** is enabled on the account. It writes its result to each blob's index tags.
  - The Claims API allows a claim to be **submitted only when every receipt's scan result is "No threats found"**. If a scan has not finished, the API returns `409 Conflict` with `Retry-After: 10`.
  - A malicious file is **soft-deleted** automatically (Defender option enabled), the claim stays **Draft** with `reason="receipt_blocked"`, and the event is sent to the security team.
- **Viewing:** managers, finance and auditors open receipts through a **read-only SAS valid for 5 minutes**, issued only after the Auth & Access Policy component confirms they may see that claim.
- **Retention:** a lifecycle rule moves receipts to the **Cool tier after 90 days** and deletes them **7 years** after the claim is paid. Uploaded files never attached to a submitted claim are deleted after **24 hours**.

## Consequences
- **Easier:** 10 MB files never pass through the Claims API, so submission stays fast; interrupted uploads resume instead of starting again; malware is caught before a finance officer opens a file; there are no storage account keys to leak or rotate.
- **Harder:** **a legitimate receipt can be wrongly flagged as malware**, and then the claimant cannot submit until they upload a different file or the security team reviews it. A staff member could be **delayed in getting their money back** because of a false alarm, which is uncomfortable for an expense tool, so we need a clear manual review process before go-live.
- **Harder:** scanning happens **after** upload and is not instant. Most scans finish in seconds, but a claimant who taps Submit immediately after uploading may get "scan in progress, try again shortly". The Web App must handle this smoothly.
- **Harder:** a signed URL **cannot enforce the 10 MB limit by itself**. Someone could upload a much larger file within the 10-minute window; we only find and delete it when the API checks. Short expiry, one path per URL, and the 24-hour cleanup limit the damage.
- **Different:** Defender for Storage is **charged per GB scanned**, a new running cost for finance to approve. Security now owns a review step in the claims process.

## Alternatives considered
- **Upload receipts through the Claims API:** rejected. Validation would be simpler, but every 10 MB file would pass through the API, tying up App Service instances, making submissions slow on poor connections, and making resumable uploads much harder to build.
- **Store receipt images inside Azure SQL:** rejected. It would make the database large, slow down backups, and raise the cost of the 7-year backup retention chosen in ADR-0003.
- **Our own virus scanner (for example ClamAV) running in a container:** rejected. It means another service to build, patch and keep up to date with virus signatures, and container hosting is exactly what ADR-0002 chose to avoid for this team.
- **Signed URLs signed with the storage account key:** rejected. An account key gives full access to the whole account, and individual URLs cannot be revoked without rotating the key. User delegation SAS is tied to Entra ID and is short-lived.
- **No malware scanning:** rejected. Finance staff open receipts from every employee, so the risk of spreading malware inside BISTEC is too high for the cost saved.
