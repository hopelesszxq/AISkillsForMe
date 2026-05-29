---
name: pg-security-hardening
description: PostgreSQL 安全加固实战：认证、SSL/TLS、行级安全、审计与加密
tags: [postgresql, security, ssl, rls, authentication, hardening]
---

## 概述

PostgreSQL 安全加固涵盖认证方式、传输加密、访问控制、数据加密和审计日志。本文覆盖生产环境必备的安全配置。

## 1. 认证方式选择

### 方法对比

| 方式 | 安全等级 | 密码存储 | 适用场景 |
|------|---------|---------|---------|
| **trust** | ❌ 极低 | — | 仅本地 Unix Socket 开发 |
| **password** | ⚠️ 低 | 明文传输 | 遗留兼容（不推荐） |
| **md5** | ⚠️ 中 | MD5 哈希 | 旧版兼容 |
| **scram-sha-256** | ✅ 高 | SCRAM 迭代哈希 | **生产推荐** |
| **cert** | ✅✅ 最高 | 客户端证书 | 内部服务间通信 |
| **ldap** | ✅ 中 | LDAP 服务器 | 企业集成 |
| **gss** | ✅ 高 | Kerberos | 域环境 |

### pg_hba.conf 推荐配置

```conf
# 本地 Unix Socket 连接（管理员用）
local   all             all                                     peer

# 本地 TCP 连接（管理员用）
host    all             all             127.0.0.1/32            scram-sha-256

# 应用连接（限制到应用网段）
host    myapp           appuser         10.0.1.0/24            scram-sha-256

# 内部服务间（证书认证）
hostssl all             replicator      10.0.0.0/8             cert
# clientcert=verify-full 确保客户端证书 CN 匹配用户名
hostssl all             +dba_role       192.168.0.0/16          scram-sha-256 clientcert=verify-full

# 拒绝所有其他连接
host    all             all             0.0.0.0/0               reject
```

### 切换到 SCRAM-SHA-256

```sql
-- 设置默认密码加密
ALTER SYSTEM SET password_encryption = 'scram-sha-256';
SELECT pg_reload_conf();

-- 更新现有用户密码
ALTER USER appuser PASSWORD 'new-strong-password';
```

## 2. SSL/TLS 传输加密

### 生成证书

```bash
# CA 自签名
openssl req -new -x509 -days 3650 -nodes \
  -out ca-cert.pem -keyout ca-key.pem \
  -subj "/C=CN/ST=Shanghai/O=MyOrg/CN=MyCA"

# 服务器证书（CN 须为服务器主机名或 IP）
openssl req -new -nodes \
  -out server.csr -keyout server-key.pem \
  -subj "/C=CN/ST=Shanghai/O=MyOrg/CN=db01.example.com"

openssl x509 -req -days 365 -in server.csr \
  -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial \
  -out server-cert.pem

# 客户端证书
openssl req -new -nodes \
  -out client.csr -keyout client-key.pem \
  -subj "/C=CN/ST=Shanghai/O=MyOrg/CN=appuser"

openssl x509 -req -days 365 -in client.csr \
  -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial \
  -out client-cert.pem
```

### postgresql.conf SSL 配置

```conf
ssl = on
ssl_cert_file = '/etc/pki/tls/certs/server-cert.pem'
ssl_key_file = '/etc/pki/tls/private/server-key.pem'
ssl_ca_file = '/etc/pki/tls/certs/ca-cert.pem'

# 最低 TLS 版本（禁止 TLS 1.0/1.1）
ssl_min_protocol_version = 'TLSv1.2'

# 客户端证书验证模式
ssl_ciphers = 'HIGH:!aNULL:!eNULL:!MD5'

# 将 REVOKE 的证书放入 CRL
ssl_crl_file = '/etc/pki/tls/certs/ca-cert.crl'
```

### 强制用户使用 SSL

```sql
-- 要求指定用户必须通过 SSL 连接
ALTER USER appuser SET sslcert = 'client-cert.pem';
-- 或在角色上设置 SSL 连接限制
ALTER ROLE appuser VALID UNTIL 'infinity';
```

## 3. 行级安全（Row-Level Security, RLS）

RLS 允许在多租户场景中按用户控制数据可见性，避免应用层过滤遗漏。

### 启用 RLS

```sql
-- 创建多租户表
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    product_name TEXT NOT NULL,
    amount NUMERIC(10,2)
);

-- 启用行级安全
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
-- 可选：对表所有者也生效（默认不生效）
ALTER TABLE orders FORCE ROW LEVEL SECURITY;

-- 创建基于当前用户租户标识的策略
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.tenant_id'));

-- 创建读写分离策略
CREATE POLICY tenant_read ON orders FOR SELECT
    USING (tenant_id = current_setting('app.tenant_id'));

CREATE POLICY tenant_write ON orders FOR INSERT
    WITH CHECK (tenant_id = current_setting('app.tenant_id'));
```

### 应用层设置租户上下文

