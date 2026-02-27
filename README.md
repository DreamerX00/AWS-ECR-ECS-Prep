# AWS ECR & ECS — Zero to Mastery Curriculum

> **Goal:** Crack interviews, design production architectures, debug live issues, optimize costs, and harden security on AWS ECS and ECR.

---

## How to Use This Curriculum

This curriculum is organized into **5 Phases**, each building on the previous one. Study them **in order** — do not skip phases.

### Recommended Study Order

1. Container Internals
2. ECR Architecture
3. ECS Core Components
4. Networking Model
5. IAM Deep Dive
6. Scheduler Mechanics
7. Deployment Patterns
8. Observability
9. Scaling Strategies
10. Failure Handling
11. Cost Optimization
12. Advanced Architecture Patterns

---

## Phase Overview

| Phase | Folder                                 | Topics                                                        |
| ----- | -------------------------------------- | ------------------------------------------------------------- |
| 0     | `Phase-0_Container-Foundations`        | Docker, Images, Layers, OCI                                   |
| 1     | `Phase-1_Amazon-ECR-Deep-Dive`         | ECR Architecture, Auth, Security, Cost                        |
| 2     | `Phase-2_Amazon-ECS-Core`              | ECS Components, Launch Types, IAM, Networking                 |
| 3     | `Phase-3_ECS-Advanced-Architecture`    | ALB, Service Discovery, Deployments, Observability, Scaling   |
| 4     | `Phase-4_Mastery-And-Interview-Prep`   | Cost Optimization, Performance, CI/CD, Edge Cases, Interviews |

---

## File Index

### Phase 0 — Container Foundations
| File | Topic |
|------|-------|
| [01_What_Is_Containerization.md](./Phase-0_Container-Foundations/01_What_Is_Containerization.md) | What containers are, namespaces, cgroups |
| [02_Docker_Architecture.md](./Phase-0_Container-Foundations/02_Docker_Architecture.md) | CLI → dockerd → containerd → runc chain |
| [03_Image_vs_Container.md](./Phase-0_Container-Foundations/03_Image_vs_Container.md) | Blueprint vs running instance, CoW |
| [04_Layers_And_UnionFS.md](./Phase-0_Container-Foundations/04_Layers_And_UnionFS.md) | OverlayFS, layer caching, build optimization |
| [05_OCI_Standard.md](./Phase-0_Container-Foundations/05_OCI_Standard.md) | Image spec, Runtime spec, Distribution spec |
| [06_Image_Immutability_Tagging_Digest.md](./Phase-0_Container-Foundations/06_Image_Immutability_Tagging_Digest.md) | Tags vs digests, immutable tags, tagging strategy |

### Phase 1 — Amazon ECR Deep Dive
| File | Topic |
|------|-------|
| [01_What_Is_ECR.md](./Phase-1_Amazon-ECR-Deep-Dive/01_What_Is_ECR.md) | ECR overview, private vs public, comparison with Docker Hub |
| [02_ECR_Internal_Architecture.md](./Phase-1_Amazon-ECR-Deep-Dive/02_ECR_Internal_Architecture.md) | S3-backed storage, layer deduplication, manifest structure |
| [03_ECR_Authentication_Model.md](./Phase-1_Amazon-ECR-Deep-Dive/03_ECR_Authentication_Model.md) | Token auth, IAM, OIDC for CI/CD, cross-account pull |
| [04_ECR_Security_Concepts.md](./Phase-1_Amazon-ECR-Deep-Dive/04_ECR_Security_Concepts.md) | Image scanning (Basic vs Enhanced), encryption, policies |
| [05_ECR_Cost_Components.md](./Phase-1_Amazon-ECR-Deep-Dive/05_ECR_Cost_Components.md) | Storage, data transfer, lifecycle policies, cost optimization |
| [06_ECR_Advanced_Topics.md](./Phase-1_Amazon-ECR-Deep-Dive/06_ECR_Advanced_Topics.md) | Replication, multi-arch images, pull-through cache |

### Phase 2 — Amazon ECS Core
| File | Topic |
|------|-------|
| [01_What_Is_ECS.md](./Phase-2_Amazon-ECS-Core/01_What_Is_ECS.md) | ECS overview, ECS vs EKS, control plane architecture |
| [02_ECS_Core_Components.md](./Phase-2_Amazon-ECS-Core/02_ECS_Core_Components.md) | Cluster, Task Definition, Task, Service, Container Agent |
| [03_ECS_Launch_Types.md](./Phase-2_Amazon-ECS-Core/03_ECS_Launch_Types.md) | Fargate vs EC2, capacity providers, when to use which |
| [04_ECS_Internal_Working.md](./Phase-2_Amazon-ECS-Core/04_ECS_Internal_Working.md) | Desired state reconciliation, task lifecycle, deployment flow |
| [05_ECS_Scheduler_Deep_Dive.md](./Phase-2_Amazon-ECS-Core/05_ECS_Scheduler_Deep_Dive.md) | Placement strategies, constraints, AZ balancing |
| [06_ECS_Security_Model.md](./Phase-2_Amazon-ECS-Core/06_ECS_Security_Model.md) | Task Role vs Execution Role, IAM, Secrets, network security |
| [07_ECS_Networking_Modes.md](./Phase-2_Amazon-ECS-Core/07_ECS_Networking_Modes.md) | awsvpc, bridge, host mode, ENI, security groups |

