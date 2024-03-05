# prereq
- cluster deployed

# set variables
```bash
export MY_CLUSTER_CONTEXT=gloo
```

# install argocd

```bash
kubectl create namespace argocd --context "${MY_CLUSTER_CONTEXT}"

until kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.9.5/manifests/install.yaml --context "${MY_CLUSTER_CONTEXT}" > /dev/null 2>&1; do sleep 2; done
```

check deployment status:
```bash
kubectl get pods -n argocd --context "${MY_CLUSTER_CONTEXT}"
```

Change password to `admin / solo.io`
```bash
# bcrypt(password)=$2a$10$79yaoOg9dL5MO8pn8hGqtO4xQDejSEVNWAGQR268JHLdrCw6UCYmy
# password: solo.io
kubectl --context "${MY_CLUSTER_CONTEXT}" -n argocd patch secret argocd-secret \
  -p '{"stringData": {
    "admin.password": "$2a$10$79yaoOg9dL5MO8pn8hGqtO4xQDejSEVNWAGQR268JHLdrCw6UCYmy",
    "admin.passwordMtime": "'$(date +%FT%T%Z)'"
  }}'
```

access argocd with port-forwarding:
At this point, we should be able to access our Argo CD server using port-forward at http://localhost:9999

```bash
kubectl port-forward svc/argocd-server -n argocd 9999:443 --context "${MY_CLUSTER_CONTEXT}"
```

# deploy an argo application

```bash
kubectl apply --context "${MY_CLUSTER_CONTEXT}" -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: workloads
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
  # -------------------------------------------------
    ## change me to your own public repo
    repoURL: https://github.com/ably77/shashank-poc
    ## change me to your own public repo
    path: workloads
  # -------------------------------------------------
    targetRevision: HEAD
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m0s
EOF
```