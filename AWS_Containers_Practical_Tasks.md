# AWS Containers — Practical Production Tasks
### 15 Hands-On Labs for Deep Mastery of ECR · Fargate · ECS · EKS
*Companion exercise set for `AWS_Containers_Master_Guide.md` — no solutions included; you build, break, and fix it yourself.*

---

## How to Use This Set

- **Access level:** These labs assume you are working with **Administrator (or near-Administrator) IAM permissions** in a sandbox/non-production AWS account — you'll be creating IAM roles, VPC resources, ECR repos, ECS/EKS clusters, KMS keys, and CloudWatch resources. In a real company you'd normally have scoped-down permissions; as an extra interview-prep exercise, try writing the **least-privilege IAM policy** each task actually needs after you finish it.
- **Order:** Roughly sequential — later tasks assume you're comfortable with earlier ones. Tasks 4, 8, and 15 are deliberately "break it, then fix it" exercises; do them last in their section.
- **Discipline:** Don't copy commands straight from the guide. Read the relevant Part, then build the task from your own understanding. If you get stuck, re-read the guide section — don't search for the exact answer immediately.
- **Always tear down** every resource (Part 12 of the guide) after each task to avoid surprise billing.
- **Interview angle:** Each task ends with a short "Be ready to explain" list — these are the kind of follow-up questions an interviewer will ask after you describe the project on your resume. Don't look up the answers — form your own explanation from what you just built.

---

## Section A — Docker & Amazon ECR

### Task 1 — Build a Hardened, Multi-Arch Production Image and Push It Through a Governed ECR Repo
Build a small web app image using a **multi-stage Dockerfile** (build stage + minimal/distroless runtime stage), running as a **non-root user**, with a `.dockerignore` in place. Build it for **both amd64 and arm64** using buildx and push it to an ECR repo you configure with: tag **immutability**, **enhanced (Inspector) scanning on push**, and a **lifecycle policy** that keeps only the last 5 tagged images and expires untagged images after 1 day. Then intentionally introduce a known-vulnerable base image or dependency, observe the scan findings, and remediate down to zero CRITICAL/HIGH findings.

**Be ready to explain:** why multi-stage builds reduce attack surface; the difference between basic and enhanced ECR scanning; why immutable tags matter for production rollback safety; how `docker history` can leak secrets and how BuildKit secret mounts avoid that.

