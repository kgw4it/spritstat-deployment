# spritstat-deployment
Deployment repository for the [SPRITSTAT](https://github.com/tgamauf/spritstat) providing configuration for Kubernetes/Argo CD.

# Setup Tanzu jumpbox
```bash
# Enable privileged access
kubectl create clusterrolebinding default-tkg-admin-privileged-binding --clusterrole=psp:vmware-system-privileged --group=system:authenticated
```

# Preparing ArgoCD
https://argo-cd.readthedocs.io/en/stable/getting_started/

## On the Jumpbox
```bash
# Setup ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Do port forwarding to allow access to the UI and CLI from the workstation
while true; do kubectl port-forward svc/argocd-server -n argocd 8080:443; done

# Get password for user `admin`
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

## On the Workstation
```bash
# Set the external cluster IP in the port forwarding script "forward_argocde.sh"

# Start port forwarding to the jumpbox
./forward_argocd.sh
```

# Setup cluster services
# Nginx Ingress Controller
https://kubernetes.github.io/ingress-nginx/deploy/

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.5.1/deploy/static/provider/cloud/deploy.yaml
```

