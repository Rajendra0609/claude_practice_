# ALB / NLB & Auto Scaling — Production-Grade Practice Tasks

> Based on: `ALB_NLB_AutoScaling_Mastery.md`
> Purpose: Build **deep, hands-on mastery** through realistic production scenarios — not toy examples.
> Rule: **No solutions included.** You build, break, diagnose, and fix everything yourself. Use the guide's sections as reference material, not as a script to copy-paste.

---

## How to Use This File

- Work through tasks **in order** — later tasks assume infrastructure/skills from earlier ones.
- For every task, before you touch the console/CLI, **write down your plan** (which resources, in what order, what could go wrong). This is exactly what interviewers ask you to do on a whiteboard.
- After completing each task, answer the **"Interview Angle"** questions out loud or in writing, as if explaining to a panel — this is what actually cements the knowledge.
- **Always tag your lab resources** (e.g. `Project=lab-tier`) and **tear them down** after each task to avoid charges.
- Keep a **run log** (a simple notes doc) of every failure you hit and how you diagnosed it — this becomes your personal troubleshooting playbook and interview story bank.

---

## Task 1 — Design and Build a Production-Pattern 3-Tier Web App

**Scenario:** You're the on-call engineer for a new customer-facing web app. Leadership wants it to survive an entire Availability Zone outage, handle a 5x traffic spike without manual intervention, and never expose backend servers directly to the internet.

**Build:**
- An ALB in public subnets, EC2 (via ASG) in private subnets, spread across **at least 2 AZs**.
- Security groups that follow least-privilege (only the ALB can reach the app port).
- A real `/health` endpoint used by ELB-type health checks.
- Min/Desired/Max capacity chosen deliberately — justify your numbers.

**Validate:**
- Confirm requests are actually distributed across instances (not always hitting one).
- Confirm instances in both AZs receive traffic.

**Interview Angle:** Be ready to explain *why* EC2 sits in private subnets, what an EC2 health check misses that an ELB health check catches, and how you'd justify your Min/Max choice to a cost-conscious manager.

---

## Task 2 — HTTPS Everywhere, the Right Way

**Scenario:** Security review flagged your Task 1 app for serving traffic over plain HTTP. You must fix this before go-live, and support **two different subdomains** on the same ALB (e.g. `www.` and `api.`) with separate certificates.

**Build:**
- ACM certificate(s) covering both subdomains (or a SAN/wildcard cert).
- HTTPS listener on 443, with an HTTP→HTTPS redirect on port 80 (not just a forward).
- A modern TLS security policy (not a legacy/weak one).

**Validate:**
- HTTP requests are redirected, not served.
- Both subdomains resolve and present the correct certificate (check via browser or `openssl s_client`).

**Interview Angle:** Explain what "TLS termination at the load balancer" means, why the backend can still use plain HTTP internally, and what changes if a security team demands end-to-end encryption to the instance.

---

## Task 3 — Multi-Service Routing on a Shared ALB

**Scenario:** Your company is consolidating three microservices (`web`, `api`, `admin`) behind **one** ALB instead of three separate load balancers, to cut cost and simplify DNS.

**Build:**
- Three target groups, each pointing at its own (mock) backend/ASG.
- Listener rules using **path-based** routing for one service, **host-header** routing for another, and a **custom HTTP header** condition for the third (e.g. internal admin access only with `X-Internal: true`).
- A fixed-response rule that serves a maintenance page for a specific path.
- Correct **rule priority ordering** so none of your rules accidentally shadow another.

**Validate:**
- Each routing method sends traffic to the correct target group and nowhere else.
- Requests that don't match any custom rule fall through to a sensible default action.

**Interview Angle:** Explain how ALB evaluates rule priority, and describe a real incident where two rules conflicting caused unexpected routing — how would you have caught it in review?

---

## Task 4 — Canary and Blue/Green Deployment with Weighted Target Groups

**Scenario:** You're deploying v2 of a payment service. The business will not tolerate a bad deploy affecting more than a small slice of users, and requires an **instant rollback** capability.

**Build:**
- Two target groups (v1, v2) behind the same listener, using weighted forwarding.
- Start with 100/0, shift gradually (e.g. 90/10 → 50/50 → 0/100) while monitoring target health and error rates at each step.
- A documented, one-command rollback procedure (shift weights back to 100/0 immediately).

