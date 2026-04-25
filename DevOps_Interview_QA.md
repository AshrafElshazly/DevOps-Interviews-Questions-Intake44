## Table of contents

1. [EKS & Kubernetes (Q1–Q8)](#eks--kubernetes)
2. [Terraform & IaC (Q9–Q15)](#terraform--iac)
3. [CI/CD & GitOps (Q16–Q22)](#cicd--gitops)
4. [AWS networking & security (Q23–Q31)](#aws-networking--security)
5. [Data, resilience & compliance (Q32–Q36)](#data-resilience--compliance)
6. [Observability, cost & incidents (Q37–Q42)](#observability-cost--incidents)
7. [Senior & behavioral (Q43–Q48)](#senior--behavioral)
8. [general job-description extensions (Q49–Q58)](#general-job-description-extensions-q49q58)
9. [Real general interview bank (Q59–Q101)](#real-general-interview-bank-q59q101)
10. [Final tips](#final-tips)

---

## EKS & Kubernetes

### Q1. How do you provision a production-grade EKS cluster with Terraform?

**Answer.** Use a modular layout: networking, control plane, node compute, add-ons, and cluster services.

- **Networking:** VPC with private subnets across ≥2 AZs; NAT per AZ for HA if required. Workloads in private subnets. Consider VPC endpoints (ECR, S3, CloudWatch, STS) to cut NAT cost and tighten egress.
- **Control plane:** Enable KMS encryption for secrets; enable control plane logging (api, audit, authenticator, controllerManager, scheduler). Prefer private-only API endpoint plus controlled access (VPN, SSO, bastion pattern).
- **Nodes:** Managed node groups and/or **Karpenter**; separate critical (on-demand) from flexible (Spot) workloads via labels/taints. IAM for nodes: minimal node role; app permissions via IRSA / Pod Identity — not on the node role.
- **Add-ons:** Align VPC CNI, CoreDNS, kube-proxy (and CSI if needed) versions with the Kubernetes version. Manage via `aws_eks_addon` or consistent Helm/GitOps.
- **State:** Remote backend (S3 + DynamoDB lock); separate state per env and per layer where it reduces blast radius.

**Design calls:** private API, encryption, logging, clear upgrade path, and autoscaling that matches workload shape.

**general fit:** Postings call out **EKS, EC2, KMS, CloudWatch** — tie answers to **encryption (KMS)**, **operational visibility (control plane logs + CloudWatch alarms/dashboards)**, and **stable multi-environment** clusters that many Vodafone-facing services depend on.

### Q2. Explain IRSA and configure a pod to read S3

**Answer.** **IRSA (IAM Roles for Service Accounts)** lets a Kubernetes service account assume an IAM role via the cluster OIDC provider — short-lived credentials, no static keys in pods.

1. Create an IAM policy (e.g. `s3:GetObject` on a known bucket/prefix).
2. Create an IAM role with a trust policy allowing `sts:AssumeRoleWithWebIdentity` from the EKS OIDC issuer, scoped with `StringEquals` on `sub` (and usually `aud` = `sts.amazonaws.com`).

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/oidc.eks.REGION.amazonaws.com/id/CLUSTER_OIDC_ID"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.REGION.amazonaws.com/id/CLUSTER_OIDC_ID:aud": "sts.amazonaws.com",
          "oidc.eks.REGION.amazonaws.com/id/CLUSTER_OIDC_ID:sub": "system:serviceaccount:NAMESPACE:SA_NAME"
        }
      }
    }
  ]
}
```

3. Annotate the service account and use it in the pod:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/s3-reader-role
```

AWS SDKs in the pod pick up credentials via the projected service account token.

### Q3. Karpenter vs Cluster Autoscaler

**Answer.**

| Aspect | Cluster Autoscaler | Karpenter |
|--------|-------------------|-----------|
| Mechanism | Scales ASG/MNG desired capacity | Provisions EC2 directly per `NodeClaim` |
| Speed | Often slower (ASG lifecycle) | Often faster, finer-grained |
| Instance choice | Tied to node group config | Flexible instance types/sizes/Spot |
| Bin-packing / consolidation | Limited | Consolidation, disruption budgets |
| Ops model | Familiar with MNG | CRDs, policies, more moving parts |

**Choose CA** when you need strict, pre-approved instance families/SKUs per compliance. **Choose Karpenter** for cost and responsiveness on heterogeneous workloads. Many teams use both patterns during migration.

### Q4. How does the AWS VPC CNI work? IP exhaustion?

**Answer.** The **amazon-vpc-cni** assigns VPC IPs to pods (often secondary IPs on ENIs on the node). Exhaustion appears as pods stuck `Pending` with IP assignment errors.

**Mitigations:** larger pod subnets; **custom networking** (pods in dedicated subnets); tune warm pool settings; prefer instance types with higher ENI/IP limits; **IPv6** where supported; design multi-account IPAM up front; avoid too-small `/24` pod subnets at scale.

### Q5. Zero-downtime EKS upgrade (control plane + nodes)

**Answer.**

- **Before:** Check API deprecations (e.g. Pluto, Kubernetes release notes); upgrade add-ons in lockstep; ensure PDBs, HPA, and enough capacity across AZs.
- **Control plane:** Bump version in Terraform/EKS API; expect brief API blips; run during low traffic with comms ready.
- **Nodes:** Blue/green or rolling: new node group (or new Karpenter NodePool) on new AMI/version → cordon/drain old nodes with PDB respect → remove old group. Validate DaemonSets, storage CSI, and CNI compatibility.

### Q6. Pod stuck in `Pending` — troubleshooting

**Answer.** `kubectl describe pod` → **Events**.

- **Resources:** insufficient CPU/memory on cluster → scale nodes or reduce requests.
- **PVC:** not bound → StorageClass, CSI driver, AZ topology.
- **Scheduling:** nodeSelector, affinity, taints/tolerations.
- **Images:** pull errors, private registry auth.
- **Autoscaler:** Karpenter/CA logs, IAM, limits.
- **Admission:** failing webhooks (timeouts or deny).

### Q7. Five EKS security practices

**Answer.**

1. **IRSA or EKS Pod Identity** — no long-lived keys in images or ConfigMaps.
2. **Pod Security** — PSS/PSA or policy engine (Kyverno/Gatekeeper): non-root, drop caps, read-only root where possible.
3. **Network policies** — CNI that supports them (e.g. Calico/Cilium) for east-west segmentation; security groups for pods where used.
4. **Encryption & audit** — KMS for etcd secrets; audit logs to S3/CloudWatch; guard admin `system:masters` usage.
5. **Supply chain** — signed images, minimal base images, ECR scanning, patch nodes and add-ons.

### Q8. Helm in GitOps with Argo CD

**Answer.** Helm packages templated manifests. general-style postings emphasize **interpreting, modifying, and managing Helm charts across development, testing, and production** — typically `values-dev.yaml`, `values-test.yaml`, `values-prod.yaml` (or folder-per-env) with the **same chart version** promoted through environments.

In GitOps, the **chart + values** live in Git; Argo CD `Application` points to the chart path and `valueFiles` per environment. Changes merge to Git → sync to cluster. Use **Kustomize overlays** instead of or alongside Helm if teams prefer patch-based env separation.

Secrets should not be plaintext in values: use External Secrets, Sealed Secrets, or Argo CD plugins with strict RBAC.

---

## Terraform & IaC

### Q9. Terraform state in a team

**Answer.** Remote state in **S3** (versioning, encryption, bucket policy), **DynamoDB** for locking. Separate state per environment and logical slice (network, cluster, data) when it improves safety and plan scope. Use OIDC-based CI roles to run plans/applies; never commit `.tfstate`.

### Q10. Structure for many microservices

**Answer.** Reusable **modules** (network, EKS, “service baseline”: IAM, SQS, SNS, parameters). **Environment folders** compose modules with backend config. For services, either a thin wrapper module per service or a service catalog pattern generating consistent tags, naming, and guardrails.

### Q11. Detect and fix drift

**Answer.** **Detect:** scheduled `terraform plan` in CI; Terraform Cloud drift notifications; Config rules on tag-based ownership. **Remediate:** intentional change → import or codify; accidental → `apply` to desired state; **prevent** SCPs/IAM boundaries so only automation can mutate production baselines.

### Q12. Secrets in Terraform

**Answer.** Never commit secrets. Use **Secrets Manager / SSM** data sources; `random_password` + immediate write to Secrets Manager for DB bootstrap; `sensitive = true` on outputs; inject via CI env (`TF_VAR_*`) without echoing logs; consider Vault for dynamic DB creds.

### Q13. Workspaces vs directories

**Answer.** **Workspaces** swap state file names inside one config — OK for small/ephemeral cases. **Directories** (or separate root modules) per env are clearer for different variables, backends, and approvals — preferred for production.

### Q14. Production workflow with approvals

**Answer.** PR → `fmt`/`validate`/`plan` artifact → human review → merge → apply pipeline with **manual gate for prod**, separate role/session, and **post-apply smoke tests**. Optionally retain plans as immutable artifacts; document rollback (previous tag + apply or feature flags).

### Q15. Enforce “only via Terraform”

**Answer.** Combine **SCPs** (deny changes unless role/tag conditions match), **least-privilege human roles**, **Config + remediation** for untagged resources, and **culture** (self-service modules, fast pipelines). Perfect enforcement is rare; depth depends on org maturity.

---

## CI/CD & GitOps

### Q16. Jenkins + GitHub pipeline to EKS

**Answer.** general listings mention **Jenkins, Argo CD, GOCD**, and **GitLab / GitHub / Git** — the pattern is the same; only the orchestrator changes.

**Typical stages:** checkout → unit tests → **SonarQube** (quality gate) → build image → **container scan** (Trivy/ECR scanning) → push to **ECR** or **Nexus** as registry (if used) → deploy dev (**Helm** upgrade or GitOps sync) → integration tests → **manual approval** → staging → approval → prod (**canary/blue-green** via Argo Rollouts/Flagger or ALB weighted targets) → smoke + **CloudWatch/vendor APM** gate.

**Same image digest** (or immutable tag) promoted across dev/test/prod. **CodeBuild/CodeCommit** can replace parts of Jenkins for AWS-native pipelines; **GOCD** emphasizes pipeline-as-value and fan-in/fan-out; **GitLab CI** uses `.gitlab-ci.yml` with similar stages.

### Q17. Jenkins agents on EKS

**Answer.** **Kubernetes plugin** with pod templates per agent label; agents as ephemeral pods; tools in sidecars or custom agent image. Benefits: elasticity, isolation, cost vs static agents. Protect controller: RBAC, network policy, credential stores (not in pod specs), and backups for Jenkins home if stateful.

### Q18. Jenkins shared library

**Answer.** Groovy in a repo, loaded with `@Library`. Centralize steps like build/push, Helm deploy, Slack notify. Version libraries (tags); test with **JenkinsPipelineUnit**; avoid unreviewed `master`-only library for regulated envs.

### Q19. Migrate Jenkins → GitHub Actions safely

**Answer.** Inventory pipelines and secrets → map stages to jobs → **OIDC to AWS** (no long-lived keys) → run **dual triggers** and compare → migrate low-risk repos first → train and retire Jenkins jobs. Keep rollback path until parity is proven.

### Q20. GitOps and Argo CD on Terraform-built EKS

**Answer.** **GitOps:** desired state in Git; controller reconciles cluster. Terraform provisions cluster, IAM, Argo CD install (Helm provider or GitOps bootstrap). Argo CD `Application`/`AppProject` enforce destinations, namespaces, and RBAC. Cluster admins don’t `kubectl apply` ad hoc for apps — they merge PRs.

**Flux CD (also listed):** Git-native reconciler (Kustomize/Helm sources); strong multi-tenancy and **OCI helm** workflows. **Argo CD** is UI-rich and widely adopted for app-of-apps. Many enterprises pick **one** as standard; hybrid is possible during migration (different clusters or tiers).

### Q21. AWS Load Balancer Controller and internet ALB

**Answer.** Controller watches **Ingress** or **Gateway API** resources and manages ALB/NLB lifecycle. Install with **IRSA**. Example ingress annotations: scheme `internet-facing`, target-type `ip` when using IP mode, health checks, SSL with ACM cert ARN, WAF association if required.

### Q22. Helm charts in Argo CD

**Answer.** Point `Application.spec.source` to Helm chart path/repo; use `helm.parameters` or `valueFiles` per env; optional `helmfile` pattern or app-of-apps. For multi-cluster, use Argo CD clusters + projects with allowed destinations.

---

## AWS networking & security

### Q23. Why IRSA matters (security angle)

**Answer.** Same mechanism as Q2, emphasized for interviews: **no static keys**, **per-workload least privilege**, **auditable role assumptions**, **automatic rotation** via OIDC. It is the default pattern for AWS API access from pods.

### Q24. Multi-account layout (dev / staging / prod)

**Answer.** **Organizations** with OUs: **Security** (logs, guardrails), **Infrastructure** (shared CI, ECR, maybe DNS), **Workloads** (separate accounts per env or per domain). Central logging account, cross-account CloudTrail, IAM Identity Center for human access. Terraform assumes into env accounts from a dedicated automation role.

### Q25. Stop manual console changes in prod

**Answer.** SCPs + break-glass roles + Config alerts + **only automation roles** can mutate tagged baselines. Pair with fast, safe Terraform workflows so people don’t bypass them.

### Q26. Security group vs NACL vs Kubernetes NetworkPolicy

**Answer.** **SG:** stateful, ENI-level, primary for AWS resources. **NACL:** subnet-level, stateless, coarse. **NetworkPolicy:** pod-level segmentation inside the cluster (depends on CNI support). Layer them: NACL for broad zoning, SG for nodes/LBs, NetworkPolicy for service-to-service.

### Q27. Application secrets on EKS

**Answer.** **External Secrets Operator** or **Secrets Store CSI** + Secrets Manager/Parameter Store; sync to K8s Secret or mount. Use IRSA/Pod Identity for the operator. Rotate at source; avoid committing to Git; restrict `get secret` RBAC.

### Q28. Karpenter consolidation and cost

**Answer.** Consolidation **repacks** workloads onto fewer/cheaper nodes when safe (respecting budgets and disruption controls), terminating underused capacity. Together with Spot and right-sized instance types, it often materially reduces compute waste versus static pools.

### Q29. EKS Pod Identity vs IRSA (conceptual)

**Answer.** **Pod Identity** is the newer AWS-native way to bind IAM roles to service accounts without managing OIDC provider wiring per role in the same way; the agent on nodes handles credential flow. **IRSA** uses OIDC trust policies on roles. Interviews may expect: both avoid static keys; migration and regional/feature availability matter; choose what your org standardizes on.

### Q30. Private ECR and cross-account pulls

**Answer.** Use **repository policies** and/or **RAM** for sharing; VPC endpoints for ECR API and DKR in private subnets; ensure node IAM allows `ecr:GetAuthorizationToken` and pulls. For multi-account, pull from a central “golden images” account with scanning enforced.

### Q31. WAF in front of EKS ingress

**Answer.** Associate **AWS WAF** with the ALB (or CloudFront in front). Use managed rule groups, rate limiting, geo/IP allow lists, and logging to Kinesis/S3. Coordinate with security for false positives and change control.

---

## Data, resilience & compliance

### Q32. RDS/Aurora with apps on EKS

**Answer.** Prefer **private subnets**, **security groups** allowing only from namespaces/node groups or service mesh egress. Use **RDS Proxy** for connection pooling and failover smoothing. Credentials via Secrets Manager + ESO. Consider IAM DB auth where supported. Plan backups, PITR, and patching windows; align AZ placement with EKS.

### Q33. Disaster recovery for EKS workloads

**Answer.** Define **RTO/RPO**. Patterns: backup etcd/apps with **Velero** to S3 (cross-region replication); replicate data (RDS cross-region, S3 CRR); **warm standby** cluster or infrastructure-as-code spin-up; Route 53 / Global Accelerator failover; runbooked game days. GitOps repo is the source of truth for workload definitions.

### Q34. S3 security for app and pipeline artifacts

**Answer.** Block public access; encryption at rest (SSE-S3 or KMS); bucket policies denying insecure transport; VPC endpoints; lifecycle to Glacier/deep archive; access logging; **Object Lock** where immutability is required; least-privilege IAM; malware scanning where needed.

### Q35. AWS Backup and Kubernetes

**Answer.** **AWS Backup** excels at AWS-native resources (EBS, RDS, etc.). For Kubernetes state, combine **Velero** (cluster resources + PV snapshots where configured) with AWS Backup for databases. Align retention with legal/compliance tags.

### Q36. Compliance touchpoints (telecom / enterprise)

**Answer.** general postings cite **ITIL**, **SOX**, and **security regulations** alongside **telecom** context.

Be ready to discuss: **data residency**, encryption in transit/at rest, **KMS** CMKs, **audit trails** (CloudTrail, K8s audit logs, pipeline audit), **access reviews** (IAM Identity Center / SSO), **vulnerability management** (ECR scanning, Inspector, SonarQube), **segregation of duties** (who approves prod deploys vs who builds), **change tickets** (Jira/Remedy-style) linked to releases, and **retention** for logs/evidence. Tie CI/CD to **approved change windows** where the business requires them.

---

## Observability, cost & incidents

### Q37. Observability stack for EKS

**Answer.** JDs list **CloudWatch** plus **Prometheus & Grafana** and enterprise APM/log tools: **Instana**, **Splunk**, **Dynatrace**, **DataDog** (and sometimes overlapping use of Instana/Dynatrace in different accounts).

**Practical layering:**

- **AWS foundation:** **CloudWatch** metrics/alarms for AWS and Container Insights; logs to log groups; dashboards for ops.
- **Kubernetes:** Prometheus + Grafana (or AMP + AMG); kube-state-metrics, node exporter, cAdvisor.
- **Logs:** Fluent Bit / OpenTelemetry Collector → **Splunk** HEC / **Datadog** / CloudWatch; **structured JSON** for correlation.
- **APM / traces:** **Instana** or **Dynatrace** agents/operators, or **OpenTelemetry** exporting to vendor or **X-Ray**.
- **SLOs:** RED/USE dashboards, error budgets, Alertmanager or vendor alerting → **PagerDuty/Slack/Jira** for incidents.

Interview tip: describe **one** end-to-end path you used (e.g. metric → dashboard → alert → ticket) and how you avoid **duplicate paging** across tools.

### Q38. EKS bill growing — what do you do?

**Answer.** Tag everything; **Cost Explorer** + **CUR**; **Kubecost** or similar for K8s allocation; right-size requests/limits; Spot + consolidation; Graviton where compatible; review NAT vs endpoints; clean unused EBS/snapshots/LBs; schedule non-prod scale-down; review data transfer.

### Q39. Pod `CrashLoopBackOff`

**Answer.** `kubectl logs --previous`, `describe` for **OOMKilled**, probe failures, env/config errors; compare limits vs actual usage; temp debug pod or `kubectl debug`; check init containers; validate ConfigMaps/Secrets and service dependencies.

### Q40. Production outage story (STAR)

**Answer.** Prepare a real example: **Situation** (symptom, blast radius), **Task** (your role), **Action** (commands, rollback, comms, Terraform/git revert), **Result** (MTTR, customer impact, follow-up). Mention **post-incident** actions: runbook, automated check, guardrail in pipeline.

### Q41. Alert fatigue vs missing pages

**Answer.** Tune alerts on **SLOs** and user-visible symptoms; dedupe/route in Alertmanager; on-call runbooks; **canary** checks; periodic alert review; kill low-value pages. Balance with error budgets and SLI-based thresholds.

### Q42. Image supply chain

**Answer.** Minimal bases, pinned digests in prod, **ECR scanning** (and CI scanning with Trivy/Grype), SBOM generation, admission policies (Kyverno verifyImages with cosign), dependency updates, and break-glass process for CVE exceptions.

---

## Senior & behavioral

### Q43. Reviewing a junior’s Terraform PR

**Answer.** (1) **Security** — IAM scope, public exposure, secrets. (2) **State & scope** — correct workspace/backend, no dangerous `force-unlock` habits. (3) **Maintainability** — tags, naming, `for_each` vs fragile `count`, variable defaults, lifecycle where needed. Run plan in CI and ask them to explain destroy risk.

### Q44. Disagreement on tools (e.g. Jenkins vs Argo CD)

**Answer.** Anchor on **outcomes**: speed, safety, audit, skill fit. Prototype or time-box comparison; document trade-offs; escalate to architecture forum if stuck; implement **hybrid** if sensible (CI in one tool, CD GitOps in another).

### Q45. Mentoring juniors

**Answer.** general explicitly asks for **mentoring and educating Dev and DevOps teams** on **infrastructure automation** and best practices.

Pair on incidents and PRs; short **lunch-and-learns** (Helm values promotion, Argo CD sync policies, Terraform modules); **safe sandboxes**; **Confluence** runbooks and **Jira** templates for changes; goal-based growth (e.g. **CKA**, **AWS DevOps Engineer Professional** / Solutions Architect); delegate ownership of a non-prod namespace or small service with guardrails and review checkpoints.

### Q46. Staying current

**Answer.** AWS/Kubernetes release notes, Well-Architected updates, CNCF landscape discipline, hands-on sandbox, internal guilds, and selective conferences or well-curated feeds — depth beats hype.

### Q47. Why general / this role

**Answer.** Personalize using JD language: **global Vodafone markets**, **secure reliable multi-client platforms**, ownership of **migrations and upgrades**, **SLA-driven** incident culture, and **continuous improvement** of CI/CD and observability. Mention hands-on fit: **EKS + Helm + Argo CD**, **Terraform**, **Bash/Python**, collaboration in **Agile** (Scrum/Kanban) with **Jira/Confluence**, and interest in **telecom-scale** resilience if applicable.

### Q48. Questions for the interviewer

**Examples:** EKS upgrade cadence and pain points; **standard GitOps tool (Argo vs Flux)** and **CI (Jenkins vs GitLab vs CodePipeline)**; **APIGEE** or other API platforms in the path to microservices; **on-call / ITIL** process and tooling (**Remedy**, Jira Service Management); how **SOX** or audit affects deploy approvals; **multi-tenant vs dedicated** cluster patterns; FinOps expectations; partnership with security and networking (DNS, proxies).

---

## general job description extensions (Q49–Q58)

### Q49. GOCD vs Jenkins for CD — how would you position them?

**Answer.** **Jenkins:** huge plugin ecosystem; shared libraries; often used for **build + deploy** in one place; more ops overhead for controllers/agents. **GOCD:** first-class **pipeline dependencies**, value-stream visibility, environment progression; good when modeling **fan-in/fan-out** and **deployment pipelines** explicitly. Many orgs use **Jenkins/GitLab CI for build**, **Argo CD for deploy** (GitOps), with GOCD where legacy pipelines already exist. Emphasize **artifact promotion**, **approvals**, and **traceability** — all map to general multi-env expectations.

### Q50. ITIL: incident vs problem management; how do you prioritize under SLA?

**Answer.** **Incident:** restore service as fast as possible (customer-impacting degradation/outage). **Problem:** find **root cause** and prevent recurrence (trend analysis, known error DB). Prioritize by **business impact**, **SLA tier**, number of users affected, and **regulatory** exposure. Communication: war-room for major incidents, **Jira/Remedy** tickets, timeline, workaround vs fix. Post-incident: **PIR**, action items into backlog, update runbooks in **Confluence**. This mirrors general language: **own incidents and problems**, vendor escalations, **documentation**.

### Q51. APIGEE / API management with EKS microservices — how do they fit together?

**Answer.** **APIGEE** (or similar API management) typically sits as the **edge/governance** layer: authentication (OAuth/OIDC), rate limits, quotas, **API products**, analytics, and sometimes **WAF**-class policies. **EKS** services expose APIs behind **Ingress/ALB** or internal NLBs; APIGee routes **external** traffic to those backends (or to an internal gateway). Kubernetes handles **service discovery** and scaling; Apigee handles **developer portal**, **consumer contracts**, and **central policy**. Discuss **mTLS**, **VPC connectivity**, and **least-privilege** from gateway to services — relevant to **multi-client** Vodafone integrations in the JD.

### Q52. Onboarding a new team’s microservice onto shared EKS — your checklist

**Answer.** Aligns with **lead customer interfacing** / **onboard applications** language.

- **Namespace & RBAC:** dedicated namespace, quotas, NetworkPolicies baseline.
- **IAM:** IRSA/Pod Identity roles; no cluster-admin for app teams.
- **CI/CD:** repo structure, image registry (**ECR/Nexus**), scan + sign policy, Helm chart standards.
- **GitOps:** Argo CD `Application`, project restrictions, env promotion path.
- **Observability:** mandatory labels, logs/metrics/traces, dashboards, on-call ownership.
- **Secrets:** External Secrets + **KMS**; rotation ownership documented.
- **Runbook:** deploy, rollback, known failure modes in **Confluence**.
- **Review:** capacity, PDBs, HPA, dependencies on **RDS/Redis/Mongo** if any.

### Q53. On-premises to AWS/EKS migration — high-level approach

**Answer.** **Assess:** app dependencies, statefulness, licensing, latency to integrated systems. **Pilot:** containerize (Dockerfiles for **frontend/backend**), externalize config (**ConfigMaps/Secrets**), health checks. **Land:** VPC, EKS, CI/CD, observability parity. **Data:** **RDS**/managed DB vs rehost; **Redis**/Mongo via managed (ElastiCache, DocumentDB) or operators with clear **backup/DR**. **Cutover:** DNS (Route 53), **blue/green** or canary, rollback plan. **Optimize:** rightsizing, Spot where safe, **cost** reviews — JD explicitly asks for **performance and/or cost reduction** improvements after landing.

### Q54. SonarQube and Nexus (or ECR) in the pipeline

**Answer.** **SonarQube:** quality gates on **bugs, vulnerabilities, code smells**, coverage — fail build on policy for **main/prod** branches. Run early in CI (Jenkins/GitLab/CodeBuild). **Nexus:** artifact repo for libraries, Docker images, or helm charts in some enterprises; **ECR** is the natural AWS-native registry for images. Pattern: build → scan (Sonar + container scanner) → push **immutable digest** to ECR → deploy via GitOps. Mention **SBOM** or **policy** if asked about supply chain.

### Q55. Terraform and CloudFormation together — how do you avoid chaos?

**Answer.** JD lists both. Pick **one source of truth per resource** (avoid double-managing the same object). Common patterns: **Terraform owns** landing zone/EKS/IAM; **CloudFormation** only for legacy stacks or service-catalog templates until migrated; or **Terraform imports** existing CFN outputs via SSM/remote state. Document boundaries in **Confluence**; enforce with **tags** and **code owners**.

### Q56. MongoDB and Redis in a general-style stack (listed as a plus)

**Answer.** Prefer **managed** when possible: **ElastiCache (Redis)**, **DocumentDB** or **MongoDB Atlas** on AWS — less day-2 ops, clearer backups. On EKS: **operators** (Redis, MongoDB Community/Enterprise) with **persistent volumes**, **backup** (Velero/snapshots), **security** (auth, TLS, network policy). Watch **connection storms** from microservices — use **connection pooling** and **proxies** where appropriate (e.g. RDS Proxy pattern conceptually for DB tier).

### Q57. SOX-oriented controls for infrastructure and releases

**Answer.** **Change management:** tickets linked to PRs/releases; **segregation of duties** — engineers cannot self-approve prod; **four-eyes** on Terraform apply or Argo CD sync for prod. **Audit trail:** immutable **CloudTrail**, pipeline logs retained, **Git** history as record. **Access:** SSO, periodic access reviews, break-glass accounts monitored. **Secrets:** no shared admin passwords; **KMS** key policies. Be honest about scope: SOX touch varies by entity; show you know how to **evidence** controls for auditors.

### Q58. DNS, TLS, and corporate proxy — what breaks in real life?

**Answer.** JD mentions **DNS and proxy**. Typical issues: **image pulls** blocked by proxy — configure `HTTP_PROXY` on nodes/daemonsets carefully or use **transparent** egress; **VPC endpoints** to reduce public egress. **Private ECR** + DNS resolution for interface endpoints. **Cert chains** wrong on Ingress → fix ACM or cert-manager issuers. **Internal DNS** (Route 53 private zones) for service-to-service; **split-horizon** for hybrid on-prem. For CI runners: same proxy/certificate trust store as corporate policy. Show structured troubleshooting: **dig/nslookup**, **curl -v**, node **containerd** pull logs, **CoreDNS** logs.

---

## Real general interview bank (Q59–Q101)

Questions below are from a **past general interview** (your notes). Answers are **merged** with earlier sections: if a topic already exists, this section adds **interview-style phrasing** and points to the earlier Q.

### Terraform & state

#### Q59. What if the remote backend (e.g. S3 state bucket) is deleted?

**Answer.** Terraform loses the **source of truth** for what it thinks is deployed. **Immediate:** stop applies that could recreate or conflict; treat env as **unknown drift**. **Recovery options:** restore the bucket from **S3 versioning** / **backup** if available; restore `terraform.tfstate` from **CI artifacts** or **Terraform Cloud** history; if gone for good, you must **reconcile reality vs code** (`terraform import` for surviving resources, or targeted `terraform state push` only if you have a trusted backup). **Prevention:** bucket **versioning**, **MFA delete**, replication, and **least-privilege** so the bucket cannot be deleted casually.

#### Q60. What tool can document Terraform?

**Answer.** **`terraform-docs`** generates Markdown/HTML tables of inputs/outputs from comments and `description` in variables/outputs. Teams often run it in **pre-commit** or CI. Complement with **diagrams** (e.g. `terraform graph` + Graphviz, or external diagrams-as-code) and **module READMEs** in Confluence/Git.

#### Q61. Explain creating an EKS cluster with Terraform (interview walkthrough).

**Answer.** High level matches **Q1**: VPC/subnets/SGs → EKS cluster resource (version, logging, encryption, endpoint access) → node groups or Karpenter IAM → addons → outputs (endpoint, CA). Mention **remote backend + lock** (**Q9**). Do not repeat full design here in the room — narrate **order of operations** and **dependencies**.

#### Q62. Add an existing AWS resource to Terraform / remove from state / stop managing without “comment hacks”.

**Answer.** **Import:** `terraform import <resource_address> <id>` then write matching HCL so `plan` is clean. **Stop tracking without destroying:** `terraform state rm <address>` — resource stays in AWS but Terraform forgets it (dangerous if someone later applies a config that recreates it). **Declarative removal (Terraform 1.7+):** use a **`removed` block** to document that a resource was intentionally deleted from config and how to handle destroy. **Ignore drift on specific attributes:** `lifecycle { ignore_changes = [attribute] }` — still manages the resource. For “ignore entire drift” use cases, prefer **splitting stacks** or **`moved`/`removed`** over ad-hoc `#` comments.

#### Q63. Locking backend — what is it?

**Answer.** With **S3 backend**, **DynamoDB** (or Terraform Cloud) provides a **lock** so two applies cannot corrupt state. Same idea as **Q9** — mention `dynamodb_table` and that **force-unlock** is break-glass only.

#### Q64. Securing sensitive data in Terraform.

**Answer.** Align with **Q12**: never commit secrets; **SSM/Secrets Manager** data sources; **`sensitive = true`** on variables/outputs; **masked CI logs**; least-privilege IAM for pipeline roles. Mention **KMS** for S3 backend encryption.

#### Q65. Terraform “unit” / automated testing.

**Answer.** **Terratest** (Go) or **terraform test** (`.tftest.hcl`, 1.6+) for module behavior; **policy tests** with **OPA/Sentinel/Checkov/tfsec** in CI. “Unit” in infra usually means **small module tests** + **mocked AWS** or **ephemeral account**, not classical function units.

#### Q66. Dynamic blocks and `locals`.

**Answer.** **`dynamic` blocks** generate nested blocks (e.g. multiple `ingress` rules) from a **list/map** — avoids copy-paste. **`locals`** compute derived values once, keep DRY, and simplify expressions. Interview: show you use them to **reduce duplication** and make modules readable.

#### Q67. Avoid re-downloading Terraform providers/modules in every pipeline run.

**Answer.** Persist **`TF_PLUGIN_CACHE_DIR`** (or `.terraform/providers`) between CI jobs; run **`terraform init -plugin-cache=...`**; cache **`.terraform`** directory per **lock file** hash; use **private registry** mirrors if allowed. Pin **provider version constraints** so cache stays stable.

---

### Jenkins & quality tools

#### Q68. Why Jenkins vs other automation tools? Master–agent architecture? Freestyle? Scripted vs declarative?

**Answer.** **Why Jenkins:** huge **plugin** ecosystem, **self-hosted** control, fits brownfield enterprises, **shared libraries** — at the cost of **maintenance**. Compare to GitLab CI, GitHub Actions, Azure DevOps (managed, less ops). **Master–controller / agents:** controller schedules work; **agents** (executors on VMs, containers, K8s) run builds — isolate untrusted build code from the controller. **Freestyle:** GUI-configured jobs, simple scripts — legacy but still common. **Declarative pipeline** (`pipeline { }`): structured, opinionated, good for reviews. **Scripted pipeline** (`node { }` + Groovy): maximum flexibility, easier to become unmaintainable. **Also see Q16, Q49.**

#### Q69. SonarQube, JaCoCo, Trivy, Apigee — what are they for?

**Answer.** **SonarQube:** static analysis + quality gates (bugs, smells, security hotspots). **JaCoCo:** **Java code coverage** (often fed into SonarQube). **Trivy:** **container image** and IaC **vulnerability** scanning (OCI, fs, K8s manifests). **Apigee:** **API management** (gateway, policies, developer portal, analytics) — **conceptual overlap with Q51**; **SonarQube in CI** also **Q54, Q42**.

---

### AWS — IAM & policies

#### Q70. What is IAM? Explain IAM policies.

**Answer.** **IAM** controls **who** can do **what** on **which resources** in AWS. An **IAM policy** is JSON (or visual editor) listing **Effect**, **Action**, **Resource**, optional **Condition**. Policies attach to **users, groups, roles**, or **resources** (resource-based).

#### Q71. IAM role vs IAM user.

**Answer.** **User:** long-term identity for a **human** or legacy app (access keys risk). **Role:** **temporary credentials** via **STS**; assumed by users, services, or **federated** identities; **preferred** for workloads (EC2 instance profile, Lambda, EKS IRSA). **Least privilege** and **no static keys** favor roles.

#### Q72. Resource-based policy vs identity-based policy.

**Answer.** **Identity-based:** attached to IAM user/group/role — “this principal may act on resources.” **Resource-based:** attached to the resource (S3 bucket policy, KMS key policy, SNS topic policy) — “these principals may access **this** resource.” **Cross-account** access often needs **both sides** (e.g. role trust + bucket policy).

---

### AWS — compute, database, storage

#### Q73. Explain RDS and how to improve performance.

**Answer.** **RDS** is managed relational DB (Multi-AZ, backups, patching). **Improve performance:** right-size **instance**; **storage** type/IOPS (gp3/io2); **read replicas** for read scale; **RDS Proxy** for pooling; tune **indexes** and queries; **Parameter groups**; **Performance Insights**; **caching** (ElastiCache) for hot reads; avoid chatty app patterns. **Also see Q32.**

#### Q74. What is EC2? How troubleshoot SSH timeout?

**Answer.** **EC2** is resizable compute in the cloud. **SSH timeout:** check **security group** (22 from your IP/bastion), **NACL**, **instance** is running, **public IP / correct subnet / route to IGW**, **user data** boot failures, **SSM Session Manager** path if SSH blocked, **corporate VPN** routing, and **host firewall** inside the guest. **Also see Q100** for a full SSH checklist.

#### Q75. AWS storage services and storage classes (S3 focus).

**Answer.** **Block:** EBS volumes (gp3, io2, st1, sc1). **File:** **EFS**, **FSx** (Lustre, ONTAP, Windows). **Object:** **S3** with classes — **Standard**, **Standard-IA**, **One Zone-IA**, **Glacier Instant/Flexible/Deep Archive**, **Intelligent-Tiering**. Choose by **latency, cost, retrieval, durability**. JD mentions **EBS, S3, Glacier** — tie answers to **lifecycle** policies.

#### Q76. What is S3 versioning?

**Answer.** Keeps **multiple variants** of an object under the same key; protects against **accidental overwrite/delete** (with MFA delete optional); enables **rollback**; increases **storage cost** — use **lifecycle** to expire noncurrent versions.

#### Q77. What are EFS and FSx?

**Answer.** **EFS:** NFS file system, **POSIX**, scales automatically, good for **many clients** (containers, Lambda with limits). **FSx:** **specialized** file systems — **Lustre** (HPC/ML), **NetApp ONTAP**, **OpenZFS**, **Windows File Server** — pick when you need **that protocol/feature set**, not generic NFS.

---

### Serverless, messaging, integration

#### Q78. Create a serverless architecture (example).

**Answer.** Example: **API Gateway** → **Lambda** → **DynamoDB**; async via **SQS**; **Step Functions** for long workflows; **EventBridge** for scheduling/events; static on **S3 + CloudFront**; auth **Cognito**. Mention **IAM least privilege**, **VPC** only if needed, **observability** (X-Ray, CloudWatch).

#### Q79. SQS, RabbitMQ, Lambda, Lambda@Edge, Step Functions — roles?

**Answer.** **SQS:** AWS-managed **queue**, decouple producers/consumers, retries, DLQ. **RabbitMQ:** common **message broker** (often **Amazon MQ**); flexible routing (exchanges); more ops unless managed. **Lambda:** run code on events, **15 min max**. **Lambda@Edge:** run at **CloudFront edge** (viewer/request/response), low latency, **limited runtime**. **Step Functions:** **orchestrate** Lambdas/services, **long-running** workflows, retries, human tasks.

#### Q80. Temporary S3 access link?

**Answer.** **Presigned URL** (`Presign` in SDK/CLI): time-limited **GET/PUT** with **your credentials** embedded as signature; no bucket public. Set **short expiry**, **HTTPS**, least-privilege IAM who can sign.

#### Q81. Define “serverless function” and “Lambda trigger.”

**Answer.** **Serverless function:** code unit executed by a **fully managed** runtime **on demand**, **scaled** by the platform; you don’t manage servers (Lambda is the classic AWS example). **Trigger:** **event source** that invokes the function — e.g. **API Gateway**, **S3** event, **SQS** poll, **EventBridge** rule, **DynamoDB Streams**, **Cron** (scheduled EventBridge).

#### Q82. Lambda maximum execution time? How extend beyond 15 minutes?

**Answer.** **Max 15 minutes** per invocation (hard limit). To go longer: **Step Functions** (Standard workflows up to a year), **split work** into chained Lambdas, **ECS/Fargate** / **Batch** for long jobs, **SQS-driven** workers, or **child workflows**.

#### Q83. How does Lambda access S3 — from internet vs VPC?

**Answer.** By default Lambda in **AWS-managed VPC** reaches S3 **over the AWS network** (public S3 endpoints) with **IAM** — no NAT needed. If Lambda is **in a VPC** (for private DB), it needs **VPC endpoints** (S3 gateway) or **NAT** for public internet/S3; **IAM** still authorizes. **Clarify in interview:** “S3 is AWS API — use VPC endpoint or NAT depending on subnet routing.”

#### Q84. Deploy a dynamic website with serverless services.

**Answer.** **API Gateway + Lambda** (REST/HTTP) for APIs; **Cognito** auth; **DynamoDB** data; **S3** for static assets; **CloudFront** in front; **SSR** optionally **Lambda@Edge** or **Amplify** / **Fargate** if you outgrow Lambda limits.

---

### Networking & load balancing

#### Q85. How connect two VPCs? Can different accounts peer?

**Answer.** **VPC peering** (same or **cross-account** with acceptance); **non-transitive** (no automatic VPC1→VPC3 via VPC2). Alternatives: **Transit Gateway**, **PrivateLink**, **VPN / Cloud WAN**. **Also see Q96.**

#### Q86. NAT gateway — what and how it works?

**Answer.** **NAT GW** in a **public subnet** lets **private subnet** instances **initiate outbound** IPv4 to internet (updates, external APIs) while **blocking unsolicited inbound**. Return traffic is **stateful**. **NAT instance** is legacy DIY. **Cost:** prefer **VPC endpoints** for AWS APIs to reduce NAT traffic (**Q4, Q58**).

#### Q87. What is Route 53?

**Answer.** **DNS** service: **hosted zones**, **routing policies** (weighted, latency, failover, geo), **health checks**, integration with **ALB/CloudFront**, **hybrid DNS** with on-prem.

#### Q88. CloudFront caching static content from S3.

**Answer.** Create **S3 origin** (OAC/OAI for private bucket), **CloudFront distribution** with **behaviors**, **TTL/cache policies**, **optional WAF**. Clients hit **edge** POPs → fewer S3 GETs, lower latency globally.

#### Q89. Forward proxy vs reverse proxy.

**Answer.** **Forward proxy:** clients send traffic **through** it to internet (corporate egress, filtering). **Reverse proxy:** sits **in front of servers**; clients talk to proxy; proxy **terminates TLS**, **load balances**, caches — **ALB/NLB**, **nginx**, **API Gateway** are reverse-proxy patterns.

#### Q90. When use ALB vs NLB — generally and with EKS?

**Answer.** **ALB:** **Layer 7** HTTP/HTTPS, **host/path routing**, **WebSocket**, integrates with **Ingress** (AWS LB Controller) for **EKS** — default for **HTTP APIs**. **NLB:** **Layer 4** TCP/UDP, **extreme performance**, static IPs, **preserving client IP**, good for **non-HTTP**, **gRPC with TLS passthrough**, **internal** high-throughput services. **EKS:** Ingress → usually **ALB**; **Service type LoadBalancer** can be NLB when L4 needed. **Also see Q21.**

---

### Git

#### Q91. `git fetch` vs `git pull`; `git rebase` vs `git merge`; stash; merge conflict.

**Answer.** **`fetch`** downloads remote refs **without** merging; **`pull`** = `fetch` + merge (or rebase with config). **`merge`** preserves branch history with a merge commit; **`rebase`** replays commits on top of another branch for **linear history** (never rebase shared public branches). **`git stash`** saves WIP without commit. **Merge conflict:** edit conflicted files, remove markers, **`git add`**, complete merge/rebase — use **`git status`** and a diff tool.

---

### Architecture concepts

#### Q92. Resilience, failover, DR, high availability, compliance — what they mean and how to achieve?

**Answer.** **HA:** service stays up during **routine failures** (Multi-AZ, ASG, health checks, no single AZ). **Resilience:** **withstand and recover** from faults (retries, timeouts, circuit breakers, chaos testing). **Failover:** **automatic switch** to standby (RDS Multi-AZ, Route53 health checks). **DR:** **regional** or major disaster recovery — **RTO/RPO**, backups, **cross-region** replicas, runbooks (**Q33, Q50**). **Compliance:** **controls + evidence** — encryption, logging, access reviews, change management (**Q36, Q57**).

---

### Hybrid cloud

#### Q93. How connect Azure with AWS using “Direct Connect”?

**Answer.** **AWS Direct Connect** is **AWS ↔ on-prem/colocation**, not native **Azure ↔ AWS**. Cross-cloud options: **VPN** site-to-site between **Azure VPN Gateway** and **AWS VGW/Transit Gateway**; **ExpressRoute + partner / Megaport / Equinix** to a **carrier** that connects both clouds; **third-party SD-WAN**. If they said “Direct Connect,” clarify terminology and give the **VPN/partner exchange** pattern.

---

### Scenario & troubleshooting (short “four-point” style)

#### Q94. App with primary RDS (RW) + read replica (RO); ~35s delay then timeout — propose four solutions.

**Answer.** (1) **RDS Proxy** toward primary/replica with **correct read/write split** and **connection pooling** to cut connection storms. (2) Tune **timeouts** (app, pool, SG) and **DNS** (replica endpoint correctness). (3) **Replica lag** — check **CloudWatch** `ReplicaLag`, scale replica, fix heavy writes blocking replay. (4) **Network path** — same **VPC/region**, avoid accidental **cross-region** reader; add **caching** (ElastiCache) for read-heavy hot data.

#### Q95. VPC1 peered to VPC2, VPC2 to VPC3 — is VPC1 connected to VPC3? How connect many VPCs without a star?

**Answer.** **Peering is non-transitive** — **no** automatic path VPC1→VPC3. Use **Transit Gateway** (**hub-spoke**), **full mesh peering** (does not scale), or **Cloud WAN**. Star topology problems (single point of failure) → **multi-region TGW**, **redundant attachments**, or **segmented** network design.

#### Q96. Image processing app stores in S3; clients see high latency — what do you do?

**Answer.** **CloudFront** in front of S3; **S3 Transfer Acceleration** if uploads dominate; **parallel multipart** uploads; **regional buckets** closer to users; **Lambda@Edge** for lightweight transforms at edge; optimize **object size** and **TLS**; **VPC endpoints** for internal processors — not for public clients.

#### Q97. Stateful app: same user must hit same machine, but machine overloaded.

**Answer.** **Sticky sessions** (ALB stickiness) **hurt** at scale — prefer **server-side session store** (Redis/DynamoDB) so **any** node serves the user. If true affinity is unavoidable, **scale out** behind LB with **consistent hashing** or **shard users**; **offload state** gradually; use **HPA** + **stronger instances** as stopgap.

#### Q98. Kubernetes app must handle high traffic — steps?

**Answer.** **HPA/VPA** on CPU/custom metrics; **PDBs** for rollouts; **cluster autoscaler/Karpenter** (**Q3**); **requests/limits** tuned; **ingress** capacity (**Q21**, ALB); **caching** and **rate limiting** at edge; **load test** (k6); **DB** scaling and **pooling**; **CDN** for static. **Capacity/cost:** **Q38**.

#### Q99. Kubernetes slow deployments — causes and fixes?

**Answer.** **Image pull** latency (mirror, warm nodes, image size); **rolling update** `maxUnavailable`/`maxSurge` too conservative; **readiness probes** too slow or failing; **admission webhooks** latency; **scheduling** pending (resources, PDB); **Argo CD** sync waves or hooks waiting; **resource quotas**; **network policies** blocking health checks. Fix: tune deployment strategy, **pre-pull** images, **faster probes** with real checks, **more capacity**, profile **control plane** if many objects.

#### Q100. EC2 not reachable via SSH — troubleshooting steps.

**Answer.** Instance **running**? **Correct key** / username? **SG** allows 22 from your IP (changes if home IP moves)? **NACL**? **Public IP / EIP** and **subnet route** to IGW? **Private instance** needs **SSM Session Manager** or **bastion**? **CPU/memory** exhausted (tty unavailable)? **Disk full**? **CloudWatch serial console** / **screenshot** for boot errors.

#### Q101. Lambda exceeding memory — troubleshoot and fix.

**Answer.** Check **CloudWatch Logs** for **runtime killed / OOM** patterns; **AWS Lambda console** shows **memory use** vs configured **Max Memory**; **increase memory** (also grants more CPU in proportion); fix **memory leaks** / large payloads; **stream** S3 objects instead of loading whole file; use **Powertools** / profiling; right-size **concurrency** to avoid accidental parallelism spikes.

---

## Final tips

- **Depth over breadth** — for each topic, know one failure mode and one mitigation you’ve actually used.
- **STAR** for behavioral questions (incidents, migrations, cost wins).
- **Draw architectures** — multi-account, pipeline, observability, **API gateway → mesh/ingress → services** — on a whiteboard confidently.
- **Numbers** — when you quote savings or MTTR, be ready to justify them.
- **Certs mentioned in JDs:** AWS **DevOps Engineer Professional**, **Solutions Architect Professional**, or **Cloud Engineer** — name what you hold or are pursuing.
- **Languages:** be ready to show **Bash** and **Python** examples (small automation, boto3, kubectl helpers).

Good luck with your interview preparation.
