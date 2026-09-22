# 🔐 AWS IAM — Advanced Practical Tasks (Production-Scenario Lab)

> Companion exercise set to your **AWS IAM Hands-On Guide**. These tasks go beyond the guide's step-by-step walkthroughs and drop you into realistic production situations — the kind you'll be expected to handle on the job and explain in an interview.
>
> **How to use this:** Do each task yourself in a sandbox/test AWS account (never production). No solutions or answer keys are included on purpose — the goal is for you to design the policy, break something, read the error, and fix it using AWS documentation, the Policy Simulator, and CloudTrail. Where a task says "document your reasoning," write a short note explaining *why* you made each decision — that's the muscle interviewers actually test.
>
> ⚠️ Use a disposable/sandbox AWS account. Several tasks intentionally involve denying access, rotating keys, or simulating a compromise.

---

## Section A — Least-Privilege Policy Design

### Task 1: Build a Least-Privilege Policy from a Vague Requirement
A developer asks for "S3 access to do my job." Your job is to translate that into a tightly scoped policy.
- Create an S3 bucket with three prefixes: `uploads/`, `processed/`, and `archive/`.
- Write a customer-managed policy that allows the developer to **upload and list objects only under `uploads/`**, read (but not delete) objects under `processed/`, and has **no access at all** to `archive/`.
- Use policy variables (e.g., `${aws:username}`) so the same policy could theoretically be reused for multiple developers with per-user folders.
- Attach it to a test user and verify each boundary using the console *and* the CLI.
- Document what you'd tell the developer if they later asked for delete access — what questions would you ask before granting it?

### Task 2: Design an ABAC (Attribute-Based Access Control) Model
Instead of writing a separate policy per project, implement tag-based access control.
- Tag several EC2 instances and S3 buckets with a `Project` tag (e.g., `Project=Phoenix`, `Project=Atlas`).
- Write a single IAM policy using `aws:ResourceTag` and `aws:PrincipalTag` conditions so that a user tagged `Project=Phoenix` can only manage resources tagged `Project=Phoenix`.
- Test that a user tagged for one project is denied access to another project's resources — without touching the policy itself.
- Note: this is a favorite senior-level interview topic ("how would you scale permissions across 50 teams without writing 50 policies?").

### Task 3: Deny-by-Exception Policy for a Regulated Workload
Simulate a compliance requirement: "No one may delete objects from any S3 bucket tagged `Compliance=SOX`, even admins, unless they assume a special `SOX-Override` role with MFA."
- Design the explicit **Deny** statement with a `Condition` that exempts only the override role.
- Attach it as a policy that would sit alongside `AdministratorAccess` (remember: explicit Deny always wins).
- Verify an account admin is blocked, then verify the override role works only when MFA is present.
- Be ready to explain how this pattern (a "guardrail Deny" policy) differs from a Service Control Policy (SCP) at the AWS Organizations level.

---

## Section B — Roles, Trust Policies & Cross-Account Access

### Task 4: Cross-Account Role Assumption
Simulate a common enterprise pattern: a central "Security" account needs read-only audit access into a "Production" account.
- Using two AWS accounts (or two separate sandbox setups if you only have one, using account aliases), create a role in "Production" whose trust policy allows a specific role/user ARN from the "Security" account to assume it.
- Add an `sts:ExternalId` condition to the trust policy (mimic the pattern used for third-party vendor access).
- From the Security account, assume the role via CLI and run read-only commands against Production.
- Document what could go wrong if you forgot the `ExternalId` condition (this is a classic "confused deputy" interview question).

### Task 5: Break Down and Rebuild a Trust Policy
Take the auto-generated trust policy from an EC2 service role and deliberately misconfigure it three different ways (wrong principal, missing `sts:AssumeRole` action, overly broad `Principal: "*"`).
- For each misconfiguration, predict what error you'll see, then test it.
- Fix each one and write a one-paragraph explanation of the difference between a **trust policy** and a **permissions policy**, using this exercise as your example.

### Task 6: Lambda Execution Role with Least Privilege
Deploy a simple Lambda function that reads from one specific DynamoDB table and writes logs to CloudWatch.
- Start by attaching `AdministratorAccess` (to prove it works), then progressively replace it with a custom least-privilege policy until the function works with the *minimum* required actions.
- Use CloudTrail or Access Advisor to confirm no unused permissions remain.
- Explain in your own words why "start broad, then narrow using logs" is a legitimate and common production technique — and its risks.

### Task 7: Instance Profile Troubleshooting
Launch an EC2 instance **without** attaching a role, install the AWS CLI, and try to run `aws s3 ls`. Observe the failure.
- Now attach a role with S3 permissions *after* the instance is already running, without rebooting it.
- Verify from inside the instance (via the metadata endpoint) that the new credentials are picked up automatically.
- Explain what "credential caching" behavior an application (e.g., a long-running Python process using boto3) might exhibit here, and how you'd handle it in production.

---

## Section C — Security Auditing, Rotation & Incident Response

### Task 8: Simulate and Respond to a Leaked Access Key
This is the single most common real-world IAM incident.
- Create a test IAM user with an access key, then deliberately treat that key as "leaked" (e.g., pretend you found it in a public GitHub repo).
- Without deleting the user, perform a full incident response: identify what the key was used for (CloudTrail `Event history` filtered by access key), deactivate it, attach a policy that would have limited blast radius (or apply an explicit deny), rotate to a new key, and document a timeline of your response.
- Write out, as if for a post-incident report, what preventative controls (IAM, GitHub, CloudTrail alarms) would have caught this faster.

