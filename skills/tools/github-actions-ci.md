---
name: github-actions-ci
description: GitHub Actions CI/CD 实战：Spring Boot 微服务多环境构建、测试与自动部署
tags: [tools, ci-cd, github-actions, devops, spring-boot]
---

## 概述

GitHub Actions 是 GitHub 内置的 CI/CD 平台，与 Spring Boot 微服务项目搭配可实现自动化构建、测试、容器镜像推送和多环境部署。本文覆盖从基础到进阶的完整 Pipeline 配置。

## 1. 基础 Maven 构建流水线

```yaml
# .github/workflows/ci.yml
name: CI Build

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        java: [17, 21]

    steps:
      - uses: actions/checkout@v4

      - name: Setup JDK ${{ matrix.java }}
        uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java }}
          distribution: temurin
          cache: maven

      - name: Build & Test
        run: ./mvnw verify -B -DskipTests=false

      - name: Upload Test Report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results-${{ matrix.java }}
          path: target/surefire-reports/
```

## 2. Docker 镜像构建与推送

```yaml
# .github/workflows/docker-build.yml
name: Build & Push Docker Image

on:
  push:
    branches: [main]
    paths:
      - 'src/**'
      - 'pom.xml'
      - 'Dockerfile'

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: 21
          distribution: temurin
          cache: maven

      - name: Build JAR
        run: ./mvnw clean package -DskipTests -B

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build & Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

## 3. 多环境部署（Kubernetes）

```yaml
# .github/workflows/deploy.yml
name: Deploy to K8s

on:
  workflow_run:
    workflows: ['Build & Push Docker Image']
    types:
      - completed

jobs:
  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v3

      - name: Configure Kubeconfig
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBE_CONFIG_STAGING }}" | base64 -d > $HOME/.kube/config

      - name: Deploy to Staging
        run: |
          kubectl set image deployment/${{ vars.APP_NAME }} \
            app=${{ env.REGISTRY }}/${{ github.repository }}:${{ github.sha }} \
            -n staging
          kubectl rollout status deployment/${{ vars.APP_NAME }} -n staging

  deploy-production:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.example.com
    needs: deploy-staging  # 仅当 staging 成功后才部署生产
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Production
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBE_CONFIG_PROD }}" | base64 -d > $HOME/.kube/config
          kubectl set image deployment/${{ vars.APP_NAME }} \
            app=${{ env.REGISTRY }}/${{ github.repository }}:${{ github.sha }} \
            -n production --record
          kubectl rollout status deployment/${{ vars.APP_NAME }} -n production
```

## 4. 缓存策略优化

```yaml
- name: Cache Maven dependencies
  uses: actions/cache@v4
  with:
    path: ~/.m2/repository
    key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
    restore-keys: |
      ${{ runner.os }}-maven-

- name: Cache Docker layers
  uses: actions/cache@v4
  with:
    path: /tmp/.buildx-cache
    key: ${{ runner.os }}-buildx-${{ github.sha }}
    restore-keys: |
      ${{ runner.os }}-buildx-
```

## 5. 并发与矩阵构建

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        module: [user-service, order-service, payment-service]
        java: [17, 21]
      # 失败快速取消
      fail-fast: false
      # 最多2个并发，避免资源耗尽
      max-parallel: 2
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java }}
          distribution: temurin
          cache: maven

      - name: Test ${{ matrix.module }}
        run: ./mvnw test -pl ${{ matrix.module }} -B

  # 合并测试结果
  coverage:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: 21
          distribution: temurin
          cache: maven
      - name: Aggregate coverage
        run: ./mvnw verify -Pcoverage -B
      - name: Upload coverage
        uses: codecov/codecov-action@v4
```

## 6. 基础设施即代码（Terraform）

```yaml
# .github/workflows/infra.yml
name: Terraform Infrastructure

on:
  push:
    branches: [main]
    paths:
      - 'terraform/**'

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ./terraform

    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.7.0

      - name: Terraform Format Check
        run: terraform fmt -check

      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        run: terraform plan -out=tfplan

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main'
        run: terraform apply -auto-approve tfplan
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

## 注意事项

### 性能优化
- **依赖缓存**：Maven/Gradle 的 `.m2` 目录用 `actions/cache`，设置 `restore-keys` 实现部分命中
- **条件触发**：用 `paths` 和 `paths-ignore` 减少不必要的构建（如仅修改 README 时跳过）
- **并发控制**：`max-parallel` 防止 runner 资源被打满

### 安全最佳实践
- **Secrets**：敏感信息（密码、Token、Kubeconfig）用 GitHub Secrets，不在 yaml 中硬编码
- **OIDC**：使用 OpenID Connect 替代长期密钥（如 AWS/Google Cloud 认证）
- **最小权限**：`permissions` 块限制 GITHUB_TOKEN 作用域，禁止 `write-all`

### 常见坑点
1. **Docker BuildX 缓存**：用 `type=gha` 模式（GitHub Actions Cache），不用本地 `--cache-from`，否则缓存不共享
2. **环境变量作用域**：`env` 块的变量不会自动在 `run` 中展开 — 用 `${{ env.VAR }}` 而不是 `$VAR`
3. **Job 间数据传递**：用 `actions/upload-artifact` + `actions/download-artifact`，或设置 `job.outputs`
4. **Maven Wrapper**：仓库必须包含 `.mvn/wrapper/maven-wrapper.properties`，否则 CI 环境可能无法下载
