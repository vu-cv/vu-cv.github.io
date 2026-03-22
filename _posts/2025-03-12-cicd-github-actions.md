---
layout: article
title: CI/CD với GitHub Actions – Tự động test & deploy
tags: [cicd, github-actions, devops, deployment, automation]
---
GitHub Actions cho phép tự động hóa toàn bộ quy trình từ khi push code đến khi deploy lên production. Bài này hướng dẫn xây dựng CI/CD pipeline hoàn chỉnh cho ứng dụng NestJS.

## 1. GitHub Actions là gì?

- **Workflow**: Tập hợp các jobs, được trigger bởi events (push, PR, schedule...)
- **Job**: Tập hợp các steps chạy trên cùng một runner
- **Step**: Một lệnh shell hoặc một Action có sẵn
- **Runner**: Máy ảo chạy job (ubuntu-latest, macos-latest, windows-latest)

## 2. Pipeline CI — Test & Lint khi push

Tạo file `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      # Khởi động MongoDB cho integration test
      mongodb:
        image: mongo:7
        ports:
          - 27017:27017

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run lint
        run: npm run lint

      - name: Run unit tests
        run: npm run test -- --coverage
        env:
          NODE_ENV: test

      - name: Run e2e tests
        run: npm run test:e2e
        env:
          NODE_ENV: test
          MONGODB_URI: mongodb://localhost:27017/test
          REDIS_HOST: localhost

      - name: Upload coverage report
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
```

## 3. Pipeline CD — Deploy lên server

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]
  workflow_dispatch:  # Cho phép trigger thủ công

jobs:
  deploy:
    runs-on: ubuntu-latest
    needs: []  # Đặt 'test' ở đây nếu muốn deploy chỉ khi CI pass

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Build
        run: |
          npm ci
          npm run build

      - name: Build Docker image
        run: |
          docker build -t ${{ secrets.REGISTRY_URL }}/my-api:${{ github.sha }} .
          docker tag ${{ secrets.REGISTRY_URL }}/my-api:${{ github.sha }} \
                     ${{ secrets.REGISTRY_URL }}/my-api:latest

      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ secrets.REGISTRY_URL }}
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}

      - name: Push Docker image
        run: |
          docker push ${{ secrets.REGISTRY_URL }}/my-api:${{ github.sha }}
          docker push ${{ secrets.REGISTRY_URL }}/my-api:latest

      - name: Deploy to server via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            docker pull ${{ secrets.REGISTRY_URL }}/my-api:latest
            docker stop my-api || true
            docker rm my-api || true
            docker run -d \
              --name my-api \
              --restart unless-stopped \
              -p 3000:3000 \
              --env-file /home/deploy/.env \
              ${{ secrets.REGISTRY_URL }}/my-api:latest
            docker image prune -f

      - name: Notify Slack on success
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          payload: '{"text":"✅ Deploy thành công: ${{ github.sha }}"}'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}

      - name: Notify Slack on failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: '{"text":"❌ Deploy thất bại: ${{ github.sha }}"}'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

## 4. Deploy lên AWS ECS

```yaml
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-1

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Push to ECR
        run: |
          IMAGE_URI=${{ steps.login-ecr.outputs.registry }}/my-api:${{ github.sha }}
          docker build -t $IMAGE_URI .
          docker push $IMAGE_URI

      - name: Update ECS Service
        run: |
          aws ecs update-service \
            --cluster my-cluster \
            --service my-api-service \
            --force-new-deployment
```

## 5. Cache dependencies để build nhanh hơn

```yaml
      - name: Cache node_modules
        uses: actions/cache@v4
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-

      - name: Cache Docker layers
        uses: actions/cache@v4
        with:
          path: /tmp/.buildx-cache
          key: ${{ runner.os }}-buildx-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-buildx-
```

## 6. Environment Secrets

Thêm secrets vào GitHub repo: **Settings → Secrets and variables → Actions**

```
SERVER_HOST         = 1.2.3.4
SERVER_USER         = ubuntu
SERVER_SSH_KEY      = -----BEGIN RSA PRIVATE KEY-----...
REGISTRY_URL        = ghcr.io/username
REGISTRY_USERNAME   = username
REGISTRY_PASSWORD   = ghp_xxx
SLACK_WEBHOOK       = https://hooks.slack.com/...
AWS_ACCESS_KEY_ID   = AKIA...
AWS_SECRET_ACCESS_KEY = xxx
```

## 7. Bảo vệ branch main

Trong **Settings → Branches → Branch protection rules**:

- ✅ Require status checks to pass (CI phải xanh)
- ✅ Require pull request reviews (1-2 reviewer)
- ✅ Dismiss stale reviews
- ✅ Require linear history

## 8. Kết luận

GitHub Actions CI/CD giúp tự động hóa hoàn toàn quy trình phát triển:

- **CI**: Tự động lint, test mỗi khi push — phát hiện bug sớm
- **CD**: Tự động build Docker image và deploy khi merge vào `main`
- **Secrets**: Bảo mật credential, không hardcode trong code
- **Branch protection**: Bắt buộc CI pass trước khi merge

Một pipeline tốt giúp team tự tin deploy nhiều lần mỗi ngày thay vì sợ hãi mỗi lần release.
