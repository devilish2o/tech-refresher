# Lesson 009 — Control-M: model and validate a dependency-driven batch flow

**Date:** 2026-11-20  
**Domain:** Control-M / workload automation / dependency management / troubleshooting  
**Duration:** 30 minutes  
**Environment:** Windows PowerShell 5.1 on Windows 11; local JSON-only exercise, no Control-M server required  
**Version assumptions:** The object model and CLI references below use BMC Control-M Automation API 9.0.21 documentation. Check the documentation shipped with your installed release before applying syntax to a different version.

## Objective

Build and locally validate a small three-job Control-M flow, then practise diagnosing why a downstream job is waiting.

You will practise:

1. Distinguishing a Folder, a Job, and a Flow in Automation API JSON.
2. Expressing a success-dependent sequence with the Flow object's Sequence array.
3. Validating the JSON structure and intended order locally without deploying or running anything.
4. Distinguishing a Flow dependency from scheduling criteria, events/conditions, resources, hosts and workload policies.
5. Knowing which read-only Automation API query helps explain a waiting job.

**Safety boundary:** This is an offline modelling exercise. Do not run `ctm deploy`, `ctm run`, rerun/force/confirm actions, or make changes to any Control-M environment. The local PowerShell checks validate JSON syntax and the intended sequence; they do not replace Control-M's schema validation.

## 30-minute plan

| Time | Activity |
|---|---|
| 0–4 min | Review the Control-M objects and confirm local tools |
| 4–12 min | Create a three-job flow definition as JSON |
| 12–20 min | Parse and validate the jobs and dependency order in PowerShell |
| 20–25 min | Inject an incorrect sequence in memory and confirm the check rejects it |
| 25–28 min | Review waiting statuses and the diagnostic query |
| 28–30 min | Self-check and record the outcome |

## References

