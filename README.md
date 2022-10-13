# gitops for local development of a cluster

## step 1 - brew install [k3d](https://k3d.io/v5.3.0)
```bash
brew install k3d
```

## step 2 - create a new local cluster
```bash

k3d cluster create kubefirstzzz --agents 3 --agents-memory 1024m --registry-create kubefirst-registryzz:64640
```
## step 3 helm install argocd
```bash
helm plugin install https://github.com/chartmuseum/helm-push

helm repo add argo https://argoproj.github.io/argo-helm
# helm repo update?

helm install argocd --namespace argocd --create-namespace -f ./argocd-values.yaml --version 4.10.2 argo/argo-cd
```

# access
### argocd
```bash
# password
kubectl -n argocd get secrets argocd-initial-admin-secret -ojson | jq -r .data.password | base64 -D
# port-forward
kubectl -n argocd port-forward svc/argocd-server 8080:80
```
user: admin   
password : $from-above-k8s-secret

### argo
```bash
# password
local kubeconfig will get you access
# port-forward
kubectl -n argo port-forward svc/argo-argo-workflows-server 2746:2746
```

### minio
```bash
# port-forward
kubectl -n minio port-forward svc/minio-console 9001:9001
```
user: k-ray   
password : feedkraystars   


# k3d registry
### find a good working directory for some garbage

```bash
git clone https://github.com/jarededwards/miller.git

cd miller 

docker build -t localhost:63630/miller:latest .
docker push localhost:63630/miller:latest
# getting ErrImgPull in kubernetes from Deployment, not successfully pulling from k3d registry, didn't look into it
```

# chartmuseum
```bash
helm repo add kubefirst-charts http://localhost:8181 --username k-ray --password feedkraystars
helm cm-push charts/miller kubefirst-charts
```

# argo workflows
this will submit an argo workflow that produces an artifact in minio
```bash
# por
kubectl -n argo port-forward svc/argo-argo-workflows-server 2746:2746
argo submit -f wf/artifacts.yaml 
```


---
# stop reading here

```bash
export AWS_ACCESS_KEY_ID=k-ray
export AWS_SECRET_ACCESS_KEY=feedkraystars

kubectl -n minio port-forward svc/minio 9000:9000

aws --endpoint-url http://localhost:9000 s3 ls




# registry
k3d registry create                                           
INFO[0000] Creating node 'k3d-registry'                 
INFO[0001] Pulling image 'docker.io/library/registry:2' 
INFO[0002] Successfully created registry 'k3d-registry' 
INFO[0002] Starting Node 'k3d-registry'                 
INFO[0002] Successfully created registry 'k3d-registry' 
# You can now use the registry like this (example):
# 1. create a new cluster that uses this registry
k3d cluster create --registry-use k3d-registry:64606

# 2. tag an existing local image to be pushed to the registry
docker tag lite:1 localhost:64606/lite:1

# 3. push that image to the registry
docker push localhost:64606/lite:1

# 4. run a pod that uses this image
kubectl run mynginx --image localhost:64606/lite:1


apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-test-registry
  labels:
    app: nginx-test-registry
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-test-registry
  template:
    metadata:
      labels:
        app: nginx-test-registry
    spec:
      containers:
      - name: nginx-test-registry
        image: k3d-registry.localhost:64606/lite:1
        ports:
        - containerPort: 80

apiVersion: v1
kind: Pod
metadata:
  name: hello-world
  labels:
    app: hello
spec:
  containers:
    - name: hello-world-container
      image: 
      command: ['sh', '-c', 'echo Hello World! && sleep 3600']


https://github.com/ContainerSolutions/trow/blob/main/docs/USER_GUIDE.md


apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: test-secrets
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  target:
    name: test-secrets
  secretStoreRef:
    kind: ClusterSecretStore
    name: vault-secrets-backend
  refreshInterval: 10s
  dataFrom:
    - extract:      
        key: /test
```

# big side step - argo admin kubectl boom
MESSAGE
Error (exit code 1): pods "testing-123-k8gmp-git-checkout-2935214226" is forbidden: User "system:serviceaccount:argo:default" cannot patch resource "pods" in API group "" in the namespace "argo"

kubectl create rolebinding default-admin --clusterrole=admin --serviceaccount=argo:default -n argo



apiVersion: apps/v1
kind: Deployment
metadata:
  name: miller-registry
  labels:
    app: miller-registry
spec:
  replicas: 1
  selector:
    matchLabels:
      app: miller-registry
  template:
    metadata:
      labels:
        app: miller-registry
    spec:
      containers:
      - name: miller-registry
        image: localhost:63630/miller:1.0.0
        ports:
        - containerPort: 80