```java
// JDBC / MyBatis 中设置
@BeforeEach
public void setTenant() {
    try (var stmt = connection.createStatement()) {
        stmt.execute("SET app.tenant_id = 'tenant-abc'");
    }
}
```

### RLS 性能注意事项

- RLS 会对每行检查策略，大表上可能影响查询性能
- 结合索引可以提高策略评估效率
- 使用 `EXPLAIN (ANALYZE, BUFFERS)` 检查 RLS 过滤代价
- 对不需要 RLS 的高级用户（管理员）可创建豁免策略

## 4. 列级安全与数据脱敏

```sql
-- 创建带加密敏感列的表
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username TEXT NOT NULL,
    email TEXT,
    phone_encrypted BYTEA,       -- 加密存储
    credit_card_last4 TEXT,      -- 仅存储后4位
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 使用扩展 pgcrypto 加密
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- 插入加密数据
INSERT INTO users (username, email, phone_encrypted)
VALUES (
    'alice',
    'alice@example.com',
    pgp_sym_encrypt('13800138000', current_setting('app.encrypt_key'))
);

-- 解密（仅在需要时）
SELECT username,
       pgp_sym_decrypt(phone_encrypted, current_setting('app.encrypt_key')) AS phone
FROM users
WHERE id = 1;
```

## 5. 审计日志（pgaudit）

### 安装配置

```bash
# 在 shared_preload_libraries 中添加
shared_preload_libraries = 'pgaudit'
```

```conf
# pgaudit 配置
pgaudit.log = 'write,ddl,role'         # 记录 DML 写操作、DDL、角色变更
pgaudit.log_level = 'notice'           # 日志级别
pgaudit.log_relation = on              # 记录涉及的表名
pgaudit.log_parameter = on             # 记录查询参数值
pgaudit.log_statement_once = off       # 每条语句都记录
```

```sql
-- 为特定角色开启详细审计
ALTER ROLE auditor_user SET pgaudit.log = 'all';
ALTER ROLE auditor_user SET pgaudit.log_level = 'info';
```

### 查询审计日志

```sql
-- 从 PostgreSQL 日志中分析（结合 csvlog）
-- 创建审计分析视图
CREATE VIEW audit_log_view AS
SELECT
    log_time,
    session_id,
    user_name,
    database_name,
    command_tag,
    object_name,
    detail
FROM pg_catalog.pg_log
WHERE command_tag IN ('SELECT', 'INSERT', 'UPDATE', 'DELETE', 'DROP', 'ALTER');
```

## 6. 网络安全

### listen_addresses 限制

```conf
# 只监听特定 IP
listen_addresses = '10.0.1.100,127.0.0.1'
# 不要设为 '*' 除非有防火墙保护
```

### 连接限制

```conf
# 最大连接数（防止连接耗尽）
max_connections = 200

# 超时配置
statement_timeout = '30s'              # SQL 执行超时
idle_in_transaction_session_timeout = '60s'  # 空闲事务超时
password_timeout = '10s'               # 密码验证超时

# 超级用户保留连接（防止管理锁死）
superuser_reserved_connections = 10
```

### 扩展安全：sepgsql

```bash
# SELinux 策略增强（CentOS/RHEL）
# 启用 sepgsql
shared_preload_libraries = 'sepgsql'

# 重新加载 SELinux 策略
load_policy -r
```

## 7. 安全清单检查

```sql
-- 1. 检查密码加密方式
SELECT name, setting FROM pg_settings WHERE name = 'password_encryption';

-- 2. 检查 SSL 是否启用
SELECT name, setting FROM pg_settings WHERE name = 'ssl';

-- 3. 列出所有用户及属性
SELECT rolname, rolsuper, rolcreatedb, rolcanlogin,
       rolvaliduntil, rolconnlimit
FROM pg_roles
WHERE rolcanlogin = true;

-- 4. 检查是否有弱密码（需要使用 pgcrypto 扩展）
-- 需要额外工具配合

-- 5. 检查公开暴露的表
SELECT schemaname, tablename,
       has_table_privilege('public', schemaname||'.'||tablename, 'SELECT') AS public_can_read
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema');

-- 6. 检查未启用 RLS 的表（如果应该启用的话）
SELECT relname, relrowsecurity
FROM pg_class
WHERE relrowsecurity = false AND relkind = 'r'
  AND relnamespace NOT IN (
    SELECT oid FROM pg_namespace
    WHERE nspname IN ('pg_catalog', 'information_schema')
  );
```

## 注意事项

- **不要在 pg_hba.conf 中使用 trust**，仅限本地开发
- **SSL 证书过期**会导致所有新连接失败，设置证书自动续期提醒
- RLS 策略会对 EXPLAIN 和 ANALYZE 增加额外开销，注意监控
- pgaudit 可能产生大量日志，设置合理的日志轮转
- pgcrypto 加密在数据库 CPU 上增加开销，敏感列建议应用层加密
- 定期进行安全审计，检查用户权限和连接来源
- 生产环境建议启用所有 `log_*` 参数并配合日志分析工具
