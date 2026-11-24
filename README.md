# DevOps Lab 4: Advanced Deployment Strategies & Enterprise DevOps

**Student:** Meenakshi | **Roll No.:** 2301730200 | **Email:** meenakship0805@gmail.com
**Course:** DevOps and Automation (ENSP461)
**Environment:** Ubuntu 24.04 LTS, Docker, Minikube (2 nodes), kubectl, Helm, Jenkins, GitHub Actions, Argo CD, HashiCorp Vault, PostgreSQL


## Layout

```
app/                 
tests/unit/          
tests/integration/   
k8s/base/            
k8s/blue-green/      
k8s/canary/          
k8s/dr/              
k8s/vault/           
gitops/lab4-app/     
argocd/              
scripts/             
Jenkinsfile          
.github/workflows/   
```

## Setup (once)

```bash
# tools
sudo apt install -y docker.io git python3-venv
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
sudo snap install kubectl --classic
sudo snap install helm --classic

# 2-node cluster, ingress, images, DB secret, PostgreSQL
bash scripts/setup_cluster.sh
```

The manifests pin PostgreSQL to the node named `minikube` (the default profile name).

## Tests

```bash
python3 -m venv .venv && . .venv/bin/activate
pip install -r requirements-dev.txt
pytest tests/unit -v --cov=app --cov-fail-under=80

export DB_PASSWORD=$(openssl rand -hex 16)
docker compose up -d db
DB_HOST=localhost pytest -m integration -v
```

## Q1 Disaster recovery

```bash
bash scripts/dr_drill.sh          # backup -> write more -> delete namespace -> restore -> RPO/RTO report
bash scripts/dr_backup.sh         # manual off-cluster backup to ./backups
bash scripts/dr_restore.sh        # restore newest backup (checksum verified)
# failover: terminal 1
bash scripts/traffic_probe.sh http://$(minikube ip):30400
# terminal 2
bash scripts/failover_demo.sh     # drains node 2; PDB keeps >= 2 pods serving
```

Targets: **RPO ≤ 15 min** (backup CronJob interval), **RTO ≤ 10 min**.

## Q3 Blue-green

```bash
bash scripts/traffic_probe.sh http://$(minikube ip):30410     # terminal 1
bash scripts/blue_green_deploy.sh lab4-app:v2                 # terminal 2
bash scripts/blue_green_deploy.sh --rollback
bash scripts/blue_green_deploy.sh --status
```

## Q4 GitOps

```bash
bash scripts/gitops_setup.sh      # install Argo CD, register the app, print UI login
bash scripts/gitops_demo.sh       # manual drift is reverted; then change Git and push
```

## Q6 Secrets

```bash
bash scripts/create_db_secret.sh  # random password -> Kubernetes Secret (never in Git)
bash scripts/vault_setup.sh       # Vault (dev mode), K8s auth, policy, injected secret
```

Pipelines: Jenkins uses the `lab4-db-password` credential (masked in logs); GitHub Actions generates a random masked password per run. gitleaks scans every push.

## Q7 Canary

```bash
bash scripts/traffic_probe.sh http://$(minikube ip) lab4.local   # terminal 1 (optional)
bash scripts/canary_rollout.sh          # 10% -> 25% -> 50% -> 100%, then promote
bash scripts/canary_rollout.sh --bad    # 30% errors in v2 -> automatic rollback
```

Rollback trigger: canary error rate above 5% (`MAX_ERROR_PCT`). Per-step metrics are saved to `canary-metrics.csv`.
