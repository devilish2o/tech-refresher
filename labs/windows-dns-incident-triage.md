# Evening lab: Windows DNS incident triage and evidence correlation

**Duration:** 30 minutes  
**Status:** Delivered; user completion checkpoint pending  
**Environment:** Windows 11, Windows PowerShell 5.1, normal non-administrator session. No installations.

## Why this lab

This is a focused extension of Lesson 008 (read-only service and Event Log triage). It adds DNS resolution and outbound TCP evidence, correlates DNS Client and Service Control Manager events, and creates a small sanitised diagnostic report. It is a single-session extension, not a new multi-evening series.

## Safety boundaries

- Read-only diagnostics only.
- Do not start, stop, restart or reconfigure services.
- Do not edit the registry, enable/disable event logs, clear logs or change firewall/DNS settings.
- Use only the reserved example.com and .invalid names for the network tests.
- The report deliberately excludes event messages and DNS payloads. Keep the output local; report only the summary fields requested in the lesson.

## Timed plan

| Minutes | Task |
|---|---|
| 0–4 | Confirm PowerShell 5.1 and inspect Dnscache |
| 4–10 | Compare a known-good DNS lookup with a controlled negative lookup |
| 10–15 | Test outbound TCP/443 without changing the machine |
| 15–24 | Collect and correlate DNS Client / Service Control Manager event metadata |
| 24–28 | Validate the JSON/CSV evidence bundle |
| 28–30 | Write a short incident conclusion |

## 1. Prepare a lab directory

Open ordinary Windows PowerShell 5.1 (not elevated):

~~~powershell
$Lab = Join-Path $HOME 'labs\tech-refresher\evening-windows-dns-triage'
New-Item -ItemType Directory -Path $Lab -Force | Out-Null
Set-Location $Lab

$PSVersionTable.PSVersion
$PSVersionTable.PSEdition
~~~

Expected: Windows PowerShell 5.1, Desktop edition. Use vim .\Collect-DnsTriage.ps1 to create the script below.

## 2. Create the read-only collector

~~~powershell
$ErrorActionPreference = 'Stop'

$Since = (Get-Date).AddHours(-24)
$OutDir = Join-Path $PSScriptRoot 'output'
New-Item -ItemType Directory -Path $OutDir -Force | Out-Null

$os = Get-CimInstance Win32_OperatingSystem |
    Select-Object Caption, Version, BuildNumber

$dnsService = Get-CimInstance Win32_Service -Filter "Name='Dnscache'" |
    Select-Object Name, State, StartMode, ProcessId

$requiredServices = @(
    Get-Service -Name Dnscache -RequiredServices -ErrorAction SilentlyContinue |
    Select-Object Name, Status
)

$dependentServices = @(
    Get-Service -Name Dnscache -DependentServices -ErrorAction SilentlyContinue |
    Select-Object Name, Status
)

# Positive control: a reserved documentation domain.
$dnsRecords = @()
$dnsStatus = 'PASS'
try {
    $dnsRecords = @(
        Resolve-DnsName -Name 'example.com' -Type A -DnsOnly -QuickTimeout -ErrorAction Stop |
        Select-Object Name, Type, IPAddress, TTL
    )
    if ($dnsRecords.Count -eq 0) { $dnsStatus = 'NO A RECORDS RETURNED' }
} catch {
    $dnsStatus = 'FAIL: ' + $_.Exception.GetType().Name
}

# Negative control: .invalid is reserved and should not resolve publicly.
$negativeName = 'no-such-host-' + (Get-Date -Format 'yyyyMMddHHmmss') + '.invalid'
$negativeStatus = 'NOT TESTED'
try {
    $negativeRecords = @(
        Resolve-DnsName -Name $negativeName -DnsOnly -QuickTimeout -ErrorAction Stop
    )
    if ($negativeRecords.Count -eq 0) {
        $negativeStatus = 'EXPECTED: no records returned'
    } else {
        $negativeStatus = 'UNEXPECTED: records returned'
    }
} catch {
    $negativeStatus = 'EXPECTED DNS FAILURE: ' + $_.Exception.GetType().Name
}

# Read-only outbound TCP check. A failure can be caused by network policy,
# routing, DNS or remote availability; it does not identify the cause alone.
$tcpSummary = $null
try {
    $tcp = Test-NetConnection -ComputerName 'example.com' -Port 443 -InformationLevel Detailed -WarningAction SilentlyContinue
    $tcpSummary = [pscustomobject]@{
        ComputerName     = $tcp.ComputerName
        RemoteAddress    = [string]$tcp.RemoteAddress
        RemotePort       = $tcp.RemotePort
        InterfaceAlias   = $tcp.InterfaceAlias
        SourceAddress    = [string]$tcp.SourceAddress
        TcpTestSucceeded = $tcp.TcpTestSucceeded
    }
} catch {
    $tcpSummary = [pscustomobject]@{
        ComputerName = 'example.com'
        ErrorType = $_.Exception.GetType().Name
    }
}

# Capture event metadata only. Do not export event Message or raw event XML.
$events = @()
$eventQueryStatus = @()
$filters = @(
    @{
        ProviderName = 'Microsoft-Windows-DNS-Client'
        Id = @(1014)
    },
    @{
        ProviderName = 'Service Control Manager'
        Id = @(7000, 7001, 7009, 7011, 7023, 7024, 7031, 7034)
    }
)

