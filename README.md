# Confluent for Kubernetes with Argo CD

A GitOps approach to deploying [Confluent for Kubernetes (CFK)](https://docs.confluent.io/operator/current/) using [Argo CD](https://argo-cd.readthedocs.io/en/stable/getting_started/) for continuous delivery. Push changes to this Git repo and Argo CD automatically syncs them to your Kubernetes cluster.

## Table of Contents

- [Confluent for Kubernetes with Argo CD](#confluent-for-kubernetes-with-argo-cd)
  - [Table of Contents](#table-of-contents)
  - [Prerequisites](#prerequisites)
  - [Repository Structure](#repository-structure)
  - [Steps to deploy Argo CD](#steps-to-deploy-argo-cd)
  - [Deploy Confluent Operator via ArgoCD](#deploy-confluent-operator-via-argocd)
    - [CFK version to Helm chart version mapping](#cfk-version-to-helm-chart-version-mapping)
  - [Steps to create CFK as an application via GitHub](#steps-to-create-cfk-as-an-application-via-github)
  - [Verifying the GitOps Workflow](#verifying-the-gitops-workflow)
  - [Useful Commands](#useful-commands)
  - [Deletion](#deletion)
  - [References](#references)

## Prerequisites

- A running Kubernetes cluster (e.g. Docker Desktop, minikube, EKS, GKE, etc.)
- [kubectl](https://kubernetes.io/docs/tasks/tools/) installed and configured
- [Helm 3](https://helm.sh/docs/intro/install/) installed
- [Homebrew](https://brew.sh/) (macOS, for installing the Argo CD CLI)

## Repository Structure

```
CFKCRsZookeeper/       # Confluent Platform with ZooKeeper
  kafka.yaml           #   Kafka broker cluster
  zookeeper.yaml       #   ZooKeeper ensemble
  connect.yaml         #   Kafka Connect workers
  schemaregistry.yaml  #   Schema Registry
  ksqldb.yaml          #   ksqlDB
  controlcenter.yaml   #   Confluent Control Center
  topic.yaml           #   KafkaTopic resource
CFKCRsKRaft/           # Confluent Platform with KRaft (no ZooKeeper)
  kraftcontroller.yaml #   KRaft controller quorum
  kafka.yaml           #   Kafka broker cluster
  connect.yaml         #   Kafka Connect workers
  schemaregistry.yaml  #   Schema Registry
  ksqldb.yaml          #   ksqlDB
  controlcenter.yaml   #   Confluent Control Center
  kafkarestproxy.yaml  #   Kafka REST Proxy
  topic.yaml           #   KafkaTopic resource
```

Point ArgoCD's `--path` at the directory matching your deployment mode — each file is a separate Confluent Platform component, making it easy to add, remove, or modify individual resources via Git.

## Steps to deploy Argo CD

1. Install Argo CD  
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side --force-conflicts
```
> **Note:** The `--server-side --force-conflicts` flags are required because the ApplicationSets CRD exceeds the 262144-byte annotation limit for client-side `kubectl apply`.
2. Download Argo CD CLI  
```bash
brew install argocd
```
3. Port Forwarding for accessing The Argo CD API Server  
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
4. Login Using The CLI
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

```bash
argocd login localhost:8080
```
Use user `admin` and password as shown above.  
Update password if needed:  
```bash
argocd account update-password
```
5.  Register A Cluster To Deploy Apps To

> **Note:** If you are deploying apps to the **same cluster** where ArgoCD is running, this step is not needed — the in-cluster target `https://kubernetes.default.svc` is already available by default. You can verify with `argocd cluster list`. This step is only required for **remote/external** clusters.

Option A — automatically add the current context:
```bash
argocd cluster add $(kubectl config current-context)
```
Option B — list all contexts and pick one manually:
```bash
kubectl config get-contexts -o name
argocd cluster add <context-name>
```


## Deploy Confluent Operator via ArgoCD

The CFK custom resources depend on the Confluent Operator, so it must be deployed first.

1. Create the `confluent` namespace:
```bash
kubectl create namespace confluent
```

2. Deploy the operator using one of the options below:

Option A — using the ArgoCD CLI:
```bash
argocd app create confluent-operator \
  --repo https://packages.confluent.io/helm \
  --helm-chart confluent-for-kubernetes \
  --revision 0.1514.1 \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace confluent \
  --sync-policy automated \
  --auto-prune
```

Option B — declarative YAML (full GitOps):

Create and apply an Application manifest:
```bash
kubectl apply -f argocd/confluent-operator.yaml
```

```yaml
# argocd/confluent-operator.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: confluent-operator
  namespace: argocd
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: confluent
  source:
    repoURL: https://packages.confluent.io/helm
    chart: confluent-for-kubernetes
    targetRevision: "0.1514.1"
  project: default
  syncPolicy:
    automated:
      prune: true
```

> **Important:** The `--revision` / `targetRevision` must use the **Helm chart version** (e.g. `0.1514.1`), not the CFK operator version (e.g. `3.2.0`). These are different versioning schemes. The mapping is shown below.

### CFK version to Helm chart version mapping

| CFK Operator Version | Helm Chart Version | App Version |
|---|---|---|
| 3.2.0 | 0.1514.1 | 3.2.0 |
| 3.1.1 | 0.1351.59 | 3.1.1 |
| 3.1.0 | 0.1351.24 | 3.1.0 |
| 3.0.3 | 0.1263.105 | 3.0.3 |
| 2.11.3 | 0.1193.70 | 2.11.3 |

For the full list and additional planning details, see the [Confluent for Kubernetes documentation](https://docs.confluent.io/operator/current/co-plan.html#co-long-image-tags).

> Check the latest chart version with: `helm search repo confluentinc/confluent-for-kubernetes --versions`

## Steps to create CFK as an application via GitHub  

1. Set the default namespace to argocd:
```bash
kubectl config set-context --current --namespace=argocd
```

2. Create an Application from a Git repository:
```bash
argocd app create <app-name> \
  --repo https://github.com/<your-org>/<your-repo>.git \
  --path <path> \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace confluent \
  --sync-policy automated \
  --auto-prune
```

For example:
```bash
argocd app create mycfk \
  --repo https://github.com/MosheBlumbergX/Argocd-CFK.git \
  --path CFKCRsKRaft \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace confluent \
  --sync-policy automated \
  --auto-prune
```

3. Open the Argo CD UI at `https://localhost:8080` to see the deployed applications.

You can also create applications through the [Argo CD web UI](https://argo-cd.readthedocs.io/en/stable/getting_started/#creating-apps-via-ui).

## Verifying the GitOps Workflow

To confirm that Argo CD is syncing changes from Git to the cluster:

1. Edit a resource in this repo, e.g. change `cleanup.policy` in `CFKCRsKRaft/topic.yaml` from `delete` to `compact` (or vice versa).
2. Commit and push the change.
3. Observe Argo CD automatically syncing — visible in the UI or via `argocd app get <app-name>`.

## Useful Commands

```bash
kubectl --namespace confluent get confluent
kubectl --namespace confluent get pods
kubectl --namespace confluent get topic
kubectl --namespace confluent exec -it kafka-2 -- kafka-topics --bootstrap-server localhost:9071 --describe --topic moshetopic
```


## Deletion 

To [delete](https://argo-cd.readthedocs.io/en/stable/user-guide/app_deletion/) the app: 

```bash
argocd app delete <app-name> --cascade -y 
```

## References

- [Confluent for Kubernetes](https://docs.confluent.io/operator/current/)
- [Argo CD](https://argo-cd.readthedocs.io/en/stable/getting_started/)
- [Confluent Platform](https://docs.confluent.io/platform/current/platform/overview.html)
