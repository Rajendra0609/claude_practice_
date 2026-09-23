# AWS Containers — Hands-On Practice Tasks
### ECR · ECS · Fargate · EKS | Basic → Medium | Production Scenarios & Interview Prep

> **How to use this file:** Each task is a scenario, not a checklist of commands. Read the scenario, decide your own approach using the master guide as reference, implement it in your AWS account, and then break something on purpose and fix it — that's where the real learning (and interview stories) come from. No solutions are provided here by design; you build, you troubleshoot, you explain.
>
> For every task, after you complete the lab, write **3–4 sentences in your own words** explaining what you did and why. That explanation is what you'll actually say in an interview — practicing it now is half the prep.

---

## Task 1 — Build a Production-Grade Image From Scratch (Basic)
**Focus area:** Docker fundamentals

**Scenario:** Your team lead asks you to containerize a simple web app before it goes anywhere near AWS. The image will be reviewed for a security sign-off before it's allowed into any registry.

**Your objective:**
- Write a Dockerfile for a small app of your choice that runs as a non-root user, uses a minimal base image, and has a proper `.dockerignore`.
- Convert it into a multi-stage build so build tools never ship to production.
- Compare the image size before and after multi-stage, and be ready to explain the difference in MB.

**Break-fix twist:** Intentionally bake a fake secret (e.g., `ENV API_KEY=test123`) into an early layer, then try to "remove" it in a later layer with `RUN rm`. Inspect the image history and see what you find.

**Interview angle:** Be ready to explain why removing a file in a later layer doesn't remove it from the image, what `docker history` reveals, and how BuildKit secret mounts solve this properly.

---

## Task 2 — Set Up a Secure, Governed ECR Repository (Basic)
**Focus area:** ECR

**Scenario:** Your organization wants two repositories: one for developers to experiment in (`workshop-app`) and one that will hold production images (`prod-app`). Production image tags must never be overwritten once pushed.

**Your objective:**
- Create both repositories with the correct tag mutability settings for their purpose.
- Push at least two versions of an image to each and demonstrate what happens when you try to overwrite an existing tag on the production repo.
- Apply a lifecycle policy to `workshop-app` that automatically removes old/untagged images.
- Enable image scanning and review the vulnerability findings.

**Break-fix twist:** Try pushing to `prod-app` with an existing tag and capture the exact error. Then figure out the correct way to update a production image.

**Interview angle:** Be ready to explain the difference between MUTABLE and IMMUTABLE repos, why production environments almost always use IMMUTABLE, and how teams version images without mutable "latest" tags.

---

## Task 3 — Deploy Your First ECS Fargate Service Behind a Load Balancer (Basic → Medium)
**Focus area:** ECS + Fargate + ALB

**Scenario:** You need to expose your containerized app to the internet with high availability — at least 2 running tasks at all times, spread across availability zones, fronted by an ALB with a health check.

**Your objective:**
- Write a task definition with correct CPU/memory sizing, execution role, and task role separated correctly.
- Create an ECS service with `desired-count=2`, `awsvpc` networking, and register it with an ALB target group.
- Confirm the service reaches steady state and the health check passes.

**Break-fix twist:** Deliberately misconfigure the health check path so it never returns a 200, deploy, and watch what happens to your service and target group. Then diagnose and fix it using only the ECS console/CLI events and CloudWatch logs — no peeking at the answer key.

**Interview angle:** Be ready to explain the difference between the ECS **execution role** and **task role**, what a target group's health check controls, and how you'd debug a service that never reaches `STEADY` state.

---

## Task 4 — Handle a Task That Won't Start (Medium)
**Focus area:** ECS troubleshooting

**Scenario:** This is a pure troubleshooting drill. Deliberately create at least three different broken deployments and diagnose each using only the tools a real on-call engineer would use.

**Your objective — create and independently diagnose each of these failure modes:**
1. A task stuck in `PROVISIONING`/`PENDING` because of a networking misconfiguration (e.g., private subnet with no NAT and no public IP).
2. A `CannotPullContainerError` caused by a missing ECR permission on the execution role.
3. A task that starts and then immediately stops because the application crashes on launch.

**Your objective:** For each scenario, use `describe-services`, `describe-tasks`, and CloudWatch logs to identify root cause before fixing it — do not fix it blindly by trial and error.

**Interview angle:** This task IS the interview question. "Tell me about a time a container wouldn't start in production — walk me through how you debugged it" is one of the most common ECS interview questions. You'll now have three real stories.

---

## Task 5 — Secrets and Config the Right Way (Basic → Medium)
**Focus area:** ECS + Secrets Manager / SSM

**Scenario:** A security review flagged that your team has been passing a database password as a plain environment variable in the task definition. You need to fix this without any downtime.

**Your objective:**
- Store a fake "database password" in AWS Secrets Manager.
- Update your task definition to inject it via `secrets` (not `environment`), referencing the Secrets Manager ARN.
- Confirm inside the running container that the secret is available at runtime but was never visible in the task definition JSON in plaintext.
- Force a new deployment and confirm zero downtime with your existing ALB setup from Task 3.

