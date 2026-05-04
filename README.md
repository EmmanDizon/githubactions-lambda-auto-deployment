# Lambda + Parameter Store + GitHub OIDC Deployment Setup

## Goal
Set up secure automated deployment from GitHub Actions to AWS Lambda without storing AWS credentials.

---

## Architecture Flow

### Deployment Flow
GitHub Actions → OIDC → IAM Role → AWS STS → Lambda Update

### Runtime Flow
Lambda → Execution Role → SSM Parameter Store → App Config

---

## 1. Setup GitHub OIDC Provider (AWS)

Go to:
IAM → Identity providers

Check if exists:
token.actions.githubusercontent.com

If not, create:

- Provider type: OpenID Connect
- Provider URL: https://token.actions.githubusercontent.com
- Audience: sts.amazonaws.com

---

## 2. Create IAM Role for GitHub Actions

Go to:
IAM → Roles → Create role

Select:
- Trusted entity type: Web identity
- Identity provider: token.actions.githubusercontent.com
- Audience: sts.amazonaws.com

Role name example:
gha-inventory-deploy-prod

---

## 3. Configure Trust Policy

Edit trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:OWNER/REPO:*"
        }
      }
    }
  ]
}
```

---

## 4. Add Lambda Deploy Permissions

Attach inline policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "lambda:UpdateFunctionCode",
        "lambda:GetFunction",
        "lambda:GetFunctionConfiguration"
      ],
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:FUNCTION_NAME"
    }
  ]
}
```

---

## 5. Setup GitHub Environment Variables

GitHub → Settings → Environments → production

Add:

- LAMBDA_ROLE_ARN
- AWS_REGION
- LAMBDA_FUNCTION_NAME

---

## 6. GitHub Actions Workflow (Example)

```yaml
permissions:
  id-token: write
  contents: read

- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ vars.LAMBDA_ROLE_ARN }}
    aws-region: ${{ vars.AWS_REGION }}
```

---

## 7. Setup Parameter Store

Go to:
Systems Manager → Parameter Store

Create parameters like:

/inventory-app/DB_HOST
/inventory-app/DB_PASSWORD

---

## 8. Lambda Execution Role Permissions

Attach inline policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters",
        "ssm:GetParametersByPath"
      ],
      "Effect": "Allow",
      "Resource": "arn:aws:ssm:REGION:ACCOUNT_ID:parameter/inventory-app/*"
    }
  ]
}
```

If using SecureString:

```json
{
  "Effect": "Allow",
  "Action": "kms:Decrypt",
  "Resource": "*"
}
```

---

## 9. Lambda Code Example (Node.js)

```ts
import { SSMClient, GetParameterCommand } from "@aws-sdk/client-ssm";

const client = new SSMClient({ region: process.env.AWS_REGION });

export async function getParam(name: string) {
  const res = await client.send(
    new GetParameterCommand({
      Name: name,
      WithDecryption: true
    })
  );
  return res.Parameter?.Value;
}
```

---

## Notes

- No AWS credentials stored in GitHub
- Uses short-lived tokens (OIDC)
- Enforces least privilege access
- Centralized config via Parameter Store