### Task 2 — Design an Enterprise ECR Governance Model
Set up a scenario with **two repos** — `dev-app` (mutable, no scan-on-push) and `prod-app` (immutable, scan-on-push, KMS-encrypted). Configure a **pull-through cache** for Docker Hub so your team never pulls directly from the internet, and a **cross-region replication rule** for `prod-` prefixed repos. Write and attach an ECR **repository policy** that allows a *second AWS account* (simulate with an IAM role in the same account if you don't have a second one) read-only pull access, and separately restrict push access to a CI role only. Add the four required **VPC endpoints** (ECR API, ECR DKR, S3, Logs) and prove that image pulls still succeed with the NAT gateway detached/removed.

**Be ready to explain:** how ECR repository policies differ from IAM policies; why private subnets need VPC endpoints for image pulls; what problem pull-through cache solves; what "OCI-compliant registry" means and what else besides images ECR can store.

---

## Section B — AWS Fargate & Amazon ECS

### Task 3 — Deploy a Multi-Container ECS Fargate Service with Zero-Downtime Blue/Green Releases
Design a task definition with **two containers** (app + a sidecar, e.g., a log router or an Envoy-style proxy), correct CPU/memory sizing, **secrets pulled from Secrets Manager** (not hardcoded env vars), and an execution role scoped to only what's needed. Put the service behind an **ALB with HTTPS** and proper health checks, run **2+ tasks across AZs**, then wire up **CodeDeploy blue/green deployments** so a new task definition revision shifts traffic gradually with automatic rollback on failed health checks. Prove the rollback works by deploying a broken revision on purpose.

**Be ready to explain:** the difference between a task definition revision and a service; how ECS decides a deployment is "steady state"; how blue/green differs from ECS's default rolling deployment; what happens to in-flight requests during a task replacement.

### Task 4 — Production Incident Simulation: ECS Won't Go Healthy
Deliberately break your Task 3 service in **four separate, undone-one-at-a-time** ways and diagnose each purely from symptoms (don't peek at what you changed until you've formed a hypothesis):
1. Remove the execution role's ECR pull permission.
2. Point the task to a private subnet with no NAT/VPC endpoint.
3. Set the ALB health check path to a route your app doesn't serve.
4. Set memory too low so the container gets OOM-killed under load.

For each, use only `describe-services` events, CloudWatch Logs, and target-group health status to identify root cause before fixing it.

**Be ready to explain:** the ECS task lifecycle states (PROVISIONING → PENDING → RUNNING → STOPPED) and what each stuck state usually means; how you'd differentiate an infra issue from an application bug under pressure; how you'd write a runbook entry from this incident.

### Task 5 — Cost-Resilient Capacity Strategy: Fargate Spot + Base On-Demand
Convert your cluster to a capacity provider strategy mixing **FARGATE** and **FARGATE_SPOT** (e.g., 1 base on-demand task, remaining tasks weighted mostly Spot). Add a graceful **SIGTERM handler** in your app so it drains in-flight requests within the 120-second Spot reclaim window. Load-test the service while manually interrupting spot capacity (or simulate by stopping tasks) and confirm the ALB never serves a 5xx during the interruption.

**Be ready to explain:** how Spot interruption differs between EC2 Spot and Fargate Spot; why `base` matters in a capacity provider strategy; what workloads should *never* run on Fargate Spot.

### Task 6 — Scheduled & Event-Driven ECS Workloads
Build a batch job as an ECS task (not a long-running service) that runs on an **EventBridge schedule**, reads a message from an SQS-style trigger (or S3 event), processes it, and exits. Configure it so failures retry with backoff and successes are logged with structured JSON to CloudWatch Logs Insights-queryable format.

**Be ready to explain:** when you'd choose an ECS scheduled task over a Lambda function; how you'd monitor and alert on a batch job that silently stops running.

---

## Section C — Amazon EKS

### Task 7 — Production-Ready EKS Cluster with Mixed Compute
Create an EKS cluster with **all three** compute models coexisting: a managed node group (EC2), an EKS **Fargate profile** for a specific namespace, and **Karpenter** for dynamic node provisioning. Deploy the same app to both the managed-node namespace and the Fargate namespace, and prove Karpenter scales nodes up under load and back down when idle.

**Be ready to explain:** why EKS Fargate can't run DaemonSets or privileged pods; how Karpenter differs from the Cluster Autoscaler; when you'd deliberately mix all three compute types in one cluster instead of picking one.

### Task 8 — Debug a Broken Kubernetes Deployment (No Hints)
Deploy an application with a Deployment, three probes (liveness/readiness/startup), a PodDisruptionBudget, and an HPA. Then, working from a partner's (or your own past self's) intentionally sabotaged manifest — or sabotage it yourself and forget the details for a day — diagnose and fix at least three of these without looking at the manifest diff first:
- Pod stuck `Pending` (resource requests too high for node capacity)
- `ImagePullBackOff` (wrong tag or node IAM role missing ECR read policy)
- `CrashLoopBackOff` (bad liveness probe killing a slow-starting app)
- Ingress with no `ADDRESS` (AWS Load Balancer Controller / IRSA misconfigured)

Use only `kubectl describe`, `kubectl logs --previous`, and `kubectl get events` to build your hypothesis before changing anything.

**Be ready to explain:** the difference between liveness and readiness probes and a real scenario where confusing them caused an outage; how you'd design a startup probe for a slow Java app; what `kubectl describe pod` tells you that `kubectl get pod` doesn't.

### Task 9 — IRSA / Pod Identity for Least-Privilege AWS Access from Pods
Give a pod access to a specific S3 bucket and a specific Secrets Manager secret using **IRSA (or EKS Pod Identity)** — no static AWS keys anywhere. Then intentionally misconfigure the trust policy (wrong service account or namespace) and diagnose the resulting `AccessDenied` purely from the pod's perspective, without checking the IAM console first.

**Be ready to explain:** how IRSA actually works under the hood (OIDC provider, trust policy, token projection); why this is more secure than mounting an access key as a Kubernetes Secret; the difference between IRSA and the newer EKS Pod Identity feature.

### Task 10 — Stateful Workloads on EKS: Postgres on EBS + Shared Storage on EFS
Deploy a StatefulSet running a database backed by an EBS-backed PersistentVolumeClaim (via the EBS CSI driver), and a separate Deployment where multiple pod replicas share a common EFS volume for shared file access. Kill the node the database pod is running on (or cordon+drain it) and confirm the StatefulSet correctly reschedules and reattaches its volume without data loss.

**Be ready to explain:** why EBS can't be shared across AZs/pods the way EFS can; what "ReadWriteOnce" vs "ReadWriteMany" access modes mean in practice; how you'd approach running a database in Kubernetes vs. just using RDS.

---

## Section D — Security, Observability, CI/CD, and Cost

