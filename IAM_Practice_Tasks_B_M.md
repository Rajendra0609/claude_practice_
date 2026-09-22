# 🔐 AWS IAM — Basic to Medium Practice Tasks

> Companion task list for the *AWS IAM — Identity & Access Management Hands-On Guide*.
> Goal: go beyond "I can click through the console" to **real operational judgment** — setting up account structure correctly, writing and debugging policies, handling security incidents, and explaining *why* you did what you did.

---

## How to Use This List

- Each task gives you a **scenario**, **what to build/do**, and **constraints** — no step-by-step solution, and no answers.
- Work through this in your own AWS account. Keep your root account MFA-protected and don't use root for any of these tasks except where explicitly stated.
- After each task, write a short note in your own words explaining what you did and why — that explanation *is* the interview answer.
- Difficulty is labeled **🟢 Basic** or **🟡 Medium** so you can gauge where you are.
- Tasks are grouped into five tracks, loosely following the guide's phase order.

---

## Track 1 — Account Foundations & Access Structure

### Task 1 🟢 Build a Correct Group-Based Permission Structure
**Scenario:** You've just been handed a brand-new AWS account. Nothing is set up yet except the root login.
**Do this:**
- Lock down root: enable MFA on the root account and confirm you have no root access keys.
- Create three groups (`Admins`, `Developers`, `ReadOnly`) with appropriate managed policies, and one user in each group.
- Deliberately try attaching a policy **directly to a user** first, then redo it the "right way" through a group — explain in your own notes why the group-based approach is considered best practice at scale.
**Interview angle:** "How would you set up IAM for a brand-new AWS account on day one?"

### Task 2 🟢 Verify the Principle of "Zero Permissions by Default"
**Scenario:** A new hire's IAM user was created but they're reporting they "can't do anything" in the console.
**Do this:**
- Create a new IAM user with no group membership and no policies attached, then log in as that user and confirm exactly what happens when they try to access any service.
- Add them to one group at a time (`ReadOnly`, then `Developers`) and note how their access changes at each step.
- Document what the console looks like/says when a user has zero effective permissions vs an explicit Deny — these look similar but are diagnostically different.
**Interview angle:** "A new employee says they can't see anything in the AWS console — what would you check first?"

### Task 3 🟡 Simulate a User Offboarding
**Scenario:** An engineer is leaving the company today. Security policy requires access to be revoked immediately, but their infrastructure automation (using an access key) must not break other running jobs unexpectedly.
**Do this:**
- Set up a user with console access, an active access key, and group membership as if they were a real employee running a scheduled script.
- Walk through a full offboarding: deactivate (not delete) the access key first, remove console access, remove from all groups, and only delete the user after confirming nothing depends on them.
- Note what you'd check *before* deleting anything irreversible (e.g., is this user's access key referenced by a Lambda function, CI/CD pipeline, or cron job?).
**Interview angle:** "Walk me through your process for offboarding a departing employee's AWS access."

---

## Track 2 — Policies & Least Privilege

### Task 4 🟢 Write a Custom Policy From Scratch (Not From a Template)
**Scenario:** A developer needs to view EC2 instances for debugging but should never be able to stop or terminate them, even by accident.
**Do this:**
- Without copying the guide's example directly from memory, write your own JSON policy granting read-only EC2 visibility while explicitly denying `StopInstances` and `TerminateInstances`.
- Attach it to a test user and validate both the "allowed" and "denied" behavior live, then validate the same result using the **IAM Policy Simulator** before touching the real console.
- Break the policy on purpose (e.g., remove the explicit Deny) and observe how the door quietly reopens — write down why this matters for change review.
**Interview angle:** "How does IAM decide whether to allow or deny a request when multiple policies apply?"

### Task 5 🟡 Diagnose Three Different "Access Denied" Root Causes
**Scenario:** Three different users report the same generic "Access Denied" error, but the underlying cause is different each time.
**Do this:**
- Deliberately create three separate broken scenarios: (1) a missing Allow permission, (2) an explicit Deny statement overriding an Allow elsewhere, and (3) a policy with a mismatched or overly narrow `Resource` ARN (e.g., policy allows one S3 bucket but the user is hitting a different one).
- For each, use the Policy Simulator and/or the actual error message to identify the true root cause before fixing it.
- Write a personal "Access Denied triage checklist" — the order of things you'd check first, second, third.
**Interview angle:** "A user gets Access Denied even though you can see a policy that should allow it — how do you debug that?"

