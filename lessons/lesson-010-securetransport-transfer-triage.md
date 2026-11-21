# Lesson 010 — Axway SecureTransport: correlate an MFT transfer end to end

**Date:** 2026-11-21  
**Domain:** Axway SecureTransport / MFT / observability / incident triage  
**Duration:** 30 minutes  
**Environment:** Windows PowerShell 5.1 on Windows 11; local synthetic-data exercise, no SecureTransport server required  
**Version assumptions:** SecureTransport 5.5 on-premises Administrator Guide, current documentation updated October 2026. UI labels and log locations can differ in earlier releases and deployment models. File Tracking is available on SecureTransport Server, not Edge.

## Objective

Trace a managed-file-transfer incident across the protocol, Transaction Manager (TM), routing and outbound-delivery stages. Practise separating a successful inbound transfer from a failed downstream delivery, and correlate evidence without assuming that one log line tells the whole story.

You will practise:

1. Choosing the correct evidence source: File Tracking, Server Log and the filesystem `xferlog`.
2. Using transfer direction, partner/site, protocol, timestamp, transfer ID and—where available—session ID to build a timeline.
3. Distinguishing protocol acceptance from TM/routing and outbound delivery.
4. Identifying the first failed boundary and writing a concise incident finding.
5. Preserving evidence without cancelling, resubmitting or changing a transfer.

**Safety boundary:** This is a read-only diagnostic lesson using synthetic data. Do not cancel or resubmit transfers, change partner/site settings, edit certificates or keys, change logging levels, or modify a SecureTransport environment. Never paste real partner names, endpoint addresses, filenames, account names, certificates, keys or customer data into the lab.

## 30-minute plan

| Time | Activity |
|---|---|
| 0–4 min | Review SecureTransport evidence sources and transfer stages |
| 4–8 min | Confirm PowerShell and create a disposable lab directory |
| 8–16 min | Build and parse a synthetic transfer timeline |
| 16–23 min | Correlate inbound and outbound legs; identify the failure boundary |
| 23–27 min | Map the local finding to the SecureTransport Administration Tool |
| 27–30 min | Self-check and record the operational conclusion |

## References

