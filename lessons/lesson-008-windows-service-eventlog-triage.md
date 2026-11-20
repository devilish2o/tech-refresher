# Lesson 008 — Windows: service and Event Log incident triage

**Date:** 2026-11-19  
**Domain:** Windows / PowerShell / observability / troubleshooting  
**Duration:** 30 minutes  
**Environment:** Windows 11, using the built-in Windows PowerShell 5.1; normal, non-elevated session  
**Version assumptions:** The commands use inbox Windows PowerShell 5.1 cmdlets and the Windows CIM/WMI service provider. They also work in PowerShell 7 on Windows, but the examples are written for 5.1.

## Objective

Practise a safe first-response workflow for a Windows service incident:

1. Collect a service's live state and startup configuration with **Get-CimInstance**.
2. Identify automatically configured services that are currently stopped, without assuming they are faulty.
3. Query the System event log efficiently for recent Service Control Manager failures with **Get-WinEvent -FilterHashtable**.
4. Correlate event timestamps and messages with the service being investigated.
5. Produce a small read-only triage summary before considering any remediation.

**Safety boundary:** This is a read-only exercise on the local Windows machine. Do not start, stop, restart, reconfigure or disable services. Do not edit the registry, clear event logs, change log permissions or export sensitive event data.

## 30-minute plan

| Time | Activity |
|---|---|
| 0–4 min | Confirm PowerShell version and read-only prerequisites |
| 4–11 min | Inspect one service and identify stopped automatic services |
| 11–21 min | Query Service Control Manager events from the last seven days |
| 21–26 min | Correlate the events with the selected service and create a triage summary |
| 26–30 min | Self-check and explain the operational diagnosis |

## References

