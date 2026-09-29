# Pinger app → AWS EKS deployment guide

Order matters. Do these top to bottom. `<AWS_ACCOUNT_ID>` appears in several
files — replace it everywhere before applying (`grep -rl AWS_ACCOUNT_ID .`).

---

## 0. Prereqs (once)

```bash
aws --version && eksctl version && kubectl version --client && helm version && docker --version
aws sts get-caller-identity   # confirms your AWS creds work, prints your account ID
```

---

## 1. Create the ECR repos

One repo per image (frontend, details, pinger-v1, pinger-v2):

```bash
for repo in learnathon-frontend learnathon-details learnathon-pinger-v1 learnathon-pinger-v2; do
  aws ecr create-repository --repository-name $repo --region ap-south-1
done
```

---

## 2. Build and push the first images manually (bootstrap)

The CI/CD pipeline only builds on future pushes — you need images in ECR
*before* the cluster can pull them the first time.

```bash
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com

# repeat for each service (frontend, details, pinger/v1, pinger/v2)
docker build -t <AWS_ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/learnathon-frontend:v1 app/frontend
docker push <AWS_ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/learnathon-frontend:v1
```

---

## 3. Create the EKS cluster

```bash
eksctl create cluster -f cluster/eksctl-cluster.yaml
```

Takes ~15-20 min. `withOIDC: true` in that file matters — it's what lets the
ALB controller and (later) GitHub Actions authenticate to AWS without static
keys.

---

## 4. Install metrics-server (required for HPA)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl top nodes   # once this returns numbers (may take a minute), metrics-server is working
```

---

## 5. Install the AWS Load Balancer Controller (required for the Ingress/ALB)

```bash
# IAM policy + service account for the controller
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json

eksctl create iamserviceaccount \
  --cluster pinger-eks \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

helm repo add eks https://aws.github.io/eks-charts && helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=pinger-eks \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller   # should show 1/1 or 2/2 ready
```

---

## 6. Deploy the app

```bash
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/01-frontend.yaml
kubectl apply -f k8s/02-details.yaml
kubectl apply -f k8s/03-pinger-v1.yaml
kubectl apply -f k8s/04-pinger-v2.yaml
kubectl apply -f k8s/05-hpa.yaml
kubectl apply -f k8s/06-ingress.yaml
```

```bash
kubectl get pods -n pinger-app -w        # wait for all pods Running/Ready
kubectl get ingress -n pinger-app        # ADDRESS column = your ALB DNS name (takes a few min to populate)
```

Open the ALB address in a browser once it resolves — that's your frontend, live.

---

## 7. Install Prometheus + Grafana

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts && helm repo update
kubectl create namespace monitoring
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring -f monitoring/prometheus-grafana-values.yaml
```

```bash
kubectl get pods -n monitoring -w   # wait for everything Running
```

Access Grafana (kept as ClusterIP on purpose — no second ALB, no extra cost):

```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3001:80
# open http://localhost:3001 — user: admin / password: changeme123 (from the values file)
```

What you'll see out of the box: node CPU/memory, pod resource usage, cluster
health dashboards (kube-prometheus-stack ships these by default). You will
**not** see per-request metrics (latency, request rate) for the pinger/details
apps themselves yet — that needs `prom-client` added to the Node code and a
`/metrics` endpoint + a ServiceMonitor pointed at it. That's a good next step
once this is up, but it's app-code work, not infra — see "Next steps" below.

---

## 8. Wire up push-based CI/CD (GitHub Actions → ECR → EKS)

### 8a. Create the IAM role GitHub Actions will assume (OIDC, no stored keys)

```bash
# one-time: register GitHub's OIDC provider with your AWS account (skip if already done for another repo)
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

Create `trust-policy.json` (replace `YOUR_GH_ORG/YOUR_REPO`):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": { "token.actions.githubusercontent.com:sub": "repo:YOUR_GH_ORG/YOUR_REPO:ref:refs/heads/main" }
    }
  }]
}
```

```bash
aws iam create-role --role-name github-actions-pinger-deploy --assume-role-policy-document file://trust-policy.json
aws iam attach-role-policy --role-name github-actions-pinger-deploy --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser

# let the role talk to the EKS cluster too
eksctl create iamidentitymapping --cluster pinger-eks \
  --arn arn:aws:iam::<AWS_ACCOUNT_ID>:role/github-actions-pinger-deploy \
  --group system:masters --username github-actions
```

### 8b. Push the pipeline

The workflow is already at `.github/workflows/ci-cd.yml`. It:
1. detects which service folder(s) changed in the push (`dorny/paths-filter`)
2. builds + pushes only those images to ECR, tagged with the git SHA
3. runs `kubectl set image` on just the matching Deployment(s) and waits for rollout

```bash
git add .
git commit -m "Add EKS manifests and CI/CD pipeline"
git push origin main
```

Watch it run under the repo's **Actions** tab. Any future push that touches
`app/frontend/**`, `app/details/**`, `app/pinger/v1/**`, or `app/pinger/v2/**`
auto-builds and redeploys just that service.

---

## Switching frontend to v2

Right now `frontend`'s `PINGER_BASE_URL` env var points at `pinger-v1-service`;
`pinger-v2` is deployed and reachable but gets no user traffic. To cut over:

```bash
kubectl set env deployment/frontend PINGER_BASE_URL=http://pinger-v2-service:3000 -n pinger-app
kubectl rollout status deployment/frontend -n pinger-app
```

(To make this permanent, also update the value in `k8s/01-frontend.yaml` so
the next `kubectl apply` doesn't revert it.)

---

## Cost notes (you're on limited credits)

- ALB (~$0.0225/hr + traffic) + EKS control plane ($0.10/hr) + 2x t3.medium
  nodes are your main recurring costs — call it ~$4-5/day while it's running.
- Nothing here provisions EBS volumes, NAT gateways, or RDS.
- **Tear down when not actively using it**, in this order:

```bash
kubectl delete -f k8s/06-ingress.yaml          # ALB first — it's billed even if idle
helm uninstall monitoring -n monitoring
helm uninstall aws-load-balancer-controller -n kube-system
eksctl delete cluster -f cluster/eksctl-cluster.yaml
```

---

## Next steps (once this is stable)

- Add `prom-client` to the Node apps, expose `/metrics`, add a ServiceMonitor
  → real Golden Signal dashboards in Grafana instead of just node metrics.
- Add a readiness gate / canary step so the pipeline doesn't roll out a broken
  image to all replicas at once.
- Revisit Envoy for actual v1/v2 traffic-splitting instead of an all-or-nothing
  env var cutover.https://github.com/RylVishal/pipeline_testings/actions/runs/36415357105/job/108915041740#logs