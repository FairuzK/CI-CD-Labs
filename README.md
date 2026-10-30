# 🚀 CI/CD – Cheat Sheet

CI/CD stands for **Continuous Integration** and **Continuous Delivery (or Continuous Deployment)**. It is a software development methodology that uses automation to build, test, and release code faster and more reliably.

**GitHub Actions** pipelines that build, test, scan, and deploy containers to **ECR** and **ECS Fargate**.

## Pipeline flow
`push / pull request` → lint and test → build Docker image → scan → push to ECR → deploy to ECS

## What's in this repo
| Path | Purpose |
|---|---|
| `.github/workflows/ci.yml` | Runs on pull requests: lint, test, build, scan |
| `.github/workflows/deploy.yml` | Runs on merge to `main`: push to ECR, deploy to ECS |

## Best practices
- Authenticate to AWS with **OIDC and an IAM role**, not long-lived access keys
- Store anything sensitive in GitHub **Secrets**; never echo it in logs
- Pin third-party actions to a version (ideally a commit SHA)
- Keep jobs small and fast; cache dependencies; fail early on lint or tests
- Tag images with the commit SHA (`${{ github.sha }}`) for traceable releases
- Protect `main`: require pull request review and passing checks
- Use **environments** with approval gates for production
- Set `concurrency` so overlapping deploys cancel safely
- Scan images (Trivy) and run `terraform fmt` / `validate` / `plan` in CI

## Example: build, push to ECR, deploy to ECS
```yaml
name: Deploy
on:
  push:
    branches: [main]

permissions:
  id-token: write   # required for OIDC
  contents: read

concurrency: deploy-prod

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<account-id>:role/github-deploy
          aws-region: us-east-1

      - id: ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push
        env:
          IMAGE: ${{ steps.ecr.outputs.registry }}/app:${{ github.sha }}
        run: |
          docker build -t $IMAGE .
          docker push $IMAGE
          echo "image=$IMAGE" >> $GITHUB_ENV

      - id: task
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: app
          image: ${{ env.image }}

      - uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task.outputs.task-definition }}
          service: app
          cluster: my-cluster
          wait-for-service-stability: true
```
