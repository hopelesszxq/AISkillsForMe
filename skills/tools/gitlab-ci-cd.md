---
name: gitlab-ci-cd
description: GitLab CI/CD 流水线最佳实践：Spring Boot 多阶段构建、K8s 部署、SonarQube 代码质量
tags: [tools, gitlab, ci-cd, devops, spring-boot, docker]
---

## 概述

GitLab CI/CD 通过 `.gitlab-ci.yml` 定义流水线，支持多阶段构建、制品传递、环境部署和安全扫描。本文从基础到高级场景，覆盖 Java/Spring Boot 项目的完整 CI/CD 实践。

## 1. 基础流水线结构

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - package
  - deploy

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository -DskipTests=false"
  DOCKER_DRIVER: overlay2

# 缓存 Maven 依赖
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - .m2/repository/

build:
  stage: build
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn clean compile
  artifacts:
    paths:
      - target/classes/
    expire_in: 1 hour

test:
  stage: test
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn test
  artifacts:
    reports:
      junit: target/surefire-reports/TEST-*.xml
    paths:
      - target/site/jacoco/  # 覆盖率报告

package:
  stage: package
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn package -DskipTests
  artifacts:
    paths:
      - target/*.jar
    expire_in: 1 week
```

## 2. Docker 镜像构建与推送

```yaml
variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  DOCKER_IMAGE_LATEST: $CI_REGISTRY_IMAGE:latest

docker-build:
  stage: package
  image: docker:26
  services:
    - docker:26-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $DOCKER_IMAGE -t $DOCKER_IMAGE_LATEST .
    - docker push $DOCKER_IMAGE
    - docker push $DOCKER_IMAGE_LATEST
  only:
    - main
```

### 最佳实践的 Dockerfile

```dockerfile
# 多阶段构建 —— 分离编译和运行环境
# Stage 1: Build
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn package -DskipTests

# Stage 2: Runtime
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=builder /app/target/*.jar app.jar
USER appuser
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## 3. 多环境部署

```yaml
stages:
  - build
  - test
  - deploy-dev
  - deploy-staging
  - deploy-production

.deploy-template: &deploy-template
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/$APP_NAME $APP_NAME=$DOCKER_IMAGE -n $NAMESPACE
    - kubectl rollout status deployment/$APP_NAME -n $NAMESPACE --timeout=5m

deploy-dev:
  <<: *deploy-template
  variables:
    NAMESPACE: dev
    APP_NAME: my-service
  environment:
    name: dev
  only:
    - develop

deploy-staging:
  <<: *deploy-template
  variables:
    NAMESPACE: staging
    APP_NAME: my-service
  environment:
    name: staging
  only:
    - main
  needs:
    - docker-build

deploy-production:
  <<: *deploy-template
  variables:
    NAMESPACE: prod
    APP_NAME: my-service
  environment:
    name: production
  when: manual  # 人工审批
  only:
    - tags
  needs:
    - docker-build
```

## 4. 代码质量与安全扫描

```yaml
sonarqube-check:
  stage: test
  image: maven:3.9-eclipse-temurin-17
  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
    GIT_DEPTH: "0"
  script:
    - mvn verify sonar:sonar
      -Dsonar.host.url=$SONAR_HOST_URL
      -Dsonar.login=$SONAR_TOKEN
      -Dsonar.qualitygate.wait=true
  allow_failure: true
  only:
    - merge_requests
    - main

dependency-scan:
  stage: test
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn org.owasp:dependency-check-maven:check
      -DfailBuildOnCVSS=7  # CVSS 7 以上失败
  artifacts:
    paths:
      - target/dependency-check-report.html
  allow_failure: true
```

## 5. 高级技巧

### 并行测试（分片）

```yaml
.test-parallel:
  stage: test
  parallel: 5  # 5 个并行 Job
  script:
    - mvn test -Dtest=Test${CI_NODE_INDEX}
```

### Review Apps（PR 环境自动创建）

```yaml
review:
  stage: deploy-dev
  script:
    - helm upgrade --install review-$CI_MERGE_REQUEST_IID ./chart
      --set image.tag=$CI_COMMIT_SHORT_SHA
      --set host=review-$CI_MERGE_REQUEST_IID.example.com
      -n review
  environment:
    name: review/$CI_MERGE_REQUEST_IID
    url: https://review-$CI_MERGE_REQUEST_IID.example.com
    on_stop: stop-review
  only:
    - merge_requests

stop-review:
  stage: deploy-dev
  script:
    - helm uninstall review-$CI_MERGE_REQUEST_IID -n review
  environment:
    action: stop
  when: manual
  only:
    - merge_requests
```

### 子流水线（Child Pipeline）

```yaml
# 主流水线触发子流水线
trigger-multi-module:
  stage: build
  trigger:
    include:
      - local: 'services/service-a/.gitlab-ci.yml'
      - local: 'services/service-b/.gitlab-ci.yml'
    strategy: depend
```

### Needs 关键字优化

```yaml
# 明确 Job 依赖关系，优化 DAG 执行
test-api:
  stage: test
  needs: ["build-api"]
  script:
    - mvn test -pl api

test-common:
  stage: test
  needs: ["build-common"]
  script:
    - mvn test -pl common

# 两个 test Job 可并行执行
```

## 6. .gitlab-ci.yml 常见变量

| 变量 | 说明 |
|---|---|
| `CI_COMMIT_SHORT_SHA` | 短 commit SHA（8 位） |
| `CI_COMMIT_REF_SLUG` | 分支名（URL 安全格式） |
| `CI_COMMIT_TAG` | Tag 名称（仅 Tag 触发时） |
| `CI_MERGE_REQUEST_IID` | MR 编号 |
| `CI_PIPELINE_ID` | 流水线 ID |
| `CI_JOB_ID` | 当前 Job ID |
| `CI_REGISTRY_IMAGE` | GitLab 容器仓库镜像地址 |
| `CI_ENVIRONMENT_NAME` | 环境名称 |

## 注意事项

1. **Docker in Docker 安全**：`docker:26-dind` 需要特权模式（`privileged: true`），或用 `kaniko` 替代以提升安全性
2. **缓存清理**：`cache` 设置 `key` 按分支区分，避免不同分支缓存冲突；定期 `mvn clean` 防止构建异常
3. **Artifacts 过期**：设置合理的 `expire_in`（构建产物 1 小时~1 天），避免 GitLab 存储空间占用
4. **流水线超时**：设置全局 `timeout`（如 `timeout: 30 minutes`），避免卡死的 Job 一直占用 Runner
5. **敏感信息**：密码、Token 使用 `CI/CD Settings > Variables` 的 **Masked** 变量，不要硬编码
6. **Runner 标签**：为不同类型的 Runner 打标签（如 `docker`、`k8s`、`shell`），Job 通过 `tags` 指定 Runner
7. **GitLab CI 与 GitHub Actions 差异**：GitLab 使用 `stages` + `jobs` DSL，多个 Job 在同一 stage 可并行执行，通过 `needs` 构建 DAG
