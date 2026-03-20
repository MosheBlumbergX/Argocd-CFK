# Argo cd CFK

This example leverage a gitop approach to deploying Confluent For Kubernetes with integration of Argo CD and CR, using [ArgosCD](https://argo-cd.readthedocs.io/en/stable/getting_started/) for CI/CD with minikube. 

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
    syncOptions:
      - CreateNamespace=true
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

1. Create An Application From A Git Repository

```bash
kubectl config set-context --current --namespace=argocd


argocd app create myconfluentfork8s \
--repo https://github.com/MosheBlumbergX/Argocd-CFK.git \
--path CFK-CRs --dest-server https://kubernetes.default.svc \
--dest-namespace confluent \
--sync-policy automated \
--auto-prune 
```

2. You can also leverage the [CLI](https://argo-cd.readthedocs.io/en/stable/getting_started/#creating-apps-via-ui)


### Some useful kubectl commands 

```bash
kubectl --namespace confluent get confluent
kubectl --namespace confluent get pods
kubectl --namespace confluent get topic
kubectl  --namespace confluent exec -it kafka-2  -- kafka-topics --bootstrap-server localhost:9071 --describe --topic moshetopic   
```


## Deletion 

To [delete](https://argo-cd.readthedocs.io/en/stable/user-guide/app_deletion/) the app: 

```bash
argocd app delete myconfluentfork8s --cascade -y 
```