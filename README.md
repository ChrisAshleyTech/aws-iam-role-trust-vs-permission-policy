# IAM Role: Trust Policy vs. Permission Policy

## Objective
Demonstrate the distinction between an IAM role's **trust policy** (who can assume
the role) and its **permission policy** (what the role can do), by hand-writing
both — no managed policies or console wizards — and verifying the role works
end-to-end using a separate, unprivileged IAM user.

## Why this matters
Every IAM role has two policies, and confusing them is one of the most common
real-world misconfigurations:

- **Trust policy** — lives on the role itself. Answers *"who is allowed to put
  on this hat?"* Specifies the principal(s) permitted to call `sts:AssumeRole`.
- **Permission policy** — answers *"once you're wearing the hat, what are you
  allowed to touch?"* The actual list of allowed actions and resources.

Roles differ from IAM users in a fundamental way: a user has permanent
credentials, while a role hands out *temporary* credentials only to whoever
the trust policy allows, for a limited session. This mechanism underpins
cross-service access (e.g., EC2 reading from S3), cross-account access, and
federation (SAML/OIDC) — making it a foundational concept for IAM/IGA work.

## What was built
- An S3 bucket (`chrisash-iam-role-test-bucket`) as a target resource, with a
  test object uploaded
- An IAM role (`S3-ReadOnly-TestBucket-Role`) with:
  - A **custom trust policy** (`trust-policy.json`) allowing any principal
    within the same AWS account to call `sts:AssumeRole`
  - A **custom inline permission policy** (`permission-policy.json`) granting
    only `s3:GetObject` and `s3:ListBucket`, scoped to the single test bucket
    (least privilege — no wildcard resources, no managed policy)

## Key concept: the two-sided permission model
Assuming a role is not governed by the trust policy alone. AWS requires
**both**:

1. The role's **trust policy** to allow the calling principal, **and**
2. The calling principal's **own identity-based policy** to explicitly allow
   `sts:AssumeRole` on that role's specific ARN

I ran into this gap directly during testing: a dedicated test IAM user with
the trust policy fully satisfied still could not switch into the role. The
console returned "Invalid information in one or more fields" — a generic
error that in practice means the calling identity lacks its own
`sts:AssumeRole` grant. Fixing it required attaching this policy to the
*user*, not the role:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::<account-id>:role/S3-ReadOnly-TestBucket-Role"
        }
    ]
}
```

This is a good illustration of why "the trust policy already allows my
account" is not sufficient reasoning on its own — permission has to be
granted from both directions.

## Verification process
- Created a dedicated IAM user (`chris-console-user`) with **zero** standing
  permissions, used solely to prove the role — deliberately not tested from
  the root account, to keep the test realistic to how a least-privilege
  environment actually behaves
- Confirmed **Switch Role** failed until the AssumeRole grant above was added
  to the user
- After the fix, confirmed the role could successfully **list and read**
  objects in the target bucket
- Confirmed the role was **denied** access everywhere else it wasn't
  explicitly granted (Billing and Cost Management, AWS Health, other AWS
  services), proving the least-privilege scoping was correctly enforced and
  not accidentally broad

## Files in this repo
| File | Purpose |
|---|---|
| `trust-policy.json` | The role's trust policy — defines who can assume it |
| `permission-policy.json` | The role's inline permission policy — defines what it can do |
| `test-user-assume-role-policy.json` | The identity-based policy attached to the test user, granting it permission to call `sts:AssumeRole` on this specific role |

*(Account ID redacted as `<account-id>` in all policy files.)*

## NIST 800-53 mapping
- **AC-6 (Least Privilege)** — permission policy scoped to the exact actions
  and single resource needed, no more
- **AC-3 (Access Enforcement)** — dual-condition enforcement (trust policy +
  identity policy) required before access is granted
- **AC-2(3) (Account Disabling)** *(conceptually related)* — same
  trust/permission split underlies how temporary role credentials expire and
  differ from standing user credentials

## Tools used
AWS IAM, AWS S3, AWS Management Console (Switch Role)