**Interview angle:** Be ready to explain the difference between `environment` and `secrets` in a task definition, why baking secrets into env vars or Docker images is a common real-world mistake, and what IAM permission the execution role needs to retrieve the secret.

---

## Task 6 — Scale a Service Under Load and Control the Blast Radius (Medium)
**Focus area:** ECS autoscaling + Fargate Spot

**Scenario:** Your app has unpredictable traffic spikes. Leadership wants scaling to be automatic but also wants to reduce compute cost, and they're nervous about relying entirely on Spot capacity in case AWS reclaims it.

**Your objective:**
- Configure Application Auto Scaling on your ECS service based on a CPU or ALB-request-count metric, with sensible min/max task counts.
- Configure a capacity provider strategy that mixes `FARGATE` and `FARGATE_SPOT`, keeping at least one guaranteed on-demand task running at all times (`base`).
- Generate synthetic load (even a simple loop of `curl` requests or a basic load tool) and observe the service scale out and back in.

**Interview angle:** Be ready to explain how Fargate Spot interruption works (the 120-second SIGTERM warning), how you'd design an app to handle it gracefully, and how a `base` + `weight` capacity provider strategy behaves under scale-out.

---

## Task 7 — Stand Up an EKS Cluster and Deploy the Same App Two Ways (Medium)
**Focus area:** EKS + Fargate profiles + managed nodes

**Scenario:** Your platform team wants to evaluate running the same workload on both EKS managed nodes and EKS Fargate profiles, to compare operational overhead and behavior.

**Your objective:**
- Create an EKS cluster with both a managed node group and a Fargate profile scoped to a specific namespace.
- Deploy the same application as a Deployment with a Service, once onto the managed nodes and once into the Fargate-profile namespace.
- Compare `kubectl describe pod` output between the two — note what's different (e.g., node info, networking).

**Break-fix twist:** Deploy a DaemonSet and see what happens when you try to schedule it into your Fargate profile namespace. Understand why it behaves the way it does before checking documentation.

**Interview angle:** Be ready to explain when you'd choose EKS Fargate profiles over managed node groups, and a concrete limitation of Fargate on EKS (e.g., no DaemonSets, no privileged pods) that would make you pick EC2 nodes instead.

---

## Task 8 — Expose an EKS App Securely with Ingress + IAM for Pods (Medium)
**Focus area:** EKS networking + IRSA/Pod Identity

**Scenario:** Your pod needs to read an object from an S3 bucket, and your app needs to be reachable over HTTPS from outside the cluster — but you've been told "no static AWS credentials in the pod, ever."

**Your objective:**
- Install the AWS Load Balancer Controller and expose your app via an Ingress backed by an ALB.
- Set up IRSA (or Pod Identity) so your pod can assume an IAM role scoped to read-only access on a single S3 bucket — no access keys anywhere in the pod spec or image.
- Prove it works: have your app (or a `kubectl exec` shell in a debug pod) successfully read from S3 using only the assumed role, and confirm access is denied for anything outside that one bucket's permissions.

**Interview angle:** Be ready to explain how IRSA actually works under the hood (OIDC provider, trust policy, service account annotation) — this is one of the most commonly asked EKS security questions, and most candidates can only describe it at a surface level.

---

## Task 9 — Diagnose a Crash-Looping Pod (Medium)
**Focus area:** EKS troubleshooting

**Scenario:** A teammate pings you: "the app pod keeps restarting in a loop and I don't know why." You need to reproduce and independently diagnose this without being told the cause.

**Your objective:**
- Intentionally misconfigure a liveness probe so it fails after the container starts successfully (e.g., wrong path or port), causing `CrashLoopBackOff`.
- Separately, in a second deployment, cause a real application-level crash (e.g., missing an expected environment variable your app requires to boot).
- For each, use `kubectl describe pod`, `kubectl logs --previous`, and `kubectl get events` to determine root cause, and document how you told the two failure modes apart.

**Interview angle:** "How do you tell the difference between a bad liveness probe and an actual app crash?" is a classic Kubernetes interview question — you'll now have hands-on proof either way.

---

## Task 10 — Enforce Security Guardrails With Policy-as-Code (Medium)
**Focus area:** Security (Kyverno / Pod Security Standards)

**Scenario:** Your cluster has no guardrails today — anyone can deploy a container running as root, using the `latest` tag, or pulling an unsigned image. You've been asked to fix that before the next audit.

**Your objective:**
- Deploy a policy engine (Kyverno, per the guide) with at least three policies: block containers running as root, block the `latest` tag, and require images to come from your ECR registry.
- Try to deploy a deliberately non-compliant pod (root user, `latest` tag) and confirm it's rejected — capture the actual rejection message.
- Fix the manifest to comply and confirm it deploys successfully.

**Interview angle:** Be ready to explain the difference between scanning an image for CVEs (detective control) and admission-control policies (preventive control), and why production clusters need both.

---

## Task 11 — Build Full Observability for a Running Service (Medium)
**Focus area:** CloudWatch Container Insights / Prometheus-Grafana / X-Ray

