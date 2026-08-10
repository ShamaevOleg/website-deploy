# Runbook — bringing the stack up and tearing it down

Steps to stand up the REMNI application on EKS from scratch, and to tear it back
down. The foundational network and the ACM certificate are left running between
sessions (a VPC, subnets, and a certificate cost nothing); only the cluster and
the database are created and destroyed each time.

## Prerequisites each session

- Your public IP is in `var.allowed_ip_for_kubectl` (it changes on a dynamic
  connection). Check with `curl ifconfig.me` and update the variable if needed.
- AWS CLI is authenticated as the principal that holds the EKS access entry
  (`aws sts get-caller-identity` should show that user).

## 1. Infrastructure (Terraform)

Order matters — the cluster and the database read the network through remote
state, so the network must exist first. (It's normally already up from a
previous session; the same is true for the ACM certificate.)

```bash
cd 00-foundation-network        && terraform apply   # only if not already up
cd ../04-managed-database       && terraform apply   # RDS — ~10 min
cd ../03-containers-ecr-eks/eks && terraform apply   # cluster, nodes, cluster OIDC provider, ESO + LB-controller IAM roles — ~10-15 min
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
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="<ESO role ARN — output of the eks module>"

kubectl get pods -n external-secrets -w   # wait until Running
```

Confirm the service account carries the role annotation (this is the IRSA link):

```bash
kubectl get sa external-secrets -n external-secrets -o yaml | grep role-arn
```

## 4. AWS Load Balancer Controller (IRSA)

Needed for the Ingress to create an ALB.

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=eks_cluster_example \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="<LB-controller role ARN — output of the eks module>" \
  --set region=eu-west-2 \
  --set vpcId=<vpc_id — output of the network module>

kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller -w
```

## 5. Deploy the app (Argo Applications)

Apply the Argo Applications that point at this repo. Order: secrets first (so the
Kubernetes Secret exists before the pods need it), then the workloads.

```bash
kubectl apply -f apps/external-secrets.yaml
kubectl apply -f apps/redis.yaml
kubectl apply -f apps/backend.yaml
kubectl apply -f apps/frontend.yaml
kubectl apply -f apps/ingress.yaml
```

Then Sync in the Argo UI (or set the Applications to auto-sync).

## 6. Point DNS at the new load balancer

The ALB DNS name changes every time the cluster is recreated, so the DNS record
must be updated after each stand-up.

```bash
kubectl get ingress -n remni-website    # copy the ADDRESS (the ALB DNS name)
```

In Cloudflare, update the records for `belts-website.com` and
`www.belts-website.com` to point at that ALB address (DNS-only / grey cloud, so
the AWS ACM certificate on the ALB is what the browser sees).

## 7. Verify

```bash
# ESO synced the secret out of Secrets Manager
kubectl get externalsecret -n remni-website          # STATUS SecretSynced, READY True
kubectl get secret rds-password -n remni-website

# the app pods are up
kubectl get pods -n remni-website -w                 # backend, frontend, redis Ready

# the app answers over HTTPS on the domain
# open https://belts-website.com — browser padlock shows the AWS-issued cert
```

If the ExternalSecret shows `SecretSyncedError`, the usual causes are: the
ClusterSecretStore doesn't exist yet, `remoteRef.key` no longer matches the
secret name (the name carries the RDS instance UUID and changes if the database
was recreated), or the IRSA role can't read that secret ARN.

## 8. Tear down (in reverse)

Argo CD, ESO, and the LB controller live in the cluster, so destroying the
cluster removes them — no separate cleanup needed. Leave the network and the ACM
certificate.

```bash
cd 03-containers-ecr-eks/eks && terraform destroy   # cluster, nodes, IAM roles
cd ../../04-managed-database  && terraform destroy   # database (all data is lost — skip_final_snapshot)
```

Confirm nothing costly is left:

```bash
aws eks list-clusters
aws rds describe-db-instances --query 'DBInstances[].DBInstanceIdentifier'
```

Both empty → nothing is billing.

## Known rough edges

- **Pod IP budget.** Argo CD (~7 pods) + ESO (~3) + the LB controller + system
  pods fill a small node fast. The node group runs 2 nodes for this reason; if
  pods sit `Pending`, that is the ENI/pod-IP limit, not memory.
- **RDS secret name changes on recreation.** `remoteRef.key` in the
  ExternalSecret carries the instance UUID. If the database is recreated, update
  the key (or move to a fixed secret name — parked decision).
- **The ALB DNS name changes on every stand-up**, so the Cloudflare records must
  be re-pointed each time (step 6). ExternalDNS would automate this.
