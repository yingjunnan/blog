---
title: Terraform 快速入门：用 IaC 管理 AWS 资源并模块化创建 DynamoDB
published: 2026-04-27
description: '从 Terraform 基础语法、常用命令到 AWS Provider 实战，使用 module 创建和更新 DynamoDB 表。'
image: '/covers/terraform-aws-dynamodb-cover.png'
tags: ['Terraform', 'AWS', 'DynamoDB', 'IaC']
category: 'AWS'
draft: false
lang: 'ZH_CN'
---

# Terraform 快速入门：用 IaC 管理 AWS 资源并模块化创建 DynamoDB

Terraform 是 HashiCorp 推出的基础设施即代码工具。它把云资源写成声明式配置文件，通过 `plan` 预览变更，再通过 `apply` 创建或更新资源。对 AWS 来说，它很适合管理 VPC、IAM、KMS、DynamoDB、S3、EKS 等资源。

这篇文章用一个小而完整的例子入门：先理解 Terraform 的基础语法，再用 AWS Provider 创建 DynamoDB 表，最后把 DynamoDB 封装成 module，演示如何创建和更新资源。

## 适合谁阅读

- 已经会使用 AWS Console 或 AWS CLI，但想用代码管理资源
- 想快速理解 `.tf` 文件、变量、输出、module、state 的关系
- 想用 Terraform 管理 DynamoDB 表结构、标签、TTL、二级索引

## Terraform 工作流

Terraform 的日常工作流通常是四步：

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
```

各命令的作用如下：

| 命令 | 作用 |
| --- | --- |
| `terraform init` | 初始化工作目录，下载 provider 和 module |
| `terraform fmt` | 格式化 `.tf` 文件 |
| `terraform validate` | 检查语法和配置是否有效 |
| `terraform plan` | 预览 Terraform 将执行的创建、更新、删除 |
| `terraform apply` | 执行变更 |
| `terraform destroy` | 删除当前配置管理的资源 |

建议养成一个习惯：任何变更先看 `plan`，确认无误后再 `apply`。

## Terraform 基础语法

Terraform 配置语言叫 HCL，核心概念并不多。

### 1. Block：配置块

Terraform 用 block 表达一组配置。常见 block 包括 `terraform`、`provider`、`resource`、`variable`、`output`、`module`。

```hcl
resource "aws_dynamodb_table" "orders" {
  name         = "demo-orders"
  billing_mode = "PAY_PER_REQUEST"
}
```

这里的结构是：

```text
resource "<资源类型>" "<本地名称>" {
  参数 = 值
}
```

`aws_dynamodb_table.orders` 是 Terraform 内部引用地址，不一定等于 AWS 上的资源名称。

### 2. Argument：参数

block 内部的 `name = "demo-orders"`、`billing_mode = "PAY_PER_REQUEST"` 都是 argument。

```hcl
region = "ap-southeast-1"
```

### 3. Expression：表达式

Terraform 支持字符串、数字、布尔值、列表、Map、对象、条件表达式和函数调用。

```hcl
name = "${var.project}-${var.environment}-orders"

tags = {
  Project     = var.project
  Environment = var.environment
  ManagedBy   = "Terraform"
}
```

在新版 Terraform 中，简单引用通常不需要 `"${...}"`：

```hcl
table_name = aws_dynamodb_table.orders.name
```

### 4. Variable：变量

变量用于把可变化的值从代码中抽出来。

```hcl
variable "environment" {
  description = "部署环境，例如 dev、staging、prod"
  type        = string
  default     = "dev"
}
```

使用变量：

```hcl
name = "demo-${var.environment}-orders"
```

### 5. Local：本地值

`locals` 适合放组合出来的值，减少重复。

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"

  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```

使用：

```hcl
name = "${local.name_prefix}-orders"
tags = local.common_tags
```

### 6. Output：输出

`output` 用来把关键信息暴露出来，例如表名、ARN。

```hcl
output "orders_table_name" {
  value = aws_dynamodb_table.orders.name
}
```

执行 `terraform apply` 后，终端会显示输出值。

### 7. Provider：云厂商插件

Provider 负责和云 API 通信。管理 AWS 资源时，需要配置 AWS Provider。

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}
```

### 8. State：状态文件

Terraform 会把“代码中的资源”和“云上的真实资源”映射到 state。默认是本地 `terraform.tfstate`，团队协作时通常要改成远程后端，例如 S3 + DynamoDB Lock。

入门阶段可以先用本地 state；生产环境建议使用远程 state，并限制访问权限。

## 项目目录结构

一个适合入门和扩展的目录可以这样组织：

```text
terraform-aws-demo/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars
└── modules/
    └── dynamodb_table/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