### Task 9: Zero-Downtime Key Rotation Under Load
Simulate a production script or scheduled job that authenticates using an access key (a local script polling `aws s3 ls` every few seconds in a loop is enough).
- While the script is running, perform the two-key rotation pattern from the guide: create key 2, switch the script to use it, deactivate key 1, and confirm zero failed calls occurred during the switch.
- Then simulate a *failed* rotation — deactivate key 1 before key 2 is actually working — and observe/troubleshoot the failure.
- Note what you'd automate (e.g., via a script or AWS Config rule) to make this safer at scale across hundreds of service accounts.

### Task 10: Full Account Access Audit
- Generate and download the IAM Credentials Report.
- Cross-reference it with Access Advisor for at least three users with different permission levels.
- Identify (in a real or seeded test account) at least three concrete findings: an unused access key, a user with no MFA, and a user with far more permissions than their Access Advisor activity justifies.
- Write a remediation plan with priorities (what you'd fix first, second, third, and why) — this is exactly the kind of answer interviewers expect for "how would you audit an AWS account you just inherited?"

### Task 11: Permissions Boundaries for Delegated Admin
Simulate giving a team lead the ability to create IAM users and roles for their own team — without letting them (or anyone they create) ever exceed a defined boundary.
- Create a permissions boundary policy capping the maximum permissions (e.g., no IAM changes outside their team's tag, no billing access, no access to other teams' S3 buckets).
- Grant the team lead `iam:CreateUser`/`iam:CreateRole` rights, but only when a permissions boundary is attached.
- Test that the team lead cannot create a new admin user that escapes the boundary, even though they technically have `iam:CreateUser`.
- Be ready to explain how permissions boundaries differ from SCPs and from a plain Deny policy.

---

## Section D — Debugging & Policy Evaluation Under Pressure

### Task 12: Diagnose a "Mystery" Access Denied
Set up a deliberately confusing scenario: attach an Allow policy directly to a user, add the user to a group with a conflicting explicit Deny, and also apply a permissions boundary that further restricts them.
- Without looking at any policy first, have a colleague (or your future self, a week later) try to reproduce and diagnose *why* a specific API call is denied, using only the Policy Simulator and `aws iam` CLI commands to inspect attached policies.
- Document the exact order you checked things in, and map your steps back to the IAM policy evaluation logic (explicit deny → SCP → permissions boundary → identity-based → resource-based → implicit deny).
- This maps directly to one of the most common IAM interview questions: *"Walk me through how you'd troubleshoot an unexpected Access Denied error."*

### Task 13: Resource-Based vs. Identity-Based Policy Conflict
- Create an S3 bucket policy (resource-based) that denies access from all principals except a specific role.
- Separately, grant a different user broad S3 permissions via an identity-based policy.
- Test and observe what happens when the identity-based policy says "allow" but the resource-based policy says "deny" (and vice versa).
- Write a short explanation of how S3 bucket policies and IAM identity policies are evaluated together — this trips up even experienced engineers.

### Task 14: Condition Keys for Contextual Access Control
Design and test at least three separate policies using different condition keys:
- Restrict console/API access to a specific source IP range (`aws:SourceIp`).
- Require MFA for any destructive action (`aws:MultiFactorAuthPresent`).
- Restrict access to a specific time window (`aws:CurrentTime`) to simulate a "business hours only" access policy for a contractor.
- For each, test both the allowed and denied scenarios, and note a realistic production use case where you'd actually deploy each one.

---

## Section E — Scenario-Based Interview Simulation

### Task 15: Design IAM from Scratch for a New Startup Environment
Given this (fictional) scenario, design the full IAM structure — group hierarchy, roles for each environment, policy strategy, and MFA/key-rotation approach — and implement as much of it as your sandbox allows:

> "We're a 15-person startup with 3 engineers, 2 DevOps, 4 developers, 1 finance/admin person, and a handful of automated CI/CD pipelines. We use dev, staging, and production AWS accounts (AWS Organizations). Engineers need broad access in dev, limited access in staging, and near-zero standing access in production (break-glass only). CI/CD needs to deploy to all three environments without ever using long-lived access keys."

- Produce a written design (groups, roles, trust relationships, naming conventions, break-glass process) before touching the console.
- Implement the core pieces: at least one cross-environment role with a tightly scoped trust policy, a break-glass role requiring MFA and time-boxed session duration, and a CI/CD role usable via OIDC federation (e.g., GitHub Actions) instead of static keys.
- Be ready to defend every design decision out loud — this task is essentially a live-coding/whiteboard interview rehearsal.

---

## How to Get the Most Out of This

- After each task, write a **3–5 sentence "if asked in an interview" summary** in your own words — what the task taught you and what real production risk it maps to.
- Intentionally break things (wrong ARNs, missing conditions, overly broad wildcards) before fixing them. Diagnosing your own mistakes is what builds real troubleshooting instinct.
- Revisit **Task 12** and **Task 15** last — they're designed to combine everything from Tasks 1–14.
- Once comfortable, try timing yourself on Task 12 and Task 15 as a mock interview exercise (15–20 minutes each, thinking out loud).

*No answers or solutions are included in this file by design — that's your work to do.*
