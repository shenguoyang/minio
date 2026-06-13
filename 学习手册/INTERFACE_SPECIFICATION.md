# MinIO 外部接口说明文档

> **项目**: MinIO Server (开源对象存储服务器)
> **版本**: 基于 `master` 分支
> **协议**: GNU AGPL v3
> **文档日期**: 2026-06-13

---

## 目录

1. [接口总览](#1-接口总览)
2. [S3 兼容 API](#2-s3-兼容-api)
3. [Admin 管理 API](#3-admin-管理-api)
4. [STS 安全令牌服务 API](#4-sts-安全令牌服务-api)
5. [Console 管理控制台](#5-console-管理控制台)
6. [Metrics 监控指标接口](#6-metrics-监控指标接口)
7. [KMS 密钥管理 API](#7-kms-密钥管理-api)
8. [Health Check 健康检查接口](#8-health-check-健康检查接口)
9. [事件通知系统](#9-事件通知系统)
10. [FTP / SFTP 接口](#10-ftp--sftp-接口)
11. [内部节点通信接口](#11-内部节点通信接口)
12. [认证与授权机制](#12-认证与授权机制)
13. [错误码体系](#13-错误码体系)

---

## 1. 接口总览

MinIO 根据启动参数提供多种外部接口，主要分为以下几类：

| 接口类别 | 默认端口 | 协议 | 说明 |
|---------|---------|------|------|
| **S3 API** | 9000 | HTTP/HTTPS | Amazon S3 兼容的对象存储 REST API |
| **Console (Web UI)** | 9090 | HTTP/HTTPS | 嵌入式 Web 管理控制台 |
| **Admin API** | (与 S3 API 同端口) | HTTP/HTTPS | 集群管理、用户策略管理等 REST API |
| **STS API** | (与 S3 API 同端口) | HTTP/HTTPS | 安全令牌服务，兼容 AWS STS |
| **KMS API** | (与 S3 API 同端口) | HTTP/HTTPS | 密钥管理服务 |
| **Metrics API** | (与 S3 API 同端口) | HTTP/HTTPS | Prometheus 兼容的监控指标 |
| **Health Check** | (与 S3 API 同端口) | HTTP/HTTPS | 存活性和就绪性探测 |
| **Peer REST/Grid** | 9000 (内部) | HTTP | 分布式模式节点间通信 |
| **FTP** | 8021 (可配) | FTP | 传统 FTP 文件传输协议 |
| **SFTP** | 8022 (可配) | SFTP/SSH | SSH 文件传输协议 |

**服务端口配置示例:**

```bash
# 基础启动
minio server /data

# 指定 S3 API 端口和 Console 端口
minio server --address ":9000" --console-address ":9090" /data

# 启用 FTP 和 SFTP
minio server --ftp="address=:8021" --ftp="passive-port-range=30000-40000" \
             --sftp="address=:8022" --sftp="ssh-private-key=${HOME}/.ssh/id_rsa" /data
```

---

## 2. S3 兼容 API

S3 API 是 MinIO 最核心的外部接口，实现了 Amazon S3 协议的主要功能。

### 2.1 请求路径

```
http://{endpoint}:{port}/{bucket}/{object}
```

支持两种访问风格：
- **路径风格 (Path-Style)**: `http://host:9000/bucket/object`
- **虚拟托管风格 (Virtual-Hosted-Style)**: `http://bucket.host:9000/object`

### 2.2 认证方式

支持以下 AWS 兼容签名认证：
- **AWS Signature V4** — 请求头中包含 `Authorization: AWS4-HMAC-SHA256 ...`
- **AWS Signature V2** (遗留兼容) — `Authorization: AWS ...`
- **Presigned URL V4** — URL 查询参数包含 `X-Amz-Credential` 等
- **Presigned URL V2** (遗留兼容) — URL 查询参数包含 `AWSAccessKeyId` 等
- **POST Policy Signature V4** — 基于表单的上传认证
- **JWT Bearer Token** — `Authorization: Bearer {token}`

### 2.3 Bucket 级别操作

#### Bucket 生命周期管理

| 操作 | 方法 | Query 参数 | 说明 |
|------|------|-----------|------|
| `PutBucket` | `PUT` | — | 创建 Bucket |
| `DeleteBucket` | `DELETE` | — | 删除 Bucket |
| `HeadBucket` | `HEAD` | — | 检查 Bucket 是否存在 |
| `GetBucketLocation` | `GET` | `?location` | 获取 Bucket 所在区域 |

#### Bucket 配置管理

| 操作 | 方法 | Query 参数 | 说明 |
|------|------|-----------|------|
| `GetBucketPolicy` | `GET` | `?policy` | 获取 Bucket 访问策略 |
| `PutBucketPolicy` | `PUT` | `?policy` | 设置 Bucket 访问策略 |
| `DeleteBucketPolicy` | `DELETE` | `?policy` | 删除 Bucket 访问策略 |
| `GetBucketPolicyStatus` | `GET` | `?policyStatus` | 获取策略状态(Public/Private) |
| `GetBucketLifecycle` | `GET` | `?lifecycle` | 获取生命周期配置 |
| `PutBucketLifecycle` | `PUT` | `?lifecycle` | 设置生命周期配置 |
| `DeleteBucketLifecycle` | `DELETE` | `?lifecycle` | 删除生命周期配置 |
| `GetBucketEncryption` | `GET` | `?encryption` | 获取服务端加密配置 |
| `PutBucketEncryption` | `PUT` | `?encryption` | 设置服务端加密配置 |
| `DeleteBucketEncryption` | `DELETE` | `?encryption` | 删除服务端加密配置 |
| `GetBucketObjectLockConfig` | `GET` | `?object-lock` | 获取对象锁定配置 |
| `PutBucketObjectLockConfig` | `PUT` | `?object-lock` | 设置对象锁定配置 |
| `GetBucketVersioning` | `GET` | `?versioning` | 获取版本控制配置 |
| `PutBucketVersioning` | `PUT` | `?versioning` | 设置版本控制配置 |
| `GetBucketTagging` | `GET` | `?tagging` | 获取 Bucket 标签 |
| `PutBucketTagging` | `PUT` | `?tagging` | 设置 Bucket 标签 |
| `DeleteBucketTagging` | `DELETE` | `?tagging` | 删除 Bucket 标签 |

#### Bucket 复制

| 操作 | 方法 | Query 参数 | 说明 |
|------|------|-----------|------|
| `GetBucketReplicationConfig` | `GET` | `?replication` | 获取复制配置 |
| `PutBucketReplicationConfig` | `PUT` | `?replication` | 设置复制配置 |
| `DeleteBucketReplicationConfig` | `DELETE` | `?replication` | 删除复制配置 |
| `GetBucketReplicationMetrics` (v2) | `GET` | `?replication-metrics=2` | 获取复制指标 v2 |
| `GetBucketReplicationMetrics` | `GET` | `?replication-metrics` | 获取复制指标（已废弃） |
| `ValidateBucketReplicationCreds` | `GET` | `?replication-check` | 验证复制目标凭证 |
| `ResetBucketReplicationStatus` | `GET` | `?replication-reset-status` | 重置复制状态 (MinIO 扩展) |
| `ResetBucketReplicationStart` | `PUT` | `?replication-reset` | 启动复制重置 (MinIO 扩展) |

#### Bucket 通知

| 操作 | 方法 | Query 参数 | 说明 |
|------|------|-----------|------|
| `GetBucketNotification` | `GET` | `?notification` | 获取通知配置 |
| `PutBucketNotification` | `PUT` | `?notification` | 设置通知配置 |
| `ListenNotification` | `GET` | `?events={events}` | SSE 监听事件通知流 |

#### 其他 Bucket 操作

| 操作 | 方法 | Query 参数 | 说明 |
|------|------|-----------|------|
| `GetBucketACL` | `GET` | `?acl` | 获取 ACL（桩实现） |
| `PutBucketACL` | `PUT` | `?acl` | 设置 ACL（桩实现） |
| `GetBucketCORS` | `GET` | `?cors` | 获取 CORS 配置（桩实现） |
| `PutBucketCORS` | `PUT` | `?cors` | 设置 CORS 配置（桩实现） |
| `DeleteBucketCORS` | `DELETE` | `?cors` | 删除 CORS 配置（桩实现） |
| `GetBucketWebsite` | `GET` | `?website` | 获取静态网站配置 |
| `DeleteBucketWebsite` | `DELETE` | `?website` | 删除静态网站配置 |
| `GetBucketAccelerate` | `GET` | `?accelerate` | 获取加速配置（桩实现） |
| `GetBucketRequestPayment` | `GET` | `?requestPayment` | 获取请求付费配置（桩实现） |
| `GetBucketLogging` | `GET` | `?logging` | 获取日志配置（桩实现） |

### 2.4 Object 级别操作

#### 基本对象操作

| 操作 | 方法 | Query 参数 | 说明 |
|------|------|-----------|------|
| `GetObject` | `GET` | — | 下载对象 |
| `PutObject` | `PUT` | — | 上传对象 |
| `DeleteObject` | `DELETE` | — | 删除对象 |
| `HeadObject` | `HEAD` | — | 获取对象元数据 |
| `CopyObject` | `PUT` | (header: `X-Amz-Copy-Source`) | 拷贝对象 |
| `GetObjectAttributes` | `GET` | `?attributes` | 获取对象属性 |
| `GetObjectLambda` | `GET` | `?lambdaArn={arn}` | 通过 Lambda 获取对象 |

#### 高级对象操作

| 操作 | 方法 | Query 参数 | 说明 |
|------|------|-----------|------|
| `PutObjectExtract` | `PUT` | (header: `X-Amz-Snowball-Extract: true`) | 自动解压 ZIP 归档 |
| `GetObjectRetention` | `GET` | `?retention` | 获取对象保留策略 |
| `PutObjectRetention` | `PUT` | `?retention` | 设置对象保留策略 |
| `GetObjectLegalHold` | `GET` | `?legal-hold` | 获取对象依法保留状态 |
| `PutObjectLegalHold` | `PUT` | `?legal-hold` | 设置对象依法保留状态 |
| `GetObjectACL` | `GET` | `?acl` | 获取对象 ACL（桩实现） |
| `PutObjectACL` | `PUT` | `?acl` | 设置对象 ACL（桩实现） |
| `GetObjectTagging` | `GET` | `?tagging` | 获取对象标签 |
| `PutObjectTagging` | `PUT` | `?tagging` | 设置对象标签 |
| `DeleteObjectTagging` | `DELETE` | `?tagging` | 删除对象标签 |
| `SelectObjectContent` | `POST` | `?select&select-type=2` | S3 Select 对象查询 |
| `PostRestoreObject` | `POST` | `?restore` | 恢复归档对象 |
| `DeleteMultipleObjects` | `POST` | `?delete` | 批量删除对象 |
| `PostPolicyBucket` | `POST` | (multipart/form-data) | 基于表单策略的上传 |

#### 分片上传 (Multipart Upload)

| 操作 | 方法 | Query 参数 | 说明 |
|------|------|-----------|------|
| `NewMultipartUpload` | `POST` | `?uploads` | 初始化分片上传 |
| `PutObjectPart` | `PUT` | `?partNumber={n}&uploadId={id}` | 上传分片 |
| `CopyObjectPart` | `PUT` | `?partNumber={n}&uploadId={id}` (header: `X-Amz-Copy-Source`) | 拷贝分片 |
| `ListObjectParts` | `GET` | `?uploadId={id}` | 列出已上传分片 |
| `CompleteMultipartUpload` | `POST` | `?uploadId={id}` | 完成分片上传 |
| `AbortMultipartUpload` | `DELETE` | `?uploadId={id}` | 取消分片上传 |
| `ListMultipartUploads` | `GET` | `?uploads` | 列出进行中的分片上传 |

#### 列出操作

| 操作 | 方法 | Query 参数 | 说明 |
|------|------|-----------|------|
| `ListBuckets` | `GET` | (根路径 `/`) | 列出所有 Bucket |
| `ListObjectsV1` | `GET` | (无 list-type) | 列出对象 (V1, 遗留) |
| `ListObjectsV2` | `GET` | `?list-type=2` | 列出对象 (V2) |
| `ListObjectsV2M` | `GET` | `?list-type=2&metadata=true` | 列出对象含元数据 (V2) |
| `ListObjectVersions` | `GET` | `?versions` | 列出对象版本 |
| `ListObjectVersionsM` | `GET` | `?versions&metadata=true` | 列出对象版本含元数据 |

### 2.5 已明确拒绝的 S3 API

以下 S3 API 在 MinIO 中有意返回 `501 Not Implemented`：

| 接口 | 说明 |
|------|------|
| `?torrent` | Torrent 种子操作 |
| `?inventory` | S3 Inventory 清单配置 |
| `?metrics` | S3 Storage Metrics |
| `?logging` (PUT/DELETE) | S3 服务端日志配置 |
| `?accelerate` (PUT/DELETE) | S3 Transfer Acceleration |
| `?requestPayment` (PUT/DELETE) | 请求付费配置 |
| `?publicAccessBlock` | Public Access Block 配置 |
| `?ownershipControls` | 所有权控制 |
| `?intelligent-tiering` | 智能分层 |
| `?analytics` | S3 Analytics |
| `?acl` (Bucket DELETE/PUT/HEAD, Object DELETE) | 部分 ACL 操作 |
| `?cors` (PUT/DELETE) | CORS 写操作 |
| `?website` (PUT) | 静态网站写配置 |

### 2.6 CORS 支持

MinIO 在 S3 API 层面配置了完整的 CORS 支持：

- **允许的方法**: `GET`, `PUT`, `HEAD`, `POST`, `DELETE`, `OPTIONS`, `PATCH`
- **允许的头**: `Date`, `ETag`, `Content-*`, `x-amz-*`, `X-Amz-*` 等
- **允许凭证**: `true`
- **来源校验**: 通过 `MINIO_API_CORS_ALLOW_ORIGIN` 环境变量配置白名单，支持通配符匹配

---

## 3. Admin 管理 API

Admin API 用于管理 MinIO 集群、用户、策略、配置等。Admin API 路径前缀为 `minio/admin`。

### 3.1 接口规范

```
POST/GET/PUT/DELETE /minio/admin/v3/{resource}?{query-params}
```

- **API 版本**: `v3`
- **路径前缀**: `/minio/admin`
- **请求格式**: JSON
- **响应格式**: JSON
- **认证**: 需要 Admin 权限的 JWT 或签名认证

### 3.2 服务管理

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `ServiceV2` | `POST` | `/service?action={action}&type=2` | 重启/停止 MinIO 服务 (v2) |
| `Service` | `POST` | `/service?action={action}` | 重启/停止 MinIO 服务（已废弃） |
| `ServerUpdateV2` | `POST` | `/update?updateURL={url}&type=2` | 更新 MinIO (v2) |
| `ServerUpdate` | `POST` | `/update?updateURL={url}` | 更新 MinIO（已废弃） |
| `ServerInfo` | `GET` | `/info` | 获取服务器信息 |
| `StorageInfo` | `GET` | `/storageinfo` | 获取存储信息 |
| `DataUsageInfo` | `GET` | `/datausageinfo` | 获取数据使用量信息 |
| `InspectData` | `GET/POST` | `/inspect-data` | 检查数据用于调试 |

### 3.3 IAM 管理 — 用户与策略

#### 用户管理

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `AddUser` | `PUT` | `/add-user?accessKey={ak}` | 创建用户 |
| `RemoveUser` | `DELETE` | `/remove-user?accessKey={ak}` | 删除用户 |
| `GetUserInfo` | `GET` | `/user-info?accessKey={ak}` | 获取用户信息 |
| `ListUsers` | `GET` | `/list-users` | 列出所有用户 |
| `ListBucketUsers` | `GET` | `/list-users?bucket={bucket}` | 列出有 Bucket 权限的用户 |
| `SetUserStatus` | `PUT` | `/set-user-status?accessKey={ak}&status={status}` | 启用/禁用用户 |
| `AccountInfo` | `GET` | `/accountinfo` | 获取当前账号信息 |

#### 服务账户 (Service Account) 管理

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `AddServiceAccount` | `PUT` | `/add-service-account` | 创建服务账户 |
| `UpdateServiceAccount` | `POST` | `/update-service-account?accessKey={ak}` | 更新服务账户 |
| `InfoServiceAccount` | `GET` | `/info-service-account?accessKey={ak}` | 查询服务账户 |
| `ListServiceAccounts` | `GET` | `/list-service-accounts` | 列出服务账户 |
| `DeleteServiceAccount` | `DELETE` | `/delete-service-account?accessKey={ak}` | 删除服务账户 |
| `TemporaryAccountInfo` | `GET` | `/temporary-account-info?accessKey={ak}` | 获取临时凭证信息 |

#### 访问密钥批量操作

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `ListAccessKeysBulk` | `GET` | `/list-access-keys-bulk?listType={type}` | 批量列出访问密钥 |
| `InfoAccessKey` | `GET` | `/info-access-key?accessKey={ak}` | 查询访问密钥信息 |

#### 策略管理

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `AddCannedPolicy` | `PUT` | `/add-canned-policy?name={name}` | 创建策略 |
| `RemoveCannedPolicy` | `DELETE` | `/remove-canned-policy?name={name}` | 删除策略 |
| `InfoCannedPolicy` | `GET` | `/info-canned-policy?name={name}` | 查询策略详情 |
| `ListCannedPolicies` | `GET` | `/list-canned-policies` | 列出所有策略 |
| `ListBucketPolicies` | `GET` | `/list-canned-policies?bucket={bucket}` | 列出 Bucket 关联策略 |
| `SetPolicyForUserOrGroup` | `PUT` | `/set-user-or-group-policy?policyName={n}&userOrGroup={name}&isGroup={bool}` | 为用户/组设置策略 |
| `AttachDetachPolicyBuiltin` | `POST` | `/idp/builtin/policy/{operation}` | 附加/分离内置策略 |
| `ListPolicyMappingEntities` | `GET` | `/idp/builtin/policy-entities` | 列出策略映射实体 |

#### 组管理

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `UpdateGroupMembers` | `PUT` | `/update-group-members` | 更新组成员 |
| `GetGroup` | `GET` | `/group?group={group}` | 获取组信息 |
| `ListGroups` | `GET` | `/groups` | 列出所有组 |
| `SetGroupStatus` | `PUT` | `/set-group-status?group={name}&status={status}` | 启用/禁用组 |

### 3.4 Identity Provider (IDP) 配置

#### 通用 IDP 配置

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `AddIdentityProviderCfg` | `PUT` | `/idp-config/{type}/{name}` | 添加 IDP 配置 |
| `UpdateIdentityProviderCfg` | `POST` | `/idp-config/{type}/{name}` | 更新 IDP 配置 |
| `ListIdentityProviderCfg` | `GET` | `/idp-config/{type}` | 列出 IDP 类型配置 |
| `GetIdentityProviderCfg` | `GET` | `/idp-config/{type}/{name}` | 获取指定 IDP 配置 |
| `DeleteIdentityProviderCfg` | `DELETE` | `/idp-config/{type}/{name}` | 删除 IDP 配置 |

#### LDAP 特定操作

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `AddServiceAccountLDAP` | `PUT` | `/idp/ldap/add-service-account` | 添加 LDAP 服务账户 |
| `ListAccessKeysLDAP` | `GET` | `/idp/ldap/list-access-keys?userDN={dn}&listType={type}` | 列出 LDAP 用户访问密钥 |
| `ListAccessKeysLDAPBulk` | `GET` | `/idp/ldap/list-access-keys-bulk?listType={type}` | 批量列出 LDAP 访问密钥 |
| `ListLDAPPolicyMappingEntities` | `GET` | `/idp/ldap/policy-entities` | 列出 LDAP 策略映射 |
| `AttachDetachPolicyLDAP` | `POST` | `/idp/ldap/policy/{operation}` | LDAP 策略操作 |

#### OpenID 特定操作

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `ListAccessKeysOpenIDBulk` | `GET` | `/idp/openid/list-access-keys-bulk?listType={type}` | 批量列出 OpenID 访问密钥 |

### 3.5 IAM 导入 / 导出

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `ExportIAM` | `GET` | `/export-iam` | 导出 IAM 数据 (ZIP) |
| `ImportIAM` | `PUT` | `/import-iam` | 导入 IAM 数据 |
| `ImportIAMV2` | `PUT` | `/import-iam-v2` | 导入 IAM 数据 (V2) |

### 3.6 配置管理

#### KV 配置操作

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `GetConfigKV` | `GET` | `/get-config-kv?key={key}` | 获取配置键值 |
| `SetConfigKV` | `PUT` | `/set-config-kv` | 设置配置键值 |
| `DelConfigKV` | `DELETE` | `/del-config-kv` | 删除配置键值 |
| `HelpConfigKV` | `GET` | `/help-config-kv?subSys={sub}&key={key}` | 获取配置帮助 |

#### 配置历史

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `ListConfigHistoryKV` | `GET` | `/list-config-history-kv?count={n}` | 列出配置变更历史 |
| `ClearConfigHistoryKV` | `DELETE` | `/clear-config-history-kv?restoreId={id}` | 清除历史记录 |
| `RestoreConfigHistoryKV` | `PUT` | `/restore-config-history-kv?restoreId={id}` | 恢复历史配置 |

#### 批量配置

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `GetConfig` | `GET` | `/config` | 导出全部配置 |
| `SetConfig` | `PUT` | `/config` | 设置全部配置 |

### 3.7 Bucket 配额管理

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `GetBucketQuota` | `GET` | `/get-bucket-quota?bucket={name}` | 获取 Bucket 配额 |
| `PutBucketQuota` | `PUT` | `/set-bucket-quota?bucket={name}` | 设置 Bucket 配额 |

### 3.8 远程目标与复制管理

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `ListRemoteTargets` | `GET` | `/list-remote-targets?bucket={name}&type={type}` | 列出远程复制目标 |
| `SetRemoteTarget` | `PUT` | `/set-remote-target?bucket={name}` | 设置远程目标 |
| `RemoveRemoteTarget` | `DELETE` | `/remove-remote-target?bucket={name}&arn={arn}` | 删除远程目标 |
| `ReplicationDiff` | `POST` | `/replication/diff?bucket={name}` | 复制差异比较 (MinIO 扩展) |
| `ReplicationMRF` | `GET` | `/replication/mrf?bucket={name}` | 复制 MRF 信息 (MinIO 扩展) |

### 3.9 存储池 (Pool) 与均衡 (Rebalance)

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `ListPools` | `GET` | `/pools/list` | 列出存储池 |
| `StatusPool` | `GET` | `/pools/status?pool={pool}` | 获取存储池状态 |
| `StartDecommission` | `POST` | `/pools/decommission?pool={pool}` | 启动存储池退役 |
| `CancelDecommission` | `POST` | `/pools/cancel?pool={pool}` | 取消存储池退役 |
| `RebalanceStart` | `POST` | `/rebalance/start` | 启动数据均衡 |
| `RebalanceStatus` | `GET` | `/rebalance/status` | 获取均衡状态 |
| `RebalanceStop` | `POST` | `/rebalance/stop` | 停止数据均衡 |

### 3.10 修复 (Heal) 操作

仅在纠删码模式下可用。

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `Heal` | `POST` | `/heal/` | 修复整个集群 |
| `Heal` | `POST` | `/heal/{bucket}` | 修复指定 Bucket |
| `Heal` | `POST` | `/heal/{bucket}/{prefix}` | 修复指定路径 |
| `BackgroundHealStatus` | `POST` | `/background-heal/status` | 后台修复状态 |

### 3.11 分批任务 (Batch Jobs)

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `StartBatchJob` | `POST` | `/start-job` | 启动分批任务 |
| `ListBatchJobs` | `GET` | `/list-jobs` | 列出分批任务 |
| `BatchJobStatus` | `GET` | `/status-job` | 查询任务状态 |
| `DescribeBatchJob` | `GET` | `/describe-job` | 描述任务详情 |
| `CancelBatchJob` | `DELETE` | `/cancel-job` | 取消任务 |

支持的任务类型: `replicate` (批量复制), `keyrotate` (密钥轮换), `expire` (批量过期)

### 3.12 Bucket 元数据迁移

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `ExportBucketMetadata` | `GET` | `/export-bucket-metadata` | 导出 Bucket 元数据 |
| `ImportBucketMetadata` | `PUT` | `/import-bucket-metadata` | 导入 Bucket 元数据 |

### 3.13 远程分层 (Tier) 管理

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `AddTier` | `PUT` | `/tier` | 添加远程分层 |
| `EditTier` | `POST` | `/tier/{tier}` | 编辑远程分层 |
| `ListTier` | `GET` | `/tier` | 列出所有分层 |
| `RemoveTier` | `DELETE` | `/tier/{tier}` | 删除远程分层 |
| `VerifyTier` | `GET` | `/tier/{tier}` | 验证远程分层 |
| `TierStats` | `GET` | `/tier-stats` | 获取分层统计 |

### 3.14 站点复制 (Site Replication)

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `SiteReplicationAdd` | `PUT` | `/site-replication/add` | 添加对等站点 |
| `SiteReplicationRemove` | `PUT` | `/site-replication/remove` | 移除对等站点 |
| `SiteReplicationInfo` | `GET` | `/site-replication/info` | 站点复制信息 |
| `SiteReplicationMetaInfo` | `GET` | `/site-replication/metainfo` | 站点复制元信息 |
| `SiteReplicationStatus` | `GET` | `/site-replication/status` | 站点复制状态 |
| `SiteReplicationEdit` | `PUT` | `/site-replication/edit` | 编辑站点复制 |
| `SiteReplicationResyncOp` | `PUT` | `/site-replication/resync/op?operation={op}` | 站点重新同步操作 |

站点间内部通信 Peer API:

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `SRPeerJoin` | `PUT` | `/site-replication/peer/join` | 对等站点加入 |
| `SRPeerBucketOps` | `PUT` | `/site-replication/peer/bucket-ops?bucket={b}&operation={op}` | Bucket 同步操作 |
| `SRPeerReplicateIAMItem` | `PUT` | `/site-replication/peer/iam-item` | IAM 项同步 |
| `SRPeerReplicateBucketItem` | `PUT` | `/site-replication/peer/bucket-meta` | Bucket 元数据同步 |
| `SRPeerGetIDPSettings` | `GET` | `/site-replication/peer/idp-settings` | 获取 IDP 设置 |
| `SRPeerEdit` | `PUT` | `/site-replication/peer/edit` | 对等站点编辑 |
| `SRPeerRemove` | `PUT` | `/site-replication/peer/remove` | 对等站点移除 |
| `SRStateEdit` | `PUT` | `/site-replication/state/edit` | 站点状态编辑 |

### 3.15 性能测试

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `ObjectSpeedTest` | `POST` | `/speedtest` 或 `/speedtest/object` | 对象 I/O 性能测试 |
| `DriveSpeedtest` | `POST` | `/speedtest/drive` | 磁盘驱动性能测试 |
| `Netperf` | `POST` | `/speedtest/net` | 网络性能测试 |
| `SitePerf` | `POST` | `/speedtest/site` | 站点间性能测试 |
| `ClientDevNull` | `POST` | `/speedtest/client/devnull` | 客户端空设备测试 |
| `ClientDevNullExtraTime` | `POST` | `/speedtest/client/devnull/extratime` | 客户端空设备扩展测试 |
| `SiteReplicationDevNull` | `POST` | `/site-replication/devnull` | 站点复制空设备测试 |
| `SiteReplicationNetPerf` | `POST` | `/site-replication/netperf` | 站点间网络性能测试 |

### 3.16 诊断与调试

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `Metrics` | `GET` | `/metrics` | Admin 级别指标 |
| `Trace` | `GET` | `/trace` | HTTP 调用跟踪 (SSE 流) |
| `ConsoleLog` | `GET` | `/log` | 控制台日志 (SSE 流) |
| `Profile` | `POST` | `/profile` | 启动性能剖析 |
| `StartProfiling` | `POST` | `/profiling/start?profilerType={type}` | 启动剖析（已废弃） |
| `DownloadProfiling` | `GET` | `/profiling/download` | 下载剖析数据 |
| `TopLocks` | `GET` | `/top/locks` | 顶级锁信息 (分布式模式) |
| `ForceUnlock` | `POST` | `/force-unlock?paths={paths}` | 强制解锁路径 |
| `HealthInfo` | `GET` | `/healthinfo` 或 `/obdinfo` | 获取健康诊断信息 |

### 3.17 KMS 管理 (Admin 路径)

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `KMSStatus` | `POST` | `/kms/status` | KMS 状态 |
| `KMSCreateKey` | `POST` | `/kms/key/create?key-id={id}` | 创建 KMS 密钥 |
| `KMSKeyStatus` | `GET` | `/kms/key/status` | 获取密钥状态 |

### 3.18 Token 吊销

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `RevokeTokens` | `POST` | `/revoke-tokens/{userProvider}` | 吊销用户所有 Token |

---

## 4. STS 安全令牌服务 API

MinIO 实现了 AWS STS 兼容的安全令牌服务，支持多种身份认证方式。

### 4.1 接口信息

- **API 版本**: `2011-06-15` (兼容 AWS STS)
- **路径**: 基于根路径，通过 Query 参数区分
- **请求格式**: `application/x-www-form-urlencoded` 或 JSON
- **响应格式**: XML (与 AWS STS 兼容)

### 4.2 STS Actions

| Action | Query 参数 | 认证方式 | 说明 |
|--------|-----------|---------|------|
| `AssumeRoleWithClientGrants` | `?Action=AssumeRoleWithClientGrants&Version=2011-06-15&Token={token}` | OpenID ClientGrants | 使用 OpenID Client Grants 换取临时凭证 |
| `AssumeRoleWithWebIdentity` | `?Action=AssumeRoleWithWebIdentity&Version=2011-06-15&WebIdentityToken={token}` | OpenID WebIdentity | 使用 Web Identity Token 换取临时凭证 |
| `AssumeRoleWithLDAPIdentity` | `?Action=AssumeRoleWithLDAPIdentity&Version=2011-06-15&LDAPUsername={user}&LDAPPassword={pwd}` | LDAP | 使用 LDAP 凭证换取临时凭证 |
| `AssumeRoleWithCertificate` | `?Action=AssumeRoleWithCertificate&Version=2011-06-15` | TLS 客户端证书 | 使用 TLS 证书认证 |
| `AssumeRoleWithCustomToken` | `?Action=AssumeRoleWithCustomToken&Version=2011-06-15` | 自定义 Token | 使用自定义 Token 认证 |
| `AssumeRoleWithSSO` | (自动检测，无 Query) | JWT Bearer Token | SSO 统一入口 |
| `AssumeRole` | (自动检测，Signature V4) | AWS Signature V4 | 标准 STS AssumeRole |

### 4.3 STS 响应格式

```xml
<AssumeRoleWithWebIdentityResponse xmlns="https://sts.amazonaws.com/doc/2011-06-15/">
  <AssumeRoleWithWebIdentityResult>
    <Credentials>
      <AccessKeyId>...</AccessKeyId>
      <SecretAccessKey>...</SecretAccessKey>
      <SessionToken>...</SessionToken>
      <Expiration>...</Expiration>
    </Credentials>
  </AssumeRoleWithWebIdentityResult>
</AssumeRoleWithWebIdentityResponse>
```

---

## 5. Console 管理控制台

MinIO 内嵌了一个基于 Web 的图形化管理界面（MinIO Console）。

### 5.1 接口信息

- **默认端口**: 9090 (通过 `--console-address` 配置)
- **协议**: HTTP/HTTPS
- **访问地址**: `http://{host}:9090`
- **功能**: Bucket 管理、对象浏览、用户管理、策略配置、监控面板、日志查看等

### 5.2 Console 特性

- 基于 React 的 SPA (Single Page Application)
- 支持登录认证
- 提供可视化的监控仪表板
- 支持对象浏览器和上传/下载
- 集成 Admin API 的所有管理功能

---

## 6. Metrics 监控指标接口

MinIO 提供了 Prometheus 兼容的指标采集端点。

### 6.1 接口信息

- **路径前缀**: `/minio`
- **认证方式**: 默认 JWT（可通过环境变量 `MINIO_PROMETHEUS_AUTH_TYPE=public` 设为公开访问）

### 6.2 指标端点

| 端点 | 版本 | 说明 |
|------|------|------|
| `/minio/prometheus/metrics` | Legacy | 旧版 Prometheus 指标 |
| `/minio/v2/metrics/cluster` | v2 | 集群级指标 |
| `/minio/v2/metrics/bucket` | v2 | Bucket 级指标 |
| `/minio/v2/metrics/node` | v2 | 节点级指标 |
| `/minio/v2/metrics/resource` | v2 | 资源使用指标 |
| `/minio/metrics/v3/{path}` | v3 | 新版指标 API，支持按类别查询 |

### 6.3 Metrics v3 指标类别

v3 版本按功能模块分类提供指标：

- **API**: API 调用统计
- **Audit**: 审计日志指标
- **Bucket Replication**: Bucket 复制指标
- **Cache**: 缓存指标
- **Cluster Config**: 集群配置指标
- **Cluster Erasure Set**: 纠删码集指标
- **Cluster Health**: 集群健康指标
- **Cluster IAM**: IAM 相关指标
- **Cluster Notification**: 通知系统指标
- **Cluster Usage**: 集群使用量指标
- **Logger Webhook**: 日志 Webhook 指标
- **Scanner**: 数据扫描指标
- **ILM**: 生命周期管理指标
- **Healing**: 修复指标
- **Batch Jobs**: 批处理任务指标
- **KMS**: 密钥管理指标

### 6.4 访问示例

```bash
# 旧版指标
curl http://minio-server:9000/minio/prometheus/metrics

# v2 集群指标
curl http://minio-server:9000/minio/v2/metrics/cluster

# v3 集群健康指标
curl http://minio-server:9000/minio/metrics/v3/cluster/health

# v3 列出所有可用指标
curl http://minio-server:9000/minio/metrics/v3?list
```

---

## 7. KMS 密钥管理 API

### 7.1 接口信息

- **路径前缀**: `/minio/kms`
- **API 版本**: `v1`

### 7.2 端点列表

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| `KMSStatus` | `GET` | `/minio/kms/v1/status` | KMS 状态 |
| `KMSMetrics` | `GET` | `/minio/kms/v1/metrics` | KMS 指标 |
| `KMSAPIs` | `GET` | `/minio/kms/v1/apis` | 列出支持的 KMS API |
| `KMSVersion` | `GET` | `/minio/kms/v1/version` | KMS 版本信息 |
| `KMSCreateKey` | `POST` | `/minio/kms/v1/key/create?key-id={id}` | 创建密钥 |
| `KMSListKeys` | `GET` | `/minio/kms/v1/key/list?pattern={pattern}` | 列出密钥 |
| `KMSKeyStatus` | `GET` | `/minio/kms/v1/key/status` | 获取密钥状态 |

---

## 8. Health Check 健康检查接口

用于 Kubernetes 等容器编排平台的探活和就绪检测。

### 8.1 端点列表

- **路径前缀**: `/minio/health`

| 端点 | 方法 | 说明 |
|------|------|------|
| `/minio/health/live` | `GET`, `HEAD` | 存活探测 — 服务进程是否运行 |
| `/minio/health/ready` | `GET`, `HEAD` | 就绪探测 — 服务是否准备好接受请求 |
| `/minio/health/cluster` | `GET`, `HEAD` | 集群探测 — 集群是否正常运行 |
| `/minio/health/cluster/read` | `GET`, `HEAD` | 集群读探测 — 集群读操作是否正常 |

### 8.2 Kubernetes 配置示例

```yaml
livenessProbe:
  httpGet:
    path: /minio/health/live
    port: 9000
  initialDelaySeconds: 120
  periodSeconds: 15

readinessProbe:
  httpGet:
    path: /minio/health/ready
    port: 9000
  initialDelaySeconds: 120
  periodSeconds: 15
```

---

## 9. 事件通知系统

MinIO 支持将 Bucket 事件推送到多种外部消息系统。

### 9.1 支持的通知目标

| 目标类型 | 配置键 | 说明 |
|---------|--------|------|
| **AMQP** (RabbitMQ) | `notify_amqp` | 通过 AMQP 协议推送事件 |
| **Elasticsearch** | `notify_elasticsearch` | 写入 Elasticsearch |
| **Kafka** | `notify_kafka` | 通过 Kafka 推送事件 |
| **MQTT** | `notify_mqtt` | 通过 MQTT 推送事件 |
| **MySQL** | `notify_mysql` | 写入 MySQL 数据库 |
| **NATS** | `notify_nats` | 通过 NATS 推送事件 |
| **NSQ** | `notify_nsq` | 通过 NSQ 推送事件 |
| **PostgreSQL** | `notify_postgresql` | 写入 PostgreSQL 数据库 |
| **Redis** | `notify_redis` | 通过 Redis Pub/Sub 推送事件 |
| **Webhook** | `notify_webhook` | HTTP POST 回调 |

### 9.2 支持的事件类型

| 事件类型 | 说明 |
|---------|------|
| `s3:ObjectCreated:*` | 所有对象创建事件 |
| `s3:ObjectCreated:Put` | PUT 对象 |
| `s3:ObjectCreated:Post` | POST 对象上传 |
| `s3:ObjectCreated:Copy` | COPY 对象 |
| `s3:ObjectCreated:CompleteMultipartUpload` | 完成分片上传 |
| `s3:ObjectRemoved:*` | 所有对象删除事件 |
| `s3:ObjectRemoved:Delete` | DELETE 对象 |
| `s3:ObjectRemoved:DeleteMarkerCreated` | 版本删除标记 |
| `s3:ObjectAccessed:*` | 所有对象访问事件 |
| `s3:ObjectAccessed:Get` | GET 对象 |
| `s3:ObjectAccessed:Head` | HEAD 对象 |
| `s3:Replication:*` | 复制事件 |
| `s3:ObjectRestore:*` | 对象恢复事件 |
| `s3:ObjectLifecycle:*` | 生命周期事件 |
| `s3:ObjectTagging:*` | 标签变更事件 |
| `s3:Scanner:*` | 扫描事件 |
| `s3:BucketCreated` | Bucket 创建 |
| `s3:BucketRemoved` | Bucket 删除 |

### 9.3 实时事件监听 (SSE)

可通过 S3 API 接口直接以 Server-Sent Events (SSE) 方式监听事件：

```
GET /{bucket}?events={event1,event2,...}
GET /?events={event1,event2,...}    (监听所有 Bucket)
```

- **认证**: 需要相应的 Bucket 权限
- **传输方式**: SSE (Server-Sent Events)

### 9.4 对象 Lambda 通知

支持配置 AWS Lambda 风格的对象转换：

- **获取触发**: `GET /{bucket}/{object}?lambdaArn={arn}` — 通过 Lambda 函数转换后返回对象

---

## 10. FTP / SFTP 接口

MinIO 支持通过传统 FTP 和 SFTP 协议访问对象存储。

### 10.1 FTP

- **启动参数**: `--ftp="address=:8021" --ftp="passive-port-range=30000-40000"`
- **默认端口**: 8021
- **认证**: 使用 MinIO Access Key / Secret Key 作为用户名/密码

### 10.2 SFTP

- **启动参数**: `--sftp="address=:8022" --sftp="ssh-private-key=${HOME}/.ssh/id_rsa"`
- **默认端口**: 8022
- **认证**: SSH 密钥 或 Access Key/Secret Key 密码认证

### 10.3 支持的操作

| 操作 | 说明 |
|------|------|
| 文件上传/下载 | 与 S3 对象存储双向互通 |
| 目录列表 | 列出 Bucket 和前缀 |
| 文件删除 | 删除对象 |
| 目录操作 | 创建/删除目录（映射为对象前缀） |

---

## 11. 内部节点通信接口

以下接口仅用于分布式部署模式下的节点间内部通信，**不应对外开放**。

### 11.1 Peer REST 接口

- **路径前缀**: `/minio/peer`
- **用途**: 节点间集群信息同步、锁协调、事件传播等
- **传输协议**: HTTP (内网)

### 11.2 Storage REST 接口

- **路径前缀**: `/minio/storage/{diskPath}`
- **用途**: 分布式存储节点间的数据读写操作
- **传输协议**: HTTP (内网)

### 11.3 Grid 通信 (RELEASE.2024+)

新一代高性能内部通信协议：

- **路径**: `/minio/lock` — 分布式锁
- **路径**: `/minio/grid` — 通用 RPC 通信
- **用途**: 替代传统 REST 的高效内部 RPC

### 11.4 Bootstrap REST 接口

- **用途**: 分布式集群的引导启动、节点发现
- **协议**: 内部 HTTP

---

## 12. 认证与授权机制

### 12.1 支持的认证方式

| 认证方式 | 使用场景 | 说明 |
|---------|---------|------|
| **AWS Signature V4** | S3 API | 标准 S3 API 认证 |
| **AWS Signature V2** | S3 API | 遗留兼容 |
| **Presigned URL V4/V2** | S3 API | 预签名 URL |
| **POST Policy Signature V4** | S3 API | 浏览器直传 |
| **JWT Bearer Token** | Admin API / Console | 内部 JWT Token 认证 |
| **OpenID Connect** | STS / Console SSO | 通过 OpenID 提供商认证 |
| **LDAP** | STS | 通过 LDAP/AD 认证 |
| **TLS 客户端证书** | STS | 通过 TLS 证书认证 |
| **Access Key / Secret Key** | FTP/SFTP | 作为 FTP/SFTP 用户凭证 |

### 12.2 授权模型

MinIO 使用基于策略的授权模型，与 AWS IAM 策略语法兼容：

- **IAM Policy**: JSON 格式的访问策略 (与 S3 兼容)
- **Bucket Policy**: Bucket 级别的访问策略
- **内置角色策略**: 预定义的管理角色
- **用户/组策略绑定**: IAM 用户和组的策略管理

### 12.3 STS 临时凭证

通过 STS API 获取的临时凭证包含：
- `AccessKeyId` — 临时访问密钥
- `SecretAccessKey` — 临时秘密密钥
- `SessionToken` — 会话令牌（JWT）
- `Expiration` — 过期时间

临时凭证的有效时间通过 `DurationSeconds` 参数配置（默认 1 小时）。

---

## 13. 错误码体系

### 13.1 S3 API 错误码

MinIO 实现了完整的 AWS S3 兼容错误码（参见 `cmd/api-errors.go`），部分常用错误码如下：

| HTTP 状态码 | 错误码 | 说明 |
|------------|--------|------|
| 400 | `AccessDenied` | 访问被拒绝 |
| 400 | `BadDigest` | 内容 MD5 不匹配 |
| 400 | `InvalidBucketName` | 无效的 Bucket 名称 |
| 400 | `InvalidPart` | 无效的分片 |
| 404 | `NoSuchBucket` | Bucket 不存在 |
| 404 | `NoSuchKey` | 对象不存在 |
| 404 | `NoSuchUpload` | 分片上传不存在 |
| 409 | `BucketAlreadyExists` | Bucket 已存在 |
| 409 | `BucketNotEmpty` | Bucket 非空 |
| 411 | `MissingContentLength` | 缺少 Content-Length |
| 500 | `InternalError` | 内部服务器错误 |
| 503 | `ServiceUnavailable` | 服务不可用 |
| 501 | `NotImplemented` | 功能未实现 |

### 13.2 STS API 错误码

| 错误码 | 说明 |
|--------|------|
| `AccessDenied` | 访问被拒绝 |
| `InvalidParameterValue` | 无效参数 |
| `NotInitialized` | 服务未初始化 |
| `InternalError` | 内部错误 |

### 13.3 Admin API 错误

Admin API 使用 JSON 格式的错误响应：

```json
{
  "Code": "XMinioAdmin...",
  "Message": "error description",
  "Resource": "/minio/admin/v3/..."
}
```

---

## 附录 A: 环境变量参考

| 变量 | 说明 |
|------|------|
| `MINIO_ROOT_USER` | 根用户 Access Key |
| `MINIO_ROOT_PASSWORD` | 根用户 Secret Key |
| `MINIO_PROMETHEUS_AUTH_TYPE` | 指标端点认证类型 (`jwt` / `public`) |
| `MINIO_IDENTITY_OPENID_CONFIG_URL` | OpenID 发现端点 URL |
| `MINIO_IDENTITY_OPENID_CLIENT_ID` | OpenID 客户端 ID |
| `MINIO_IDENTITY_LDAP_SERVER_ADDR` | LDAP 服务端地址 |
| `MINIO_KMS_SECRET_KEY` | KMS 主密钥 |
| `MINIO_NOTIFY_WEBHOOK_ENDPOINT_*` | Webhook 通知端点配置 |
| `MINIO_NOTIFY_KAFKA_BROKERS_*` | Kafka Broker 配置 |
| `MINIO_FTP_ADDRESS` | FTP 监听地址 |
| `MINIO_SFTP_ADDRESS` | SFTP 监听地址 |

---

## 附录 B: 相关资源

- [MinIO 官方文档](https://min.io/docs/)
- [AWS S3 API 参考](https://docs.aws.amazon.com/AmazonS3/latest/API/)
- [AWS STS API 参考](https://docs.aws.amazon.com/STS/latest/APIReference/)
- [MinIO Go Admin SDK (madmin-go)](https://github.com/minio/madmin-go)
- [MinIO Go Client SDK (minio-go)](https://github.com/minio/minio-go)

---

> **文档说明**: 本文档基于 MinIO 源码 (`cmd/api-router.go`, `cmd/admin-router.go`, `cmd/sts-handlers.go`, `cmd/kms-router.go`, `cmd/metrics-router.go`, `cmd/healthcheck-router.go` 等) 分析整理，版本信息请参考 `master` 分支最新提交。
