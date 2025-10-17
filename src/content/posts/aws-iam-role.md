---
title: AWS IAM Role 操作指南
published: 2025-10-17
description: ' 介绍如何使用AWS CLI查看和管理IAM角色，包括查看角色列表、详细信息、附加策略以及策略内容。'
image: ''
tags: ['aws', 'iam', 'role']
category: 'AWS'
draft: false 
lang: ''
---
# AWS IAM Role 操作指南

## 目录
1. [查看IAM角色列表](#查看iam角色列表)
2. [查看角色详细信息](#查看角色详细信息)
3. [查看角色附加策略](#查看角色附加策略)
- [托管策略](#托管策略)
- [内联策略](#内联策略)
4. [查看策略具体内容](#查看策略具体内容)
5. [常用命令速查表](#常用命令速查表)
6. [最佳实践建议](#最佳实践建议)

---

## 查看IAM角色列表

### 列出所有IAM角色
```bash
aws iam list-roles
```

### 仅显示角色名称
```bash
aws iam list-roles --query 'Roles[].RoleName' --output text
```

### 按路径过滤角色
```bash
aws iam list-roles --path-prefix "/service-role/"
```

---

## 查看角色详细信息

### 获取角色完整信息
```bash
aws iam get-role --role-name ROLE_NAME
```

### 查看角色信任关系（Assume Role策略）
```bash
aws iam get-role --role-name ROLE_NAME --query 'Role.AssumeRolePolicyDocument'
```

### 格式化输出信任策略
```bash
aws iam get-role --role-name ROLE_NAME --query 'Role.AssumeRolePolicyDocument' | jq
```

---

## 查看角色附加策略

### 托管策略

#### 列出角色附加的托管策略
```bash
aws iam list-attached-role-policies --role-name ROLE_NAME
```

#### 查看托管策略内容
```bash
# 获取策略ARN
POLICY_ARN=$(aws iam list-attached-role-policies --role-name ROLE_NAME --query 'AttachedPolicies[0].PolicyArn' --output text)

# 获取策略默认版本ID
VERSION_ID=$(aws iam get-policy --policy-arn $POLICY_ARN --query 'Policy.DefaultVersionId' --output text)

# 查看策略文档
aws iam get-policy-version --policy-arn $POLICY_ARN --version-id $VERSION_ID --query 'PolicyVersion.Document'
```

### 内联策略

#### 列出角色内联策略
```bash
aws iam list-role-policies --role-name ROLE_NAME
```

#### 查看内联策略内容
```bash
aws iam get-role-policy --role-name ROLE_NAME --policy-name POLICY_NAME
```

---

## 查看策略具体内容

### 获取策略文档（格式化输出）
```bash
aws iam get-policy-version \
--policy-arn "arn:aws:iam::ACCOUNT_ID:policy/POLICY_NAME" \
--version-id "v1" \
--query 'PolicyVersion.Document' | jq
```

### 示例（查看S3访问策略）
```bash
aws iam get-policy-version \
--policy-arn "arn:aws:iam::675079658645:policy/pp.aws.s3.opr" \
--version-id "v1" \
--query 'PolicyVersion.Document' | jq
```

---

## 常用命令速查表

| 操作 | 命令 |
|------|------|
| 列出所有角色 | `aws iam list-roles` |
| 查看角色详情 | `aws iam get-role --role-name ROLE_NAME` |
| 列出附加的托管策略 | `aws iam list-attached-role-policies --role-name ROLE_NAME` |
| 列出内联策略 | `aws iam list-role-policies --role-name ROLE_NAME` |
| 查看托管策略内容 | `aws iam get-policy-version --policy-arn ARN --version-id VERSION` |
| 查看内联策略内容 | `aws iam get-role-policy --role-name ROLE_NAME --policy-name POLICY_NAME` |
| 查看信任策略 | `aws iam get-role --role-name ROLE_NAME --query 'Role.AssumeRolePolicyDocument'` |

---

## 最佳实践建议

1. **最小权限原则**：只授予角色完成任务所需的最小权限
2. **使用托管策略**：优先使用AWS托管策略而非内联策略，便于统一管理
3. **定期审计**：使用`aws iam get-account-authorization-details`定期审查权限
4. **命名规范**：采用一致的命名规范（如`<service>-<environment>-<purpose>`）
5. **使用条件限制**：在策略中添加条件限制（如IP限制、MFA要求）
6. **删除未使用角色**：定期清理不再使用的角色（`aws iam list-roles`配合手动检查）

---

## 文档使用说明

1. 将`ROLE_NAME`替换为您的实际角色名称
2. 将`ACCOUNT_ID`替换为您的AWS账户ID
3. 安装`jq`工具可获得更好的JSON格式化输出
4. 对于生产环境，建议在测试环境验证命令后再执行

> 提示：可以将此文档保存为`AWS_IAM_Role_Operations.md`，方便随时查阅。

---

## 扩展阅读

- [AWS IAM 官方文档](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)
- [IAM策略语法参考](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_syntax.html)
- [IAM最佳实践](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)