**Validate:**
- Confirm you can observe which version served a given request (e.g. from response body/hostname).
- Deliberately make v2 "unhealthy" mid-rollout and confirm your monitoring would have caught it before 100% cutover.

**Interview Angle:** Differentiate canary vs blue/green vs rolling deployment. Explain how you'd decide the shift schedule and what metrics would make you abort a rollout.

---

## Task 5 — NLB for a Fixed-IP, High-Throughput Requirement

**Scenario:** A banking partner requires your API to be reachable through a **static IP allow-listed on their firewall**, and needs raw TCP performance for a custom binary protocol (not HTTP).

**Build:**
- An NLB with Elastic IP(s) attached, fronting a target group of your choosing.
- A second variant: NLB in front of an ALB (Pattern C from the guide) so you retain L7 smart routing for other clients while giving this partner a fixed IP.
- Correctly handle **source IP preservation** in your target security groups (this is a very common real-world gotcha).

**Validate:**
- Confirm the IP truly never changes (stop/start won't help you here — understand *why* it doesn't need to).
- Confirm target security groups allow the *actual client IP*, not the NLB's IP, and understand why that distinction matters.

**Interview Angle:** Explain why NLB preserves source IP by default and ALB does not, and what problems that causes for security groups if not handled correctly.

---

## Task 6 — Scaling Policy Bake-Off

**Scenario:** Your app has two traffic patterns: (1) a fairly predictable daily curve (low at night, high during business hours) and (2) occasional unpredictable spikes from marketing campaigns. One scaling policy alone won't handle both well.

**Build:**
- A **target tracking** policy on CPU.
- A second target tracking policy on **`ALBRequestCountPerTarget`** — compare its scaling behavior to the CPU-based one under the same load test.
- A **scheduled scaling** action for your predictable daily pattern.
- Deliberately generate load and observe scale-out speed vs scale-in speed.

**Validate:**
- Document which metric reacted faster/more accurately to your specific load test.
- Confirm scheduled and dynamic scaling don't fight each other (i.e., check what happens if scheduled min conflicts with a dynamic scale-in).

**Interview Angle:** Be ready to argue for CPU vs request-count-per-target vs a custom metric depending on the workload type. Explain instance warm-up and why skipping it causes over-scaling.

---

## Task 7 — Decoupled Worker Fleet That Scales on Queue Depth

**Scenario:** A separate "jobs" service processes uploaded files asynchronously via a queue. Unlike the web tier, this fleet should be able to **scale to zero** when there's no work, and scale aggressively when the backlog grows.

**Build:**
- A worker ASG (no load balancer attached) with a custom/customized-metric target tracking policy based on backlog-per-instance.
- Min size of 0, and a plan for how new jobs get picked up once the fleet scales back from zero.

**Validate:**
- Confirm the fleet actually scales down to zero when idle.
- Load the queue and time how long it takes to scale out and start draining the backlog.

**Interview Angle:** Explain why CPU is often the *wrong* metric for a worker fleet, and how you compute "backlog per instance." Discuss the cold-start problem when scaling from zero.

---

## Task 8 — Zero-Downtime Deployment with Instance Refresh (and a Rollback Drill)

**Scenario:** You need to roll out a new AMI/app version to a live ASG without dropping a single request, and you must be able to prove to your team you can roll back cleanly if something goes wrong mid-rollout.

**Build:**
- A new Launch Template version with a visible change (e.g. different response text).
- An Instance Refresh with a `MinHealthyPercentage` you can justify.
- Mid-rollout, **deliberately trigger a rollback** and confirm the ASG returns to the previous known-good state.

**Validate:**
- While the refresh runs, continuously hit the ALB and confirm zero failed requests (or document exactly how many and why).
- Confirm deregistration delay/connection draining is tuned so in-flight requests aren't cut off during instance replacement.

**Interview Angle:** Contrast Instance Refresh with a manual "terminate half the fleet" approach. Explain what `MinHealthyPercentage` actually protects against.

---

## Task 9 — Lifecycle Hooks for Custom Startup/Shutdown Logic