顶层目录描述“当前环境需要哪些资源”，`modules/dynamodb_table` 描述“如何创建一个标准 DynamoDB 表”。

## 案例一：直接创建 DynamoDB 表

先看不使用 module 的版本，便于理解资源本身。

### main.tf

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-southeast-1"
}

resource "aws_dynamodb_table" "orders" {
  name         = "demo-dev-orders"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "pk"
  range_key    = "sk"

  attribute {
    name = "pk"
    type = "S"
  }

  attribute {
    name = "sk"
    type = "S"
  }

  ttl {
    attribute_name = "expires_at"
    enabled        = true
  }

  point_in_time_recovery {
    enabled = true
  }

  tags = {
    Project     = "terraform-demo"
    Environment = "dev"
    ManagedBy   = "Terraform"
  }
}
```

### 执行创建

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
```

确认 `plan` 中只包含你期望创建的 DynamoDB 表后，再输入 `yes`。

## 案例二：用 module 创建 DynamoDB 表

直接写资源适合学习，但真实项目里通常会把通用资源封装成 module。module 的价值是统一命名、标签、安全配置和默认能力。

### modules/dynamodb_table/variables.tf

```hcl
variable "name" {
  description = "DynamoDB 表名"
  type        = string
}

variable "hash_key" {
  description = "分区键名称"
  type        = string
}

variable "range_key" {
  description = "排序键名称，可选"
  type        = string
  default     = null
}

variable "attributes" {
  description = "DynamoDB attribute 定义"
  type = list(object({
    name = string
    type = string
  }))
}

variable "ttl_attribute_name" {
  description = "TTL 字段名，留空则不启用 TTL"
  type        = string
  default     = null
}

variable "global_secondary_indexes" {
  description = "全局二级索引配置"
  type = list(object({
    name            = string
    hash_key        = string
    range_key       = optional(string)
    projection_type = optional(string, "ALL")
  }))
  default = []
}

variable "tags" {
  description = "资源标签"
  type        = map(string)
  default     = {}
}
```

### modules/dynamodb_table/main.tf

```hcl
resource "aws_dynamodb_table" "this" {
  name         = var.name
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = var.hash_key
  range_key    = var.range_key

  dynamic "attribute" {
    for_each = var.attributes

    content {
      name = attribute.value.name
      type = attribute.value.type
    }
  }

  dynamic "ttl" {
    for_each = var.ttl_attribute_name == null ? [] : [var.ttl_attribute_name]

    content {
      attribute_name = ttl.value
      enabled        = true
    }
  }

  dynamic "global_secondary_index" {
    for_each = var.global_secondary_indexes

    content {
      name            = global_secondary_index.value.name
      hash_key        = global_secondary_index.value.hash_key
      range_key       = try(global_secondary_index.value.range_key, null)
      projection_type = try(global_secondary_index.value.projection_type, "ALL")
    }
  }

  point_in_time_recovery {
    enabled = true
  }

  tags = var.tags
}
```

这里用到了两个重要语法：

- `dynamic`：用于动态生成重复嵌套块，比如多个 `attribute`、多个 `global_secondary_index`
- `for_each`：遍历集合，为每个元素生成一段配置

### modules/dynamodb_table/outputs.tf

```hcl
output "name" {
  description = "DynamoDB 表名"
  value       = aws_dynamodb_table.this.name
}

output "arn" {
  description = "DynamoDB 表 ARN"
  value       = aws_dynamodb_table.this.arn
}
```

### 顶层 variables.tf

```hcl
variable "aws_region" {
  description = "AWS 区域"
  type        = string
  default     = "ap-southeast-1"
}

variable "project" {
  description = "项目名"
  type        = string
  default     = "terraform-demo"
}

variable "environment" {
  description = "环境名"
  type        = string
  default     = "dev"
}
```

### 顶层 main.tf

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

