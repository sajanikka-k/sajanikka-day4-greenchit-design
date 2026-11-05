# Sequence — Submit and Approve a Claim

Day 4 — Deliverable 2

This diagram follows one claim through GreenChit over time. The lifelines use exactly the same names as the container diagram in `sajanikka-day4-greenchit-design.md`, so each lifeline can be matched to one box.

| Lifeline | Element in the architecture pack |
|----------|----------------------------------|
| Claimant (staff) | Person |
| Web App | Container: React SPA |
| Claims API | Container: ASP.NET Core 8 on App Service |
| Claims Database | Container: Azure SQL Database |
| Receipt Storage | Container: Azure Blob Storage |
| Message Queue | Container: Azure Service Bus |
| Notification Worker | Container: WebJob on App Service |
| Microsoft Teams | External system: Adaptive Card webhook |
| Line manager | Person |

**How to read the arrows**

| Arrow | Meaning | Example |
|-------|---------|---------|
| Solid line, filled arrowhead (`->>`) | **Synchronous** call: the caller waits for an answer | `POST /claims/{id}/submit` |
| Dashed line, filled arrowhead (`-->>`) | **Return** of a synchronous call | `200 OK` |
| Solid line, open arrowhead (`-)`) | **Asynchronous** message: the sender does not wait | `Publish claim.submitted` |
| Line ending in a cross (`-x`) | A call that **failed** | Upload interrupted |

Tall thin bars on a lifeline (activation bars) show when the Claims API is working on a request.

**One difference from the brief's starting example:** in this design the browser uploads receipts **straight to Receipt Storage** with signed URLs, before the claim is submitted, rather than the API uploading them. That keeps 10 MB files out of the API and lets an interrupted upload resume (see the container diagram). So a claim is first created as a **Draft**, and only becomes **Submitted** once its receipts are confirmed.

## Happy path

```mermaid
sequenceDiagram
  autonumber
  actor U as Claimant (staff)
  participant FE as Web App
  participant API as Claims API
  participant DB as Claims Database
  participant BLOB as Receipt Storage
  participant SB as Message Queue
  participant WK as Notification Worker
  participant TEAMS as Microsoft Teams
  actor MGR as Line manager

  U->>FE: Fill in claim, attach 2 receipt photos
  FE->>+API: POST /claims (JWT)
  API->>DB: INSERT claim (status=Draft) + audit row
  API-->>-FE: 201 Created (claimId)
  FE->>+API: POST /claims/{id}/receipts/upload-urls (JWT)
  API-->>-FE: 200 OK (2 signed URLs, valid 10 min)
  FE->>BLOB: PUT receipt blocks (signed URL)
  BLOB-->>FE: 201 Created
  Note over BLOB: Defender for Storage scans each new receipt
  U->>FE: Tap "Submit claim"
  FE->>+API: POST /claims/{id}/submit (JWT)
  API->>BLOB: Check receipts exist and scan is clean
  BLOB-->>API: 2 receipts, no threats found
  API->>DB: UPDATE status=Submitted + audit row + outbox event (one transaction)
  API-->>-FE: 200 OK (status=Submitted)
  FE-->>U: "Claim submitted"
  Note over U,API: Steps 9 to 15 (tap Submit to confirmation) must take under 1.5 s (p95)
  API-)SB: Publish claim.submitted (from outbox)
  SB-)WK: Deliver claim.submitted
  WK->>TEAMS: POST Adaptive Card (HTTPS webhook)
  TEAMS-->>WK: 200 OK
  TEAMS-)MGR: Approval card in Teams chat
  MGR->>FE: Open claim from card, tap "Approve"
  FE->>+API: POST /claims/{id}/approve (JWT)
  Note right of API: Auth and Access Policy checks the caller is this claimant's line manager
  API->>DB: UPDATE status=Approved + audit row + outbox event (one transaction)
  API-->>-FE: 200 OK (status=Approved)
  FE-->>MGR: "Claim approved"
  API-)SB: Publish claim.decided (from outbox)
  SB-)WK: Deliver claim.decided
  WK->>TEAMS: POST card "Your claim was approved"
  TEAMS-->>WK: 200 OK
  TEAMS-)U: Approval message in Teams chat
```

**What happens, in order**

1. **Create as Draft (steps 1–4).** The Web App creates the claim. The API saves it as **Draft** and writes the first audit row.
2. **Upload receipts (steps 5–8).** The API hands out short-lived signed URLs, and the browser uploads the photos directly to Receipt Storage. Defender for Storage scans each one.
3. **Submit (steps 9–15).** The API confirms the receipts exist and are clean, then changes the status to **Submitted**. The status change, its audit row and the "claim submitted" event are saved in **one database transaction**, so none of them can be lost without the others. The claimant sees "Claim submitted" right away.
4. **Notify the manager (steps 16–20), asynchronously.** The event goes through the Message Queue to the Notification Worker, which posts an Adaptive Card to Teams. The claimant never waits for this part, which keeps submission under 1.5 seconds.
5. **Approve (steps 21–25).** The manager opens the claim and approves it. The API checks the manager is allowed to approve this claim, saves **Approved** with an audit row, and answers `200 OK`.
6. **Tell the claimant (steps 26–30),** again asynchronously through the queue and Teams.