**Scenario:** Compliance requires every instance to register itself in an external asset-tracking system on launch, and to **flush logs to S3** before termination — neither of which can happen in user data alone because they need to block the instance from going into/out of service until complete.

**Build:**
- A launch lifecycle hook that pauses new instances until your "registration" step signals completion.
- A terminate lifecycle hook that pauses termination until your "log flush" step signals completion.
- A failure scenario: what happens if your automation *never* calls `complete-lifecycle-action`? Trigger it on purpose and observe.

**Validate:**
- Confirm an instance genuinely does not receive traffic until the launch hook completes.
- Confirm you understand the default result (`ABANDON` vs `CONTINUE`) and when each is appropriate.

**Interview Angle:** Explain a real use case for lifecycle hooks beyond this scenario, and what the heartbeat timeout protects against.

---

## Task 10 — Troubleshooting Lab: Break It on Purpose

**Scenario:** You've inherited someone else's ALB/ASG setup and users are reporting intermittent 502s and slow scale-out. Nobody left documentation.

**Build a deliberately broken environment**, one issue at a time, and diagnose each using only CLI output (`describe-target-health`, `describe-scaling-activities`, alarm states, etc.) — **do not look at the console first**:
1. Health check path returns a 404 instead of 200.
2. Instance security group blocks the ALB's security group on the app port.
3. Target group port doesn't match the port your app actually listens on.
4. A scaling policy references a broken/non-existent CloudWatch alarm.
5. Max capacity is reached, so the ASG "refuses" to scale further.
6. (NLB-specific) Targets show healthy, but real clients can't connect.

**Validate:** For each scenario, write down: the *symptom* a user would report, the *diagnostic command* that reveals the root cause, and the *fix* — but implement the fix yourself; don't look one up.

**Interview Angle:** This entire task **is** the interview question "walk me through how you'd troubleshoot a 502 from an ALB." Practice narrating your diagnostic steps out loud, in order, without skipping straight to the answer.

---

## Task 11 — Cross-AZ Traffic Distribution and Cross-Zone Load Balancing

**Scenario:** You notice one AZ's instances are consistently handling far more traffic (and CPU) than another AZ's instances, even though instance counts are equal.

**Build:**
- An unbalanced setup (e.g. more targets registered in one AZ than another, or an NLB with cross-zone load balancing toggled off).
- Measure the imbalance (via CloudWatch per-instance metrics or manual request sampling).
- Fix it, and re-measure.

**Validate:** Quantify the before/after difference. Understand the **cost implication** of enabling cross-zone load balancing on an NLB (data transfer charges) vs ALB (on by default, no extra charge).

**Interview Angle:** Explain the default cross-zone behavior differences between ALB and NLB, and a scenario where you'd deliberately choose to keep it off.

---

## Task 12 — Security Hardening Pass

**Scenario:** A pre-launch security audit needs the following answered with evidence, not assumptions.

**Build/Verify:**
- Confirm the instance security group allows the app port **only** from the ALB's security group (not `0.0.0.0/0`, not a broad CIDR).
- Confirm EC2 instances have **no direct public IP/SSH exposure**; verify you can still access them via SSM Session Manager.
- Attach **AWS WAF** to the ALB with at least one managed rule group, and prove it blocks a malicious-looking request without blocking legitimate traffic.
- Review IAM: confirm the launch template's instance profile follows least privilege (no wildcard `*` permissions).

**Validate:** Attempt to access an instance directly, bypassing the ALB, and confirm it fails as designed.

**Interview Angle:** Explain defense-in-depth for this architecture layer by layer (network, SG, WAF, IAM), and how you'd explain a WAF false-positive incident to a frustrated product team.

---

## Task 13 — Cost Optimization Under a Budget Constraint

**Scenario:** Finance has capped your fleet's monthly compute spend. You must keep the same availability guarantees while cutting cost meaningfully.

**Build:**
- Convert your ASG to a **mixed instances policy** combining On-Demand and Spot, with multiple instance types for flexibility.
- Enable **capacity rebalancing** and explain what problem it solves versus relying only on the 2-minute Spot interruption notice.
- Add scheduled scale-down for predictable off-peak hours.
- Calculate (roughly) the cost difference vs your Task 1 all-On-Demand baseline.

