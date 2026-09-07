# AWS VPC Networking Lab — Hands-On Console Guide

A 15-task, click-by-click lab for learning AWS VPC networking (subnets, routing, NAT, Security Groups, NACLs, Peering, Transit Gateway, VPC Endpoints, and Flow Logs) by **doing it yourself in the AWS Console**, in real time. This file is a guide, not an automation script — every task below tells you exactly which console screen to open and which buttons to click, then tells you what to observe once you get there. Work through it in order; later tasks build on resources created in earlier ones.

> **Do this manually.** The value of this lab is muscle memory — clicking through the actual console, watching a route table update, watching a Session Manager terminal, watching a NACL rule silently drop a packet. Don't script it. Type the values in yourself.

---

## ⚠️ Before You Start

- **Cost warning.** This lab intentionally creates billable resources: NAT Gateway(s), Elastic IPs, EC2 instances, Interface VPC Endpoints, and a Transit Gateway. None of these are in the AWS Free Tier's "always free" category (NAT Gateway and Transit Gateway attachments bill by the hour even when idle). Check the [AWS Pricing page](https://aws.amazon.com/vpc/pricing/) for current rates before you begin, and **do the cleanup checklist at the end of every session** you don't plan to continue immediately.
- **Region.** Pick one region and stay in it for the whole lab (e.g. `us-east-1`). Switching regions mid-lab is the #1 cause of "I can't find my VPC" confusion.
- **Permissions.** You'll need an IAM identity with permissions for VPC, EC2, Systems Manager (SSM), IAM (to create an instance role), CloudWatch Logs, and Transit Gateway. Admin or PowerUserAccess is easiest for a lab account.
- **SSM prerequisite.** Tasks 7, 8, 9, and 15 use **SSM Session Manager** instead of SSH keys/bastions to reach the private instance. Before Task 6, create an IAM role:
  - IAM Console → Roles → Create role → Trusted entity: **AWS service** → Use case: **EC2**
  - Attach policy: `AmazonSSMManagedInstanceCore`
  - Name it e.g. `lab-ssm-instance-role` → Create role
  - You'll attach this role to your EC2 instances in Task 6 (as the "IAM instance profile").
- **AMI.** Use **Amazon Linux 2023** for all instances — it ships with the SSM Agent and AWS CLI pre-installed, which every later task depends on.

---

## Table of Contents