### Phase 3 — ECS Advanced Architecture
| File | Topic |
|------|-------|
| [01_ECS_ALB_Integration.md](./Phase-3_ECS-Advanced-Architecture/01_ECS_ALB_Integration.md) | ALB target groups, health checks, path-based routing |
| [02_Service_Discovery_And_Connect.md](./Phase-3_ECS-Advanced-Architecture/02_Service_Discovery_And_Connect.md) | Cloud Map, ECS Service Connect, Envoy proxy |
| [03_Deployment_Strategies.md](./Phase-3_ECS-Advanced-Architecture/03_Deployment_Strategies.md) | Rolling update, Blue/Green (CodeDeploy), Canary |
| [04_Failure_Scenarios.md](./Phase-3_ECS-Advanced-Architecture/04_Failure_Scenarios.md) | Common failures, debugging, crash loop patterns |
| [05_Observability_Logs_Metrics_Tracing.md](./Phase-3_ECS-Advanced-Architecture/05_Observability_Logs_Metrics_Tracing.md) | CloudWatch, FireLens, Container Insights, X-Ray |
| [06_Scaling_Strategies.md](./Phase-3_ECS-Advanced-Architecture/06_Scaling_Strategies.md) | Target tracking, step scaling, KEDA, Fargate Spot |
| [07_Secrets_Handling.md](./Phase-3_ECS-Advanced-Architecture/07_Secrets_Handling.md) | Secrets Manager, Parameter Store, injection at task start |
| [08_Advanced_Architecture_Patterns.md](./Phase-3_ECS-Advanced-Architecture/08_Advanced_Architecture_Patterns.md) | Sidecar, event-driven, multi-region, batch |

### Phase 4 — Mastery and Interview Prep
| File | Topic |
|------|-------|
| [01_Cost_Optimization.md](./Phase-4_Mastery-And-Interview-Prep/01_Cost_Optimization.md) | Fargate Spot, right-sizing, Savings Plans, lifecycle policies |
| [02_Performance_Engineering.md](./Phase-4_Mastery-And-Interview-Prep/02_Performance_Engineering.md) | Image optimization, startup time, connection pooling |
| [03_CI_CD_Full_Pipeline.md](./Phase-4_Mastery-And-Interview-Prep/03_CI_CD_Full_Pipeline.md) | GitHub Actions / CodePipeline end-to-end pipeline |
| [04_Edge_Cases_And_Debugging.md](./Phase-4_Mastery-And-Interview-Prep/04_Edge_Cases_And_Debugging.md) | ENI limits, stuck deployments, ECS Exec, task failures |
| [05_Interview_Questions_Mastery.md](./Phase-4_Mastery-And-Interview-Prep/05_Interview_Questions_Mastery.md) | Curated Q&A, design questions, cheat sheets |

---

## Learning Philosophy

Each topic file follows this structure:

```
Concept Explanation     — What is it?
Internal Architecture   — How does it work internally?
Analogy                 — Real-world comparison for retention
Real-World Scenario     — Production use case
Hands-on                — CLI commands / config examples
Gotchas & Edge Cases    — What trips people up
Interview Angle         — How interviewers ask about this
```

---

## Quick Reference Links

- [ECR Auth Flow](./Phase-1_Amazon-ECR-Deep-Dive/03_ECR_Authentication_Model.md)
- [ECS IAM Roles — Task Role vs Execution Role](./Phase-2_Amazon-ECS-Core/06_ECS_Security_Model.md)
- [Fargate vs EC2 Launch Type](./Phase-2_Amazon-ECS-Core/03_ECS_Launch_Types.md)
- [Deployment Strategies](./Phase-3_ECS-Advanced-Architecture/03_Deployment_Strategies.md)
- [Interview Questions](./Phase-4_Mastery-And-Interview-Prep/05_Interview_Questions_Mastery.md)

---

## Mastery Checklist

After completing this curriculum, you should be able to:

- [ ] Explain Docker UnionFS and why image layers matter for build speed and storage
- [ ] Authenticate to ECR without static credentials in CI/CD pipelines (OIDC)
- [ ] Design a multi-account ECR replication strategy
- [ ] Explain the exact difference between Task Role and Execution Role
- [ ] Explain how ECS Fargate achieves ENI-per-task network isolation
- [ ] Design a zero-downtime Blue/Green deployment with CodeDeploy
- [ ] Debug a rolling update that gets stuck at 50%
- [ ] Calculate Fargate costs for a given workload
- [ ] Set up FireLens for centralized log routing to multiple destinations
- [ ] Design a multi-AZ ECS service with capacity providers and auto-scaling

---

*"The expert in anything was once a beginner." — Start with Phase 0.*
