# 🚀 AWS EC2 — Advanced Hands-On Practice Tasks

> Companion task list for the *AWS EC2 & Storage — Hands-On Learning Guide*.
> Goal: move beyond following console steps toward **production-grade judgment** — designing, breaking, diagnosing, and defending EC2 architectures the way you'd need to on the job and in an interview.

---

## How to Use This List

- Each task gives you a **scenario**, **what to build/do**, and **constraints** — no step-by-step solution, and no answers.
- Do the work in your own AWS account (Free Tier where possible — watch your Budget alerts from Phase 1).
- After each task, write your own short "runbook" or postmortem in your notes — this is exactly what interviewers want to hear you talk through.
- Tasks are grouped into six tracks. You don't need to do them in strict order, but Track 1 assumes you've completed Guide Phase 1.
- **Always tear down resources when done** (see the Guide's Section 3.10 cleanup checklist).

---

## Track 1 — Core Compute & Access Resilience

### Task 1: Recover Access After Losing Your Key Pair
**Scenario:** You SSH into a production instance regularly, but today your `.pem` file is gone (laptop wiped) and the instance has no other admin access configured.
**Do this:**
- Simulate the situation on a running instance and regain SSH access **without terminating the instance or losing its data**.
- Explore at least two different recovery approaches (e.g., involving the root volume, a temporary instance, EC2 Instance Connect, or user data) and note the trade-offs of each.
- Document what pre-emptive setup would have prevented this entirely.
**Interview angle:** "A teammate locked themselves out of a prod box and the key is gone — walk me through what you'd do."

### Task 2: Diagnose a "Website Not Loading" Incident
**Scenario:** Your `my-web-server` instance from Guide 1.3 was working yesterday. Today, the public IP loads nothing in the browser.
**Do this:**
- Deliberately break the setup in **three different ways** (one at a time): a Security Group misconfiguration, a stopped/crashed service, and a changed Public IP after a stop/start.
- For each break, practice a structured diagnosis: is it a network reachability issue, an OS/service issue, or a DNS/IP issue? Use `curl`, `systemctl`, Security Group rules, and the metadata endpoint to narrow it down.
- Write a short troubleshooting checklist you'd hand to a junior engineer for "site is down" tickets.
**Interview angle:** "How do you systematically troubleshoot an EC2-hosted app that's unreachable?"

### Task 3: Least-Privilege IAM Role Design
**Scenario:** An app on EC2 needs to read from one specific S3 bucket and write logs to CloudWatch — nothing else.
**Do this:**
- Instead of attaching `AmazonS3ReadOnlyAccess` (Guide 1.6), write a **custom least-privilege IAM policy** scoped to a single bucket/prefix and the minimum CloudWatch Logs actions.
- Attach it via an instance profile, then verify from inside the instance that allowed actions succeed and disallowed actions (e.g., listing a different bucket, deleting objects) are denied.
- Test what happens to running processes if you swap the IAM role on a live instance — do existing AWS CLI sessions need a restart?
**Interview angle:** "Why shouldn't you just attach `AdministratorAccess` to make things work?" and "How do temporary credentials from an instance profile actually get delivered to the instance?"

### Task 4: IMDSv2 Hardening Against SSRF
**Scenario:** A security review flags that your instances allow IMDSv1, which is vulnerable to credential theft via Server-Side Request Forgery (SSRF) in a web app.
**Do this:**
- On a test instance, first retrieve instance metadata using the old IMDSv1-style unauthenticated GET request.
- Enforce **IMDSv2-only** (token-required) access via the instance's metadata options, then confirm the old-style request now fails.
- Explain, in your own notes, how a real SSRF vulnerability in an application could be used to steal IAM role credentials from IMDS, and why IMDSv2's token requirement mitigates it.
**Interview angle:** "What is the metadata service, and what's the security risk if it's misconfigured?" — a very common EC2 security interview question.

---

## Track 2 — Networking & High Availability

### Task 5: Build a Failover Pair Using ENI Detach/Attach
**Scenario:** You run a legacy stateful app that clients connect to via a fixed private IP. It cannot sit behind a Load Balancer. You need a manual failover mechanism.
**Do this:**
- Launch a primary and a standby instance in the same AZ. Create a dedicated ENI with a fixed private IP and attach it to the primary.
- Simulate a primary failure, then detach the ENI and attach it to the standby within a measured time window.
- Time the full failover and identify what state (in-flight connections, application state) is lost vs preserved.
- Note explicitly why this approach cannot work across AZs, and what you'd use instead for a cross-AZ failover requirement.
**Interview angle:** "How would you design failover for a legacy app that can't use a load balancer?"

### Task 6: Elastic IP vs DNS-Based Failover Cost/Speed Comparison
**Scenario:** Leadership wants a fixed public endpoint for a critical service and is debating Elastic IP re-association vs Route 53 DNS failover.
**Do this:**
- Implement both approaches against two instances: (a) reassociating an Elastic IP, (b) updating a DNS record to point to a new Public IP.
- Measure and record the effective "time to recover" for each approach, including DNS TTL effects.
- Produce a one-page comparison (cost, speed, operational complexity, blast radius) you could present to a team lead.
**Interview angle:** "When would you choose an Elastic IP over DNS-based failover, and what are the trade-offs?"

### Task 7: Placement Group Failure-Domain Experiment
**Scenario:** A data engineering team wants low inter-node latency for a Hadoop-like cluster but is worried about a single rack failure taking down the whole cluster.
**Do this:**
- Launch instances into a **Cluster** Placement Group and measure inter-instance latency/throughput (e.g., with `ping`, `iperf`, or timed file transfer) vs instances with no placement group.
- Repeat with a **Partition** Placement Group across multiple partitions, and reason through how partition count relates to fault tolerance.
- Attempt to move a *running* instance into a placement group after the fact, and document what actually happens and why planning ahead matters.
**Interview angle:** "What's the difference between Cluster, Spread, and Partition placement groups, and when would you pick each for a real workload?"

### Task 8: Secondary ENI for Segregated Management Traffic
**Scenario:** Compliance requires that SSH/admin traffic never traverse the same network path as application traffic.
**Do this:**
- Attach a second ENI (in a separate subnet) to an instance and configure the OS to route SSH traffic over `eth1` while app traffic stays on `eth0`.
- Verify the separation using `ip route` / `ip addr` and by testing connectivity from each subnet independently.
- Identify the ENI limits for your chosen instance type and what would break if you tried to exceed them.
**Interview angle:** "How would you isolate management traffic from application traffic on the same EC2 instance?"

---

## Track 3 — Storage Engineering & Data Resilience

### Task 9: Diagnose and Fix an EBS Performance Bottleneck
**Scenario:** A database on a `gp2` volume is experiencing latency spikes during peak load; the app team suspects "disk is too slow."
**Do this:**
- Provision a small `gp2` volume, generate sustained I/O load against it (e.g., with `fio` or `dd`), and observe baseline IOPS/throughput ceilings.
- Migrate the volume to `gp3`, independently tune IOPS and throughput, and re-run the same load test.
- Build a short decision matrix: at what point would you reach for `io1`/`io2` instead of `gp3`, and why?
**Interview angle:** "A team says their database disk is too slow — what EBS-level things would you check and try before assuming it's a code problem?"

### Task 10: Cross-AZ Disaster Recovery via Snapshots
**Scenario:** Your only instance's AZ has an outage. You need to bring the workload back up in a different AZ within your recovery time objective.
**Do this:**
- Create a snapshot of a live volume with data on it, then restore it as a new volume **in a different AZ**, and boot a new instance from it.
- Time the entire recovery process end-to-end and identify what data (if any) would be lost given normal snapshot scheduling.
- Design (on paper) an automated snapshot + cross-AZ/cross-region copy schedule using Data Lifecycle Manager that would meet a defined RPO.
**Interview angle:** "Walk me through how you'd recover an EC2 workload if an entire Availability Zone went down."

### Task 11: Build and Roll Out a Golden AMI Pipeline
**Scenario:** Every new server currently gets manually configured after boot, causing config drift and slow scale-out.
**Do this:**
- Fully configure an instance (packages, hardened settings, monitoring agent, app baseline) and bake it into a custom AMI.
- Launch several new instances purely from the AMI and verify zero manual post-boot steps are needed.
- Simulate a "patch Tuesday": update the base AMI, and design a rollout plan for replacing running instances with the new AMI version with minimal downtime.
**Interview angle:** "How do you handle OS/security patching across a fleet of EC2 instances without manual SSH into each one?"

### Task 12: Shared Storage with EFS — Consistency and Locking
**Scenario:** Two web servers behind a load balancer need to share uploaded user files.
**Do this:**
- Mount the same EFS file system on two instances in different AZs and validate read-after-write visibility between them.
- Simulate two processes on different instances writing to the same file concurrently and observe what happens (data corruption risk, need for app-level locking).
- Compare EFS Standard vs EFS-IA cost/access-latency trade-offs using the Lifecycle Management feature, and decide where the crossover point is for your data access pattern.
**Interview angle:** "When would you choose EFS over an EBS Multi-Attach volume for shared storage, and what are the consistency implications?"

### Task 13: Retrofit Encryption on an Unencrypted Production Volume
**Scenario:** An audit discovers a production EBS volume holding sensitive data was never encrypted.
**Do this:**
- Starting from an unencrypted volume with live data on it, perform the full remediation (snapshot → copy with encryption → new volume → swap) **without data loss and with minimal downtime**.
- Time how long the swap takes and think through how you'd do this for a volume too large to tolerate the downtime window — what's your mitigation?
- Enable account-level default encryption and verify it applies automatically to a newly created volume.
**Interview angle:** "How do you encrypt an existing unencrypted EBS volume in production without losing data?"

---

## Track 4 — Cost Engineering & Capacity Planning

### Task 14: Run a Real Cost Audit and Justify Every Recommendation
**Scenario:** You've inherited an AWS account with unclear spend and no idea what's idle vs needed.
**Do this:**
- Use Cost Explorer to identify your top cost drivers by instance type/service, then cross-reference with Compute Optimizer recommendations.
- Deliberately create some "waste" first (an unattached EBS volume, an idle Elastic IP, an oversized instance running at low CPU) so you have real findings to detect and clean up.
- Produce a short cost-optimization report: what you found, estimated savings, and the risk/trade-off of each recommended change (e.g., what could break if you right-size too aggressively).
**Interview angle:** "How would you reduce EC2 costs by 30% for a team that's been running everything on-demand?"

### Task 15: Design a Mixed-Purchase Fleet with Spot Interruption Handling
**Scenario:** A batch-processing workload is fault-tolerant and cost-sensitive but must never fully stall out.
**Do this:**
- Configure a fleet (Spot Fleet or an Auto Scaling group with mixed instance policy) combining On-Demand for baseline and Spot for burst capacity, using the `priceCapacityOptimized` allocation strategy.
- Write and test a script that polls the instance metadata for the 2-minute Spot interruption warning and performs a graceful shutdown (e.g., finishing/checkpointing in-flight work) before termination.
- Force an interruption scenario (or simulate one) and verify your fleet self-heals to target capacity.
**Interview angle:** "How do you architect a workload to safely use Spot Instances without risking job failures?"

---

## Track 5 — Chaos & Troubleshooting Drills

### Task 16: "Break It On Purpose" Peer Exercise
**Scenario:** The fastest way to get interview-sharp is to fix things you didn't break yourself.
**Do this:**
- If possible, have a friend/colleague (or do this to yourself and forget the details after a day) introduce **one** unannounced fault into a running environment: e.g., a revoked IAM permission, a changed Security Group rule, a full disk, a corrupted `fstab` entry, an expired/rotated credential, or a placement group instance that failed to launch.
- Diagnose and resolve it using only AWS Console/CLI tools and logs (`/var/log/cloud-init-output.log`, `journalctl`, CloudTrail if enabled) — no hints.
- Afterward, write a "5 whys" root-cause note: what broke, why, how you found it, and what monitoring/alerting would have caught it sooner.
**Interview angle:** This is essentially a live "debug this system" interview exercise — practicing it directly builds the muscle for it.

### Task 17: Full-Stack Resilience Simulation
**Scenario:** Combine everything into one end-to-end production simulation.
**Do this:**
- Stand up a two-instance web tier (different AZs) behind a shared EFS mount, with encrypted `gp3` root volumes, least-privilege IAM roles, a custom AMI for consistent config, and a snapshot lifecycle policy for backups.
- Kill one instance's AZ conceptually (stop it and pretend it's unavailable) and validate the remaining instance keeps serving traffic.
- Estimate the monthly cost of this setup, then re-design it with one purchase-model change (e.g., Reserved Instance for baseline) and quantify the savings.
**Interview angle:** This mirrors "design a highly available, cost-efficient EC2 architecture" system-design interview questions almost exactly.

---

## Suggested Practice Cadence

```
Week 1: Track 1 (Access + Troubleshooting fundamentals)
Week 2: Track 2 (Networking + HA)
Week 3: Track 3 (Storage engineering)
Week 4: Track 4 (Cost + Spot resilience)
Week 5: Track 5 (Chaos drills + full simulation) + mock interview Q&A using the "Interview angle" prompts above
```

## Before You Call a Task "Done"

- [ ] Could you explain your fix to a non-technical manager in 2 sentences?
- [ ] Could you explain the same fix to a senior engineer, including trade-offs you considered and rejected?
- [ ] Did you capture the relevant CLI commands/console paths in your own notes (not this doc)?
- [ ] Did you clean up every resource you created?

---
*Do the labs yourself — the value is in the debugging, not the destination. Good luck! ☁️*
