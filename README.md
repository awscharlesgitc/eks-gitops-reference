# AWS EKS + GitHub Actions + ECR + Helm + Argo CD reference implementation

This is a starter reference repository for a Python Flask service deployed to Amazon EKS using GitHub Actions for CI and Argo CD for GitOps CD.

## Flow

Developer push -> GitHub Actions -> tests -> Trivy scan -> Docker build -> ECR push -> update Helm image tag in Git -> Argo CD detects Git change -> Helm render -> EKS rollout.

Supporting services: Terraform-provisioned VPC/EKS/ECR, AWS Secrets Manager + External Secrets Operator (optional integration manifests), AWS Load Balancer Controller (optional ingress), CloudWatch/Prometheus/Grafana (operational extensions).

## Repository layout

- `app/`: sample Flask application and tests
- `Dockerfile`, `.dockerignore`: container build
- `.github/workflows/ci.yml`: CI, image scan, ECR push, GitOps update
- `helm/sample-app/`: Helm chart
- `argocd/application.yaml`: Argo CD Application
- `terraform/`: VPC, EKS and ECR starter
- `k8s/`: optional ExternalSecret example

## Prerequisites

1. AWS account and permissions to create VPC, EKS, ECR, IAM and related resources.
2. Terraform >= 1.6, AWS CLI v2, kubectl, Helm.
3. GitHub repositories:
   - application repository containing this project
   - GitOps repository containing `helm/sample-app/` and `argocd/`
4. Configure GitHub repository variables:
   - `AWS_REGION` (for example `ap-south-1`)
   - `AWS_ACCOUNT_ID`
   - `ECR_REPOSITORY` (for example `sample-python-app`)
   - `AWS_ROLE_ARN` (IAM role trusted for this GitHub repository)
   - `GITOPS_REPOSITORY` (for example `ORG/sample-app-gitops`)
   - `GITOPS_BRANCH` (for example `main`)
5. For cross-repository GitOps updates, configure `GITOPS_TOKEN` as a fine-grained PAT with Contents: Read and write on the GitOps repository. For a same-repository layout, adapt the workflow to use `GITHUB_TOKEN`.

## Build and test locally

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r app/requirements.txt
pytest -q
docker build -t sample-python-app:local .
docker run --rm -p 8080:8080 sample-python-app:local
curl http://localhost:8080/health
```

## Provision AWS infrastructure

```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars
terraform init
terraform plan
terraform apply
```

The Terraform starter uses upstream community modules. Review module versions, costs, account quotas, CIDRs, IAM permissions, endpoint access and security requirements before applying. NAT Gateways, EKS control plane and worker nodes incur charges.

After provisioning, configure kubectl using the cluster name and region from Terraform outputs:

```bash
aws eks update-kubeconfig --region ap-south-1 --name sample-eks
kubectl get nodes
```

Create the ECR repository name in Terraform to match `ECR_REPOSITORY`.

## Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

For a quick lab only, port-forward the UI:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Retrieve the initial password:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

Change the initial password and configure SSO/RBAC for shared environments. Register the GitOps repository credentials in Argo CD if it is private. Then apply:

```bash
kubectl apply -f argocd/application.yaml
```

The example Application assumes the GitOps repository URL and branch are updated in that file.

## GitHub Actions OIDC

Create an IAM OIDC provider for `https://token.actions.githubusercontent.com` if it does not already exist. The IAM role trust policy must constrain `sub` to the exact repository and branch, for example:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"},
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
        "token.actions.githubusercontent.com:sub": "repo:ORG/sample-python-app:ref:refs/heads/main"
      }
    }
  }]
}
```

Attach only the permissions required to push to the named ECR repository. Do not grant broad administrator permissions. Set the role ARN in GitHub variable `AWS_ROLE_ARN`.

## GitOps repository

The workflow checks out the GitOps repository, changes `helm/sample-app/values.yaml` image tag and pushes a commit. Ensure that path exists in the GitOps repository. Copy `helm/sample-app/` and `argocd/` there, or adjust the workflow paths.

## Important production hardening

- Use private EKS API access where feasible, restricted admin access, private worker nodes and controlled egress.
- Use immutable image tags and consider ECR tag immutability and lifecycle policies.
- Add branch protection, required reviews, environment approvals and signed commits/artifacts.
- Add SAST, dependency/SBOM scanning, policy-as-code and image signing/verification.
- Use a dedicated IAM role for GitHub OIDC, EKS node role, Argo CD and External Secrets Operator.
- Configure backups, encryption, audit logging, alerting, resource requests/limits, Pod Security Standards and NetworkPolicies.
- Use separate AWS accounts/clusters or isolated environments for dev/test/prod.
- This sample is a learning baseline, not a security-certified production blueprint.
