# Lesson 014 — AWS IAM policy evaluation and least-privilege S3 access

**Domain:** AWS IAM / security  
**Duration:** 30 minutes  
**Environment:** Windows PowerShell; AWS account access is optional. No AWS resources are created.

## Objective

Write and inspect an identity-based IAM policy that permits a workload to list and read only objects under one S3 bucket's `reports/` prefix. Validate the JSON locally, then (if you have permitted AWS console access) simulate the policy without attaching it to an identity or making an S3 request.

By the end, you should be able to explain how IAM combines **Action**, **Resource**, and **Condition**, and distinguish an implicit deny from an explicit deny.

## Exact items to practise

- Scope `s3:ListBucket` to a bucket ARN and `s3:GetObject` to object ARNs under a specific prefix.
- Use the S3-specific `s3:prefix` condition to constrain listing.
- Read the basic evaluation model: default deny, matching allow, explicit deny, and the limiting effect of permissions boundaries and AWS Organizations SCPs.
- Use the IAM Policy Simulator's **Custom** mode to test a draft policy safely.

## Timed plan

| Time | Activity |
|---:|---|
| 0–4 min | Review IAM evaluation: implicit deny, explicit deny, identity/resource policies, boundaries and SCPs |
| 4–12 min | Create the least-privilege S3 policy below and inspect its scope |
| 12–16 min | Parse the JSON in PowerShell and review each statement |
| 16–25 min | Simulate allowed and denied requests in IAM Policy Simulator, or complete the offline decision matrix |
| 25–30 min | Explain the results and complete the self-check |

## 1. Policy scenario

A workload needs read-only access to reports stored in this example bucket:

- Bucket: `example-company-reports`
- Permitted prefix: `reports/`
- Example permitted object: `reports/2026/summary.csv`
- Example out-of-scope object: `finance/payroll.csv`

The policy below is an **identity-based policy draft**. It is not attached to an IAM user or role during this exercise.

## 2. Create the policy on Windows

Open PowerShell. Create a disposable lesson directory and move into it:

```powershell
$Lab = "$env:USERPROFILE\labs\tech-refresher\lesson-014-aws-iam"
New-Item -ItemType Directory -Path $Lab -Force | Out-Null
Set-Location $Lab
```

Create `s3-reports-readonly-policy.json`:

```powershell
@'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListOnlyReportsPrefix",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::example-company-reports",
      "Condition": {
        "StringLike": {
          "s3:prefix": [
            "reports/",
            "reports/*"
          ]
        }
      }
    },
    {
      "Sid": "ReadOnlyReportsObjects",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::example-company-reports/reports/*"
    }
  ]
}
'@ | Set-Content -Path .\s3-reports-readonly-policy.json -Encoding utf8
```

### Why the two resources differ

- `s3:ListBucket` is a bucket-level action, so its resource is the bucket ARN without a trailing object path.
- `s3:GetObject` is an object-level action, so its resource is the object ARN pattern ending in `reports/*`.
- The `s3:prefix` condition limits which key prefixes can be listed. It does not grant object reads; the second statement does that.
- No `s3:PutObject`, `s3:DeleteObject`, or broad `s3:*` permission is present.

## 3. Validate the JSON locally

Run:

```powershell
$Policy = Get-Content -Raw .\s3-reports-readonly-policy.json |
    ConvertFrom-Json -ErrorAction Stop

$Policy.Version
$Policy.Statement |
    Select-Object Sid, Effect, Action, Resource |
    Format-Table -AutoSize
```

Expected:
- The first command prints `2012-10-17`.
- Two statements appear: `ListOnlyReportsPrefix` and `ReadOnlyReportsObjects`.
- No JSON parsing error appears.

This confirms JSON syntax and lets you inspect the policy structure. It does **not** prove that AWS will grant the intended access; that is what policy simulation is for.

## 4. Simulate the policy (optional AWS console access)

If you have an AWS account and permission to use the IAM Policy Simulator:

1. Open **IAM → Policy Simulator** in the AWS Management Console.
2. Select **Custom** mode.
3. Paste the complete contents of `s3-reports-readonly-policy.json` into the identity-policy editor.
4. Add the S3 actions and resources below. For `s3:ListBucket`, supply the service-specific condition key `s3:prefix` for each test.
5. Run each test and record Allow or Deny.

Use these test cases:

| Action | Resource ARN | Condition input | Expected |
|---|---|---|---|
| `s3:ListBucket` | `arn:aws:s3:::example-company-reports` | `s3:prefix = reports/2026/` | Allow |
| `s3:ListBucket` | `arn:aws:s3:::example-company-reports` | `s3:prefix = finance/` | Deny |
| `s3:GetObject` | `arn:aws:s3:::example-company-reports/reports/2026/summary.csv` | None | Allow |
| `s3:GetObject` | `arn:aws:s3:::example-company-reports/finance/payroll.csv` | None | Deny |
| `s3:PutObject` | `arn:aws:s3:::example-company-reports/reports/2026/summary.csv` | None | Deny |
| `s3:DeleteObject` | `arn:aws:s3:::example-company-reports/reports/2026/summary.csv` | None | Deny |

**Safety:** Custom mode simulates the draft; it does not attach the policy or send a request to S3. Do not create a bucket, user, role, access key, or other AWS resource for this lesson. If you cannot access the simulator, use the table as an offline policy-evaluation exercise.

## 5. Understand the decision

For this simple identity-policy exercise:

1. AWS starts with an implicit deny.
2. A request is allowed only when an applicable Allow matches its action, resource and conditions, and no applicable explicit Deny overrides it.
3. A permissions boundary and an applicable Organizations SCP can further restrict what an identity policy allows; they do not grant permissions by themselves.
4. Resource-based policies can also grant access. Their interaction with identity policies and boundaries has important principal-specific nuances, so do not assume every policy type is always a simple intersection or union.

The simulator is a useful pre-deployment check, not proof of live behaviour. AWS documents cases where simulation can differ from the actual environment; verify the effective permissions and relevant resource policies through your normal controlled review before a real deployment.


## Equivalent Linux shell workflow

Assumptions: Bash and Python 3; jq is optional. No AWS credentials or resources are required.

Create the same lab directory:

```bash
mkdir -p "$HOME/labs/tech-refresher/lesson-014-aws-iam"
cd "$HOME/labs/tech-refresher/lesson-014-aws-iam"
```

Create the same policy JSON:

```bash
cat > s3-reports-readonly-policy.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListOnlyReportsPrefix",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::example-company-reports",
      "Condition": {
        "StringLike": {
          "s3:prefix": ["reports/", "reports/*"]
        }
      }
    },
    {
      "Sid": "ReadOnlyReportsObjects",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::example-company-reports/reports/*"
    }
  ]
}
JSON
```

Validate JSON using Python 3 (standard library; no install required):

```bash
python3 -m json.tool s3-reports-readonly-policy.json
```

Optional statement inspection with jq:

```bash
jq -r '.Version, (.Statement[] | [.Sid, .Effect, (.Action|tostring), .Resource] | @tsv)' s3-reports-readonly-policy.json
```

Expected: formatted JSON with no parsing error and two policy statements. This validates syntax, not AWS authorization. Use the same IAM Policy Simulator steps and six-case decision matrix in the main exercise. There is no equivalent local Bash command that can replace AWS policy evaluation.

## Expected outcome

You should have:
- A local JSON policy file that parses successfully.
- A clear explanation of why the bucket ARN is used for listing and the object ARN for reading.
- The six expected Allow/Deny outcomes above, either simulated or reasoned through offline.
- No AWS resources created and no policy attached to an identity.

## Quick self-check

- [ ] Can you explain why `s3:ListBucket` does not use the object ARN?
- [ ] Can you explain what the `s3:prefix` condition restricts?
- [ ] Can you identify why `PutObject` is denied even though the policy contains an Allow?
- [ ] Can you distinguish implicit deny from explicit deny?
- [ ] Can you explain why an SCP or permissions boundary can still block an identity-policy Allow?

## Production and interview relevance

This is the difference between “the role has S3 access” and “the role can read only the intended data.” The same reasoning is used for CI/CD deployment roles, application workload roles, cross-account access reviews and incident triage for `AccessDenied`. In an interview, be ready to explain the action/resource distinction, condition evaluation, explicit-deny precedence and why a simulator result must still be checked against the real account's guardrails and resource policies.

## For Dummies 🧸

Think of an S3 bucket as a filing cabinet. The first permission lets you look at the labels on only one drawer called `reports`. The second lets you read papers inside that drawer. It does not let you write, delete, or open the `finance` drawer. AWS checks the request against all the rules before opening anything.

## Completion checkpoint

Report **“Lesson 014 completed”** when done, and include:
- JSON parse: pass/fail
- Simulator: completed / unavailable (offline matrix used)
- Results: the six Allow/Deny outcomes
- One sentence explaining the difference between an implicit deny and an explicit deny

Do not mark the lesson complete until the exercise and self-check have actually been performed.
