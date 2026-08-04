# gha-composite-custom-aws-login

Custom GitHub Composite Action which handles the AWS Login accordingly to the org architecture.

## Description

This composite action performs AWS login by assuming an IAM role derived from the target environment and repository name. It optionally logs in to Amazon ECR as well.

## Prerequisites

The following environment variables must be set in the calling workflow:

| Variable | Description |
|---|---|
| `ENV_NAME` | Target environment name (e.g. `dev`, `staging`, `prod`) |
| `AWS_REGION` | AWS region to use |
| `REPO_NAME` | Repository name (used to construct the IAM role name) |

## Inputs

| Input | Description | Required | Default |
|---|---|---|---|
| `account_id` | AWS account ID for the login | Yes | — |
| `role_secret` | AWS IAM Role secret suffix for the login | Yes | — |
| `ecr_login` | Whether to perform ECR login (`true`/`false`) | No | `false` |
| `aws_access_key_id` | AWS access key ID used to assume the role (e.g. for LocalStack); if empty, OIDC is used | No | — |
| `aws_secret_access_key` | AWS secret access key used to assume the role (e.g. for LocalStack); if empty, OIDC is used | No | — |
| `sts_endpoint` | Custom STS endpoint (e.g. for LocalStack) | No | — |

## Outputs

| Output | Description |
|---|---|
| `tfvars` | TF var name for the target environment |
| `artifact_bucket` | Artifact bucket name for the target environment |
| `ecr_registry` | ECR registry URL (only populated when `ecr_login` is `true`) |

## Usage

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
      ENV_NAME: dev
      AWS_REGION: eu-west-1
      REPO_NAME: ${{ github.event.repository.name }}
    steps:
      - name: AWS Login
        id: aws_login
        uses: faccomichele/gha-composite-custom-aws-login@main
        with:
          account_id: ${{ secrets.AWS_ACCOUNT_ID }}
          role_secret: ${{ secrets.AWS_ROLE_SECRET }}
          ecr_login: 'true'

      - name: Use ECR registry
        run: echo "ECR registry is ${{ steps.aws_login.outputs.ecr_registry }}"
```

To use static credentials instead of OIDC (e.g. against LocalStack), provide the optional inputs:

```yaml
        with:
          account_id: ${{ secrets.AWS_ACCOUNT_ID }}
          role_secret: ${{ secrets.AWS_ROLE_SECRET }}
          aws_access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws_secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          sts_endpoint: http://localhost:4566
```

## How it works

1. **Validates environment variables** — ensures `ENV_NAME` and `AWS_REGION` are set; fails the step otherwise.
2. **Constructs the IAM role ARN** — builds the role ARN in the format:
   ```
   arn:aws:iam::<account_id>:role/gha-role-for-<repo_name>-<ENV_NAME>-GHARole-<role_secret>
   ```
   The repository name is truncated to 26 characters when constructing the role name.
3. **Assumes the IAM role** — uses [`aws-actions/configure-aws-credentials`](https://github.com/aws-actions/configure-aws-credentials) with a 1-hour session duration. If `aws_access_key_id` and `aws_secret_access_key` are provided, they are used as the source credentials to assume the role (with `sts-endpoint` applied when set); otherwise, the default OIDC-based flow is used.
4. **Logs in to Amazon ECR** *(optional)* — when `ecr_login` is `true`, uses [`aws-actions/amazon-ecr-login`](https://github.com/aws-actions/amazon-ecr-login) and exposes the registry URL via the `ecr_registry` output.

## License

[MIT](LICENSE)

