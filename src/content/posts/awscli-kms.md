---
title: 使用 AWS CLI 创建和管理 KMS 密钥
published: 2025-10-09
description: '详细指南：如何使用 AWS CLI 创建、管理和优化 KMS 密钥'
image: ''
tags: ['AWS', 'KMS']
category: 'aws'
draft: false 
lang: 'ZH_CN'
---

# 使用 AWS CLI 创建和管理 KMS 密钥

AWS Key Management Service (KMS) 提供了安全管理加密密钥的服务，可用于保护您的数据。本文将详细介绍如何使用 AWS CLI 创建和管理 KMS 密钥。

## 前提条件

- 已安装并配置 [AWS CLI](https://aws.amazon.com/cli/)
- 拥有适当的 IAM 权限来创建和管理 KMS 密钥

## 1. 创建 KMS 密钥

创建 KMS 密钥需要提供描述信息和区域：

```bash
aws kms create-key \
  --description "My production encryption key" \
  --region us-west-2
```

执行后会返回详细的密钥元数据：

```json
{
  "KeyMetadata": {
    "AWSAccountId": "123456789012",
    "KeyId": "1234abcd-12ab-34cd-56ef-1234567890ab",
    "Arn": "arn:aws:kms:us-west-2:123456789012:key/1234abcd-12ab-34cd-56ef-1234567890ab",
    "CreationDate": "2023-04-01T12:00:00.000000+00:00",
    "Enabled": true,
    "Description": "My production encryption key",
    "KeyUsage": "ENCRYPT_DECRYPT",
    "KeyState": "Enabled",
    "Origin": "AWS_KMS",
    "KeyManager": "CUSTOMER",
    "CustomerMasterKeySpec": "SYMMETRIC_DEFAULT",
    "EncryptionAlgorithms": [
      "SYMMETRIC_DEFAULT"
    ]
  }
}
```

## 2. 为密钥创建别名

别名使密钥更易于识别和管理。使用上一步返回的 KeyId 创建别名：

```bash
aws kms create-alias \
  --alias-name alias/my-production-key \
  --target-key-id 1234abcd-12ab-34cd-56ef-1234567890ab \
  --region us-west-2
```

## 3. 一步完成创建和别名设置

您可以使用以下脚本在一个操作中完成密钥创建和别名设置：

```bash
#!/bin/bash

# 创建密钥并获取 KeyId
KEY_ID=$(aws kms create-key \
  --description "My production encryption key" \
  --region us-west-2 \
  --query 'KeyMetadata.KeyId' \
  --output text)

# 为密钥创建别名
aws kms create-alias \
  --alias-name alias/my-production-key \
  --target-key-id $KEY_ID \
  --region us-west-2

echo "✅ 已创建 KMS 密钥，ID: $KEY_ID，别名: alias/my-production-key"
```

## 4. 验证创建结果

创建完成后，您可以验证密钥和别名是否正确创建：

```bash
# 通过别名查询密钥详情
aws kms describe-key \
  --key-id alias/my-production-key \
  --region us-west-2

# 列出所有别名
aws kms list-aliases \
  --region us-west-2

# 筛选特定密钥的别名
aws kms list-aliases \
  --key-id 1234abcd-12ab-34cd-56ef-1234567890ab \
  --region us-west-2
```

## 5. 密钥轮换和管理

### 启用自动密钥轮换

```bash
aws kms enable-key-rotation \
  --key-id 1234abcd-12ab-34cd-56ef-1234567890ab \
  --region us-west-2
```

### 检查密钥轮换状态

```bash
aws kms get-key-rotation-status \
  --key-id 1234abcd-12ab-34cd-56ef-1234567890ab \
  --region us-west-2
```

### 更新密钥描述

```bash
aws kms update-key-description \
  --key-id 1234abcd-12ab-34cd-56ef-1234567890ab \
  --description "Updated production encryption key" \
  --region us-west-2
```

## 重要说明

1. **别名规则**：
   - 别名名称必须以 `alias/` 前缀开头
   - 别名在一个 AWS 账户和区域内必须唯一
   - 一个密钥可以关联多个别名
   - 创建别名成功时不会返回任何输出

2. **密钥类型**：
   - 默认创建的是对称加密密钥
   - 若需要非对称密钥，可添加 `--key-spec` 参数

3. **密钥策略**：
   - 默认策略允许根账户完全控制密钥
   - 可通过 `--policy` 参数自定义策略

## 最佳实践

### 命名和组织

- **使用描述性别名**：为密钥创建清晰、有意义的别名，如 `alias/app-name-environment-purpose`
- **环境隔离**：为不同环境（开发、测试、生产）使用不同的密钥，并在别名中明确标识
- **标签管理**：使用标签对密钥进行分类和管理，便于成本分配和权限控制

```bash
# 为密钥添加标签
aws kms tag-resource \
  --key-id 1234abcd-12ab-34cd-56ef-1234567890ab \
  --tags TagKey=Environment,TagValue=Production \
  --region us-west-2
```

### 安全控制

- **最小权限原则**：仅授予应用程序所需的最小权限
- **启用密钥轮换**：对所有生产环境的密钥启用自动轮换
- **监控和审计**：使用 CloudTrail 和 CloudWatch 监控密钥使用情况

```bash
# 创建具有特定权限的密钥策略
aws kms create-key \
  --description "Restricted production key" \
  --policy file://key-policy.json \
  --region us-west-2
```

### 成本优化

- **合理使用别名**：使用别名引用密钥可减少代码修改
- **删除未使用的密钥**：定期审查并计划删除未使用的密钥
- **监控 API 调用**：KMS 按 API 调用次数计费，优化应用程序减少不必要的调用

```bash
# 计划删除不再需要的密钥（默认等待期为30天）
aws kms schedule-key-deletion \
  --key-id 1234abcd-12ab-34cd-56ef-1234567890ab \
  --pending-window-in-days 7 \
  --region us-west-2
```

### 灾难恢复

- **跨区域备份**：考虑在多个区域创建密钥，以支持跨区域灾难恢复
- **密钥材料备份**：对于导入的密钥材料，确保安全备份
- **文档化流程**：记录密钥管理流程，包括创建、轮换和删除程序

## 常见问题排查

- **权限错误**：检查 IAM 策略和密钥策略是否正确配置
- **别名冲突**：确保别名在账户和区域内唯一
- **API 限流**：大量操作时可能遇到 API 限流，实施指数退避重试策略

## 总结

通过 AWS CLI 管理 KMS 密钥提供了灵活且强大的方式来保护您的数据。遵循本文的最佳实践，可以确保您的密钥管理既安全又高效。随着应用规模的增长，考虑使用 AWS CloudFormation 或 Terraform 等基础设施即代码工具来管理密钥，以提高一致性和可重复性。