locals {
  name_prefix = "${var.project}-${var.environment}"

  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

module "orders_table" {
  source = "./modules/dynamodb_table"

  name      = "${local.name_prefix}-orders"
  hash_key  = "pk"
  range_key = "sk"

  attributes = [
    {
      name = "pk"
      type = "S"
    },
    {
      name = "sk"
      type = "S"
    },
    {
      name = "customer_id"
      type = "S"
    }
  ]

  ttl_attribute_name = "expires_at"

  global_secondary_indexes = [
    {
      name      = "customer-index"
      hash_key  = "customer_id"
      range_key = "sk"
    }
  ]

  tags = local.common_tags
}
```

### 顶层 outputs.tf

```hcl
output "orders_table_name" {
  value = module.orders_table.name
}

output "orders_table_arn" {
  value = module.orders_table.arn
}
```

执行：

```bash
terraform init
terraform plan
terraform apply
```

创建完成后，Terraform 会输出表名和 ARN。

## 如何更新 DynamoDB 表

Terraform 的更新方式不是重新执行一段“更新脚本”，而是修改 `.tf` 配置，然后让 Terraform 计算差异。

### 示例 1：增加一个 GSI

假设要按订单状态查询，可以给 module 调用增加 `status-index`：

```hcl
attributes = [
  {
    name = "pk"
    type = "S"
  },
  {
    name = "sk"
    type = "S"
  },
  {
    name = "customer_id"
    type = "S"
  },
  {
    name = "status"
    type = "S"
  }
]

global_secondary_indexes = [
  {
    name      = "customer-index"
    hash_key  = "customer_id"
    range_key = "sk"
  },
  {
    name      = "status-index"
    hash_key  = "status"
    range_key = "sk"
  }
]
```

然后执行：

```bash
terraform plan
terraform apply
```

注意：DynamoDB 的 GSI 创建和回填需要时间，生产环境需要关注表容量、写入流量和索引构建状态。

### 示例 2：更新标签

修改 `local.common_tags`：

```hcl
locals {
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
    Owner       = "platform-team"
  }
}
```

执行 `terraform plan` 后，你会看到 Terraform 只计划更新 tags。

### 示例 3：关闭或更换 TTL 字段

如果不需要 TTL，可以把：

```hcl
ttl_attribute_name = "expires_at"
```

改为：

```hcl
ttl_attribute_name = null
```

或者干脆不传该参数。变更前建议先确认应用是否仍然写入 TTL 字段，以及是否依赖自动过期能力。

## terraform.tfvars 示例

如果不想把环境值写死在 `main.tf`，可以使用 `terraform.tfvars`：

```hcl
aws_region  = "ap-southeast-1"
project     = "order-service"
environment = "dev"
```

执行 `terraform plan` 时，Terraform 会自动读取当前目录下的 `terraform.tfvars`。

也可以为不同环境准备不同变量文件：

```bash
terraform plan -var-file=dev.tfvars
terraform apply -var-file=dev.tfvars
```

## 常见坑

### 1. attribute 不是字段清单

`aws_dynamodb_table` 里的 `attribute` 只需要声明表主键和索引用到的 attribute，不需要把业务里的所有字段都写进去。DynamoDB 是 schemaless 的，普通字段由应用写入。

### 2. 修改 key schema 通常不是普通更新

DynamoDB 表的主键设计非常关键。创建后如果要改 `hash_key` 或 `range_key`，通常意味着需要新建表并迁移数据。不要把主键设计当成可以随时修改的参数。

### 3. 生产环境不要随便 destroy

`terraform destroy` 会删除当前 state 管理的资源。DynamoDB 表里通常有业务数据，生产环境建议启用删除保护策略、备份和变更审批。

### 4. 团队协作要使用远程 state

多人使用本地 state 容易互相覆盖状态。生产项目建议使用 S3 backend，并配合 state lock，避免同时 apply。

## 推荐实践

1. `plan` 必须进 Code Review，把 Terraform diff 当成变更单看。
2. module 保持小而稳定，一个 module 只做一类资源。
3. 变量加上 `type` 和 `description`，降低误用概率。
4. 对生产资源使用清晰标签：`Project`、`Environment`、`Owner`、`ManagedBy`。
5. 不要手动改 Terraform 已管理的资源；如果改了，下一次 `plan` 会出现 drift。
6. 重要资源先测试环境验证，再推广到生产环境。

## 一份最小命令清单

```bash
# 初始化
terraform init

# 格式化
terraform fmt -recursive

# 校验
terraform validate

# 预览
terraform plan -out tfplan

# 执行指定 plan
terraform apply tfplan

# 查看当前 state 管理的资源
terraform state list

# 查看某个资源状态
terraform state show module.orders_table.aws_dynamodb_table.this
```

## 总结

Terraform 入门的关键不是记住所有资源参数，而是理解它的工作方式：

- 用 HCL 描述期望状态
- 用 Provider 调用云 API
- 用 State 记录资源映射
- 用 `plan` 审查差异
- 用 `module` 把可复用的基础设施模式沉淀下来

DynamoDB 是一个很好的练习对象：表名、主键、TTL、PITR、GSI、标签都能覆盖 Terraform 的常见语法。掌握这个例子后，再去管理 S3、KMS、IAM、EKS 等资源，思路基本一致。

## 参考资料

- [Terraform Language Documentation](https://developer.hashicorp.com/terraform/language)
- [Terraform Modules Documentation](https://developer.hashicorp.com/terraform/language/modules)
- [AWS Provider: aws_dynamodb_table](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/dynamodb_table)
- [Amazon DynamoDB Developer Guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html)
