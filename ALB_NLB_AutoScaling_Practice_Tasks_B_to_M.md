# ALB / NLB & Auto Scaling — Hands-On Practice Tasks

> Companion lab guide to `ALB_NLB_AutoScaling_Mastery.md`.
> **No answers or solution steps are included** — you build, break, and fix everything yourself.
> Each task tells you the *scenario* and the *goal*. How you get there is up to you (Console or CLI).

---

## How to Use This File

- Work roughly in order — later tasks assume infrastructure from earlier ones (reuse it, don't rebuild from scratch each time).
- After each task, write **3–5 interview-style bullet points** in your own words explaining what you did and why. This is more valuable than the lab itself.
- Where a task says "break it on purpose," always fix it afterward and note the *symptom → root cause → fix* — this becomes your personal troubleshooting playbook.
- **Tear down resources after each session** (LBs, NAT gateways, running EC2) to avoid charges. Note this as its own habit to practice.
- Relevant guide sections are referenced in `()` so you can go back and re-read theory if you get stuck — but try first before peeking.

---

## Tier 1 — Foundation (Basic)

### Task 1: Build the Core Stack Manually (Console)
Using the AWS Console only (no CLI), build: a VPC-default-or-custom setup with 2+ public subnets in 2 AZs, a target group, an ALB with an HTTP listener, and 2 manually launched EC2 instances registered as targets serving a simple web page that shows the instance ID/AZ.
**Goal:** Confirm you understand every clickable field in the ALB/TG/listener wizards — not just "next, next, finish." *(Refs: §5–§7)*

### Task 2: Reproduce Task 1 Entirely via CLI
Tear down Task 1 and rebuild the exact same stack using only `aws elbv2` and `aws ec2` commands, capturing each ARN into a shell variable as you go.
**Goal:** Be able to narrate every CLI flag's purpose without looking it up. *(Refs: §26)*

### Task 3: Health Check Tuning & Failure Simulation
On your target group, deliberately misconfigure the health check path so both targets go `unhealthy`, observe the behavior, then fix it. Separately, stop the web server process on one instance only (leave the OS running) and watch what the ALB does versus what an EC2-only health check would have done.
**Goal:** Be able to explain the exact difference between "unhealthy target" and "unreachable target," and how `HealthyThresholdCount` / `UnhealthyThresholdCount` / interval affect detection speed. *(Refs: §6, §17.1, §24)*

### Task 4: Launch Template + Manual ASG, No Scaling Policy Yet
Create a Launch Template (with user data that installs/starts a web app automatically) and an ASG with Min=2/Desired=2/Max=4, attached to the target group from Task 1. Manually change desired capacity up and down and observe registration/deregistration timing.
**Goal:** Understand the full wiring loop (Launch Template → ASG → Target Group → Listener) before adding any "intelligence." *(Refs: §13, §14, §16)*

---

## Tier 2 — Core Skills (Intermediate)

### Task 5: Target Tracking Scaling Under Real Load
Add a CPU-based target-tracking policy to your ASG, then generate real CPU load on the instances (e.g., a stress tool) until it scales out, and let load subside until it scales in.
**Goal:** Capture and explain the full timeline: alarm breach → scaling activity → new instance launch → health check → traffic starts flowing. Note where cooldown/warm-up affected the timing. *(Refs: §15.1, §17)*

### Task 6: Scale on Requests-Per-Target Instead of CPU
Replace or add a policy that scales based on `ALBRequestCountPerTarget` instead of CPU, and generate enough concurrent HTTP requests to trigger it.
**Goal:** Be able to explain *when* request-based scaling is a better signal than CPU-based scaling, with a real example from your test. *(Refs: §15.1)*

### Task 7: HTTPS Everywhere with ACM
Request/validate an ACM certificate (use a domain you control, or a self-signed workaround if you don't), attach an HTTPS listener to your ALB, and configure an HTTP listener rule that redirects to HTTPS.
**Goal:** Explain TLS termination at the ALB versus end-to-end encryption to the backend, and when you'd need the latter. *(Refs: §11)*

### Task 8: Path- and Host-Based Routing for Microservices
Build a second target group backed by a different app (or the same app returning a different message), and configure ALB listener rules so `/api/*` goes to one target group and everything else goes to another. Add a host-based rule as well if you have a second domain/subdomain available.
**Goal:** Simulate a real microservices front door and explain rule evaluation order/priority. *(Refs: §12)*

### Task 9: Build and Compare an NLB
Stand up an NLB (TCP or TLS listener) in front of the same or a different set of targets, and compare its behavior to the ALB: check the static IP, test source-IP preservation against your target's security group, and try to make it fail by blocking the wrong CIDR.
**Goal:** Be able to state — with evidence from your own test, not just theory — the concrete scenarios where NLB is required over ALB. *(Refs: §8, §10, §24)*

### Task 10: Zero-Downtime Deployment via Instance Refresh
Publish a new Launch Template version (change the AMI, user data, or app version), then trigger an Instance Refresh on the ASG with a `MinHealthyPercentage` setting, while continuously curling the ALB DNS name from another terminal.
**Goal:** Prove zero (or near-zero) dropped requests during the rollout, and explain what `MinHealthyPercentage` and deregistration delay were doing behind the scenes to make that possible. *(Refs: §17, §25)*

---

## Tier 3 — Production-Grade & Advanced

### Task 11: Break It on Purpose — Troubleshooting Drills
Independently reproduce **each** of the following failures on your own stack, diagnose it using `describe-target-health` / `describe-scaling-activities` / CloudWatch, fix it, and write down the exact symptom-to-cause chain for each:
- All targets healthy in the target group, but the ALB still returns **502**.
- ALB returns **503**.
- ALB returns **504**.
- ASG refuses to scale out even though CPU is clearly high.
- Traffic is landing unevenly across AZs behind an NLB.
- A brand-new deploy never shows up — old app version keeps serving.
**Goal:** This becomes your interview troubleshooting script — practice explaining each one out loud in under 60 seconds. *(Refs: §24)*

### Task 12: Lifecycle Hooks for Custom Bootstrapping/Draining Logic
Add a lifecycle hook on instance launch (e.g., pause until a config-pull or warm-up script completes) and one on instance terminate (e.g., pause to deregister from an external system or flush logs before the instance dies).
**Goal:** Explain a real production use case for each hook type and what happens if the hook's heartbeat/timeout is never completed. *(Refs: §17)*

### Task 13: Mixed Instances Policy with Spot + On-Demand
Convert your ASG to a Mixed Instances Policy using a base of On-Demand capacity plus Spot for the rest, across multiple instance types. Force a simulated Spot interruption (or read how AWS signals one) and observe ASG behavior.
**Goal:** Explain how you'd design a fleet that's both cost-optimized and resilient to Spot interruptions, and where you would (and wouldn't) use Spot in production. *(Refs: §14, §23, §25)*

### Task 14: Blue/Green Cutover Using Target Group Swap
Stand up a second, fully independent ASG + target group running a "v2" version of the app, attach it to the same ALB, and perform a controlled cutover from v1 to v2 by shifting listener rule weights (weighted target groups) gradually from 100/0 to 0/100. Roll back mid-way to prove you can.
**Goal:** Be able to whiteboard a blue/green vs canary release strategy end-to-end, including how you'd detect a bad release before shifting 100% of traffic. *(Refs: §12, §16, §18)*

### Task 15: Multi-Tier Architecture — Public ALB + Internal NLB
Design and build a two-tier setup: an internet-facing ALB in public subnets fronting an app tier ASG in private subnets, which in turn calls an **internal** NLB fronting a backend/service tier ASG (also private). No component should be reachable except through the intended path.
**Goal:** Defend your security group chain (who's allowed to talk to whom, and only whom) as if a security reviewer were grilling you on it. *(Refs: §18, §22)*

### Task 16: Full Observability Pass
On your existing stack, enable ALB access logs to S3, set CloudWatch alarms on 5XX rate, target response time, and unhealthy host count, and build a simple dashboard. Deliberately cause a 5XX spike (e.g., kill an app process under load) and confirm your alarm fires and you can trace it back to a cause using the access logs.
**Goal:** Practice describing your production monitoring setup the way you would in a system-design interview — what you alarm on and why those specific signals. *(Refs: §21)*

### Task 17: Capstone — Rebuild Everything as Code
Pick Terraform, CloudFormation, or CDK, and codify the entire multi-tier stack from Task 15 (or your best working environment) as IaC: VPC/subnets, security groups, launch template, ASG with a target-tracking policy, ALB/NLB, listeners, rules, and an ACM cert. Destroy it and rebuild it from code twice to prove it's repeatable.
**Goal:** Be able to explain your resource dependency graph and what you'd change to make this environment support multiple deployment environments (dev/stage/prod). *(Refs: §19, §20)*

### Task 18: Cost Teardown & Optimization Review
Audit your lab environment for anything that's been left running or over-provisioned (idle LBs, oversized instance types, Min capacity higher than needed, cross-zone charges on NLB, unused Elastic IPs). Right-size it, add a scheduled scaling action for a predictable "night" scale-down, and then fully delete everything.
**Goal:** Practice explaining a cost-optimization pass on a load-balanced fleet as if presenting it to a manager, with real numbers from your own account where possible. *(Refs: §23, §25)*

---

## Tier 4 — Interview Simulation

### Task 19: Mock System Design — Verbal Only
Without touching the console, verbally (or in writing, from memory, no notes) design a highly available, cost-optimized web tier end to end: load balancer choice, listener/routing design, target group health check strategy, launch template contents, ASG sizing and scaling policy choice, deployment strategy, and monitoring. Time yourself — aim for a clear 5–7 minute explanation.
**Goal:** This is interview question #24 from the guide (§27) — rehearse it until it's fluent, not memorized. *(Refs: §27)*

### Task 20: Rapid-Fire Q&A Self-Test
Go through all interview questions in §27 of the guide one by one, answer each out loud from memory in under 90 seconds, and flag any you couldn't answer confidently. Go back and close those specific gaps with a targeted mini-lab (not just re-reading).
**Goal:** Walk away with a personal list of your 3–5 weakest areas and a concrete plan (a task above, or a new mini-lab) to fix each one. *(Refs: §27)*

---

## Self-Tracking Table

Copy this into your notes and fill it in as you go — it doubles as interview prep evidence of hands-on experience.

| # | Task | Date Done | Confidence (1–5) | Notes / Gotchas |
|---|------|-----------|-------------------|------------------|
| 1 | | | | |
| 2 | | | | |
| ... | | | | |
| 20 | | | | |