## Error path — receipt upload fails after the claim was created

```mermaid
sequenceDiagram
  autonumber
  actor U as Claimant (staff)
  participant FE as Web App
  participant API as Claims API
  participant DB as Claims Database
  participant BLOB as Receipt Storage

  U->>FE: Fill in claim, attach 3 receipt photos
  FE->>+API: POST /claims (JWT)
  API->>DB: INSERT claim (status=Draft) + audit row
  API-->>-FE: 201 Created (claimId)
  FE->>+API: POST /claims/{id}/receipts/upload-urls (JWT)
  API-->>-FE: 200 OK (3 signed URLs, valid 10 min)
  FE->>BLOB: PUT receipts 1 and 2 (signed URLs)
  BLOB-->>FE: 201 Created
  FE-xBLOB: PUT receipt 3 (Wi-Fi drops mid-upload)
  loop Up to 3 retries, waiting 1 s, 2 s, then 4 s
    FE-xBLOB: PUT only the missing blocks of receipt 3
  end
  FE-->>U: "Receipt 3 has not uploaded yet"
  U->>FE: Tap "Submit claim"
  FE->>+API: POST /claims/{id}/submit (JWT)
  API->>BLOB: Check receipts exist and scan is clean
  alt all receipts uploaded and clean
    BLOB-->>API: 3 receipts, no threats found
    API->>DB: UPDATE status=Submitted + audit row + outbox event
    API-->>FE: 200 OK (status=Submitted)
  else receipt upload failed
    BLOB-->>API: Receipt 3 missing or incomplete
    API->>DB: UPDATE status=Draft, reason="upload_failed" + audit row
    API-->>FE: 502 Bad Gateway (Retry-After: 5)
    FE-->>U: "Receipt 3 didn't upload. Your claim is saved as a Draft."
  else receipt fails the malware scan
    BLOB-->>API: Receipt 2 flagged as malicious
    API->>DB: UPDATE status=Draft, reason="receipt_blocked" + audit row
    API-->>FE: 422 Unprocessable Entity (receipt 2 rejected)
    FE-->>U: "Receipt 2 was blocked. Please attach a new photo."
  end
  deactivate API
  opt Claimant retries after the Retry-After time
    FE->>+API: POST /claims/{id}/receipts/upload-urls (JWT)
    API-->>-FE: 200 OK (new signed URL for receipt 3)
    FE->>BLOB: PUT receipt 3 (signed URL)
    BLOB-->>FE: 201 Created
    FE->>+API: POST /claims/{id}/submit (JWT)
    API-->>-FE: 200 OK (status=Submitted)
    Note over FE,DB: From here the happy path continues at step 16 (notify the manager)
  end
```

**What happens, in order**

1. **The claim already exists (steps 1–6).** It was created as a Draft, so nothing the claimant typed is lost when the upload fails.
2. **The upload is interrupted (steps 7–11).** The connection drops while receipt 3 is uploading. The Web App retries **only the missing blocks**, up to 3 times with growing waits (1 s, 2 s, 4 s), and tells the claimant it hasn't finished yet.
3. **Submit is checked (steps 12–14).** When the claimant taps Submit, the API asks Receipt Storage whether every receipt is there and clean. The `alt` fragment shows the three possible answers:
   - **All uploaded and clean:** normal submission, as in the happy path.
   - **Upload failed:** the claim stays **Draft** with `reason="upload_failed"`, an audit row records it, and the API returns `502 Bad Gateway` with `Retry-After: 5`, matching the error contract in the brief.
   - **Malware found:** the claim stays **Draft** with `reason="receipt_blocked"`, and the API returns `422`, because retrying the same file will not help: the claimant must attach a new photo.
4. **Recovery (`opt` fragment).** After the Retry-After time, the Web App asks for a new signed URL (the old one may have expired), uploads receipt 3, and submits again. The flow then joins the happy path at step 16.

## Exported images

The required export, with both paths in one image:

![Sequence diagram — happy path and error path](sequence-submit-approve.png)

The same two diagrams exported separately from Mermaid Chart, which confirms both blocks above render without syntax errors:

| File | Content |
|------|---------|
| `sequence-happy-path.png` | Happy path, steps 1–30 |
| `sequence-error-path.png` | Error path, steps 1–31, with the `loop`, `alt` and `opt` fragments |
