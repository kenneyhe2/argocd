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
kubectl create deploy nginx --image=nginx --dry-run=client -n apps -o yaml > nginx-deploy.yaml
kubectl create svc -n apps nodeport nginx --tcp=80:80 -o yaml --dry-run=client > nginx-svc.yaml
mv *yaml apps

#unittest
#cat d*yaml | kubectl create -f  -
#cat s*yaml | kubectl create -f  -
```

## WEBUI STEPS
```
https://www.linkedin.com/learning/kubernetes-gitops-with-argocd/deploying-an-application-using-argo-cd?autoSkip=true&autoplay=true&resume=false
```
