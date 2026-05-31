---
name: docker-image-security
description: Docker 镜像安全实践：漏洞扫描、镜像加固、SBOM 生成与供应链安全
tags: [tools, docker, security, container, devsecops, sbom]
---

## 一、镜像漏洞扫描

### 1. Trivy（推荐，支持多种格式）

```bash
# 安装 Trivy
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh

# 扫描本地镜像
trivy image nginx:1.27-alpine

# 扫描并输出 JSON（对接 CI）
trivy image --severity CRITICAL,HIGH --format json myapp:latest > scan-result.json

# 扫描容器文件系统
trivy fs --severity CRITICAL --ignore-unfixed /path/to/project

# 扫描仓库
trivy repo https://github.com/org/my-service.git
```

### 2. Docker Scout（Docker 官方）

```bash
# 启用 Docker Scout
docker scout quickview nginx:latest

# 查看详细漏洞
docker scout cves nginx:latest

# 查看推荐的修复版本
docker scout recommendations nginx:latest
```

### 3. Grype（Anchore）

```bash
# 安装 Grype
curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /usr/local/bin

# 扫描
grype myapp:latest

# 输出 CycloneDX SBOM
grype myapp:latest -o cyclonedx > sbom.json
```

## 二、镜像构建安全实践

### 1. 最小基础镜像

```dockerfile
# ❌ 不推荐：500MB+，大量攻击面
FROM openjdk:17-jdk

# ✅ 推荐：基于 Distroless，仅包含运行时
FROM eclipse-temurin:17-jre-alpine

# ✅ 最佳：Distroless 基础镜像（无 shell、无包管理器）
FROM gcr.io/distroless/java17-debian12:latest

# ✅ 极致精简：基于静态编译（Spring Boot 3.x + GraalVM）
FROM golang:1.23-alpine AS builder
# ... 编译
FROM scratch
COPY --from=builder /app/server /server
```

### 2. 多阶段构建（移除构建工具）

```dockerfile
# 第一阶段：构建
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn package -DskipTests -B

# 第二阶段：运行（仅含 JRE 和 Jar）
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=builder /build/target/*.jar app.jar
USER appuser
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 3. 非 root 运行

```dockerfile
# 创建专用用户
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# 或者设置用户 ID（避免主机 UID 冲突）
RUN adduser -u 10001 -D -H appuser
USER appuser
```

## 三、SBOM（软件物料清单）

### 生成 SBOM

```bash
# Syft 生成 SPDX 格式 SBOM
syft myapp:latest -o spdx-json > sbom.spdx.json

# Trivy 生成 CycloneDX 格式
trivy image --format cyclonedx --output sbom.cdx.json myapp:latest

# Docker Scout
docker scout sbom myapp:latest
```

### CI 中集成 SBOM

```yaml
# .github/workflows/security.yml
jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3

      - name: Build image
        run: docker build -t myapp:ci .

      - name: Trivy scan
        uses: aquasecurity/trivy-action@0.28.0
        with:
          image-ref: myapp:ci
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL

      - name: Generate SBOM
        uses: anchore/sbom-action@v0.17.0
        with:
          image: myapp:ci
          format: cyclonedx
          output-file: sbom.cdx.json

      - name: Upload SBOM
        uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom.cdx.json
```

## 四、镜像签名与验证

### Cosign 签名

```bash
# 安装 Cosign
wget -O cosign https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64
chmod +x cosign && mv cosign /usr/local/bin/

# 生成密钥对
cosign generate-key-pair

# 签名镜像
cosign sign --key cosign.key myregistry.com/myapp:latest

# 验证签名
cosign verify --key cosign.pub myregistry.com/myapp:latest
```

### 在 K8s 中强制验证

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-image-signature
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-cosign
      match:
        resources:
          kinds:
            - Pod
      verifyImages:
        - image: "myregistry.com/*"
          key: |-
            -----BEGIN PUBLIC KEY-----
            ...
            -----END PUBLIC KEY-----
```

## 五、Dockerfile 安全最佳实践

```dockerfile
# 1. 锁定基础镜像版本（不用 latest）
FROM eclipse-temurin:17.0.12_7-jre-alpine@sha256:abc123...
# 或用 digest 保证不可变性

# 2. 最小化层
RUN apt-get update && apt-get install -y \
    curl \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*    # ← 清理缓存减少层大小

# 3. 不要存储敏感信息
# ❌ 错误
ENV DB_PASSWORD=secret123
COPY .env .env

# ✅ 正确：运行时通过 Secret 注入
# docker run -e DB_PASSWORD=$(cat /run/secrets/db_password)

# 4. 只复制需要的内容
COPY target/app.jar app.jar    # 而非 COPY . .
# 或 .dockerignore
```
```dockerignore
# .dockerignore
**/*.class
**/*.jar
target/
.git/
.gitignore
*.md
node_modules/
```

## 六、运行时的安全加固

```bash
# 1. 避免 --privileged 模式
docker run --security-opt=no-new-privileges \
           --cap-drop=ALL \
           --cap-add=NET_BIND_SERVICE \
           --read-only \
           --tmpfs /tmp \
           myapp:latest

# 2. 只读文件系统
# 在 docker-compose 中
services:
  myapp:
    image: myapp:latest
    read_only: true
    tmpfs:
      - /tmp
      - /var/run

# 3. 资源限制
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
```

## 注意事项

1. **定期扫描**：即使基础镜像没变，新的 CVE 可能随时出现，建议每周至少扫描一次生产镜像
2. **基础镜像更新策略**：关注 Distroless/Alpine 的安全更新，设置 Dependabot/Renovate 自动提 PR
3. **不要忽视 LOW/MEDIUM 漏洞**：虽然优先级低，但可能被链式利用，建议设置 SLSA 3+ 策略
4. **私有仓库的安全扫描**：Harbor、GitLab Container Registry、AWS ECR 都内置扫描能力，优先启用
5. **CI/CD 阻断策略**：CRITICAL 漏洞直接阻断构建，HIGH 漏洞告警+人工确认，MEDIUM 记录
6. **层缓存安全**：CI 中使用 `--cache-from` 时，确保缓存来自可信构建（防止缓存投毒）
