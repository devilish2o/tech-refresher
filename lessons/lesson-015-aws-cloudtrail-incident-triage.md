# Lesson 015 — AWS CloudTrail incident triage: S3 data events and AccessDenied

**Domain:** AWS operations / security / observability  
**Duration:** 30 minutes  
**Assumptions:** Windows PowerShell 5.1 or PowerShell 7. No AWS account, CLI, credentials, or cloud resources are required. This exercise uses synthetic CloudTrail-shaped records only.

## Objective

Triage S3 object-access events and decide whether observed denials are consistent with the least-privilege policy from Lesson 014. Practise distinguishing CloudTrail management events from data events, extracting useful fields, and avoiding the mistake of treating every AccessDenied as an IAM identity-policy failure.

## Exact items to learn

- Management events record control-plane operations; data events record operations on or within resources.
- S3 GetObject, PutObject and DeleteObject are object-level data events.
- CloudTrail Event history shows the last 90 days of management events in one Region; it does not show data events.
- Trails and event data stores do not log data events by default. S3 object data-event logging must be explicitly configured and incurs additional charges.
- Inspect eventTime (UTC), userIdentity, eventSource, eventName, awsRegion, sourceIPAddress, userAgent, requestParameters, errorCode, errorMessage, eventID and requestID.
- An assumed-role session ARN identifies the session; userIdentity.sessionContext.sessionIssuer.arn identifies the IAM role behind it.

## Timed plan

| Time | Task |
|---:|---|
| 0–5 min | Review management versus data events and the Event history limitation |
| 5–10 min | Identify key CloudTrail fields and UTC timestamp |
| 10–18 min | Create a synthetic three-event fixture |
| 18–26 min | Parse failures and correlate identity/source |
| 26–30 min | Explain the likely cause, alternatives and operational next step |

## Exercise — offline PowerShell triage

### 1. Create the lab directory

Run in Windows PowerShell:

~~~powershell
$Lab = "$env:USERPROFILE/labs/tech-refresher/lesson-015-cloudtrail"
New-Item -ItemType Directory -Path $Lab -Force | Out-Null
Set-Location $Lab
~~~

### 2. Create three synthetic S3 data events

The IP address below is from a documentation-only test range. The account ID, role and request identifiers are fictitious.

~~~powershell
$Sample = @'
[
  {
    "eventVersion": "1.11",
    "eventTime": "2026-11-26T18:10:59Z",
    "eventSource": "s3.amazonaws.com",
    "eventName": "GetObject",
    "awsRegion": "ca-central-1",
    "sourceIPAddress": "203.0.113.25",
    "userAgent": "aws-cli/2.x",
    "userIdentity": {
      "type": "AssumedRole",
      "principalId": "AROAEXAMPLE:reporting-job",
      "arn": "arn:aws:sts::111122223333:assumed-role/reports-reader/reporting-job",
      "accountId": "111122223333",
      "sessionContext": {
        "sessionIssuer": {
          "type": "Role",
          "arn": "arn:aws:iam::111122223333:role/reports-reader",
          "accountId": "111122223333",
          "userName": "reports-reader"
        }
      }
    },
    "requestParameters": {
      "bucketName": "example-company-reports",
      "key": "reports/2026/summary.csv"
    },
    "responseElements": null,
    "eventCategory": "Data",
    "managementEvent": false,
    "eventType": "AwsApiCall",
    "readOnly": true,
    "eventID": "11111111-1111-4111-8111-111111111111",
    "requestID": "SYNTH-REQUEST-001"
  },
  {
    "eventVersion": "1.11",
    "eventTime": "2026-11-26T18:11:03Z",
    "eventSource": "s3.amazonaws.com",
    "eventName": "GetObject",
    "awsRegion": "ca-central-1",
    "sourceIPAddress": "203.0.113.25",
    "userAgent": "aws-cli/2.x",
    "userIdentity": {
      "type": "AssumedRole",
      "principalId": "AROAEXAMPLE:reporting-job",
      "arn": "arn:aws:sts::111122223333:assumed-role/reports-reader/reporting-job",
      "accountId": "111122223333",
      "sessionContext": {
        "sessionIssuer": {
          "type": "Role",
          "arn": "arn:aws:iam::111122223333:role/reports-reader",
          "accountId": "111122223333",
          "userName": "reports-reader"
        }
      }
    },
    "requestParameters": {
      "bucketName": "example-company-reports",
      "key": "finance/payroll.csv"
    },
    "responseElements": null,
    "errorCode": "AccessDenied",
    "errorMessage": "Access Denied",
    "eventCategory": "Data",
    "managementEvent": false,
    "eventType": "AwsApiCall",
    "readOnly": true,
    "eventID": "22222222-2222-4222-8222-222222222222",
    "requestID": "SYNTH-REQUEST-002"
  },
  {
    "eventVersion": "1.11",
    "eventTime": "2026-11-26T18:11:05Z",
    "eventSource": "s3.amazonaws.com",
    "eventName": "PutObject",
    "awsRegion": "ca-central-1",
    "sourceIPAddress": "203.0.113.25",
    "userAgent": "aws-cli/2.x",
    "userIdentity": {
      "type": "AssumedRole",
      "principalId": "AROAEXAMPLE:reporting-job",
      "arn": "arn:aws:sts::111122223333:assumed-role/reports-reader/reporting-job",
      "accountId": "111122223333",
      "sessionContext": {
        "sessionIssuer": {
          "type": "Role",
          "arn": "arn:aws:iam::111122223333:role/reports-reader",
          "accountId": "111122223333",
          "userName": "reports-reader"
        }
      }
    },
    "requestParameters": {
      "bucketName": "example-company-reports",
      "key": "reports/2026/output.csv"
    },
    "responseElements": null,
    "errorCode": "AccessDenied",
    "errorMessage": "Access Denied",
    "eventCategory": "Data",
    "managementEvent": false,
    "eventType": "AwsApiCall",
    "readOnly": false,
    "eventID": "33333333-3333-4333-8333-333333333333",
    "requestID": "SYNTH-REQUEST-003"
  }
]
'@

