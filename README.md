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
