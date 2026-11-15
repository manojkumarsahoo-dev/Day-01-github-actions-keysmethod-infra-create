# Terraform Deploy with GitHub Actions

This repo provisions AWS infrastructure (a VPC and a subnet) with Terraform and deploys it through a GitHub Actions workflow with a **manual trigger** and a **manual approval gate** before anything is applied.

## Project structure

```
.
├── .github/
│   └── workflows/
│       └── terraform.yml   # the workflow
├── main.tf                 # aws_vpc + aws_subnet resources
├── variables.tf            # cidr, name_tags, subnet_cidr, subnet_tag
└── provider.tf             # AWS provider (region)
```

## What gets created

| Resource | Variable | Default |
|---|---|---|
| `aws_vpc.name` | `cidr`, `name_tags` | `10.0.0.0/16`, `dev-vpc` |
| `aws_subnet.dev` | `subnet_cidr`, `subnet_tag` | `10.0.1.0/24`, `dev-subnet` |

The subnet is attached to the VPC through `vpc_id = aws_vpc.name.id`.

## Key GitHub Actions concepts used

### 1. Manual trigger: `workflow_dispatch`
```yaml
on:
  workflow_dispatch:
```
The workflow only runs when you click **Actions → Terraform Deploy → Run workflow**. The `push` trigger is commented out, so commits never deploy automatically. To deploy on every push to `main`, uncomment:
```yaml
  push:
    branches: [ main ]
```

### 2. Secrets for AWS credentials
```yaml
env:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  AWS_DEFAULT_REGION: us-east-1
```
Add the secrets under **Settings → Secrets and variables → Actions → New repository secret**. Never commit keys to the repo. Terraform's AWS provider reads these environment variables automatically.

### 3. Two jobs: `plan` and `apply`
Splitting the pipeline lets you review the plan before changing real infrastructure.
`needs: plan` makes `apply` wait until `plan` succeeds.

### 4. Plan steps
| Step | Command | Purpose |
|---|---|---|
| Checkout | `actions/checkout` | Pulls the repo code onto the runner |
| Setup Terraform | `hashicorp/setup-terraform` | Installs the Terraform CLI |
| Init | `terraform init` | Downloads providers, sets up backend |
| Validate | `terraform validate` | Catches syntax and config errors early |
| Plan | `terraform plan -out=tfplan` | Saves the exact planned changes to a file |

### 5. Passing the plan between jobs with artifacts
Each job runs on a fresh machine, so files are not shared automatically.
```yaml
# plan job
- uses: actions/upload-artifact@v7
  with:
    name: tfplan
    path: tfplan

# apply job
- uses: actions/download-artifact@v7
  with:
    name: tfplan
```
This guarantees `apply` executes **exactly** the plan that was reviewed.

### 6. Manual approval with `environment`
```yaml
apply:
  environment: production
```
To make this actually pause for approval:
1. Go to **Settings → Environments → New environment** and name it `production`.
2. Enable **Required reviewers** and add yourself or your team.

The `apply` job now waits until a reviewer approves it.

### 7. Apply
```yaml
run: terraform apply -auto-approve tfplan
```
`-auto-approve` is safe here because the approval already happened at the environment gate, and a saved plan file is applied as is.

## How to run

1. Add the AWS secrets to the repo.
2. Create the `production` environment with required reviewers.
3. Open **Actions → Terraform Deploy → Run workflow**.
4. Wait for `plan` to finish and review its logs.
5. Approve the `apply` job when prompted.

## Important improvements to make

1. **Use a remote backend (critical).** Without one, Terraform state is stored on the runner and is lost when the job ends. The next run would try to create the same resources again, and you could never destroy them. Use an S3 bucket (with DynamoDB locking or S3 native locking):
   ```hcl
   terraform {
     backend "s3" {
       bucket = "my-tf-state-bucket"
       key    = "dev/terraform.tfstate"
       region = "us-east-1"
     }
   }
   ```
2. **Prefer OIDC over long-lived access keys.** Use `aws-actions/configure-aws-credentials` with an IAM role so no static keys are stored in GitHub.
3. **Treat the `tfplan` artifact as sensitive.** Plan files can contain secret values. Set a short `retention-days` on the upload step.
4. **Pin versions.** Set `terraform_version` in `setup-terraform` and a `required_providers` version constraint so runs are reproducible. Check that the action versions you use (`checkout`, `upload-artifact`, `download-artifact`, `setup-terraform`) exist and are current.
5. **Add `terraform fmt -check`** before validate to enforce formatting.
6. **Add a destroy workflow** (separate `workflow_dispatch`) so you can clean up and avoid AWS charges.

## Tips

- Override defaults without editing code: `terraform plan -var="cidr=10.1.0.0/16"` or use a `terraform.tfvars` file.
- Rename `aws_vpc.name` to something meaningful such as `aws_vpc.main`, since `name` is a confusing resource label.