### Task 6 🟡 Scope a Policy to a Single Resource, Not a Whole Service
**Scenario:** A policy currently grants `s3:*` on `Resource: "*"` for a reporting application that only ever needs to read from one specific bucket/prefix.
**Do this:**
- Start with the overly broad policy, and using Access Advisor (or manual review) narrow it down to only the specific actions and specific bucket ARN actually needed.
- Test that the app/user can still do its job after tightening, and that it now correctly fails for buckets it should not touch.
- Note the blast-radius difference in your own words: what's the worst-case impact if this identity's credentials were leaked *before* vs *after* your change?
**Interview angle:** "How do you actually apply the principle of least privilege in a real policy, not just as a slogan?"

---

## Track 3 — Authentication, Access Keys & CLI

### Task 7 🟢 Enforce MFA With a Policy Condition, Not Just a Checkbox
**Scenario:** Simply "recommending" MFA hasn't worked — several users still haven't enabled it. Leadership wants it technically enforced.
**Do this:**
- Enable MFA on at least two IAM users using a virtual MFA app.
- Write and attach a policy statement that **denies all actions** unless MFA is present (using the `aws:MultiFactorAuthPresent` condition), and test it against a user with MFA on vs a user without it.
- Identify one operational gotcha this kind of policy can cause (e.g., for break-glass access, or for a user setting up MFA for the first time) and how you'd handle it.
**Interview angle:** "How do you enforce MFA at the policy level instead of just asking users nicely?"

