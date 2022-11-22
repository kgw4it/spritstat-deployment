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
## Sealed Secrets
https://github.com/bitnami-labs/sealed-secrets#public-key--certificate

### On the Jumpbox
```bash
# Install sealed-secrets to the cluster
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.19.2/controller.yaml

# Install kubeseal
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.19.2/kubeseal-0.19.2-linux-amd64.tar.gz
tar -xvzf kubeseal-0.19.2-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal

# Export public cert for use with kubeseal on the workstation
kubeseal --fetch-cert >mycert.pem
```

### On the workstation
```bash
# Install kubeseal
brew install kubeseal

# Download cert from jumpbox

# Create sealed secret from a secret definition
kubeseal --cert mycert.pem <mysecret.json >mysealedsecret.json
```

## Nginx Ingress Controller with Cert Manager
https://kubernetes.github.io/ingress-nginx/deploy/

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.5.1/deploy/static/provider/cloud/deploy.yaml
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.9.2/cert-manager.yaml
```