1. [BMC — Control-M Automation API: Folder and Flow objects](https://docs.bmc.com/xwiki/bin/view/Control-M-Orchestration/Control-M/workloadautomation/Control-M-Automation-API/ctmapi9211/Code-Reference/Folder/) — Flow sequence semantics and examples.
2. [BMC — Control-M Automation API: Run service](https://docs.bmc.com/xwiki/bin/view/Control-M-Orchestration/Control-M/workloadautomation/Control-M-Automation-API/ctmapi921/Services/Run-service/) — job statuses and `ctm run job::waitingInfo`.

## Practical exercise

### 1. Preflight — 4 minutes

Use Windows PowerShell 5.1 (not Command Prompt). Confirm the local JSON parser is available:

```powershell
$PSVersionTable.PSVersion
Get-Command ConvertFrom-Json, Set-Content
```

Create the lab directory:

```powershell
$lab = Join-Path $HOME 'labs\tech-refresher\lesson-009-controlm-flow'
New-Item -ItemType Directory -Path $lab -Force | Out-Null
$jsonPath = Join-Path $lab 'controlm-flow.json'
$lab
```

Expected directory:

`C:\Users\ashvi\labs\tech-refresher\lesson-009-controlm-flow`

### 2. Create the Control-M flow definition — 8 minutes

Create the file with three command jobs and one Flow object:

```powershell
@'
{
  "ControlMDependencyLab": {
    "Type": "Folder",
    "Extract": {
      "Type": "Job:Command",
      "Command": "echo EXTRACT_OK"
    },
    "Validate": {
      "Type": "Job:Command",
      "Command": "echo VALIDATE_OK"
    },
    "Publish": {
      "Type": "Job:Command",
      "Command": "echo PUBLISH_OK"
    },
    "Flow": {
      "Type": "Flow",
      "Sequence": ["Extract", "Validate", "Publish"]
    }
  }
}
'@ | Set-Content -LiteralPath $jsonPath -Encoding UTF8

Get-Item -LiteralPath $jsonPath | Select-Object FullName, Length
```

The jobs are deliberately harmless placeholders. They are not being submitted to an Agent or executed.

Read the structure:

- `ControlMDependencyLab` is a Folder containing the job definitions.
- `Extract`, `Validate` and `Publish` are Command jobs.
- `Flow` defines their order.
- In a Flow sequence, each job must end successfully before the next job in that sequence can run.

The resulting dependency chain is:

`Extract (success) → Validate (success) → Publish`

A Flow is not the same thing as a calendar or a `When` scheduling rule. Scheduling criteria determine when work is eligible to be ordered; the Flow determines the required order between jobs.

### 3. Parse and validate locally — 8 minutes

Load the JSON and inspect the sequence:

```powershell
$definition = Get-Content -LiteralPath $jsonPath -Raw | ConvertFrom-Json
$folder = $definition.ControlMDependencyLab
$sequence = @($folder.Flow.Sequence)

$sequence
```

Expected order:

```text
Extract
Validate
Publish
```

Define a small local validator:

```powershell
function Test-ControlMFlow {
    param(
        [Parameter(Mandatory = $true)]
        $Folder
    )

    $expected = @('Extract', 'Validate', 'Publish')
    $actual = @($Folder.Flow.Sequence)

    if (($actual -join '|') -ne ($expected -join '|')) {
        throw "Unexpected Flow sequence: $($actual -join ' -> ')"
    }

    foreach ($jobName in $expected) {
        $property = $Folder.PSObject.Properties[$jobName]

        if ($null -eq $property) {
            throw "Missing job definition: $jobName"
        }

        if ($property.Value.Type -ne 'Job:Command') {
            throw "Unexpected job type for $jobName"
        }

        if ([string]::IsNullOrWhiteSpace($property.Value.Command)) {
            throw "Command is empty for $jobName"
        }
    }

    Write-Host 'PASS: three Command jobs exist in the expected dependency order.'
}

Test-ControlMFlow -Folder $folder
```

Expected:

`PASS: three Command jobs exist in the expected dependency order.`

This validator checks the lab's expected structure. It does not validate every Control-M property or replace the Automation API `ctm build` command in an appropriately configured Workbench/test environment.

### 4. Fault-inject an ordering error — 5 minutes

Change the sequence in memory only. This does not modify the JSON file:

```powershell
$originalSequence = @($folder.Flow.Sequence)
$folder.Flow.Sequence = @('Extract', 'Publish', 'Validate')

try {
    Test-ControlMFlow -Folder $folder
}
catch {
    Write-Host "Expected rejection: $($_.Exception.Message)"
}

$folder.Flow.Sequence = $originalSequence
Test-ControlMFlow -Folder $folder
```

Expected:
- The first validation reports the unexpected order.
- The second validation passes after the original sequence is restored.

This is a simple form of negative testing: prove that the check catches a known-bad dependency order, not just that it accepts the happy path.

### 5. Diagnose a waiting job — 3 minutes

In a connected test environment, `ctm run status <runId>` returns the job statuses and job IDs for a run. A job ID has the form `<Control-M-Server>:<orderId>`.

BMC documents this read-only diagnostic command:

```text
ctm run job::waitingInfo YOUR_SERVER:YOUR_ORDER_ID
```

Replace the placeholder only with a job ID from an authorised test environment. Do not execute it against production as part of this lesson.

The command explains why a waiting job has not started, including missing conditions, resources, hosts, users or workload capacity. Common statuses include:

- **Wait Condition:** a required condition/dependency is not satisfied.
- **Wait Resource:** the required quantitative resource is not available.
- **Wait Host:** an eligible host/Agent is not available.
- **Wait Workload:** workload policy capacity is preventing execution.
- **Wait User:** the required execution user is not currently eligible/available.

Do not treat every waiting status as a job failure. It can be a correct scheduler decision while a prerequisite is outstanding. Diagnose the specific gate before considering recovery.

A safe operational sequence is: inspect the job status and waiting information → identify the unmet prerequisite → verify the predecessor/condition/resource/host/user/policy with its owner → follow the approved recovery procedure → confirm the dependency is satisfied before allowing downstream work to proceed. Do not manually force a downstream job past an unmet business dependency.

## Expected outcome

You should have a local JSON definition containing three Command jobs and a success-dependent Flow, plus a PowerShell validation that accepts the correct sequence and rejects a deliberately incorrect one.

No Control-M environment is modified and no job is executed.

## Quick self-check

1. What does the Folder represent, and what does the Flow represent?
2. What must happen to a job before the next job in a Flow sequence can run?
3. Why is a Flow dependency different from a calendar/When scheduling rule?
4. Which command explains why a particular Control-M job is waiting?
5. Why should you investigate an unmet dependency before forcing a downstream job?

## Production / interview relevance

Control-M incidents often present as a downstream job that has not started. A strong operator distinguishes a scheduler wait from an execution failure, identifies the exact unmet gate, checks the predecessor and operational constraint, and preserves the intended business dependency.

A concise interview answer is: **“I inspect the active job status and its waiting information, trace the predecessor or unmet condition/resource/host/workload gate, verify the upstream outcome, then apply the approved recovery path. I do not force a downstream job past a business dependency without validating the data and process impact.”**

## Completion checkpoint

Report:

- **JSON parse:** passed/failed.
- **Correct sequence:** passed/failed.
- **Negative test:** did the deliberately incorrect sequence get rejected?
- **Restored sequence:** passed/failed.
- **One takeaway:** the difference between a waiting job and a failed job.

Do not share credentials, tokens, private hostnames, production job IDs or customer data.

## 🧸 For Dummies

Imagine three people preparing a parcel:

1. Extract gets the items.
2. Validate checks that the items are correct.
3. Publish sends the parcel.

The third person must wait until the second person says everything is correct. Control-M is the supervisor who makes sure people work in the right order.

If someone is waiting, it does not always mean they failed. Maybe they are waiting for the previous person, the right equipment, an available room or permission. First find out what they are waiting for. Don't tell the last person to send the parcel before it has been checked.