foreach ($filter in $filters) {
    $query = @{
        LogName = 'System'
        StartTime = $Since
        ProviderName = $filter.ProviderName
        Id = $filter.Id
    }

    try {
        $found = @(Get-WinEvent -FilterHashtable $query -ErrorAction Stop)
        foreach ($event in $found) {
            $events += [pscustomobject]@{
                TimeCreated = $event.TimeCreated.ToString('o')
                Id = $event.Id
                ProviderName = $event.ProviderName
                LevelDisplayName = $event.LevelDisplayName
                RecordId = $event.RecordId
            }
        }
        $eventQueryStatus += $filter.ProviderName + ': query succeeded'
    } catch {
        if ($_.Exception.Message -match 'No events were found') {
            $eventQueryStatus += $filter.ProviderName + ': no matching events'
        } else {
            $eventQueryStatus += $filter.ProviderName + ': query error ' + $_.Exception.GetType().Name
        }
    }
}

$report = [pscustomobject]@{
    CapturedAt = (Get-Date).ToString('o')
    ComputerName = $env:COMPUTERNAME
    PowerShellVersion = $PSVersionTable.PSVersion.ToString()
    OperatingSystem = $os
    LookbackStart = $Since.ToString('o')
    Dnscache = $dnsService
    RequiredServices = $requiredServices
    DependentServices = $dependentServices
    PositiveDnsTest = [pscustomobject]@{
        Name = 'example.com'
        Status = $dnsStatus
        Records = $dnsRecords
    }
    NegativeDnsTest = [pscustomobject]@{
        Name = $negativeName
        Status = $negativeStatus
    }
    Tcp443Test = $tcpSummary
    EventQueryStatus = $eventQueryStatus
    EventCount = $events.Count
}

$summaryPath = Join-Path $OutDir 'summary.json'
$eventsPath = Join-Path $OutDir 'events.csv'

$report | ConvertTo-Json -Depth 6 |
    Set-Content -Path $summaryPath -Encoding UTF8

if ($events.Count -gt 0) {
    $events |
        Sort-Object TimeCreated |
        Export-Csv -Path $eventsPath -NoTypeInformation -Encoding UTF8
}

$report |
    Select-Object ComputerName, PowerShellVersion, Dnscache,
        PositiveDnsTest, NegativeDnsTest, Tcp443Test, EventCount |
    Format-List

Write-Host "Summary: $summaryPath"
if ($events.Count -gt 0) {
    Write-Host "Event metadata: $eventsPath"
} else {
    Write-Host 'No matching event metadata was returned; inspect EventQueryStatus in summary.json.'
}
~~~

Run it from the lab directory:

~~~powershell
& .\Collect-DnsTriage.ps1
~~~

If script execution is blocked by policy, do not change the machine policy or bypass an organisation's controls; run the diagnostic sections interactively instead.

## 3. Expected result and validation

The script creates:

- output\summary.json: host, service, DNS, TCP and event-query summary.
- output\events.csv: event metadata only, when matching events exist.

Validate the output:

~~~powershell
Test-Path .\output\summary.json
Get-Content .\output\summary.json -Raw | ConvertFrom-Json |
    Select-Object ComputerName, PowerShellVersion, EventCount

if (Test-Path .\output\events.csv) {
    Import-Csv .\output\events.csv |
        Sort-Object TimeCreated |
        Format-Table TimeCreated, ProviderName, Id, LevelDisplayName -AutoSize
}
~~~

Expected:
- Dnscache state and start mode are reported without changing them.
- The positive DNS test usually returns one or more A records.
- The .invalid test reports an expected DNS failure or no records.
- TCP/443 reports TcpTestSucceeded as true or false.
- Event metadata is sorted chronologically; zero events is a valid result, not proof that no incident occurred.

## 4. Interpret the evidence

Use this diagnostic sequence:

1. **Service:** Is Dnscache present and running? Is its start mode expected?
2. **DNS:** Does example.com resolve? Does the reserved negative test fail as expected?
3. **TCP:** Does the outbound TCP handshake to port 443 succeed?
4. **Events:** Are DNS Client 1014 or Service Control Manager failure events present in the same time window?
5. **Correlation:** Do timestamps overlap? Is there evidence for a DNS issue, a service issue, a network-path issue, or insufficient evidence?

Do not claim root cause from one failed check. For example, TCP/443 failure after successful DNS resolution suggests a later network/remote-connectivity stage, not necessarily a DNS failure.

## Troubleshooting

- Resolve-DnsName unavailable: confirm Windows PowerShell 5.1 on Windows 11 and run Get-Command Resolve-DnsName.
- Test-NetConnection unavailable: run Get-Command Test-NetConnection; do not install modules for this lab.
- No event results: check EventQueryStatus; the selected events may simply not have occurred in the last 24 hours.
- Access denied: preserve the error type; do not elevate just to complete this read-only exercise.
- Network test fails: distinguish name resolution, route, TCP handshake and remote service. Do not change DNS, proxy or firewall settings.

## Checkpoint to record

Report only:
- PowerShell version / edition.
- Dnscache state and StartMode.
- Positive DNS result: PASS / FAIL.
- Negative DNS control: EXPECTED FAILURE / UNEXPECTED RESULT.
- TCP/443: true / false / error.
- DNS Client 1014 count and SCM failure-event count (metadata only).
- One-sentence conclusion and confidence (1–5).

Do not paste the JSON/CSV files, event messages, IP addresses, internal hostnames or any private payload.

## Operational relevance

This turns a one-off service check into a repeatable incident evidence bundle. In operations and interviews, it demonstrates layered diagnosis, time-window correlation, non-invasive evidence collection and disciplined separation of symptoms from root cause.

## Official references

- [Get-WinEvent — Windows PowerShell 5.1](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent?view=powershell-5.1)
- [Resolve-DnsName — Windows 10 / PowerShell reference](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname?view=win10-ps)