### Task 11 — Supply-Chain Security Gate: Scan → Sign → Enforce
Build a pipeline stage (can be a local script or real CI) that: scans an image with Trivy and **fails the build** on any CRITICAL CVE, generates an SBOM with Syft, signs the image with Cosign + KMS, and pushes only signed, scanned images to `prod-app`. On the EKS cluster, install **Kyverno** (or an equivalent policy engine) with a policy that **rejects unsigned images and images running as root** at admission time. Prove it by trying to deploy an unsigned image and watching it get blocked.

**Be ready to explain:** what a "shift-left" security pipeline means in practice; how admission control differs from CI-time scanning as a defense layer; what a supply-chain attack via a compromised base image would actually look like.

### Task 12 — Full-Stack Observability Across ECS and EKS
Set up **Container Insights** for your ECS cluster and **Prometheus + Grafana** (or Container Insights for EKS) for your EKS cluster, feeding the same dashboards. Add **X-Ray** (or OpenTelemetry) distributed tracing so a request that flows through an ALB → ECS service → EKS service → database is traceable end-to-end with one trace ID. Deliberately introduce latency in one downstream service and confirm you can find it from the trace alone, without checking application logs first.

**Be ready to explain:** the difference between metrics, logs, and traces and when each is the right tool; how you'd set a meaningful CloudWatch alarm (not just "CPU > 80%") for a production service; what a "the service *looks* healthy but customers are complaining" incident usually turns out to be.

### Task 13 — CI/CD with OIDC Federation and GitOps (No Long-Lived Keys)
Build a GitHub Actions (or equivalent) pipeline that authenticates to AWS via **OIDC federation** (no stored access keys), builds and scans an image, pushes to ECR, and deploys to ECS via CodeDeploy blue/green **on merge to main**. Separately, set up **ArgoCD** on your EKS cluster so a change pushed to a Git repo automatically syncs to the cluster with no `kubectl apply` from a human. Force a drift (manually `kubectl edit` a live resource) and observe ArgoCD detect and correct it.

**Be ready to explain:** why OIDC federation is preferred over IAM access keys in CI/CD; the core GitOps principle of "Git as source of truth"; how you'd roll back a bad GitOps deployment in under a minute.

### Task 14 — Cost Optimization Audit
Take an existing (over-provisioned) ECS/EKS workload and produce a real optimization report: identify tasks/pods with CPU/memory requests far above actual usage (via Container Insights/Compute Optimizer data), calculate the savings from moving from `x86` to **Graviton (ARM64)**, calculate savings from shifting a portion of traffic to **Fargate Spot**, and right-size accordingly. Apply the changes and confirm the app still meets its SLA under the same load test as before.

**Be ready to explain:** how you'd justify a cost-optimization change to a team worried it'll hurt reliability; the trade-off between over-provisioning "for safety" and right-sizing; how Savings Plans/Compute Savings differ from Spot for containers.

---

## Section E — Capstone

### Task 15 — War-Room Simulation: Multi-Layer Production Incident
This is the big one — combine everything above. Set up a realistic two-service production environment (one on ECS Fargate, one on EKS) wired together with the ALB, IAM/IRSA, Secrets Manager, and observability stack from earlier tasks. Then have someone else (or a scripted randomizer / future-you) inject **one hidden fault** from a pool that spans networking, IAM, application, and Kubernetes/ECS-config layers — without telling you which layer. You have a fixed time budget (e.g., 30 minutes) to:
1. Detect the incident using only dashboards/alarms (not by reading config).
2. Localize the failing layer using logs/traces/events.
3. Fix it.
4. Write a one-page incident postmortem: timeline, root cause, contributing factors, and one concrete prevention action.

Repeat with a different injected fault at least twice.

**Be ready to explain:** walk an interviewer through this incident end-to-end as if it were a real production outage — what you saw first, what you ruled out, what you ruled in, and what you'd change about your monitoring so it's caught faster next time. This is the exact story format ("Tell me about a time you debugged a production issue") interviewers ask for.

---

## Self-Assessment Checklist

You're ready for both real production ownership and container-focused interviews when you can, **without notes**:

- [ ] Explain the full lifecycle of a container image from `docker build` to a running ECS task or EKS pod
- [ ] Diagnose any of the ECS/EKS failure states in the guide's troubleshooting cheat sheet from symptoms alone
- [ ] Justify Fargate vs. EC2, and ECS vs. EKS, for a given workload with real trade-offs (not just "it depends")
- [ ] Design least-privilege IAM for a task/pod using execution roles, task roles, and IRSA/Pod Identity correctly
- [ ] Describe how you'd secure a supply chain from source code to running container
- [ ] Whiteboard a production reference architecture (ALB → ECS/EKS → data layer) including HA, secrets, logging, and CI/CD
- [ ] Tell a clear, structured story about a real incident you debugged during these labs

---
*Pair this file with `AWS_Containers_Master_Guide.md` — read the relevant Part before attempting each task, but build the solution yourself.*
