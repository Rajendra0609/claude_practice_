# ☁️ AWS EC2 — Basic to Medium Practice Tasks

> Companion task list for the *AWS EC2 & Storage — Hands-On Learning Guide*.
> Goal: move past "I followed the console steps" toward **real operational understanding** — knowing why each setting exists, what breaks when it's wrong, and how to explain your reasoning out loud in an interview.

---

## How to Use This List

- Each task gives you a **scenario**, **what to build/do**, and **constraints** — no step-by-step solution, and no answers.
- Work through this in your own AWS account, ideally Free Tier (`t2.micro`/`t3.micro`), and keep your Phase 1 Budget alert active.
- After each task, write a short note in your own words explaining what happened and why — that explanation *is* the interview answer.
- Difficulty is labeled **🟢 Basic** or **🟡 Medium** so you can gauge where you are.
- Tasks are grouped into six tracks, loosely following the guide's phase order. **Always clean up resources when done** (Guide Section 3.10).

---

## Track 1 — Compute Fundamentals & Access

### Task 1 🟢 Launch, Customize, and Explain Every Choice
**Scenario:** You're asked to launch a basic web server and justify each configuration choice to a reviewer, not just click through defaults.
**Do this:**
- Launch an instance using User Data (like Guide 1.3), but this time write your own script from scratch (don't copy it) that installs a web server and prints the instance's hostname *and* current date dynamically.
- For each setting you pick during launch — AMI, instance type, key pair, Security Group rules — write one sentence justifying why you chose it over an alternative.
- Check `/var/log/cloud-init-output.log` to confirm your script actually ran, and intentionally introduce a typo in the script to see what a *failed* User Data run looks like in the logs.
**Interview angle:** "Walk me through what happens, step by step, from clicking Launch Instance to your web page being reachable."

### Task 2 🟢 Diagnose a Basic Connectivity Failure
**Scenario:** Your instance was reachable yesterday; today SSH times out and the website won't load.
**Do this:**
- Deliberately break connectivity in two separate, controlled ways: remove the port 80 rule from the Security Group, and separately, stop the `httpd`/web service on the instance itself while keeping the Security Group intact.
- For each broken state, practice distinguishing "network-level unreachable" from "service-level down" using `curl`, `systemctl status`, and the Security Group rules — don't just guess, verify.
- Write a short two-item checklist: what you check first when a *site* is unreachable vs when *SSH* is unreachable.
**Interview angle:** "A website hosted on EC2 suddenly stops responding — how do you start troubleshooting?"

### Task 3 🟢 SSH Access the Two Different Ways
**Scenario:** You need a documented, repeatable way to access an instance that doesn't always depend on having your `.pem` file handy.
**Do this:**
- Connect to the same running instance twice: once with your `.pem` key via standard SSH, and once via **EC2 Instance Connect** in the browser.
- Deliberately try connecting with the wrong username (e.g., `ubuntu` on an Amazon Linux box) and read the actual error message you get.
- Fill out the guide's SSH troubleshooting checklist against your own instance and note which check would have caught each mistake you made.
**Interview angle:** "What are the different ways to access an EC2 instance, and when would you use each?"

### Task 4 🟡 Attach an IAM Role Instead of Hardcoding Credentials
**Scenario:** A teammate suggests just running `aws configure` on the EC2 instance with their own access key "to get it working quickly." You need to explain — and demonstrate — the better way.
**Do this:**
- Attach an IAM Role with `AmazonS3ReadOnlyAccess` to a running instance and confirm `aws s3 ls` works without any stored credentials.
- From inside the instance, query the metadata endpoint to see the temporary credentials the role provides, and note their expiry.
- Try (and fail, on purpose) to perform an action the role doesn't allow (e.g., `s3 cp` a file up) and read the resulting error carefully.
**Interview angle:** "Why shouldn't you put access keys on an EC2 instance, and what's the standard alternative?"

---

## Track 2 — Networking Basics

### Task 5 🟢 Watch a Public IP Change, Then Fix It
**Scenario:** A junior teammate is confused why their instance's IP address is "different every morning."
**Do this:**
- Note your instance's Public IP, stop it, start it again, and confirm the Public IP actually changed.
- Allocate an Elastic IP, associate it with the instance, then repeat the stop/start — confirm the address now stays fixed.
- Calculate (using AWS pricing) what an unattached Elastic IP would cost you per month if you forgot to release it, and add "release unused EIPs" to your personal cleanup habit.
**Interview angle:** "What's the difference between a Public IP and an Elastic IP, and when do you actually need an Elastic IP?"

### Task 6 🟡 Compare a Placement Group to "No Placement Group"
**Scenario:** A teammate wants to know if placement groups are "worth it" for a small internal app.
**Do this:**
- Launch two instances with no placement group and measure basic latency/throughput between them (e.g., `ping`, or timing a file copy).
- Launch two more instances inside a **Cluster** Placement Group and repeat the same measurement.
- Based on your own numbers (not assumptions), write one paragraph on when a Cluster Placement Group is worth the added failure-domain risk, and when it clearly isn't.
**Interview angle:** "What's a placement group, and would you recommend one for a typical two-tier web app?"

### Task 7 🟡 Move a Private IP Between Instances With an ENI
**Scenario:** You want to understand, hands-on, how a "virtual network card" can move independently of the instance it's attached to.
**Do this:**
- Create a secondary ENI, attach it to a running instance, and confirm it shows up as `eth1` when you run `ip addr`.
- Detach it and attach it to a *different* instance in the same AZ, and confirm the same private IP now appears there instead.
- Try to attach the ENI to an instance in a *different* AZ and document the exact error you get — explain in your own words why this restriction exists.
**Interview angle:** "How would an ENI help you build a manual failover between two EC2 instances?"

### Task 8 🟢 Hibernate vs Stop vs Terminate — See the Difference Yourself
**Scenario:** A colleague uses "stop" and "hibernate" interchangeably and isn't sure which one to use for a long-running analysis job.
**Do this:**
- Launch an instance with an **encrypted** root volume and Hibernate enabled, start a background process (`sleep 1000 &`), then Hibernate it and start it again — confirm the process survived with `ps aux`.
- Repeat the same experiment but this time just **Stop/Start** the instance instead, and confirm the process did *not* survive.
- Write a one-line rule of thumb for choosing Stop vs Hibernate vs Terminate that you could say out loud in an interview.
**Interview angle:** "What's the practical difference between stopping and hibernating an EC2 instance?"

---

## Track 3 — Storage Basics

### Task 9 🟢 Create, Attach, Format, and Verify an EBS Volume
**Scenario:** A database needs additional disk space, and you're asked to add and mount a new volume without touching the existing root disk.
**Do this:**
- Create a `gp3` EBS volume in the same AZ as a running instance, attach it, then format and mount it yourself using `lsblk`, `mkfs`, and `mount` (don't just copy commands blindly — understand what each one does).
- Write a test file to it, then **detach and reattach** the volume and confirm the data is still there.
- Deliberately try to attach the same volume to an instance in a *different* AZ and document what happens.
**Interview angle:** "Walk me through adding a new disk to an EC2 instance from scratch."

### Task 10 🟢 Use a Snapshot to Move Data Across AZs
**Scenario:** You need to bring a copy of an existing volume's data into a different Availability Zone for a new instance.
**Do this:**
- Create a snapshot of a volume that has real data on it.
- Create a **new volume from that snapshot in a different AZ**, attach it to a new instance there, and confirm the data made the trip intact.
- Note how long the whole process took and think through what you'd need to automate if this had to happen on a schedule.
**Interview angle:** "How would you move data from an EBS volume in one AZ to another?"

### Task 11 🟡 Build a Custom AMI and Launch From It
**Scenario:** New servers currently require manual setup after every launch, which is slow and inconsistent.
**Do this:**
- Fully configure one instance (install and configure a web server, add a config file, etc.), then create a Custom AMI from it.
- Launch a **new** instance purely from your custom AMI and confirm it's pre-configured with zero manual steps.
- Deregister the AMI when done and note what happens to snapshots associated with it — is deregistering enough, or is there another cleanup step?
**Interview angle:** "Why would you create a custom AMI instead of just using a launch script every time?"

### Task 12 🟡 Retrofit Encryption on an Existing Unencrypted Volume
**Scenario:** You discover a volume with real (test) data on it was never encrypted, and you need to fix that without losing the data.
**Do this:**
- Start with an unencrypted volume containing a test file.
- Follow the snapshot → copy-with-encryption → new volume → attach flow to end up with an **encrypted** volume containing the same data.
- Enable account-level default EBS encryption afterward and confirm a brand-new volume is encrypted automatically without you specifying it.
**Interview angle:** "How do you encrypt an EBS volume that was created without encryption?"

### Task 13 🟡 Share a File Between Two Instances With EFS
**Scenario:** Two web servers need access to the same set of uploaded files.
**Do this:**
- Create an EFS file system, mount it on two instances (ideally in different AZs), and confirm a file written from one instance is immediately readable from the other.
- Compare this experience directly to what you did with EBS in Task 9 — write down, specifically, what EFS let you do that EBS could not.
- Look up (don't guess) whether EFS works on Windows instances, and note the implication for a mixed OS environment.
**Interview angle:** "When would you reach for EFS instead of EBS?"

---

## Track 4 — Cost Awareness & Cleanup

### Task 14 🟢 Find and Eliminate Waste in Your Own Account
**Scenario:** You're asked to do a quick pass through an account to catch obvious wasted spend before month-end billing.
**Do this:**
- Intentionally create a small amount of "waste" first: an unattached EBS volume, an unassociated Elastic IP, and a stopped-but-not-terminated instance you no longer need.
- Find all three using the EC2 console (Volumes filtered by `available` state, Elastic IPs, and Instances filtered by state) and clean them up.
- Check AWS Budgets/Cost Explorer to see whether your test resources showed up as cost drivers, even at Free Tier scale.
**Interview angle:** "What are the most common sources of wasted EC2/EBS spend, and how do you find them?"

### Task 15 🟡 Compare On-Demand, Spot, and Right-Sizing on the Same Workload
**Scenario:** Your manager wants a simple, evidence-based recommendation on where the team is overspending.
**Do this:**
- Launch a small instance as a normal On-Demand instance, then launch an equivalent one as a **Spot Instance**, and compare the console's shown pricing/savings for both.
- Deliberately oversize an instance for a trivial workload (e.g., launch an `m5.large` to run a task that barely uses CPU), then use **AWS Compute Optimizer** (or manual CPU monitoring) to see what it recommends.
- Write a short recommendation memo: which of these three levers (purchase model, right-sizing, or Spot) would you pull first for a fault-tolerant batch job vs a steady-state web server?
**Interview angle:** "How would you reduce EC2 costs for a team without hurting reliability?"

---

## Track 5 — Troubleshooting Drills

### Task 16 🟡 "Break It, Then Fix It" Self-Challenge
**Scenario:** The fastest way to build real troubleshooting instinct is to diagnose something you don't already know the cause of.
**Do this:**
- Pick **one** fault at random and introduce it without writing down which one you chose: a wrong Security Group rule, a stopped service, a full disk (fill `/` with a large dummy file), or a detached IAM role.
- Wait a few hours (or have a friend do it for you), then diagnose and fix it using only the tools available on the instance and in the console — no shortcuts, no peeking at what you changed.
- Write a "5 whys" note afterward: what broke, how you found it, and what you'd monitor going forward to catch it faster next time.
**Interview angle:** This is essentially a live "debug this" interview exercise — doing it to yourself is the best practice you can get.

---

## Suggested Practice Cadence

```
Day 1–2: Track 1 (Compute fundamentals + access + IAM role basics)
Day 3–4: Track 2 (Networking — IPs, placement groups, ENI, hibernate)
Day 5–6: Track 3 (Storage — EBS, snapshots, AMI, encryption, EFS)
Day 7:   Track 4 (Cost awareness + cleanup)
Day 8:   Track 5 (Troubleshooting drill) + mock interview using the "Interview angle" prompts above
```

## Before You Call a Task "Done"

- [ ] Could you explain what you did to a non-technical manager in 2 sentences?
- [ ] Could you explain the same thing to a senior engineer, including one alternative approach you considered?
- [ ] Did you verify the result yourself (logs, `curl`, `ps aux`, console state) rather than assuming it worked?
- [ ] Did you clean up every resource you created?

---
*Do the labs yourself — the value is in watching things break and fixing them, not in reading about it. Good luck! ☁️*
