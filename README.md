# Lab 2 — Push Docker Image to Amazon ECR

A minimal Spring Boot REST application containerized and automatically deployed to Amazon ECR via GitHub Actions using OIDC authentication (no long-lived AWS credentials).

---

## Repository structure

```
.
├── src/                          # Java Spring Boot application
├── pom.xml                       # Maven build descriptor
├── Dockerfile                    # Multi-stage, non-root build
├── .dockerignore
├── infra/
│   └── ecr-oidc.yaml             # CloudFormation: ECR + OIDC + IAM role
└── .github/
    └── workflows/
        └── build-push-ecr.yml    # CI/CD pipeline
```

---

## One-time setup

### 1. Deploy the CloudFormation stack

This creates the ECR repository, GitHub OIDC provider, and the least-privilege IAM role in one command.

```bash
aws cloudformation deploy \
  --stack-name ecr-lab \
  --template-file infra/ecr-oidc.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
      GitHubOrg=<your-github-username> \
      GitHubRepo=<your-repo-name> \
      ECRRepositoryName=hello-app \
  --region <your-aws-region>
```

After deployment, grab the outputs:

```bash
aws cloudformation describe-stacks \
  --stack-name ecr-lab \
  --query "Stacks[0].Outputs" \
  --output table
```

Note down:
- **GitHubActionsRoleArn** → `arn:aws:iam::<account>:role/GitHubActions-ECRPush-<repo>`
- **ECRRepositoryUri** → `<account>.dkr.ecr.<region>.amazonaws.com/hello-app`

### 2. Add GitHub Actions variables

In your GitHub repository go to **Settings → Secrets and variables → Actions → Variables** and add:

| Variable name    | Value                                    |
|------------------|------------------------------------------|
| `AWS_ROLE_ARN`   | ARN from CloudFormation output above     |
| `AWS_REGION`     | e.g. `us-east-1`                         |
| `ECR_REPOSITORY` | `hello-app`                              |

> **No secrets needed.** Authentication is handled entirely via OIDC.

### 3. Push to trigger the pipeline

```bash
git add .
git commit -m "initial commit"
git push origin main
```

The workflow will:
1. Check out the code
2. Authenticate to AWS via OIDC (no keys stored anywhere)
3. Build the Docker image using the multi-stage Dockerfile
4. Tag it `bonheurmahoro_helloapp`
5. Push it to ECR

---

## Authentication design

```
GitHub Actions runner
        │
        │  OIDC JWT (contains repo claim)
        ▼
token.actions.githubusercontent.com
        │
        │  sts:AssumeRoleWithWebIdentity
        ▼
AWS IAM (validates JWT + condition on repo)
        │
        │  short-lived session credentials
        ▼
Amazon ECR  ←  docker push
```

The IAM role trust policy restricts issuance to **your repository only** via:

```json
"StringLike": {
  "token.actions.githubusercontent.com:sub": "repo:<org>/<repo>:*"
}
```

The role's permission policy grants only the seven ECR actions needed for a push, scoped to the single repository ARN.

---

## Security checklist

- [x] No AWS access keys or secrets stored in GitHub
- [x] OIDC token scoped to a single repository
- [x] IAM role grants least-privilege (push only, single repo)
- [x] Container runs as non-root user
- [x] Minimal `eclipse-temurin:21-jre-alpine` runtime image
- [x] ECR image scanning on push enabled (via CloudFormation)
- [x] ECR lifecycle policy retains only the 10 most recent images
- [x] `.dockerignore` prevents leaking build artifacts or secrets into the image

---

## Local development

```bash
# Build
docker build -t hello-app:local .

# Run
docker run -p 8080:8080 hello-app:local

# Test
curl http://localhost:8080/hello
```
