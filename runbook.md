# Runbook — bringing the stack up and tearing it down

Steps to stand up the REMNI backend on EKS from scratch, and to tear it back
down. The `network/` module is left running between sessions (a VPC and subnets
cost nothing); only the cluster and the database are created and destroyed each
time.

## Prerequisites each session

- Your public IP is in `var.allowed_ip_for_kubectl` (it changes on a dynamic
  connection). Check with `curl ifconfig.me` and update the variable if needed.
- AWS CLI is authenticated as the principal that has the EKS access entry
  (`aws sts get-caller-identity` should show that user).

## 1. Infrastructure (Terraform)

Order matters — `eks` and `rds` read `network` through remote state, so network
must exist first. (If `network/` is already up from a previous session, skip it.)

```bash
cd network      && terraform apply    # only if not already up
cd ../module05/rds  && terraform apply # RDS — ~10 min
cd ../module04/eks  && terraform apply # cluster + nodes + OIDC provider + ESO role — ~10-15 min
```

Point kubectl at the new cluster:

```bash
aws eks update-kubeconfig --region eu-west-2 --name eks_cluster_example
kubectl get nodes            # wait for nodes Ready
```

## 2. Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd -w   # wait until all Running
```

Access the UI (keep this terminal open — the forward lives while it runs):

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

UI at https://localhost:8080 (accept the self-signed cert warning).
Admin password:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```

Login `admin`, password from the command above.

## 3. External Secrets Operator (IRSA)

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm upgrade --install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --set installCRDs=true \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="arn:aws:iam::474013238842:role/external-secrets-operator"

kubectl get pods -n external-secrets -w   # wait until Running
```

Confirm the service account carries the role annotation (this is the IRSA link):

```bash
kubectl get sa external-secrets -n external-secrets -o yaml | grep role-arn
```

## 4. Deploy the app (Argo Application)

Apply the Argo Applications that point at the `website-deploy` repo — one for
the backend chart, one for the external-secrets manifests (ClusterSecretStore +
ExternalSecret). Order: secrets first, so the k8s Secret exists before the pod
needs it.

```bash
kubectl apply -f <argo-application-external-secrets>.yaml
kubectl apply -f <argo-application-backend>.yaml
```

Then Sync in the Argo UI (or set them to auto-sync).

## 5. Verify

```bash
# ESO synced the secret out of Secrets Manager
kubectl get externalsecret -n remni-website          # STATUS SecretSynced, READY True
kubectl get secret rds-password -n remni-website

# the app pod is up and connected to RDS
kubectl get pods -n remni-website -w                 # Ready 1/1
kubectl logs -n remni-website <pod>                  # gunicorn workers hold, no DB errors
```

If the ExternalSecret shows `SecretSyncedError`, the usual causes are: the
ClusterSecretStore doesn't exist yet, `remoteRef.key` no longer matches the
secret name (the name carries the RDS instance UUID and changes if the database
was recreated), or the IRSA role can't read that secret ARN.

## 5. Install LoadBalancerControlles

```
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=eks_cluster_example \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="arn:aws:iam::474013238842:role/aws-load-balancer-controller" \
  --set region=eu-west-2 \
  --set vpcId=vpc-05663472ab0ff866b
```

## 6. Tear down (in reverse)

Argo and ESO live in the cluster, so destroying the cluster removes them — no
separate cleanup needed. Leave `network/` running.

```bash
cd module04/eks && terraform destroy   # cluster, nodes, OIDC provider, ESO role
cd ../module05/rds && terraform destroy # database
```

Confirm nothing costly is left:

```bash
aws eks list-clusters
aws rds describe-db-instances --query 'DBInstances[].DBInstanceIdentifier'
```

Both empty → nothing is billing.

## Known rough edges

- **Pod IP budget.** Argo (~7 pods) + ESO (~3) + system pods fill a small node
  fast. The node group runs 2 nodes for this reason; if pods sit `Pending`, that
  is the ENI/pod-IP limit, not memory.
- **RDS secret name changes on recreation.** `remoteRef.key` in the
  ExternalSecret carries the instance UUID. If the database is recreated, update
  the key (or move to a fixed secret name — parked decision).
- **App namespace is `remni-website`** — SecretStore target and the pod must
  share it.