Set-Content -Path ./cloudtrail-events.json -Value $Sample -Encoding UTF8

$Events = @(Get-Content -Raw ./cloudtrail-events.json |
    ConvertFrom-Json -ErrorAction Stop)

"Loaded $($Events.Count) events"
~~~

**Expected:** Loaded 3 events.

### 3. Extract failed requests

~~~powershell
$Failures = @($Events | Where-Object {
    -not [string]::IsNullOrWhiteSpace($_.errorCode)
})

$Failures | Select-Object `
    @{Name='TimeUTC'; Expression={$_.eventTime}},
    @{Name='Role'; Expression={$_.userIdentity.sessionContext.sessionIssuer.arn}},
    @{Name='Action'; Expression={$_.eventName}},
    @{Name='ObjectKey'; Expression={$_.requestParameters.key}},
    @{Name='SourceIP'; Expression={$_.sourceIPAddress}},
    errorCode |
    Format-Table -AutoSize -Wrap
~~~

**Expected:** two failed rows: GetObject on finance/payroll.csv and PutObject on reports/2026/output.csv.

### 4. Correlate the failures

~~~powershell
$Failures |
    Group-Object { $_.userIdentity.arn } |
    Select-Object Name, Count |
    Format-Table -AutoSize

$Events |
    Group-Object eventCategory |
    Select-Object Name, Count |
    Format-Table -AutoSize
~~~

**Expected:** one assumed-role session with two failures; all three fixture records have event category Data.

### 5. Interpret the evidence

Compare the events with the Lesson 014 policy:

| Event | Policy expectation | Interpretation |
|---|---|---|
| GetObject on reports/2026/summary.csv | Allow | Successful read is consistent with the reports-prefix grant |
| GetObject on finance/payroll.csv | Deny | Expected: outside the permitted reports prefix |
| PutObject on reports/2026/output.csv | Deny | Expected: the policy grants GetObject, not PutObject |

The two failures share the same assumed-role session and source IP in this synthetic example. That is a useful correlation, not proof that the source or identity is legitimate.

**Important:** AccessDenied does not by itself prove that the identity policy caused the denial. Check the identity policy, bucket policy, permissions boundary, AWS Organizations SCP, S3 VPC endpoint policy and (for SSE-KMS objects) KMS key policy and kms:Decrypt permissions. Compare the application's intended operation with the actual API call before changing permissions.

## AWS operational trap to remember

Do not search only CloudTrail Event history when investigating an S3 GetObject or PutObject failure. Event history contains management events for the last 90 days in a Region; it does not contain data events. S3 object-level data events are not logged by default. A trail or event data store must be configured to collect them, and data-event logging incurs additional charges. Do not enable broad data-event logging in a real account as part of this exercise.

If a real account is available, use read-only inspection of its existing logging configuration. Do not create or modify a trail or event data store for this lesson.

## Expected outcome and self-check

You should be able to:
- parse the fixture and report exactly three events and two failures;
- identify the assumed IAM role separately from its STS session ARN;
- state why the two denials are consistent with the Lesson 014 least-privilege policy;
- explain why Event history alone may not contain the object-level request;
- name at least three other policy layers or dependencies that can cause AccessDenied.

## Production / interview relevance

This is a common cloud incident pattern: an application reports S3 AccessDenied, but the first task is to establish which principal made which API call, against which object, from where and when. Knowing the CloudTrail event category prevents wasted time searching the wrong log source. In regulated environments, preserve event IDs, request IDs, UTC timestamps and the original evidence before changing access controls.

## Official documentation

1. [AWS CloudTrail — Working with CloudTrail event history](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/view-cloudtrail-events.html) — 90-day, Region-scoped management-event history and its limitations.
2. [AWS CloudTrail — Logging data events](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/logging-data-events-with-cloudtrail.html) — S3 object-level data events, selectors and additional charges.

## Completion checkpoint

Report:
- Lesson 015 completed
- JSON parsing: passed / failed
- Events loaded: expected 3
- Failed events: expected 2
- Your explanation of why Event history may not show these S3 object requests

Do not mark the lesson complete until you have performed the exercise.