**Validate:** Simulate a Spot interruption if your test tooling allows it, or at minimum explain exactly what would happen end-to-end when AWS reclaims one of your Spot instances.

**Interview Angle:** Explain how you'd design a workload to be Spot-safe (stateless, fast bootstrap, graceful interruption handling) and when Spot is a *bad* fit.

---

## Task 14 — Observability: Prove the System Is Healthy (or Isn't)

**Scenario:** Leadership wants a single dashboard answering "is the app healthy right now?" and wants to be **alerted before customers complain**, not after.

**Build:**
- Enable ALB access logs to S3 and query them for 5xx errors and top slow requests.
- CloudWatch alarms on: elevated 5XX rate, high target response time, and unhealthy host count > 0.
- Wire at least one alarm to a real notification (SNS email/Slack).
- Intentionally cause each alarm to fire (e.g. break health checks, generate errors) and confirm you get notified.

**Validate:** Time how long it takes from "problem starts" to "you get notified" — that gap is your real-world MTTD (mean time to detect).

**Interview Angle:** Explain the difference between monitoring the load balancer layer vs the application layer, and what blind spots exist if you only alarm on ASG-level metrics.

---

## Task 15 (Stretch) — Codify Everything as Infrastructure as Code

**Scenario:** Your manager says "if you got hit by a bus, could someone rebuild this from scratch in 10 minutes?" Right now the honest answer is no — it's all ad-hoc CLI history.

**Build:**
- Rebuild your Task 1 (or a combined) architecture using **Terraform** (or CloudFormation/CDK — pick one and justify why).
- Introduce **manual drift** (change something in the console) and detect it (`terraform plan`).
- Destroy and recreate the entire stack from code alone, confirming nothing was manually patched afterward.

**Validate:** The stack must come back up healthy with zero manual console steps.

**Interview Angle:** Explain the risks of "ClickOps" in production and how IaC changes your incident-response and change-review process.

---

## Task 16 (Stretch) — Multi-Region Failover

**Scenario:** A major customer requires your service to survive a **full regional outage**, not just an AZ outage.

**Build:**
- Duplicate your web tier (ALB + ASG) into a second region.
- Configure Route 53 with health checks and a **failover** (or latency-based) routing policy across both regional ALBs.
- Simulate a regional failure (e.g. deregister all targets in the primary region's target group) and time the failover.

**Validate:** Measure real failover time end-to-end and identify the biggest contributor to that delay (DNS TTL, health check interval, etc.).

**Interview Angle:** Explain the tradeoffs of active-active vs active-passive multi-region design, and why DNS-based failover is not instantaneous.

---

## Final Capstone Challenge — Design Under Pressure

Without writing any code or CLI first, spend 20 minutes on a whiteboard (or paper) designing the load-balancing + auto-scaling architecture for this brief, exactly as you would in a system-design interview:

> *"We're launching a flash-sale e-commerce feature. Traffic will be near-zero most of the year, then spike 50x for a 2-hour window twice a year. The checkout API must never silently drop a request. Some backend services are legacy TCP-only. Engineering leadership wants to keep monthly idle cost near-zero. You have 2 weeks."*

Then build the real thing and see how close your design held up. Note every place your first instinct was wrong — that gap is exactly what interviews are designed to probe.

---

## Quick Self-Check Before Calling Yourself "Interview-Ready"

You should be able to explain each of these **without notes**, ideally by referencing a task above where you actually did it:

- Why Launch Templates replaced Launch Configurations.
- The exact difference between EC2 and ELB health checks, and why it matters for self-healing.
- What deregistration delay protects against, and the risk of setting it too high or too low.
- Why NLB preserves source IP and ALB doesn't, and the security group implication.
- How target tracking scaling actually works under the hood (CloudWatch alarms AWS creates for you).
- The failure mode of a scaling policy pointing at a broken alarm, and how you'd catch it.
- How Instance Refresh differs from just terminating instances manually.
- Why a worker fleet should almost never scale on CPU alone.
- The cost and availability tradeoffs of Spot in an ASG.
- How you'd design this whole stack to survive an AZ outage vs a full region outage — and what changes between the two.

---

*Tear down every lab resource before closing this out. If in doubt, run through the guide's §24 Troubleshooting Playbook and §25 Best Practices Cheat Sheet one more time before your next interview.*
