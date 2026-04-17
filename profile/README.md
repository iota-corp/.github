## Architecture

**iota-corp** ships a detection engine (**iota**), how it runs in Kubernetes (**iota-deployments**), AWS/IaC around it (**iota-infra**), and optional GitHub Actions runners (**iota-arc**). The AWS account used in public material is a **test / lab** account.

### Repositories

| Repository | Purpose |
|------------|---------|
| [**iota**](https://github.com/iota-corp/iota) | Application: Go runtime, Python rules, `Dockerfile`, CI — builds and publishes **container images**. |
| [**iota-deployments**](https://github.com/iota-corp/iota-deployments) | **GitOps**: canonical Kustomize **base**, per-environment **overlays**, **Argo CD** apps — what runs **where** in each cluster. |
| [**iota-infra**](https://github.com/iota-corp/iota-infra) | **Terraform**: homelab IAM (`eks-lab/`), queues, buckets, optional **EKS-oriented modules** (`modules/iota-eks-helm`, `modules/iota-system-iam`), **homelab-arc** Helm for ARC bootstrap. |
| [**iota-arc**](https://github.com/iota-corp/iota-arc) | **Actions Runner Controller**: manifests and Helm values for **self-hosted runners** (e.g. k3d homelab); complements `iota-infra/homelab-arc`. |

### How they interact

```mermaid
flowchart TB
  subgraph dev["Developer / CI"]
    GH["GitHub Actions<br/>(org workflows)"]
    ARC["Self-hosted runners<br/>(iota-arc on k3d)"]
  end

  subgraph iota_repo["Repo: iota"]
    CODE["App + rules + Dockerfile"]
    CI["CI: test, build, push image"]
    CODE --> CI
  end

  subgraph deploy_repo["Repo: iota-deployments"]
    BASE["Kustomize base"]
    CLUSTERS["clusters/* overlays"]
    ARGO["Argo CD Applications"]
    BASE --> CLUSTERS
    CLUSTERS --> ARGO
  end

  subgraph infra_repo["Repo: iota-infra"]
    EKSLAB["Terraform eks-lab<br/>(IAM, queues, buckets)"]
    MODS["modules/<br/>iota-eks-helm, iota-system-iam"]
    HARC["homelab-arc<br/>(cert-manager + ARC Helm)"]
  end

  subgraph arc_repo["Repo: iota-arc"]
    RD["RunnerDeployment manifests<br/>+ helm values"]
  end

  subgraph aws["AWS (test / lab account)"]
    IAM["IAM users / roles"]
    SQS["SQS"]
    S3["S3 CloudTrail / lake"]
  end

  subgraph k8s_main["Cluster: workload (e.g. k3s / EKS)"]
    POD["iota pods"]
    SEC["Secrets e.g. iota-aws-*"]
  end

  subgraph k8s_arc["Cluster: ARC (e.g. k3d homelab)"]
    RUN["Runner pods"]
  end

  GH --> iota_repo
  ARC --> iota_repo
  ARC --> arc_repo

  CI -->|"container image"| REG["Container registry"]
  REG -->|"image: tag"| CLUSTERS

  CI -.->|"optional: bump tags<br/>(PAT)"| deploy_repo

  EKSLAB --> IAM
  EKSLAB --> SQS
  EKSLAB --> S3

  EKSLAB -.->|"outputs: keys, ARNs, URLs"| SEC
  CLUSTERS -->|"kubectl / GitOps sync"| POD
  SEC --> POD
  POD --> SQS
  POD --> S3

  HARC --> k8s_arc
  RD --> k8s_arc
  RUN --> GH

  MODS -.->|"optional EKS installs"| k8s_main