**Scenario:** Your service passed its load test, but during a recent incident nobody could answer "was it the app, the database, or the network?" within 15 minutes. Leadership wants proper observability before the next release.

**Your objective:**
- Enable Container Insights (or set up Prometheus + Grafana) on your cluster/service and confirm you can see CPU, memory, and network metrics per task/pod.
- Wire up centralized logging so you can search logs from all running tasks/pods in one place without SSHing anywhere.
- Add basic distributed tracing (X-Ray/OpenTelemetry) to at least one request path and confirm you can see a trace end-to-end.

**Interview angle:** Be ready to explain the difference between logs, metrics, and traces, and describe a realistic scenario where you'd need all three to diagnose a single incident.

---

## Task 12 — Automate Build → Push → Deploy With Zero Stored Credentials (Medium)
**Focus area:** CI/CD (GitHub Actions + OIDC)

**Scenario:** Your team currently pushes images manually from a laptop using long-lived AWS access keys stored in a `.env` file — a finding your security team flagged as high risk.

**Your objective:**
- Set up a GitHub Actions (or equivalent CI) pipeline that authenticates to AWS using OIDC federation — no static access keys anywhere in your repo or CI secrets.
- Pipeline should build the image, scan it (fail the build on a critical CVE), push it to ECR, and trigger a new ECS/EKS deployment automatically on a push to your main branch.
- Confirm a critical-CVE image genuinely fails the pipeline before it ever reaches ECR.

**Interview angle:** Be ready to explain how OIDC federation eliminates the need for stored AWS credentials in CI, and walk through what the IAM trust policy for the GitHub OIDC provider actually checks.

---

## Task 13 — Execute a Blue/Green Deployment and a Rollback (Medium)
**Focus area:** ECS + CodeDeploy / deployment strategy

**Scenario:** Your team is about to ship a risky release to a customer-facing service and wants the ability to shift traffic gradually and roll back instantly if something goes wrong — no "redeploy the old task definition and pray."

**Your objective:**
- Configure a blue/green deployment (CodeDeploy with ECS, or an equivalent rolling strategy with alarms) for your service from Task 3.
- Deploy a new version, verify traffic shifts as expected, and confirm the health check gate works.
- Deploy a deliberately broken version (e.g., one that fails its health check) and confirm the deployment rolls back automatically rather than taking your whole service down.

**Interview angle:** Be ready to explain the practical difference between a rolling deployment and a blue/green deployment, and how automated rollback on a CloudWatch alarm actually gets triggered.

---

## Task 14 — Right-Size and Cost-Optimize a Running Workload (Basic → Medium)
**Focus area:** Cost optimization

**Scenario:** Finance flags that your container spend has grown 3x in two months. You're asked to bring costs down without hurting reliability.

**Your objective:**
- Review CPU/memory utilization of your existing ECS or EKS workloads (Container Insights/CloudWatch metrics) and identify at least one task definition that's clearly over-provisioned.
- Right-size it and redeploy, then compare the before/after utilization and estimated cost.
- Convert one non-critical, stateless workload to run on Fargate Spot or Graviton (ARM64) and document the expected savings.

**Interview angle:** Be ready to walk through a real example of "here's a workload I found over-provisioned and how I proved it before making the change" — this is a very common senior-level cost-optimization interview question, and vague answers stand out immediately to interviewers.

---

## Task 15 — End-to-End Incident Simulation (Capstone, Medium)
**Focus area:** Everything above, combined

**Scenario:** This is your final exam. Take the ECS or EKS service you've built across the previous tasks (with ALB, secrets, autoscaling, and observability in place) and simulate a full production incident from the customer's perspective.

**Your objective:**
- Pick ONE failure to inject without telling yourself in advance which layer it's in — for example: revoke an IAM permission the running service depends on, corrupt an environment variable, or scale a dependency to zero.
- Using only your observability stack from Task 11 (not the console's "recent changes" history), detect that something is wrong, localize the failure to the correct layer (app, task/pod, network, IAM, or load balancer), and fix it.
- Write a short "postmortem" — what broke, how you found it, how long it took, and what guardrail (alarm, policy, or test) would have caught it sooner.

**Interview angle:** This entire exercise IS a "tell me about a time you resolved a production incident" answer, but genuinely yours. Practice telling it out loud in under 3 minutes — that's the real interview test.

---

## How to Turn This Into Interview Readiness

- After each task, write down the **exact AWS CLI commands and console clicks you used** — recruiters and interviewers love candidates who can describe a real workflow, not just theory.
- Keep a short "failure log": one line per bug you hit and how you fixed it. This becomes your bank of STAR-format interview stories ("Tell me about a time you debugged a production issue").
- Revisit Part 11 (Troubleshooting Cheat Sheet) and Part 9 (Cost Optimization) in the master guide *after* finishing these tasks, not before — you'll understand them far better having broken things yourself first.
- Once comfortable, try explaining each task out loud to someone non-technical in 60 seconds — if you can't simplify it, you don't fully understand it yet.
