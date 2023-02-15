## REQUIREMENTS
```
docker desktop
k3d
```

## Starting Docker
```
open /Applications/Docker.app/Contents/MacOS/Docker\ Desktop.app
k3d cluster start argocd-cluster
```

## STEPS TO SETUP K3S FOR UNITTESTING AND ARGOCD SERVER
```
cat > cluster-config << EOF
apiVersion: k3d.io/v1alpha2
kind: Simple
servers: 1
agents: 2
EOF
k3d cluster create argocd-cluster --config ./cluster-config.yaml
brew install kubectl
kubectl cluster-info
kubectl get ns
kubectl create ns argocd
kubectl -n argocd apply -f https://raw.githubusercontent.com/argoproj/argo-cd/master/manifests/install.yaml
argocd version
argocd version --client
kubectl patch -n argocd  svc argocd-server -p '{"spec": {"type":"NodePort"}}'
kubectl port-forward svc/argocd-server -n argocd 8080:443
# password
kubectl -n argocd get secrets argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d && echo

kubectl create ns apps
kubectl create deploy nginx --image=nginx --dry-run=client -n apps -o yaml > deploy.yaml
kubectl create svc -n apps  nodeport nginx --tcp=80:80 -o yaml --dry-run=client  > svc.yaml
#unittest
#cat d*yaml | kubectl create -f  -
#cat s*yaml | kubectl create -f  -
```

### ... APPS K3S
```
cat > cluster-config-apps << EOF
apiVersion: k3d.io/v1alpha2
kind: Simple
servers: 1
agents: 2
EOF

k3d cluster create apps-cluster --config ./cluster-config-apps.yaml
argocd login localhost:8080 
argocd cluster add k3d-apps-cluster --name k3d-staging-apps-cluster --server localhost:8080 --insecure
```

### ... private docker
```
docker login -u kenneyhe
...
docker push kenneyhe/traefik:latest
```

### STEPS TO ADD RASPI
    ```
    argocd login localhost:8080 --insecure --username admin --password `kubectl -n argocd get secrets argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d && echo`

    argocd cluster add k3s-rpi --kubeconfig ~/.kube/raspimaster -y
    # remove if UI is buggy
    # argocd cluster rm k3d-staging-apps-cluster -y
    # create project ie:
    argocd proj list
    argocd proj create raspi -s https://github.com/kenneyhe2/argocd.git -d https://192.168.86.26:6443,apps
    # create application to sync up ie:
    argocd app list

    argocd app create nginx2 --repo https://github.com/kenneyhe2/argocd.git --path app --dest-namespace apps --dest-server https://192.168.86.26:6443 --project raspi
    # must create namespace apps in infrastructure code
    # delete argocd app delete nginx3 -y
    ```