1. [Microsoft Learn — Get-WinEvent (Windows PowerShell 5.1)](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent?view=powershell-5.1) — especially FilterHashtable, MaxEvents and the filtering examples.
2. [Microsoft Learn — Win32_Service class](https://learn.microsoft.com/en-us/windows/win32/cimwin32prov/win32-service) — service state, start mode, delayed start, process ID and exit-code properties.

## Practical exercise

### 1. Preflight — 4 minutes

Open **Windows PowerShell** normally; do not choose **Run as administrator** for this first pass.

Run:

```powershell
$PSVersionTable.PSVersion
$PSVersionTable.PSEdition
$env:COMPUTERNAME
Get-Command Get-CimInstance, Get-Service, Get-WinEvent |
    Select-Object Name, Source
```

Expected:
- Windows PowerShell reports version **5.1** and edition **Desktop**.
- The computer name is displayed.
- All three commands are found.

No installation is required. If you normally use PowerShell 7, that is also fine; just note which edition you used.

### 2. Inspect a service — 7 minutes

Use the Windows DNS Client service as the example. Its internal service name is **Dnscache**.

```powershell
$serviceName = 'Dnscache'

$service = Get-CimInstance -ClassName Win32_Service -Filter "Name='$serviceName'"

if ($null -eq $service) {
    throw "Service '$serviceName' was not found. Choose a name shown by Get-Service."
}

$service |
    Select-Object Name, DisplayName, State, StartMode, DelayedAutoStart,
                  ProcessId, ExitCode, ServiceSpecificExitCode |
    Format-List
```

Read the fields:

- **State:** what the service is doing now (Running, Stopped, or a pending state).
- **StartMode:** its configured startup mode (Auto, Manual, Disabled, and so on).
- **DelayedAutoStart:** whether an automatic service is configured to start after other automatic services.
- **ProcessId:** the process currently hosting the service; it is normally 0 when stopped.
- **ExitCode:** the Windows service exit status. 1066 indicates that a service-specific code may be available in ServiceSpecificExitCode.

Now inspect the service's dependencies using the simpler ServiceController view:

```powershell
$sc = Get-Service -Name $serviceName

[pscustomobject]@{
    Service     = $sc.Name
    Status      = $sc.Status
    DependsOn   = ($sc.ServicesDependedOn.Name -join ', ')
    Dependents  = ($sc.DependentServices.Name -join ', ')
} | Format-List
```

The two views complement each other: **Get-Service** is convenient for status and dependency relationships; **Win32_Service** exposes startup configuration and additional service metadata.

### 3. Find automatic services that are stopped — 3 minutes

```powershell
$autoStopped = @(
    Get-CimInstance -ClassName Win32_Service |
        Where-Object {
            $_.StartMode -eq 'Auto' -and $_.State -eq 'Stopped'
        } |
        Sort-Object Name
)

"Automatic services currently stopped: $($autoStopped.Count)"

$autoStopped |
    Select-Object -First 15 Name, State, StartMode, DelayedAutoStart,
                          ExitCode, ServiceSpecificExitCode |
    Format-Table -AutoSize
```

**Important:** a stopped service is not automatically an incident. Some services stop after completing work, and some are demand/trigger-start services. Check the service's role and its events before deciding it is unhealthy. A service with StartMode = Manual is not expected to run continuously.

### 4. Query recent Service Control Manager failures — 10 minutes

The System log records Service Control Manager (SCM) events. These commonly useful IDs are a starting filter, not a complete list of every possible service problem:

- **7000, 7001:** service start or dependency-start failures.
- **7009, 7011:** startup/control-response timeouts.
- **7023, 7024:** service stopped after reporting an error.
- **7031, 7034:** service terminated unexpectedly.

The message and surrounding timeline matter more than the ID alone.

Build a server-side filter for the last seven days:

```powershell
$since = (Get-Date).AddDays(-7)

$failureIds = @(7000, 7001, 7009, 7011, 7023, 7024, 7031, 7034)

$filter = @{
    LogName      = 'System'
    ProviderName = 'Service Control Manager'
    Id           = $failureIds
    StartTime    = $since
}

$scmFailures = @()

try {
    $scmFailures = @(
        Get-WinEvent -FilterHashtable $filter -ErrorAction Stop
    )
}
catch {
    if ($_.FullyQualifiedErrorId -match 'NoMatchingEventsFound') {
        $scmFailures = @()
    }
    else {
        throw
    }
}

if ($scmFailures.Count -eq 0) {
    Write-Host "No matching SCM failure events were found since $since."
}

if ($scmFailures.Count -gt 0) {
    $scmFailures |
        Select-Object -First 15 TimeCreated, Id, LevelDisplayName,
                              RecordId, Message |
        Format-List
}
```

The two result checks below are deliberately separate **if** blocks rather than an if/else pair. In the interactive PowerShell console, the first block can execute as soon as its closing brace is entered; a separately pasted `else` may then be treated as a new command.

Why this is preferable to reading the entire System log and filtering with Where-Object:
- **FilterHashtable** applies the time, provider and event-ID criteria while Windows retrieves the records.
- **Where-Object** would first receive a much larger set of events and filter them afterwards.
- The try/catch treats “no matching events” as a valid result but lets genuine access or query errors surface.

If the query reports access denied, do not change event-log permissions. On a machine you are authorised to administer, you may repeat this **read-only** query from an elevated PowerShell session. Otherwise, record the access limitation.

### 5. Correlate the selected service and produce a summary — 5 minutes

Search the returned event messages for either the service's internal name or display name:

```powershell
$serviceEvents = @(
    $scmFailures |
        Where-Object {
            $_.Message -match [regex]::Escape($service.Name) -or
            $_.Message -match [regex]::Escape($service.DisplayName)
        }
)

if ($serviceEvents.Count -eq 0) {
    Write-Host "No matching SCM failure message mentions $($service.DisplayName)."
}

if ($serviceEvents.Count -gt 0) {
    $serviceEvents |
        Select-Object TimeCreated, Id, LevelDisplayName, RecordId, Message |
        Format-List
}
```

Create a concise local triage object:

```powershell
[pscustomobject]@{
    CollectedAt             = Get-Date
    Computer                = $env:COMPUTERNAME
    Service                 = $service.Name
    DisplayName             = $service.DisplayName
    State                   = $service.State
    StartMode               = $service.StartMode
    DelayedAutoStart        = $service.DelayedAutoStart
    ProcessId               = $service.ProcessId
    ExitCode                = $service.ExitCode
    SCMFailuresLast7Days    = $scmFailures.Count
    EventsMentioningService = $serviceEvents.Count
} | Format-List
```

If a relevant event exists, note its timestamp and read the **complete message**. Then inspect nearby events in the Application log and the service's own product logs around the same time. A failure event tells you what Windows observed; it does not necessarily identify the underlying cause.

Do not respond to a timeout by automatically increasing ServicesPipeTimeout, changing startup mode or repeatedly restarting the service. First establish whether the service eventually started, which dependency or resource it was waiting for, and whether the failure repeats.

## Expected outcome

By the end, you should have:
- A live snapshot of the DNS Client service's state and configuration.
- A list (possibly empty) of automatic services that are currently stopped.
- A filtered set of recent SCM failure events, or a clear “no matching events” result.
- A summary that distinguishes current service state from historical failure evidence.

Your machine may have no matching SCM failure events in the last seven days. That is a perfectly valid result; the objective is to demonstrate a precise, repeatable diagnostic query, not to manufacture an incident.

## Quick self-check

1. Why is Win32_Service useful in addition to Get-Service?
2. Why is Get-WinEvent -FilterHashtable preferable to reading the entire System log and filtering with Where-Object?
3. Why does StartMode = Auto plus State = Stopped not prove a service is broken?
4. If event 7001 names a failed dependency, which service should you investigate first?
5. Why should you capture the event message and timestamp before restarting a service?

## Production / interview relevance

For a Windows service incident, a strong operator does not begin by restarting the service. First capture the current state, startup mode, process ID and dependencies; then build a timestamped timeline from SCM and Application events. This preserves evidence and helps distinguish a failed dependency, timeout, application exit, account/configuration problem or a harmless stop after completed work.

A concise interview answer is: **“I inspect the service through SCM/CIM, filter the System log by provider, event IDs and incident window, correlate the full message with Application/product logs, then validate the dependency and startup configuration before selecting a controlled remediation.”**

## Completion checkpoint

Report:
- **PowerShell edition/version:** 5.1 Desktop or PowerShell 7.
- **Service snapshot:** whether Dnscache was Running or Stopped, and its StartMode.
- **Stopped automatic services:** count only; do not paste a long inventory.
- **SCM result:** number of matching events, or “none in the last seven days”.
- **One takeaway:** why a stopped service is not automatically a failed service.

Do not paste account names, full binary paths, private event payloads or other sensitive machine details.

## 🧸 For Dummies

Imagine Windows is a big building with lots of workers.

- A **service** is one worker doing a job, like delivering letters.
- **Get-CimInstance** asks, “Is this worker working now? Is it supposed to start by itself?”
- The **Event Log** is the building's diary. It records when a worker could not start, took too long, or suddenly stopped.
- **Get-WinEvent** searches that diary for the useful notes instead of making us read every page.
- We compare the worker's current situation with what the diary says before deciding what went wrong.

A worker who is sitting down is not always broken. Maybe that worker only gets called when somebody needs a special job done. In this lesson, we only look and investigate—we do not tell any workers to stop or start.