### Task 8 🟡 Perform a Zero-Downtime Access Key Rotation
**Scenario:** A script running on a schedule (e.g., a nightly CLI job) uses a long-lived access key that's now 100+ days old and needs rotation — without breaking the schedule.
**Do this:**
- Configure the AWS CLI with an initial access key and confirm a command works end-to-end (e.g., `aws sts get-caller-identity`, `aws s3 ls`).
- Follow the two-key rotation pattern from the guide: create key 2, switch usage to it, verify, deactivate key 1 (don't delete), wait, then delete key 1 — and actually verify at each step that the "old" key still technically works until you deactivate it.
- Document what would have gone wrong if you deleted the old key immediately instead of deactivating it first.
**Interview angle:** "How do you rotate an access key used by a production system without causing downtime?"

### Task 9 🟢 Manage Multiple Environments Safely With CLI Profiles
**Scenario:** You regularly need to run commands against both a "dev" and a "prod" AWS account/user, and accidentally running a command against the wrong one is a real risk.
**Do this:**
- Configure two named CLI profiles (e.g., `dev` and `prod`) with different credentials.
- Practice switching between them via `--profile` and via the `AWS_PROFILE` environment variable, and run `aws sts get-caller-identity` each time to confirm which identity is active before running anything else.
- Design a personal habit/safeguard (e.g., shell prompt customization, alias, or a pre-command check) that would prevent you from ever running a destructive command against the wrong profile — describe it even if you don't fully implement it.
**Interview angle:** "How do you avoid running a command against production when you meant to target dev?"

---

## Track 4 — IAM Roles & Temporary Credentials

### Task 10 🟢 Replace Hardcoded Credentials on EC2 With a Role
**Scenario:** You find a script on an EC2 instance with `aws_access_key_id` and `aws_secret_access_key` hardcoded in a config file — a real and common production security finding.
**Do this:**
- Reproduce the "bad" setup first: get an instance working with a hardcoded access key so you understand the risk concretely.
- Create an IAM role with the equivalent permissions, attach it to the instance, remove the hardcoded credentials entirely, and confirm the script still works using only the role.
- From inside the instance, inspect the metadata endpoint to see the temporary credentials being issued, and note their expiry — explain what happens automatically when they expire.
**Interview angle:** "Why is an IAM role considered more secure than an access key for an application running on EC2?"

### Task 11 🟡 Understand Trust Policy vs Permission Policy by Breaking Each One
**Scenario:** A role that should be assumable only by EC2 is somehow not working, and separately, a role that EC2 *can* assume doesn't seem to grant the access the app needs.
**Do this:**
- Create a role and deliberately misconfigure the **trust policy** (e.g., wrong principal/service) — confirm EC2 cannot assume it and understand the specific error.
- Fix the trust policy but leave the **permission policy** too narrow (e.g., missing an action) — confirm the role can now be assumed but calls still fail.
- Fix both, and in your own words explain the difference between "who can wear this badge" (trust policy) and "what the badge lets you do" (permission policy).
**Interview angle:** "What's the difference between a role's trust policy and its permission policy?"

### Task 12 🟡 Manually Assume a Role via STS
**Scenario:** A teammate needs temporary, time-limited access to perform one specific task in your account rather than being given a permanent user.
**Do this:**
- Create a role with a trust policy that allows a specific IAM user (not a service) to assume it.
- From the CLI, use `aws sts assume-role` to obtain temporary credentials, and use those temporary credentials (not your normal ones) to run a command.
- Let the temporary session expire (or reason through what happens when it does) and note how this differs from a normal IAM user's long-lived credentials.
**Interview angle:** "How would you give a contractor or teammate short-term access without creating a permanent IAM user for them?"

---

## Track 5 — Auditing, Incident Response & Best Practices

### Task 13 🟢 Run a Security Audit With the Credentials Report and Access Advisor
**Scenario:** You've inherited an account with several existing users and no idea who's following security best practices.
**Do this:**
- Generate and download the IAM Credentials Report; identify users without MFA, access keys older than 90 days, and users who've never logged in.
- Pick one over-permissioned user and use Access Advisor to see which services they've actually used, then propose (and apply) a tighter policy based on real usage.
- Turn your findings into a short written audit summary as if presenting it to a manager: what's risky, why, and what you changed.
**Interview angle:** "How would you audit an existing AWS account's IAM setup for security gaps?"

### Task 14 🟡 Respond to a Leaked Access Key Incident
**Scenario:** An access key was accidentally committed to a public GitHub repository. You need to respond before it's abused (or after — simulate either).
**Do this:**
- Create a throwaway access key specifically for this drill, then treat it as "leaked."
- Perform an incident response: identify the affected user, immediately deactivate (not just note) the compromised key, review what that key had access to, and issue a replacement key through the proper rotation flow.
- Write a short "what would I check for damage" list — e.g., unexpected resources created, unusual `CloudTrail`/billing activity — even if you don't have full CloudTrail set up, describe what you'd look for.
**Interview angle:** "Walk me through what you'd do in the first 15 minutes after discovering a leaked AWS access key."

### Task 15 🟡 Design (and Justify) an IAM Structure for a Small Team
**Scenario:** You're asked to design IAM from scratch for a 6-person startup: 1 founder/admin, 2 backend developers, 1 data analyst (read-only across most services), 1 contractor (needs temporary, narrow access), and 1 CI/CD pipeline (needs to deploy to EC2 and S3).
**Do this:**
- Design and actually build the groups, users, policies, and at least one role (for the CI/CD case) that this scenario calls for — don't just diagram it, implement it.
- For each identity, justify in writing why they get exactly the access they get — tie each decision back to least privilege.
- Identify which identity type should use a **role with temporary credentials** vs a **long-lived user**, and why.
**Interview angle:** This is close to a real "design IAM for this team" system-design interview prompt — practicing it end-to-end is the whole point.

---

## Suggested Practice Cadence

```
Day 1–2: Track 1 (Account foundations + offboarding drill)
Day 3–4: Track 2 (Policies, least privilege, Access Denied triage)
Day 5:   Track 3 (MFA enforcement, key rotation, CLI profiles)
Day 6–7: Track 4 (Roles, trust vs permission policy, AssumeRole)
Day 8:   Track 5 (Audit + incident response + final design task)
Day 9:   Mock interview — answer every "Interview angle" prompt above out loud, unscripted
```

## Before You Call a Task "Done"

- [ ] Could you explain your decision to a manager in 2 plain-English sentences?
- [ ] Could you explain the same decision to a senior engineer, including what you considered and rejected?
- [ ] Did you test both the "should work" and the "should be denied" case, not just the happy path?
- [ ] Did you clean up test users, keys, and policies you created for the drill?

---
*Do the labs yourself — the value is in writing the policy and watching it fail before you fix it. Good luck! 🔐*
