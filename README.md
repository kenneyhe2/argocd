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

## STEPS TO SETUP K3S FOR UNITTESTING AND ARGOCD SERVER.. only >4.0.0 will work. ref: https://brettmostert.medium.com/k3d-kubernetes-istio-service-mesh-286a7ba3a64f
```

curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | TAG=v4.4.8 sh

k3d cluster create argocd-cluster --servers 1 --agents 1 --port 9080:80@loadbalancer --port 9443:443@loadbalancer --api-port 6443 --k3s-server-arg '--no-deploy=traefik'

export ISTIO_VERSION=1.11.5
export PATH="/Users/kenneyhe/ArgoCD/public/istio-1.11.5/bin:$PATH"
curl -L https://istio.io/downloadIstio | sh -
# istioctl x uninstall --purge -y
# change from 2G to 1G k3s kubectl -n istio-system edit deploy istiod
kubectl patch deployment istiod -n istio-system --type merge --patch "$(echo '{
  "spec": {
    "template": {
      "spec": {
        "containers": [
          {
            "image": "docker.io/istio/pilot:1.11.5",
            "name": "discovery",
            "resources": {
              "requests": {
                "cpu": "500m",
                "memory": "1Gi"
              }
            }
          }
        ]
      }
    }
  }
}')"


kubectl patch -n istio-system deploy istiod -p '{"spec": {"containers": {"resources": {"requests": {"Memory":"2Gi"}}}}}'
istioctl version
istioctl profile dump default
istioctl install --set profile=default -y --readiness-timeout 10m0s
kubectl label namespace default istio-injection=enabled
cat > deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echoserver-v1
  labels:
    app: echoserver
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echoserver
      version: v1
  template:
    metadata:
      labels:
        app: echoserver
        version: v1
    spec:
      containers:
      - name: echoserver
        image: gcr.io/google_containers/echoserver:1.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
EOF

# TBD
cat > deployment1.yaml << EOF
EOF
cat > service.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: echoserver
  labels:
    app: echoserver
    service: echoserver
spec:
  selector:
    app: echoserver
  ports:
  - port: 80
    targetPort: 8080
    name: http
EOF

#TBD
cat > service1.yaml << EOF
EOF

# TBD
cat > ingress.yaml << EOF
EOF
kubectl apply -f ./deployment.yaml
kubectl apply -f ./service.yaml
kubectl apply -f ./deployment1.yaml
kubectl apply -f ./service1.yaml
kubectl apply -f ./ingress.yaml
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
### delete k3d cluster delete --all
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
    # edit playbooks/k3s-main.yml for ip addr all in https://github.com/HI-INTL/argocd-projects/tree/main
    ansible-playbook -vvv -e 'ansible_python_interpreter="/usr/bin/env python"' playbook/main.yml

    kubectl port-forward svc/argocd-server -n argocd 8080:443
    argocd login localhost:8080 --insecure --username admin --password `kubectl -n argocd get secrets argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d && echo`
    
    # if out of sync as redundancy
    argocd app sync pihole --prune

    # need to enable force namespace creation
    # must create namespace apps in infrastructure code
    # delete argocd app delete nginx3 -y

    # issues to address.. need a cronjob to scrub all unknown and stuck pods for SRE or get alerts
    #
    # kr='kubectl --kubeconfig ~/.kube/raspimaster2'
    # for p in $(kr -n kube-system get pods | grep Terminating | awk '{print $1}'); do kr -n kube-system  delete pod $p --grace-period=0 --force;done
    ```

### Modules to create individual apps with own branch
    ```
    # if not using github private runner
    kr create secret generic doppler-token-secret \
      --namespace doppler-operator-system \
      --from-literal=serviceToken=$(doppler -p github -c dev configs tokens create doppler-kubernetes-operator --plain --copy)

    argocd proj create raspi-dev-doppler -s git@github.com:kenneyhe2/argocd-projects.git  -d https://192.168.86.26:6443,doppler-operator-system
    argocd app create raspi-doppler --repo git@github.com:kenneyhe2/argocd-projects.git  --path apps/doppler-kubernetes-operator --dest-namespace doppler-operator-system --dest-server https://192.168.86.26:6443 --project raspi-dev-doppler  --revision feat-doppler-crd  --sync-policy auto
    ```