1. [Build the VPC with "VPC and more"](#task-1)
2. [Identify public vs. private subnets via route tables](#task-2)
3. [Create (but don't attach) a Security Group](#task-3)
4. [Trace the traffic path in the Resource Map](#task-4)
5. [Allocate and release an Elastic IP](#task-5)
6. [Launch a public and a private EC2 instance](#task-6)
7. [Verify NAT outbound / no inbound from the private instance](#task-7)
8. [NACL deny rule vs. Security Group allow — stateless wins](#task-8)
9. [Break and fix SG-to-SG connectivity](#task-9)
10. [VPC Peering with non-overlapping CIDRs](#task-10)
11. [VPC Peering rejection with overlapping CIDRs](#task-11)
12. [3-tier VPC + Flow Logs + Logs Insights REJECT query](#task-12)
13. [Transit Gateway across 3 VPCs](#task-13)
14. [Replace NAT Gateway with VPC Endpoints for S3 + Secrets Manager](#task-14)
15. [Blind fault injection + Networking Decision Tree](#task-15)
16. [Cleanup checklist](#cleanup)

---

<a id="task-1"></a>
## Task 1 — Build the VPC with "VPC and more"

**Goal:** stand up the base network you'll use for most of the lab: 2 AZs, 2 public + 2 private subnets, 1 NAT Gateway, an S3 Gateway endpoint.

**Steps:**
1. Console search bar → type `VPC` → open the **VPC** service.
2. Left sidebar → **Your VPCs** → click **Create VPC**.
3. At the top, choose **VPC and more** (not "VPC only") — this is the wizard that provisions subnets, route tables, an IGW, and NAT together.
4. **Name tag auto-generation:** `lab` (this prefixes all generated resource names, e.g. `lab-vpc`, `lab-subnet-public1-...`).
5. **IPv4 CIDR block:** `10.0.0.0/16`.
6. **IPv6 CIDR block:** No IPv6 CIDR block (leave default — keep the lab IPv4-only).
7. **Tenancy:** Default.
8. **Number of Availability Zones (AZs):** `2`. Click **Customize AZs** if you want to pick specific ones (e.g. `us-east-1a`, `us-east-1b`); otherwise the wizard picks two for you.
9. **Number of public subnets:** `2`.
10. **Number of private subnets:** `2`.
11. Leave the subnet CIDR blocks at their auto-generated defaults (you'll read the actual values back in Task 2).
12. **NAT gateways:** select **In 1 AZ** (this is the "1 NAT Gateway" the task calls for — it goes in one of the public subnets and is shared by both private subnets).
13. **VPC endpoints:** select **S3 Gateway**.
14. **DNS options:** leave both **Enable DNS hostnames** and **Enable DNS resolution** checked.
15. Scroll down and review the **Preview** panel — it renders a live diagram of everything about to be created.
16. Click **Create VPC**. This takes 1–3 minutes; the wizard shows a progress list for each sub-resource (VPC, subnets, route tables, IGW, NAT GW, endpoint).

**Confirm:** once it says "Create VPC succeeded," go to **Your VPCs** and confirm you see 1 VPC, 4 subnets, 1 IGW, 1 NAT Gateway, and 1 endpoint, all tagged `lab-*`.

---

<a id="task-2"></a>
## Task 2 — Identify public vs. private subnets via route tables

**Goal:** stop trusting subnet *names* and learn to read the route table, which is the actual source of truth for "is this subnet public or private."

**Steps:**
1. VPC Console → **Route tables** (left sidebar).
2. You should see two route tables tagged something like `lab-rtb-public` and `lab-rtb-private` (names are a hint here, but verify below rather than trust them).
3. Click the first route table → **Routes** tab (bottom panel).
4. Look for a row with **Destination `0.0.0.0/0`**. Check the **Target** column:
   - Target starts with `igw-...` → this route table sends all outbound traffic straight to the **Internet Gateway** → subnets using this table are **public**.
   - Target starts with `nat-...` → outbound traffic goes to the **NAT Gateway** first → subnets using this table are **private**.
5. Click the **Subnet associations** tab on the same route table to see which subnet ID(s) use it.
6. Repeat for the second route table.
7. Cross-check: go to **Subnets** (left sidebar) — there's a **Route table** column you can click straight through to the same route table, which is a faster way to do this lookup once you understand the underlying logic.

**Confirm:** you can point to each of the 4 subnets and state, from the route table alone (not the name), whether it's public or private, and why.

---

<a id="task-3"></a>
## Task 3 — Create (but don't attach) a Security Group

**Goal:** get comfortable authoring SG rules before they're live on anything.

**Steps:**
1. EC2 Console → left sidebar → **Network & Security → Security Groups**.
2. Click **Create security group**.
3. **Security group name:** `lab-sg-web`.
4. **Description:** `Lab SG: SSH from my IP, HTTP from anywhere`.
5. **VPC:** select `lab-vpc`.
6. **Inbound rules** → **Add rule**:
   - Rule 1 — Type: `SSH`, Source: click the **Source** dropdown → **My IP** (auto-fills your current public IP as a `/32`).
   - Rule 2 — Type: `HTTP`, Source: **Anywhere-IPv4** (`0.0.0.0/0`).
7. **Outbound rules:** leave the default (`All traffic` → `0.0.0.0/0`) — don't change this yet.
8. Click **Create security group**.
9. **Do not** attach it to any instance or ENI yet — just open it back up afterward and re-read the two inbound rules you created, confirming the source values are what you intended.

**Confirm:** the SG exists, shows "0 network interfaces" (not attached to anything), and the inbound rules read `22 ← <your-ip>/32` and `80 ← 0.0.0.0/0`.

---

<a id="task-4"></a>
## Task 4 — Trace the traffic path in the Resource Map

**Goal:** visually connect Internet → IGW → public subnet using the console's built-in diagram instead of clicking between five separate screens.

**Steps:**
1. VPC Console → **Your VPCs** → click `lab-vpc`.
2. Open the **Resource map** tab on the VPC detail page.
3. You'll see a diagram with the VPC as the outer box, containing: the Internet Gateway (top, connected to a line labeled "Internet"), route tables in the middle, and subnets (with their attached NAT Gateway / endpoint icons) at the bottom.
4. Follow the line from **Internet** → **Internet Gateway** → the **public route table** → the **public subnets**. This is the literal path a packet takes.
5. Now follow **private subnets** → **private route table** → **NAT Gateway** → **public subnet** → **Internet Gateway** → **Internet**. Notice private subnets have no direct line to the IGW — everything funnels through the NAT Gateway first.
6. Hover over/click individual nodes in the map — it opens a side panel with that resource's details, letting you jump straight to it without leaving the diagram.

**Confirm:** you can narrate the full path out loud for both a public-subnet resource and a private-subnet resource, using only what's on the diagram.

---

<a id="task-5"></a>
## Task 5 — Allocate and release an Elastic IP

**Goal:** see the idle-EIP charge warning firsthand — this is one of the most common "surprise AWS bill" line items.

**Steps:**
1. VPC Console → **Elastic IPs** (left sidebar).
2. Click **Allocate Elastic IP address**.
3. **Network Border Group:** leave default. **Public IPv4 address pool:** **Amazon's pool of IPv4 addresses**.
4. Click **Allocate**.
5. Once allocated, look at the **Elastic IPs** list page — AWS shows a banner/column noting that an EIP **not associated with a running instance** (or associated with a stopped instance / unattached ENI) accrues an hourly charge. Read it.
6. Select the checkbox next to your new EIP → **Actions** → **Release Elastic IP addresses** → confirm.

**Confirm:** the EIP no longer appears in the list, and you can explain in your own words why AWS charges for *idle* EIPs specifically (to discourage hoarding scarce IPv4 address space).

---

<a id="task-6"></a>
## Task 6 — Launch a public and a private EC2 instance

**Goal:** put a real instance in each subnet type and prove the public/private distinction with actual reachability, not just route table theory.

**Steps — public instance:**
1. EC2 Console → **Instances** → **Launch instances**.
2. **Name:** `lab-public-instance`.
3. **AMI:** Amazon Linux 2023 (free-tier eligible).
4. **Instance type:** `t3.micro` (or `t2.micro` if that's your free-tier default).
5. **Key pair:** you can select "Proceed without a key pair" since you'll primarily use EC2 Instance Connect or SSM — but a key pair doesn't hurt if you want SSH too.
6. **Network settings** → click **Edit**:
   - VPC: `lab-vpc`.
   - Subnet: one of the **public** subnets you identified in Task 2.
   - Auto-assign public IP: **Enable**.
   - Security group: select **existing** → `lab-sg-web` from Task 3.
7. **Advanced details** → **IAM instance profile** → select `lab-ssm-instance-role`.
8. Click **Launch instance**.

**Steps — private instance:**
1. **Launch instances** again.
2. **Name:** `lab-private-instance`.
3. Same AMI/instance type.
4. **Network settings:**
   - VPC: `lab-vpc`.
   - Subnet: one of the **private** subnets.
   - Auto-assign public IP: leave as default (it should show **Disable**, inherited from the subnet setting — private subnets shouldn't auto-assign public IPs).
   - Security group: create a new one, `lab-sg-private`, inbound: SSH from the VPC CIDR (`10.0.0.0/16`) only, for now.
5. **Advanced details** → **IAM instance profile** → `lab-ssm-instance-role` (this is what lets you reach it in Task 7 — there is no SSH key path into this instance by design).
6. Click **Launch instance**.

**Confirm reachability:**
- Select `lab-public-instance` → **Connect** → **EC2 Instance Connect** tab → **Connect**. You should get a working terminal.
- Try to reach `lab-private-instance`: note it has no public IP at all in the instance details — there is nothing to connect *to* from the internet. Confirm the **Public IPv4 address** field is blank/`—` for it. That absence is the actual reason it's "not reachable," independent of any SG/NACL rule.

---

<a id="task-7"></a>
## Task 7 — Verify NAT outbound / no inbound, from the private instance

**Goal:** prove the private instance can reach the internet *outbound* (via NAT) while nothing can reach it *inbound*.

**Steps:**
1. Systems Manager Console → **Session Manager** (left sidebar, under **Node Management**) → **Start session**.
2. Select `lab-private-instance` from the list → **Start session**. (If it doesn't appear, wait 1–2 minutes for the SSM Agent to register, and double-check the IAM instance profile from Task 6.)
3. In the browser-based terminal that opens, run:
   ```
   curl -s https://checkip.amazonaws.com
   ```
   This should return a public IP address — **the NAT Gateway's Elastic IP**, not the instance's own private IP. That confirms outbound internet access is working and is being source-NAT'd through the NAT Gateway.
4. Also try:
   ```
   curl -Is https://www.amazon.com | head -1
   ```
   You should get an `HTTP/2 200` (or similar) response, confirming general outbound web access.
5. To confirm **inbound is not possible**: note the private instance's private IP (visible on the EC2 console **Details** tab, or run `hostname -I` in the SSM session). From your own laptop (outside AWS), there is no route to that `10.0.x.x` address at all — it isn't attempt-able, let alone blocked. This is different from a Security Group *denying* a connection attempt; there's simply no path.
6. If you want to demonstrate an actual *blocked* attempt rather than an *unroutable* one, that's what Task 8 and Task 9 do next, from inside the VPC.

**Confirm:** you got the NAT Gateway's IP back from `checkip.amazonaws.com` (not the instance's private IP), and you can explain the difference between "unreachable because unroutable" (this task) and "unreachable because blocked by a rule" (Tasks 8–9).

---

<a id="task-8"></a>
## Task 8 — NACL deny rule vs. Security Group allow: stateless wins

**Goal:** directly observe that a Network ACL deny overrides a Security Group allow, and understand *why* (NACLs are stateless and evaluated at the subnet boundary; Security Groups are stateful and evaluated at the ENI).

**Setup — a "tester" instance in the same subnet:**
1. Launch a third instance, `lab-tester`, in the **same private subnet** as `lab-private-instance` (same subnet, so subnet-level NACL rules apply to traffic between them too), with SG `lab-sg-private` and the SSM instance profile, same as before.
2. On `lab-private-instance`'s security group (`lab-sg-private`), confirm the inbound rule allows SSH (port 22) from the VPC CIDR `10.0.0.0/16` — if you didn't add this in Task 6, add it now: EC2 → Security Groups → `lab-sg-private` → Inbound rules → Edit → Add rule → SSH → Source `10.0.0.0/16`.

**Baseline (SG allows, no NACL block yet):**
3. Open an SSM session to `lab-tester`, and from its terminal try:
   ```
   nc -zv <lab-private-instance-private-ip> 22
   ```
   (If `nc` isn't installed: `sudo dnf install -y nmap-ncat`.) This should succeed (`Connection succeeded`) — the SG allows it and nothing is blocking it yet.

**Add the NACL deny:**
4. VPC Console → **Network ACLs** (left sidebar).
5. Find the NACL associated with the private subnet (check the **Subnet associations** tab on each NACL, or look it up from the **Subnets** page's Network ACL column).
6. Select it → **Inbound rules** tab → **Edit inbound rules** → **Add new rule**:
   - Rule number: `90` (must be **lower** than the existing `100 ALL Traffic ALLOW` rule — NACL rules are evaluated in ascending numeric order, and the first match wins).
   - Type: `SSH (22)`.
   - Source: `0.0.0.0/0`.
   - Allow/Deny: **Deny**.
7. Save.

**Re-test:**
8. Back on `lab-tester`'s SSM session, run the same command again:
   ```
   nc -zv <lab-private-instance-private-ip> 22
   ```
   This should now **fail / hang / time out**, even though:
   - `lab-sg-private`'s inbound rule still explicitly **allows** SSH from `10.0.0.0/16`.
   - Both instances are in the same VPC and same subnet.

**Why this happens (write this in your own words after observing it, then compare to the explanation below):**
- **Security Groups are stateful and instance-scoped.** They only see traffic that's already made it to the ENI, and once an inbound flow is allowed, the matching return traffic is automatically allowed back out — you never write an outbound rule for a reply.
- **NACLs are stateless and subnet-scoped.** They evaluate every packet independently in *both* directions, at the boundary of the subnet — before it ever reaches an instance's SG. A NACL deny drops the packet before the SG is ever consulted, so it doesn't matter what the SG says.
- Because NACL rules are numeric first-match, put your explicit deny at a **lower number** than any broader allow, or it'll never be reached.

**Cleanup for this task:** remove the deny rule (or set it back to allow) once you're done, so it doesn't interfere with later tasks that need this subnet reachable.

---

<a id="task-9"></a>
## Task 9 — Break and fix SG-to-SG connectivity

**Goal:** deliberately misconfigure two instances so they can't talk to each other, then fix it the *right* way — by referencing the peer's Security Group ID as the source, not its IP.

**Steps:**
1. You should already have `lab-private-instance` and `lab-tester` from Task 8, each with their own SG (or create two fresh SGs, `lab-sg-a` and `lab-sg-b`, one per instance, if you want a clean pair).
2. Confirm **neither SG currently allows inbound traffic from the other's SG** (e.g. remove/avoid any `10.0.0.0/16`-wide rule so you're testing SG-to-SG specifically, not the whole VPC CIDR). Each SG should only allow SSH from your IP (or nothing at all inbound besides that).
3. On instance B (`lab-private-instance`), start a simple listener so you have something to actually test against:
   ```
   python3 -m http.server 8080
   ```
4. From instance A (`lab-tester`)'s SSM session, try to reach it:
   ```
   curl -m 5 http://<instance-B-private-ip>:8080
   ```
   This should **time out** — instance B's SG has no rule allowing inbound port 8080 from instance A (or from anything).

**Fix it:**
5. EC2 Console → **Security Groups** → select instance B's SG (`lab-sg-private` / `lab-sg-b`) → **Inbound rules** → **Edit inbound rules** → **Add rule**:
   - Type: `Custom TCP`.
   - Port range: `8080`.
   - **Source:** click the source field and start typing — instead of an IP/CIDR, select instance A's **security group** from the dropdown (it'll show as `sg-xxxxxxxx (lab-sg-a)`). This is the "allow the peer SG as an ingress source" pattern: instance B now trusts *any* instance that has SG A attached, regardless of that instance's IP — which is far more durable than hardcoding an IP that can change on stop/start.
6. Save.
7. Re-run the `curl` from instance A. It should now succeed and return the directory listing HTML from Python's `http.server`.

**Confirm:** you can explain why "allow SG-A as a source" is preferred over "allow instance A's private IP" in production (IP addresses on EC2 instances can change; SG membership is a stable, semantic label — e.g. "anything tagged as part of the app tier").

---

<a id="task-10"></a>
## Task 10 — VPC Peering with non-overlapping CIDRs

**Goal:** connect two separate VPCs and get real instance-to-instance reachability across the peering connection.

**Steps:**
1. Create a second VPC: VPC Console → **Your VPCs** → **Create VPC** → this time you can use the simple **VPC only** option (or "VPC and more" again if you want subnets auto-built) — name it `lab-vpc-b`, CIDR `10.1.0.0/16` (deliberately **non-overlapping** with `lab-vpc`'s `10.0.0.0/16`).
   - If you used "VPC only," manually add at least one subnet, e.g. `10.1.1.0/24`, and launch a small test instance in it (`lab-vpcb-instance`) with a default SG allowing SSH from `10.1.0.0/16` and ICMP from `10.0.0.0/16`.
2. VPC Console → **Peering connections** → **Create peering connection**.
   - **Name:** `lab-peer-a-b`.
   - **VPC (Requester):** `lab-vpc`.
   - **Account:** My account. **Region:** This region.
   - **VPC (Accepter):** `lab-vpc-b`.
   - Click **Create peering connection**.
3. The connection shows status **Pending Acceptance**. Select it → **Actions** → **Accept request** → confirm.
4. **Update route tables on both sides** — this is the step people most often forget; peering doesn't route anything by itself:
   - Go to `lab-vpc`'s **private** (and/or public, depending which instance you're testing from) route table → **Routes** → **Edit routes** → **Add route**: Destination `10.1.0.0/16`, Target → **Peering Connection** → select `pcx-...`.
   - Go to `lab-vpc-b`'s route table → **Add route**: Destination `10.0.0.0/16`, Target → the same peering connection.
5. Update Security Groups on both sides to allow the traffic you want to test (e.g. allow ICMP/SSH from the peer VPC's CIDR).

**Confirm:** from an SSM session on an instance in `lab-vpc`, `ping` or `curl` the private IP of the instance in `lab-vpc-b` — it should succeed, proving traffic is flowing across the peering connection with no NAT/IGW involved at all (peered traffic stays on the AWS backbone).

---

<a id="task-11"></a>
## Task 11 — VPC Peering rejection with overlapping CIDRs

**Goal:** see the console actively refuse to let you create an unroutable network, and understand why overlapping CIDRs make peering (and most routing) undefined.

**Steps:**
1. Create a third VPC, `lab-vpc-c`, and **deliberately** give it the **same CIDR** as `lab-vpc`: `10.0.0.0/16`.
2. VPC Console → **Peering connections** → **Create peering connection**:
   - Requester: `lab-vpc`. Accepter: `lab-vpc-c`.
   - Click **Create peering connection**.
3. Read the error the console returns — it will refuse to create the connection because the CIDR blocks overlap, since a route table can't have two different `10.0.0.0/16` targets that mean different things.
4. **Fix it:** delete `lab-vpc-c` (VPC Console → Your VPCs → select it → Delete VPC — you'll need to first delete any subnets/instances/IGW inside it if you added any), then recreate it with a **non-overlapping** CIDR, e.g. `10.2.0.0/16`.
5. Retry the peering connection creation between `lab-vpc` and the new `lab-vpc-c` — it should now succeed and move to **Pending Acceptance** normally.

**Confirm:** you can state in one sentence why overlapping CIDRs break peering (a route table entry can only point one CIDR block at one target — if both VPCs claim the same address range, the router has no way to know which VPC a given `10.0.x.x` packet is actually destined for).

---

<a id="task-12"></a>
## Task 12 — 3-tier VPC + Flow Logs + Logs Insights REJECT query

**Goal:** build a public/app/data three-tier network (the wizard doesn't do 3 tiers natively, so this one's manual), turn on Flow Logs, and hunt down every rejected packet for one specific instance using CloudWatch Logs Insights.

**Build the VPC (manual, since "VPC and more" only offers 2 tiers):**
1. **Your VPCs** → **Create VPC** → **VPC only** → Name `lab-vpc-3tier`, CIDR `10.3.0.0/16` → Create.
2. **Subnets** → **Create subnet**, repeat 6 times (2 AZs × 3 tiers), e.g.:
   - `public-1a` `10.3.0.0/24` (AZ a), `public-1b` `10.3.1.0/24` (AZ b)
   - `app-1a` `10.3.2.0/24` (AZ a), `app-1b` `10.3.3.0/24` (AZ b)
   - `data-1a` `10.3.4.0/24` (AZ a), `data-1b` `10.3.5.0/24` (AZ b)
3. **Internet Gateway**: create one, attach it to `lab-vpc-3tier`.
4. **NAT Gateway**: allocate an EIP, create a NAT Gateway in `public-1a`.
5. **Route tables** — create three:
   - `rtb-public`: `0.0.0.0/0` → `igw-...`; associate with both public subnets.
   - `rtb-app`: `0.0.0.0/0` → `nat-...`; associate with both app subnets (this tier needs outbound internet for patches/package installs, but no direct inbound).
   - `rtb-data`: **no internet route at all** — only the default local `10.3.0.0/16` route; associate with both data subnets (data tier should be fully internet-isolated, reachable only from inside the VPC).
6. Launch one small instance per tier if you want live traffic to inspect (e.g. `app-instance` with the SSM role, in `app-1a`), so Flow Logs have something interesting to capture.

**Enable Flow Logs:**
7. **Your VPCs** → select `lab-vpc-3tier` → **Flow logs** tab → **Create flow log**.
   - **Filter:** **All** (so you capture both ACCEPT and REJECT — you need ACCEPT too for context, even though you're specifically querying REJECT next).
   - **Destination:** **Send to CloudWatch Logs**.
   - **Destination log group:** create new, e.g. `/vpc/lab-vpc-3tier/flowlogs`.
   - **IAM role:** create/select a role with permission to publish to CloudWatch Logs (the console offers to create this for you).
   - Click **Create flow log**.
8. Generate some traffic to log: from your `app-instance`, run a few `curl` commands to different destinations (some that should succeed, some that should be blocked by a SG you tighten temporarily) so you get a mix of ACCEPT/REJECT entries. Wait 5–10 minutes for logs to start flowing (Flow Logs are not instant).

**Query with Logs Insights:**
9. CloudWatch Console → **Logs** → **Logs Insights**.
10. **Select log group(s):** `/vpc/lab-vpc-3tier/flowlogs`.
11. Enter this query (adjust the field names if your log format differs from the default):
    ```
    fields @timestamp, srcAddr, dstAddr, dstPort, protocol, action
    | filter action = "REJECT"
    | filter srcAddr = "<your-app-instance-private-ip>" or dstAddr = "<your-app-instance-private-ip>"
    | sort @timestamp desc
    | limit 50
    ```
12. Click **Run query**.

**Confirm:** the results table shows every rejected flow involving that instance, with timestamp, source/destination, port, and protocol — enough detail to tell you *exactly* which rule (SG or NACL) is worth checking next for that traffic pattern.

---

<a id="task-13"></a>
## Task 13 — Transit Gateway across 3 VPCs

**Goal:** connect three VPCs through a single hub and prove transitive routing (A → C with **no** direct peering between them), then compare the route-table burden to full-mesh peering.

**Steps:**
1. You should already have three non-overlapping VPCs from earlier tasks (e.g. `lab-vpc` `10.0.0.0/16`, `lab-vpc-b` `10.1.0.0/16`, `lab-vpc-c` `10.2.0.0/16`). If not, create a third with a fresh CIDR now.
2. VPC Console → **Transit Gateways** → **Create Transit Gateway**.
   - Name: `lab-tgw`.
   - Leave default ASN, DNS support enabled, default route table association/propagation **enabled** (simplest for this lab — one shared route table auto-populated by all attachments).
   - Click **Create Transit Gateway**. Wait for state **Available** (a minute or two).
3. **Transit Gateway Attachments** → **Create transit gateway attachment**, once per VPC (3 times total):
   - Transit Gateway: `lab-tgw`.
   - Attachment type: **VPC**.
   - VPC: select the VPC.
   - Subnets: pick one subnet per AZ you want attached (typically one per AZ is enough for the lab).
   - Create.
4. Because you left default route table association/propagation on, the TGW's route table should now automatically have routes to all three VPC CIDRs — check **Transit Gateway Route Tables** → the default table → **Routes** to confirm all three CIDRs are listed with the correct attachment as target.
5. **Update each VPC's subnet route table** to send traffic destined for the *other two* CIDRs to the TGW attachment:
   - In `lab-vpc`'s route table: add routes for `10.1.0.0/16` → `tgw-...` and `10.2.0.0/16` → `tgw-...`.
   - In `lab-vpc-b`'s route table: add routes for `10.0.0.0/16` → `tgw-...` and `10.2.0.0/16` → `tgw-...`.
   - In `lab-vpc-c`'s route table: add routes for `10.0.0.0/16` → `tgw-...` and `10.1.0.0/16` → `tgw-...`.
6. Update Security Groups on your test instances in each VPC to allow ICMP/SSH from the other VPCs' CIDRs as needed.

**Prove transitivity:**
7. From an SSM session on an instance in `lab-vpc` (call it "A"), `ping`/`curl` the private IP of an instance in `lab-vpc-c` ("C").
8. Confirm it succeeds, **and** confirm there is no VPC Peering connection at all between `lab-vpc` and `lab-vpc-c` — the only path is A → TGW → C.

**Compare to 3-way full mesh peering:**
9. For 3 VPCs fully meshed with peering, you'd need: **3 peering connections** (A-B, B-C, A-C), and **route table edits in all 3 VPCs, each needing an entry per peer** (2 entries per VPC = 6 entries total). For *n* VPCs, full-mesh peering needs `n(n-1)/2` peering connections and each VPC needs `n-1` route entries — this grows quadratically.
10. With Transit Gateway: **3 attachments** (one per VPC) and (with default route propagation) the route tables largely populate themselves. Adding a 4th VPC later means **1 new attachment**, not 3 new peering connections — TGW scales linearly (`n` attachments) instead of quadratically.

**Confirm:** you can sketch (on paper or in a text file) the peering-connection count vs. TGW-attachment count for `n = 3, 5, 10` VPCs and see the gap widen.

---

<a id="task-14"></a>
## Task 14 — Replace NAT Gateway with VPC Endpoints for S3 + Secrets Manager

**Goal:** remove the NAT Gateway entirely, replace it with a Gateway Endpoint (S3) and an Interface Endpoint (Secrets Manager), and confirm the private instance keeps access to *those two services specifically* while losing general internet access.

**Steps:**
1. Confirm the S3 **Gateway** endpoint from Task 1 exists and is associated with the private route table: VPC Console → **Endpoints** → select the S3 endpoint → **Route tables** tab → confirm the private route table is listed.
2. Create a Secrets Manager **Interface** endpoint:
   - VPC Console → **Endpoints** → **Create endpoint**.
   - **Service category:** AWS services.
   - Search for `secretsmanager`, select `com.amazonaws.<region>.secretsmanager` (it'll show as type **Interface**).
   - **VPC:** `lab-vpc`.
   - **Subnets:** select the private subnets (one per AZ) — the endpoint gets an ENI with a private IP in each selected subnet.
   - **Security group:** create/select one that allows inbound **HTTPS (443)** from the private subnet CIDR(s), since the interface endpoint's ENI needs an SG like any other ENI.
   - **Policy:** leave **Full access** for the lab.
   - **Additional settings:** keep **Enable DNS name** checked — this is what makes the standard `secretsmanager.<region>.amazonaws.com` hostname *automatically* resolve to the endpoint's private IP instead of the public service endpoint, so you don't have to change any application code.
   - Click **Create endpoint**. Wait for state **Available**.
3. Create at least one test secret to query: Secrets Manager Console → **Store a new secret** → any simple key/value → name it `lab/test-secret` → finish the wizard.
4. **Remove the NAT Gateway:**
   - VPC Console → **NAT Gateways** → select `lab-vpc`'s NAT Gateway → **Actions** → **Delete NAT gateway** → confirm (type `delete` if prompted). This takes a few minutes.
   - Go to the **private** route table → **Routes** → find the now-dangling `0.0.0.0/0 → nat-...` route (it'll likely show as blackholed once the NAT GW is gone) → **Edit routes** → remove that row → **Save**.
   - (Optional but good practice) release the NAT Gateway's Elastic IP now that nothing uses it.

**Confirm behavior from the private instance (SSM session):**
5. S3 still works (via the Gateway endpoint):
   ```
   aws s3 ls
   ```
   should succeed (region-appropriate CLI config assumed; if it errors on region, add `--region <your-region>`).
6. Secrets Manager still works (via the Interface endpoint):
   ```
   aws secretsmanager get-secret-value --secret-id lab/test-secret --region <your-region>
   ```
   should succeed and return the secret you stored.
7. General internet is now gone:
   ```
   curl -m 5 -Is https://www.amazon.com
   ```
   should **time out / fail** — there's no NAT Gateway and no IGW route left for this subnet, so any destination other than S3 or Secrets Manager (reached via their private endpoints) is unreachable.

**Quantify the monthly savings:**
8. Open the [AWS VPC Pricing page](https://aws.amazon.com/vpc/pricing/) (or the Pricing Calculator) for your region and note the **current** hourly NAT Gateway charge and per-GB data processing charge, plus the **current** hourly Interface Endpoint charge (per AZ) and its per-GB data processing charge. These figures do change over time, so pull live numbers rather than relying on memory.
9. Build a quick comparison, e.g.:
   - **Before:** 1 NAT Gateway × 730 hrs/month × (hourly rate) + (GB processed × NAT data rate).
   - **After:** 1 Interface Endpoint × (number of AZs) × 730 hrs/month × (hourly rate) + (GB processed × interface data rate). The S3 Gateway Endpoint itself has **no hourly charge and no data processing charge** — that side is effectively free.
   - As a rule of thumb going in: for a workload that only needs S3 + a small number of other AWS services (not general internet egress), this swap is very often cheaper than a NAT Gateway, especially at low-to-moderate data volumes — but always confirm against current published rates for your actual traffic pattern, since a high-egress-volume workload with many services could tip the other way (interface endpoints are billed per-AZ, so a 3-service, 3-AZ setup means 9 hourly charges).

**Confirm:** you have a private instance that can `aws s3 ls` and read a secret, but cannot `curl` an arbitrary internet host — and you have real numbers (pulled from the pricing page on the day you did this) for what you saved or spent.

---

<a id="task-15"></a>
## Task 15 — Blind fault injection + Networking Decision Tree

**Goal:** simulate a real on-call scenario — something in the path is broken, you don't know what, and you have 10 minutes to find it using a structured decision tree instead of randomly clicking around.

**Set up the blind test:**
1. Pick a partner if you have one (they inject the fault, you diagnose) — or do it solo with a personal "commitment device": write all 4 fault options on paper slips, or list them in a text file, close your eyes and pick one (or use `python3 -c "import random; print(random.choice(['sg','nacl','route','igw']))"` from a *separate* terminal you don't watch), then go make **exactly one** of the changes below without writing down which one you picked.
2. The four possible single-hop faults, corresponding to the four things that must all be correct for a connection to work:
   - **SG fault:** remove or narrow the inbound rule on the target instance's Security Group so the specific traffic you'll test is no longer allowed.
   - **NACL fault:** add a low-numbered `DENY` rule (like in Task 8) on the subnet's NACL for the specific traffic/direction you'll test.
   - **Route table fault:** delete or point-somewhere-wrong the relevant route (e.g. remove the `0.0.0.0/0 → igw-...` route from a public route table, or the `0.0.0.0/0 → nat-...` route from a private one).
   - **IGW attachment fault:** detach the Internet Gateway from the VPC entirely (VPC Console → Internet Gateways → select it → **Actions** → **Detach from VPC**).
3. Pick a concrete test target ahead of time, e.g. "public instance must be reachable on port 80 from the internet" or "private instance must reach the internet via NAT" — know your test *before* the fault is injected so you're not guessing what "broken" even means.

**Diagnose using the Networking Decision Tree below.** Work top-down; each question is designed to eliminate roughly half the remaining possibilities.

### Networking Decision Tree

```
START: "Traffic that should work isn't working."

1. Is the destination instance's Security Group correctly allowing
   the traffic (right port/protocol, right source)?
   ├─ NO  → Fix the SG rule. STOP.
   └─ YES → go to 2

2. Is there a NACL DENY rule (on either the source or destination
   subnet, for either direction) with a LOWER rule number than the
   ALLOW that would otherwise apply?
   Remember: NACLs are stateless — check BOTH the inbound rule for
   the request AND the outbound rule for the reply, on BOTH subnets
   involved.
   ├─ YES → Remove/renumber the deny rule. STOP.
   └─ NO  → go to 3

3. Does the SUBNET's route table have a route to the destination?
   - For internet-bound traffic from a public subnet: is there a
     0.0.0.0/0 → igw-... route?
   - For internet-bound traffic from a private subnet: is there a
     0.0.0.0/0 → nat-... route, and does it point to a NAT Gateway
     that actually exists and is in "Available" state?
   - For VPC-to-VPC traffic: is there a route for the peer/TGW CIDR
     pointing at a pcx-... or tgw-... target?
   ├─ NO / WRONG TARGET → Fix the route. STOP.
   └─ YES, LOOKS CORRECT → go to 4

4. Is the Internet Gateway actually ATTACHED to this VPC (not just
   created, but attached — check VPC Console → Internet Gateways →
   State column)?
   ├─ NOT ATTACHED → Attach it (Actions → Attach to VPC). STOP.
   └─ ATTACHED     → go to 5

5. (If all 4 above check out) Is the target instance actually
   running, and does it have a public IP if the test requires one
   (EC2 → instance → Details tab)? Also sanity-check you're testing
   from where you think you are (right subnet/AZ, right instance).
   → If this also checks out, re-read questions 1–4 more carefully —
     the fault is almost certainly one of those four, and it's easy
     to eyeball a rule as "fine" when a single character (port
     number, CIDR, rule ordering) is the actual problem.
```

**Why this order:** SG is checked first because it's the most common real-world misconfiguration and the fastest to read. NACL is second because it's the "gotcha" most engineers forget (Task 8 taught you why — it silently overrides SG **and** is stateless, so the reply direction can be broken separately from the request direction). Route table is third because a missing/wrong route affects a whole subnet, not just one instance, so it's a good "does everyone in this subnet have the problem" filter. IGW attachment is checked last because it's rare to get *un*attached but catastrophic (breaks the entire public side of the VPC) when it happens, and it's a single toggle.

**After you diagnose:** reveal which fault was actually injected and compare it to your diagnosis path. Time yourself — the 10-minute target is meant to force you to work the tree in order rather than randomly poking at things.

---

<a id="cleanup"></a>
## Cleanup Checklist

Work through this whenever you're done for the day — several of these resources bill hourly whether or not you're using them.

- [ ] **EC2 instances** — terminate all lab instances (`lab-public-instance`, `lab-private-instance`, `lab-tester`, any 3-tier/TGW test instances).
- [ ] **NAT Gateway(s)** — delete any remaining NAT Gateways (Task 14 should have already removed the main one).
- [ ] **Elastic IPs** — release any unattached EIPs (check the whole **Elastic IPs** list, not just the ones you remember allocating).
- [ ] **VPC Endpoints** — delete the Secrets Manager interface endpoint (S3 gateway endpoint is free, but delete it too if you're tearing down the whole VPC).
- [ ] **Transit Gateway** — delete all TGW attachments first, then the Transit Gateway itself (it won't delete with attachments still present).
- [ ] **Peering connections** — delete the peering connections from Tasks 10/11 if no longer needed.
- [ ] **CloudWatch Log group** — delete `/vpc/lab-vpc-3tier/flowlogs` if you don't need the Flow Logs history, and disable the Flow Log itself on the VPC.
- [ ] **Secrets Manager secret** — schedule deletion of `lab/test-secret` (Secrets Manager enforces a recovery window before permanent deletion, even if you ask for immediate deletion).
- [ ] **IAM role** — delete `lab-ssm-instance-role` if you're fully done with the lab.
- [ ] **Security Groups, NACL custom rules, subnets, route tables, IGWs, VPCs** — once instances/NAT/endpoints/TGW attachments are gone, these have no ongoing cost, but delete the VPCs last (Your VPCs → select → **Delete**, which cascades to its own subnets/route tables/IGW) to fully reclaim the CIDR ranges for reuse.

---

### A note on console UI drift

AWS updates console screens and menu wording periodically. If a button or tab name in this guide doesn't match exactly what you see, look for the nearest equivalent — the underlying concepts (route tables, SGs, NACLs, endpoints, TGW attachments) and their relationships to each other don't change even when the UI does.
