# spritstat-deployment
Deployment repository for the [SPRITSTAT](https://github.com/tgamauf/spritstat) providing configuration for Kubernetes/Argo CD.

# Preparing the Tanzu jumpbox
```bash
# Install helm
#  https://helm.sh/docs/intro/install/
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm

# Enable privileged access for the whole cluster
#  https://docs.vmware.com/en/VMware-vSphere/7.0/vmware-vsphere-with-tanzu/GUID-4CCDBB85-2770-4FB8-BF0E-5146B45C9543.html
kubectl create clusterrolebinding default-tkg-admin-privileged-binding --clusterrole=psp:vmware-system-privileged --group=system:authenticated
```

# Preparing Argo CD
https://argo-cd.readthedocs.io/en/stable/getting_started/

## On the Jumpbox

```bash
# Setup ArgoCD
kubectl create namespace argocd
helm repo add argo https://argoproj.github.io/argo-helm
helm install --namespace argocd argo argo/argo-cd

# Do port forwarding to allow access to the UI and CLI from the workstation
while true; do kubectl port-forward svc/argocd-server -n argocd 8080:443; done

# Get password for user `admin`
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

## On the Workstation

The IP address/hostname in the `forward_ports.sh` script must be set to the IP/hostname
of the jumpbox. After that all relevant ports are forwarded to the jumpbox in order to
allow local access to the non-public services running there (Argo CD, Grafana,
Prometheus, ...).

```bash
# Start port forwarding to the jumpbox
./forward_ports.sh
```

# Set up the infrastructure project in Argo CD

In order to access the Argo CD web interface the `forward_ports.sh` script must be running
on the workstation and port `443` must be forwarded from the service to the jumpbox:
```bash
while true; do kubectl port-forward svc/argocd-server -n argocd 8080:443; done
```

After that the Argo CD web interfac can be accessed via `localhost`: `http://localhost:8080`.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: infra-root
spec:
  destination:
    name: ''
    namespace: argocd
    server: 'https://kubernetes.default.svc'
  source:
    path: k8s/infrastructure
    repoURL: 'git@github.com:tgamauf/spritstat-deployment.git'
    targetRevision: test
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

After all services are deployed, install *kubeseal* on the jumpbox and retrieve the 
Sealed Secrets public key:
### On the Jumpbox
```bash

# Install kubeseal
version=0.19.2
wget "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${version}/kubeseal-${version}-linux-amd64.tar.gz"
tar -xvzf "kubeseal-${version}-linux-amd64.tar.gz" kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal

# Export public cert for use with kubeseal on the workstation
kubeseal --controller-namespace infrastructure --controller-name sealed-secrets --fetch-cert >sealedsecrets.pem
```

### On the workstation

Download the Sealed Secrets public key from the jumpbox and store it as `resources/sealedsecrets.pem`.
After that install `kubeseal` on the workstation:
```bash
# Install kubeseal
brew install kubeseal
```

# Sealed secrets

All secrets used in the project are stored as sealed secrets in the repository. As the
secrets can only be unsealed on a cluster that possesses the correct, automatically
generated key, the secrets have to be updated if a new instance of Sealed Secrets is
installed in the cluster (or if the cluster is changed). This makes it necessary to
recreate the sealed secrets.

Secrets can be created on the jumpbox by using Kubernetes secrets - either from an
environment file or a complete file:
```bash
# Create a secret from an environment files (will provide key/value pairs)
kubectl create secret generic my-secret --dry-run=client --from-env-file=my_secret.env -o yaml >my_secret.yaml

# Create a secret from a normal file (will encrypt the whole file)
kubectl create secret generic my-secret --dry-run=client --from-file=my_secret.text -o yaml >my_secret.yaml
```

The secrets created on the jumpbox need then be converted to sealed secrets using the
stored Sealed Secrets public key:
```bash
kubeseal --cert resources/sealedsecrets.pem <my_secret.yaml >path/to/app/my_sealedsecret.yaml
```

Make sure to add the correct namespace to the sealed secrets as this isn't done automatically
due to the `--dry-run=client` flag of the `kubectl create secret` command (prevents that
the secret is added to the cluster).

Secrets for the following services/apps need to be recreated:
1. Grafana administrator credentials
2. Alertmanager config (contains SMTP credentials)
3. PostgreSQL credentials
4. Spritstat secrets

In the following the structure of each sealed secret is highlighted.

## Grafana credentials

- Storage location of the sealed secret: `k8s/prometheus-stack/templates/grafana-sealedsecret.yaml`
- Namespace: `monitoring`
- Secret file contains key-value pairs:
```
admin-user=<admin username>
admin-password=<admin password>
```

## Alertmanager config

The Prometeus Alertmanager config needs to be stored in a Sealed Secret as a whole. This
is a real pain as 1. the config won't be visible in the repo, and 2. the service is
extremely sensitive to whitespace in the config. **If the config is incorrect the 
Alertmanager pod simply won't start without any overt error and the service will be 
shown as healthy despite that.**

Use this yaml config with the whitespace as shown below and only replace the secret parts,
or it might break ...

- Storage location of the sealed secret: `k8s/prometheus-stack/templates/alertmanager-sealedsecret.yaml`
- Namespace: `monitoring`
- Content:
```yaml
global:
  resolve_timeout: 5m
  smtp_from: root@sprit.thga.at
  smtp_smarthost: <smtp_server>:587
  smtp_auth_username: <username>
  smtp_auth_password: <password>
  smtp_auth_identity: "Spritstat Alertmanager" 
route:
  receiver: "null"
  group_by:
  - alertname
  routes:
  - receiver: "null"
    matchers:
    - alertname =~ "InfoInhibitor|Watchdog"
  - receiver: email
    match:
      alertname: spritstat_errors
  - receiver: email
    match:
      alertname: spritstat_not_reachable
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
inhibit_rules:
- target_matchers:
  - severity =~ warning|info
  source_matchers:
  - severity = critical
  equal:
  - namespace
  - alertname
- target_matchers:
  - severity = info
  source_matchers:
  - severity = warning
  equal:
  - namespace
  - alertname
- target_matchers:
  - severity = info
  source_matchers:
  - alertname = InfoInhibitor
  equal:
  - namespace
receivers:
- name: "null"
- name: email
  email_configs:
  - send_resolved: true
    to: <target_email>
```

## PostgreSQL credentials

- Storage location of the sealed secret: `k8s/postgresql/templates/sealedsecret.yaml`
- Namespace: `default`
- Secret file contains key-value pairs:
```
password=<spritstat user password>
postgres-password=<admin password>
```

## Spritstat secrets

Secrets of the SPRITSTAT app: https://github.com/tgamauf/spritstat

- Storage location of the sealed secret: `k8s/spritstat/sealedsecret.yaml`
- Namespace: `default`
- Secrets are key-value pairs (mounted into the container as env variables):
```
DJANGO_POSTGRES_PASSWORD=<Postgres spritstat user password>
DJANGO_EMAIL_PASSWORD=<SMTP server user password>
DJANGO_SECRET=<Django server secret>
GOOGLE_MAPS_API_KEY=<Google Maps API key>
```

# Monitoring access

To check the logs, metrics, and alerts of the system it is possible to connect to
Grafana and Prometheus running on the cluster via the jumpbox. In order to be able to
access it the `forward_ports.sh` script must be running on the workstation.

## On the jumpbox
```bash
# Forward port of Grafana pod to the jumpbox
while true; do kubectl port-forward -n monitoring svc/prometeus-stack-grafana 8000:80; done

# Forward port of the Prometheus pod to the jumpbox
while true; do kubectl port-forward -n monitoring svc/prometheus-stack-prometheus 9090; done
```

The services can then be accessed via `localhost` in the browser:
- Grafana: `http://localhost:8080`
- Prometheus: `http://localhost:9090`
- 