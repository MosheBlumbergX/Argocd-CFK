# Argo cd CFK

This example leverage a gitop approach to deploying Confluent For Kubernetes with integration of Argo CD and CR, using [ArgosCD](https://argo-cd.readthedocs.io/en/stable/getting_started/) for CI/CD with minikube. 

## Steps to deploy Argo CD

1. Install Argo CD  
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
2. Download Argo CD CLI  
```
brew install argocd
```
3. Port Forwarding for accessing The Argo CD API Server  
```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
4. Login Using The CLI
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

```
argocd login localhost:8080
```
Use user admin and password as shown above.  
Update password if needed:  
```
argocd account update-password
```
5.  Register A Cluster To Deploy Apps To
```
kubectl config get-contexts -o name
argocd cluster add minikube # or any other that you wanted 
```


## Steps to create CFK as an application via GitHub  

1. Create An Application From A Git Repository

```
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

```
kubectl --namespace confluent get confluent
kubectl --namespace confluent get pods
kubectl --namespace confluent get topic
kubectl  --namespace confluent exec -it kafka-2  -- kafka-topics --bootstrap-server localhost:9071 --describe --topic moshetopic   
```


## Deletion 

To [delete](https://argo-cd.readthedocs.io/en/stable/user-guide/app_deletion/) the app: 

```
argocd app delete myconfluentfork8s --cascade -y 
```