1. [Axway — Operations menu in SecureTransport 5.5](https://docs.axway.com/bundle/SecureTransport_55_AdministratorGuide_allOS_en_HTML5/page/Content/AdministratorsGuide/operations/operations_menu.htm) — explains File Tracking, Server Log, Event Queue and Support Tool. File Tracking exposes transfer status/details and is available on SecureTransport Server only.
2. [Axway — SecureTransport 5.5 log file list](https://docs.axway.com/bundle/SecureTransport_55_AdministratorGuide_allOS_en_HTML5/page/Content/AdministratorsGuide/server_logs/r_st_logFileList.htm) — identifies `xferlog` under `<FILEDRIVEHOME>/var/logs` and protocol/service log locations.

## Practical exercise

### 1. Know where to look — 4 minutes

For a real incident, use this evidence map:

| Evidence source | What it answers | Read-only focus |
|---|---|---|
| **Operations → File Tracking** (Server only) | What transfer was recorded, its direction/status/attributes and detailed transfer information | Search a narrow time range and inspect the relevant transfer details |
| **Operations → Server Log** | What TM and protocol components reported around the same time | Filter by time, component, severity and available session ID |
| **`<FILEDRIVEHOME>/var/logs/xferlog`** | Transfer records for FTP(S), HTTP(S), SFTP, AS2, Connect:Direct, Folder Monitor and PeSIT | Search a bounded time window; do not dump or share the full file |
| Protocol-specific service logs | What the protocol daemon observed | Inspect the relevant daemon and time window only |

On SecureTransport Server, Server Log records can include a `SESSIONID`; this field is not available in the same way on Edge. Use it when present, but do not assume every log source has the same correlation fields.

A useful investigation order is:

**Inbound protocol → inbound transfer result → TM/event processing → routing decision → outbound protocol/site → receiver acknowledgement (where applicable).**

Do not assume that a successful inbound receipt means the file was delivered to the downstream partner. A single business file flow can involve separate inbound and outbound transfer records.

### 2. Prepare a local lab — 4 minutes

Open Windows PowerShell 5.1. Confirm the parser is available:

```powershell
$PSVersionTable.PSVersion
Get-Command Import-Csv, Set-Content
```

Create the lab directory:

```powershell
$lab = Join-Path $HOME 'labs\tech-refresher\lesson-010-securetransport-triage'
New-Item -ItemType Directory -Path $lab -Force | Out-Null
$csvPath = Join-Path $lab 'transfer-events.csv'
$lab
```

Expected directory:

`C:\Users\ashvi\labs\tech-refresher\lesson-010-securetransport-triage`

### 3. Create a synthetic transfer timeline — 8 minutes

The following CSV is deliberately **normalised lab data**. It is not a literal SecureTransport `xferlog` or Server Log export, and its example status labels are analyst-friendly values rather than a claim about every native UI status.

It represents two business files. The first arrives successfully but fails on its outbound SFTP leg. The second succeeds end to end.

```powershell
@'
TimestampUtc,FileName,TransferId,Direction,Partner,Protocol,LogArea,Component,SessionId,Outcome,Message
2026-11-21T09:00:00.100Z,orders_1042.csv,TR-IN-1042,IN,PARTNER_A,SFTP,FileTracking,SSHd,sess-in-1042,START,Inbound SFTP session accepted
2026-11-21T09:00:02.200Z,orders_1042.csv,TR-IN-1042,IN,PARTNER_A,SFTP,FileTracking,SSHd,sess-in-1042,ACTIVE,Receiving inbound file
2026-11-21T09:00:04.000Z,orders_1042.csv,TR-IN-1042,IN,PARTNER_A,SFTP,FileTracking,SSHd,sess-in-1042,RECEIVED,Inbound transfer completed
2026-11-21T09:00:04.100Z,orders_1042.csv,TR-OUT-1042,OUT,ERP_SFTP,SFTP,ServerLog,TM,sess-in-1042,INFO,Route ORDERS_TO_ERP selected
2026-11-21T09:00:04.500Z,orders_1042.csv,TR-OUT-1042,OUT,ERP_SFTP,SFTP,ServerLog,SSHd,sess-out-1042,ERROR,Remote SSH host key verification failed
2026-11-21T09:00:04.520Z,orders_1042.csv,TR-OUT-1042,OUT,ERP_SFTP,SFTP,FileTracking,SSHd,sess-out-1042,ERROR,Outbound transfer failed
2026-11-21T10:00:00.100Z,orders_1043.csv,TR-IN-1043,IN,PARTNER_A,SFTP,FileTracking,SSHd,sess-in-1043,START,Inbound SFTP session accepted
2026-11-21T10:00:02.000Z,orders_1043.csv,TR-IN-1043,IN,PARTNER_A,SFTP,FileTracking,SSHd,sess-in-1043,RECEIVED,Inbound transfer completed
2026-11-21T10:00:02.100Z,orders_1043.csv,TR-OUT-1043,OUT,ERP_SFTP,SFTP,ServerLog,TM,sess-in-1043,INFO,Route ORDERS_TO_ERP selected
2026-11-21T10:00:03.100Z,orders_1043.csv,TR-OUT-1043,OUT,ERP_SFTP,SFTP,ServerLog,SSHd,sess-out-1043,INFO,Remote SSH handshake completed
2026-11-21T10:00:05.200Z,orders_1043.csv,TR-OUT-1043,OUT,ERP_SFTP,SFTP,FileTracking,SSHd,sess-out-1043,SUCCESS,Outbound transfer completed
'@ | Set-Content -LiteralPath $csvPath -Encoding UTF8

Get-Item -LiteralPath $csvPath | Select-Object FullName, Length
```

The synthetic `TransferId` and `SessionId` values are intentionally different for the inbound and outbound legs. In a real investigation, do not assume a single identifier spans every stage. Correlate using the fields actually available: transfer details, filename, partner/account, direction, protocol, timestamps, route context and session ID where present.

### 4. Parse and correlate the evidence — 7 minutes

Import and sort the events chronologically:

```powershell
$events = Import-Csv -LiteralPath $csvPath

$timeline = @(
    $events |
        Sort-Object {
            [datetimeoffset]::Parse($_.TimestampUtc).UtcDateTime
        }
)

$timeline |
    Format-Table TimestampUtc, FileName, TransferId, Direction,
                 LogArea, Component, Outcome, Message -AutoSize -Wrap
```

Now isolate the failed business file and separate its inbound and outbound legs:

```powershell
$case = @(
    $timeline |
        Where-Object { $_.FileName -eq 'orders_1042.csv' }
)

$inbound = @(
    $case |
        Where-Object { $_.Direction -eq 'IN' }
)

$outbound = @(
    $case |
        Where-Object { $_.Direction -eq 'OUT' }
)

$inboundLast = $inbound | Select-Object -Last 1
$outboundLast = $outbound | Select-Object -Last 1
$outboundFailure = $outbound |
    Where-Object { $_.Outcome -eq 'ERROR' } |
    Select-Object -First 1

[pscustomobject]@{
    FileName             = 'orders_1042.csv'
    InboundFinalOutcome  = $inboundLast.Outcome
    OutboundFinalOutcome = $outboundLast.Outcome
    FirstOutboundError   = $outboundFailure.Message
    FailureBoundary      = 'Outbound SFTP session, after route selection'
} | Format-List
```

Expected findings:

- Inbound final outcome: `RECEIVED`.
- Outbound final outcome: `ERROR`.
- First outbound error: remote SSH host key verification failed.
- Failure boundary: outbound SFTP session, after the route was selected.

Compare the successful control file:

```powershell
$timeline |
    Where-Object { $_.FileName -eq 'orders_1043.csv' } |
    Format-Table TimestampUtc, TransferId, Direction, Component,
                 Outcome, Message -AutoSize -Wrap
```

Expected: the inbound leg is received, the route is selected, the outbound SSH handshake succeeds, and the outbound leg completes.

### 5. Translate the result into a real SecureTransport investigation — 4 minutes

For a real incident, replace the synthetic CSV with evidence gathered under your authorised access:

1. In **File Tracking**, search the narrow incident window and identify the inbound and outbound transfer records. Record transfer ID, direction, partner/account, protocol, filename, status and timestamps. Open details before drawing a conclusion.
2. In **Server Log**, inspect the same time window. Filter to the relevant components (for example TM and SSH/protocol component). On SecureTransport Server, use the session ID when it is available.
3. Check the routing/application evidence between receipt and outbound initiation. If an event appears stuck, inspect the Event Queue read-only before proposing recovery.
4. For an outbound SFTP host-key failure, verify the configured partner/site host key and the remote server's published fingerprint through the approved change/partner process. Do not disable host-key verification to make a transfer pass.
5. Preserve the original failure evidence. Do not resubmit until the cause is understood and the business owner confirms whether duplicate delivery is safe.

If the issue is on Edge, remember that File Tracking is a Server-only Operations feature. Correlate the Edge protocol evidence with the backend Server/TM evidence using the available timestamps and identifiers; do not expect every field to exist on both nodes.

## Expected outcome

You should have a local `transfer-events.csv`, a chronological view of the events, and a concise diagnosis showing that one inbound transfer succeeded but its outbound SFTP leg failed during host-key verification. The second file provides a successful comparison.

No SecureTransport server, partner account, route, certificate, key, transfer or production system is accessed or changed.

## Quick self-check

1. Which SecureTransport Operations page shows transfer status and attributes?
2. Where is the default `xferlog` location under `FILEDRIVEHOME`?
3. Why is an inbound `RECEIVED` result insufficient proof of downstream delivery?
4. Which components would you correlate after inbound receipt but before outbound delivery?
5. Why must host-key verification not be disabled as a quick fix?

## Production / interview relevance

A strong MFT operator separates transport acceptance, TM processing, routing and outbound delivery. They correlate the relevant transfer records and component logs, identify the first failed boundary, preserve evidence, and avoid blind resubmission that could create duplicate business processing.

A concise interview answer is: **“I trace the file from inbound protocol completion through TM/routing to the outbound protocol and acknowledgement. I correlate the separate transfer records using timestamps, partner, filename, direction and available session IDs, then remediate only the failing boundary under the approved partner/change process.”**

## Completion checkpoint

Report:

- **CSV import and sort:** passed/failed.
- **Inbound result for orders_1042.csv:** value shown.
- **Outbound result for orders_1042.csv:** value shown.
- **Failure boundary:** your one-sentence diagnosis.
- **One takeaway:** why inbound success does not prove end-to-end delivery.

Do not paste real SecureTransport logs, partner identifiers, hostnames, filenames, account names, certificates, keys or customer data.

## 🧸 For Dummies

Imagine a parcel company:

1. The first driver picks up the parcel. That is the inbound transfer.
2. The sorting centre decides where it must go. That is TM and routing.
3. A second driver takes it to the final address. That is the outbound transfer.

The first driver can do a perfect job, but the parcel can still fail to reach its destination if the second driver cannot get through the gate.

So we check each part of the journey. We do not declare the whole delivery successful just because the parcel arrived at the sorting centre. 🧸
