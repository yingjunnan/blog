---
title: 使用 Argo Workflows 编排 AI Agent 自动开发与 Review 工作流
published: 2026-04-24
description: 调研 Argo Workflows 是否适合编排 AI Agent 自动接收需求、生成代码、运行测试、创建 PR 与进入人工 Review，并给出一套可落地的 Argo Events + WorkflowTemplate YAML 案例。
image: ''
tags: [Argo Workflows, Argo Events, Kubernetes, AI Agent, DevOps, CI/CD, Workflow]
category: Cloud Platform
draft: false
lang: 'ZH_CN'
---

# 使用 Argo Workflows 编排 AI Agent 自动开发与 Review 工作流

## 摘要

这篇文章整理一个问题：能不能用 `Argo Workflows` 来做 AI Agent 自动化开发流水线，例如自动接收需求、拉代码、让 Agent 修改代码、运行测试、安全扫描、创建 Pull Request，再进入 Review？

结论先说：

**可以做，而且 Argo Workflows 很适合作为这条链路的“编排层”。** 但它不是 AI Agent 框架，也不是代码托管平台。比较合理的定位是：

- `Argo Events`：接收外部事件，例如 GitHub Issue、PR comment、Webhook、消息队列事件；
- `Argo Workflows`：按 DAG/步骤编排每个自动化阶段，并管理状态、重试、日志、产物；
- `AI Agent Runner`：真正执行需求理解、代码修改、测试修复、Review 总结的容器或服务；
- `GitHub/GitLab`：承载 Issue、Branch、Commit、Pull Request、Code Review；
- `Artifact Repository`：保存需求快照、Agent 日志、测试报告、Patch、Review 报告；
- `人工审批点`：通过 Argo `suspend`、代码平台 PR Review、ChatOps 或内部审批系统接住。

如果你已经在 Kubernetes 上运行 CI、批处理、数据任务或平台自动化，那么用 Argo 来编排 AI Agent 工作流是顺手的。它最大的价值不是“让 Agent 更聪明”，而是让 Agent 的每一步可观测、可重试、可审计、可暂停。

本文会从能力判断、推荐架构、落地步骤和完整 YAML 示例四个部分展开。

## 1. Argo Workflows 能不能满足这个需求

### 1.1 适合的部分

Argo Workflows 官方定位是 Kubernetes 上的容器原生工作流引擎，用来编排并行作业。它的核心资源是 `Workflow`，工作流定义在 `spec` 中，执行状态也保存在同一个对象上。一个 Workflow 由多个 `template` 组成，`entrypoint` 指定入口模板。

这正好适合 AI Agent 自动开发链路中的这些动作：

| 需求 | Argo 能力 | 适配方式 |
| --- | --- | --- |
| 接收需求事件 | Argo Events `EventSource` + `Sensor` | 从 Webhook/GitHub/消息队列触发 Workflow |
| 多阶段执行 | `steps` 或 `dag` | 需求解析、规划、编码、测试、Review 分阶段跑 |
| 并行任务 | DAG 依赖 | 单测、lint、安全扫描、文档检查并行执行 |
| 保存中间产物 | `artifacts` | 保存需求 JSON、patch、测试报告、Agent 日志 |
| 参数传递 | `parameters` | repo、branch、issue id、模型名、任务描述 |
| 人工确认 | `suspend` template | 在创建 PR 前或合并前暂停等待人工恢复 |
| 通知收尾 | `onExit` exit handler | 成功/失败都通知 Slack、飞书、GitHub comment |
| 定时巡检 | `CronWorkflow` | 定时扫描待处理需求或重跑失败任务 |
| 操作 Kubernetes 资源 | `resource` template | 动态创建 Job、临时环境、沙箱 Runner |
| 复用流程定义 | `WorkflowTemplate` | 把 AI 开发流程模板化，事件只传参数 |

Argo 文档里明确说明，DAG 可以通过依赖关系表达任务图，并让可并行的任务最大化并发执行；`WorkflowTemplate` 可以把模板保存在集群里并被其他 Workflow 引用；`suspend` 可以让工作流暂停，之后从 CLI、API 或 UI 恢复；`onExit` 可以在成功或失败后统一执行通知/清理逻辑。

### 1.2 不适合单独承担的部分

Argo 本身不解决这些问题：

1. **不会理解需求**  
   需求解析、代码生成、修复测试、Review 判断，要由你的 Agent 程序完成。

2. **不是 Git 平台**  
   分支、commit、PR、review comment、merge policy 仍然应该交给 GitHub/GitLab/Gitea。

3. **不是权限边界本身**  
   Argo 能用 Kubernetes ServiceAccount/RBAC 隔离权限，但 Agent 访问仓库、模型、云资源的权限仍要单独设计。

4. **不天然防止 Agent 乱改代码**  
   需要配合沙箱、最小权限 token、只允许创建 PR、强制测试、人工审批、分支保护等机制。

所以推荐的判断是：

> Argo Workflows 适合做 AI Agent 自动化开发的“流水线控制面”，不适合把它当成完整的 AI 编程平台。

## 2. 推荐链路设计

我们设计一个最小但可扩展的案例：

用户在 GitHub Issue 中打上 `ai-agent` 标签，或通过内部需求系统发起一个 webhook。Argo Events 接收事件后触发 `WorkflowTemplate`，Workflow 自动完成：

1. 解析需求；
2. 克隆目标仓库；
3. 创建工作分支；
4. 调用 AI Agent 生成修改；
5. 运行 lint 和 test；
6. 生成 Review 报告；
7. 若检查通过，创建 Pull Request；
8. 暂停等待人工 Review；
9. 人工恢复后执行收尾通知。

整体链路如下：

```text
GitHub Issue / Webhook
        |
        v
Argo Events EventSource
        |
        v
Argo Events Sensor
        |
        v
WorkflowTemplate: ai-agent-dev-review
        |
        +--> parse-requirement
        +--> clone-repo
        +--> agent-plan
        +--> agent-code
        +--> lint --------+
        +--> test --------+--> generate-review-report
        +--> security ----+
        +--> create-pr
        +--> wait-human-review
        +--> notify
```

这里有几个重要设计取舍：

- **Workflow 只编排，不把 prompt 全塞在 YAML 里**：YAML 里只传任务参数，复杂 prompt 和 Agent 逻辑放到镜像内。
- **Agent 永远在临时分支工作**：不要让 Agent 直接 push 到主分支。
- **PR 是 Review 的主入口**：Argo 的 `suspend` 只是流水线等待点，代码 Review 仍然在 Git 平台完成。
- **失败也要生成报告**：测试失败不代表流程没有价值，Agent 的 patch、日志和失败原因都应该留存。
- **权限要分层**：触发器、Workflow、Agent、Git token、模型 token 分开管理。

## 3. 安装组件

截至 2026-04-24，Argo Workflows GitHub release 页面显示最新版本为 `v4.0.4`。生产环境建议固定版本，不要在安装 YAML 中使用漂移的 `latest`。

### 3.1 安装 Argo Workflows

```bash
kubectl create namespace argo
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/download/v4.0.4/install.yaml
```

检查：

```bash
kubectl get pods -n argo
kubectl get crd | grep workflows.argoproj.io
```

如果只是本地体验，可以先用 quick start 的方式；如果准备给团队用，建议进一步配置：

- Argo Server SSO；
- Workflow Archive；
- Artifact Repository；
- Controller HA；
- namespace 级别 RBAC；
- workflow pod 的默认安全上下文。

### 3.2 安装 Argo Events

Argo Events 用来接收外部事件并触发 Workflow。它需要 `EventBus`、`EventSource`、`Sensor` 三类核心资源。

```bash
kubectl create namespace argo-events
kubectl apply -n argo-events -f https://github.com/argoproj/argo-events/releases/download/v1.9.8/install.yaml
```

创建默认 EventBus：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventBus
metadata:
  name: default
  namespace: argo-events
spec:
  nats:
    native:
      replicas: 3
```

应用：

```bash
kubectl apply -f eventbus.yaml
```

> 官方文档说明，EventBus 是 namespace 级资源，EventSource 和 Sensor 要正常工作，需要所在 namespace 里有 EventBus。

## 4. 准备密钥和权限

### 4.1 Git Token Secret

Agent 需要读取仓库、创建分支、push commit、创建 PR。建议使用权限最小化的 GitHub App token 或细粒度 PAT。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ai-agent-git-token
  namespace: argo
type: Opaque
stringData:
  token: ghp_xxx_replace_me
```

### 4.2 模型 API Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ai-agent-model-token
  namespace: argo
type: Opaque
stringData:
  apiKey: sk-xxx_replace_me
```

### 4.3 Workflow 执行 ServiceAccount

下面给一个最小示例。真实生产里建议按 namespace 和 workflow 类型拆分权限。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ai-agent-workflow
  namespace: argo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ai-agent-workflow
  namespace: argo
rules:
  - apiGroups: ["argoproj.io"]
    resources: ["workflows", "workflowtemplates"]
    verbs: ["get", "list", "watch", "create", "patch"]
  - apiGroups: [""]
    resources: ["pods", "pods/log", "secrets", "configmaps"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ai-agent-workflow
  namespace: argo
subjects:
  - kind: ServiceAccount
    name: ai-agent-workflow
    namespace: argo
roleRef:
  kind: Role
  name: ai-agent-workflow
  apiGroup: rbac.authorization.k8s.io
```

## 5. WorkflowTemplate：AI 自动开发主流程

下面是核心模板。为了让示例容易理解，假设你已经有一个 Agent 镜像：

```text
ghcr.io/example/ai-agent-runner:0.1.0
```

这个镜像内部提供几个命令：

- `agent parse-requirement`
- `agent clone`
- `agent plan`
- `agent code`
- `agent lint`
- `agent test`
- `agent security-scan`
- `agent review-report`
- `agent create-pr`
- `agent notify`

真实落地时，可以用你自己的 Python/Node/Go 程序实现这些命令。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: ai-agent-dev-review
  namespace: argo
spec:
  serviceAccountName: ai-agent-workflow
  entrypoint: main
  onExit: exit-notify
  arguments:
    parameters:
      - name: repo_url
      - name: base_branch
        value: main
      - name: requirement_id
      - name: requirement_text
      - name: requester
        value: unknown
      - name: model
        value: gpt-4.1
      - name: dry_run
        value: "false"

  artifactRepositoryRef:
    configMap: artifact-repositories
    key: default-v1

  templates:
    - name: main
      dag:
        failFast: false
        tasks:
          - name: parse-requirement
            template: parse-requirement

          - name: clone-repo
            dependencies: [parse-requirement]
            template: clone-repo
            arguments:
              artifacts:
                - name: requirement
                  from: "{{tasks.parse-requirement.outputs.artifacts.requirement}}"

          - name: agent-plan
            dependencies: [clone-repo]
            template: agent-plan
            arguments:
              artifacts:
                - name: workspace
                  from: "{{tasks.clone-repo.outputs.artifacts.workspace}}"
                - name: requirement
                  from: "{{tasks.parse-requirement.outputs.artifacts.requirement}}"

          - name: agent-code
            dependencies: [agent-plan]
            template: agent-code
            arguments:
              artifacts:
                - name: workspace
                  from: "{{tasks.clone-repo.outputs.artifacts.workspace}}"
                - name: plan
                  from: "{{tasks.agent-plan.outputs.artifacts.plan}}"

          - name: lint
            dependencies: [agent-code]
            template: lint
            arguments:
              artifacts:
                - name: workspace
                  from: "{{tasks.agent-code.outputs.artifacts.workspace}}"

          - name: test
            dependencies: [agent-code]
            template: test
            arguments:
              artifacts:
                - name: workspace
                  from: "{{tasks.agent-code.outputs.artifacts.workspace}}"

          - name: security-scan
            dependencies: [agent-code]
            template: security-scan
            arguments:
              artifacts:
                - name: workspace
                  from: "{{tasks.agent-code.outputs.artifacts.workspace}}"

          - name: generate-review-report
            dependencies: [lint, test, security-scan]
            template: generate-review-report
            arguments:
              artifacts:
                - name: workspace
                  from: "{{tasks.agent-code.outputs.artifacts.workspace}}"
                - name: lint-report
                  from: "{{tasks.lint.outputs.artifacts.lint-report}}"
                - name: test-report
                  from: "{{tasks.test.outputs.artifacts.test-report}}"
                - name: security-report
                  from: "{{tasks.security-scan.outputs.artifacts.security-report}}"

          - name: create-pr
            dependencies: [generate-review-report]
            template: create-pr
            when: "{{workflow.parameters.dry_run}} == false"
            arguments:
              artifacts:
                - name: workspace
                  from: "{{tasks.agent-code.outputs.artifacts.workspace}}"
                - name: review-report
                  from: "{{tasks.generate-review-report.outputs.artifacts.review-report}}"

          - name: wait-human-review
            dependencies: [create-pr]
            template: wait-human-review

          - name: post-review-summary
            dependencies: [wait-human-review]
            template: post-review-summary

    - name: parse-requirement
      container:
        image: ghcr.io/example/ai-agent-runner:0.1.0
        command: [sh, -c]
        args:
          - |
            agent parse-requirement \
              --id "$REQUIREMENT_ID" \
              --text "$REQUIREMENT_TEXT" \
              --requester "$REQUESTER" \
              --output /tmp/requirement.json
        env:
          - name: REQUIREMENT_ID
            value: "{{workflow.parameters.requirement_id}}"
          - name: REQUIREMENT_TEXT
            value: "{{workflow.parameters.requirement_text}}"
          - name: REQUESTER
            value: "{{workflow.parameters.requester}}"
      outputs:
        artifacts:
          - name: requirement
            path: /tmp/requirement.json

    - name: clone-repo
      inputs:
        artifacts:
          - name: requirement
            path: /tmp/requirement.json
      container:
        image: ghcr.io/example/ai-agent-runner:0.1.0
        command: [sh, -c]
        args:
          - |
            agent clone \
              --repo "$REPO_URL" \
              --base-branch "$BASE_BRANCH" \
              --work-branch "ai-agent/${REQUIREMENT_ID}" \
              --requirement /tmp/requirement.json \
              --output /workspace/repo
        env:
          - name: REPO_URL
            value: "{{workflow.parameters.repo_url}}"
          - name: BASE_BRANCH
            value: "{{workflow.parameters.base_branch}}"
          - name: REQUIREMENT_ID
            value: "{{workflow.parameters.requirement_id}}"
          - name: GIT_TOKEN
            valueFrom:
              secretKeyRef:
                name: ai-agent-git-token
                key: token
      outputs:
        artifacts:
          - name: workspace
            path: /workspace/repo

    - name: agent-plan
      inputs:
        artifacts:
          - name: workspace
            path: /workspace/repo
          - name: requirement
            path: /tmp/requirement.json
      container:
        image: ghcr.io/example/ai-agent-runner:0.1.0
        command: [sh, -c]
        args:
          - |
            agent plan \
              --repo /workspace/repo \
              --requirement /tmp/requirement.json \
              --model "$MODEL" \
              --output /tmp/plan.md
        env:
          - name: MODEL
            value: "{{workflow.parameters.model}}"
          - name: MODEL_API_KEY
            valueFrom:
              secretKeyRef:
                name: ai-agent-model-token
                key: apiKey
      outputs:
        artifacts:
          - name: plan
            path: /tmp/plan.md

    - name: agent-code
      inputs:
        artifacts:
          - name: workspace
            path: /workspace/repo
          - name: plan
            path: /tmp/plan.md
      container:
        image: ghcr.io/example/ai-agent-runner:0.1.0
        command: [sh, -c]
        args:
          - |
            agent code \
              --repo /workspace/repo \
              --plan /tmp/plan.md \
              --model "$MODEL" \
              --max-files 20 \
              --output /workspace/repo
        env:
          - name: MODEL
            value: "{{workflow.parameters.model}}"
          - name: MODEL_API_KEY
            valueFrom:
              secretKeyRef:
                name: ai-agent-model-token
                key: apiKey
      outputs:
        artifacts:
          - name: workspace
            path: /workspace/repo

    - name: lint
      inputs:
        artifacts:
          - name: workspace
            path: /workspace/repo
      container:
        image: ghcr.io/example/ai-agent-runner:0.1.0
        command: [sh, -c]
        args:
          - |
            agent lint \
              --repo /workspace/repo \
              --output /tmp/lint-report.json
      outputs:
        artifacts:
          - name: lint-report
            path: /tmp/lint-report.json

    - name: test
      inputs:
        artifacts:
          - name: workspace
            path: /workspace/repo
      container:
        image: ghcr.io/example/ai-agent-runner:0.1.0
        command: [sh, -c]
        args:
          - |
            agent test \
              --repo /workspace/repo \
              --output /tmp/test-report.json
      outputs:
        artifacts:
          - name: test-report
            path: /tmp/test-report.json

    - name: security-scan
      inputs:
        artifacts:
          - name: workspace
            path: /workspace/repo
      container:
        image: ghcr.io/example/ai-agent-runner:0.1.0
        command: [sh, -c]
        args:
          - |
            agent security-scan \
              --repo /workspace/repo \
              --output /tmp/security-report.json
      outputs:
        artifacts:
          - name: security-report
            path: /tmp/security-report.json

    - name: generate-review-report
      inputs:
        artifacts:
          - name: workspace
            path: /workspace/repo
          - name: lint-report
            path: /tmp/lint-report.json
          - name: test-report
            path: /tmp/test-report.json
          - name: security-report
            path: /tmp/security-report.json
      container:
        image: ghcr.io/example/ai-agent-runner:0.1.0
        command: [sh, -c]
        args:
          - |
            agent review-report \
              --repo /workspace/repo \
              --lint /tmp/lint-report.json \
              --test /tmp/test-report.json \
              --security /tmp/security-report.json \
              --output /tmp/review-report.md
      outputs:
        artifacts:
          - name: review-report
            path: /tmp/review-report.md

    - name: create-pr
      inputs:
        artifacts:
          - name: workspace
            path: /workspace/repo
          - name: review-report
            path: /tmp/review-report.md
      container:
        image: ghcr.io/example/ai-agent-runner:0.1.0
        command: [sh, -c]
        args:
          - |
            agent create-pr \
              --repo /workspace/repo \
              --base-branch "$BASE_BRANCH" \
              --branch "ai-agent/${REQUIREMENT_ID}" \
              --title "AI Agent: ${REQUIREMENT_ID}" \
              --body-file /tmp/review-report.md
        env:
          - name: BASE_BRANCH
            value: "{{workflow.parameters.base_branch}}"
          - name: REQUIREMENT_ID
            value: "{{workflow.parameters.requirement_id}}"
          - name: GIT_TOKEN
            valueFrom:
              secretKeyRef:
                name: ai-agent-git-token
                key: token

    - name: wait-human-review
      suspend: {}

    - name: post-review-summary
      container:
        image: ghcr.io/example/ai-agent-runner:0.1.0
        command: [sh, -c]
        args:
          - |
            agent notify \
              --status "human-review-finished" \
              --requirement-id "$REQUIREMENT_ID"
        env:
          - name: REQUIREMENT_ID
            value: "{{workflow.parameters.requirement_id}}"

    - name: exit-notify
      container:
        image: ghcr.io/example/ai-agent-runner:0.1.0
        command: [sh, -c]
        args:
          - |
            agent notify \
              --status "{{workflow.status}}" \
              --workflow "{{workflow.name}}" \
              --requirement-id "$REQUIREMENT_ID"
        env:
          - name: REQUIREMENT_ID
            value: "{{workflow.parameters.requirement_id}}"
```

应用模板：

```bash
kubectl apply -f workflowtemplate-ai-agent-dev-review.yaml
```

手动提交一次测试：

```bash
argo submit \
  --from workflowtemplate/ai-agent-dev-review \
  -n argo \
  -p repo_url=https://github.com/example/demo-service.git \
  -p base_branch=main \
  -p requirement_id=ISSUE-123 \
  -p requirement_text='给用户列表接口增加 name 模糊搜索，并补充单元测试' \
  -p requester=junnan
```

当流程跑到 `wait-human-review` 时会暂停。人工 Review 完成后恢复：

```bash
argo resume -n argo @latest
```

## 6. Argo Events：从 Webhook 自动触发

如果需求来自内部系统，可以先用通用 Webhook。请求体约定如下：

```json
{
  "repo_url": "https://github.com/example/demo-service.git",
  "base_branch": "main",
  "requirement_id": "REQ-2026-001",
  "requirement_text": "给订单查询接口增加按状态过滤，并补充测试",
  "requester": "alice"
}
```

### 6.1 EventSource

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: ai-agent-webhook
  namespace: argo-events
spec:
  service:
    ports:
      - port: 12000
        targetPort: 12000
  webhook:
    requirement:
      port: "12000"
      endpoint: /requirements
      method: POST
```

应用：

```bash
kubectl apply -f eventsource-ai-agent-webhook.yaml
```

测试时可以先 port-forward：

```bash
kubectl -n argo-events port-forward svc/ai-agent-webhook-eventsource-svc 12000:12000
```

发送事件：

```bash
curl -X POST http://localhost:12000/requirements \
  -H 'Content-Type: application/json' \
  -d '{
    "repo_url": "https://github.com/example/demo-service.git",
    "base_branch": "main",
    "requirement_id": "REQ-2026-001",
    "requirement_text": "给订单查询接口增加按状态过滤，并补充测试",
    "requester": "alice"
  }'
```

### 6.2 Sensor RBAC

Sensor 需要有权限在 `argo` namespace 创建 Workflow。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ai-agent-sensor
  namespace: argo-events
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ai-agent-sensor-submit-workflow
  namespace: argo
rules:
  - apiGroups: ["argoproj.io"]
    resources: ["workflows", "workflowtemplates"]
    verbs: ["get", "list", "create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ai-agent-sensor-submit-workflow
  namespace: argo
subjects:
  - kind: ServiceAccount
    name: ai-agent-sensor
    namespace: argo-events
roleRef:
  kind: Role
  name: ai-agent-sensor-submit-workflow
  apiGroup: rbac.authorization.k8s.io
```

### 6.3 Sensor

Sensor 从 EventBus 中消费事件，并把 JSON 字段映射为 Workflow 参数。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: ai-agent-requirement-sensor
  namespace: argo-events
spec:
  template:
    serviceAccountName: ai-agent-sensor
  dependencies:
    - name: requirement-webhook
      eventSourceName: ai-agent-webhook
      eventName: requirement
  triggers:
    - template:
        name: submit-ai-agent-workflow
        argoWorkflow:
          operation: submit
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: ai-agent-dev-
                namespace: argo
              spec:
                workflowTemplateRef:
                  name: ai-agent-dev-review
                arguments:
                  parameters:
                    - name: repo_url
                      value: ""
                    - name: base_branch
                      value: main
                    - name: requirement_id
                      value: ""
                    - name: requirement_text
                      value: ""
                    - name: requester
                      value: ""
          parameters:
            - src:
                dependencyName: requirement-webhook
                dataKey: body.repo_url
              dest: spec.arguments.parameters.0.value
            - src:
                dependencyName: requirement-webhook
                dataKey: body.base_branch
              dest: spec.arguments.parameters.1.value
            - src:
                dependencyName: requirement-webhook
                dataKey: body.requirement_id
              dest: spec.arguments.parameters.2.value
            - src:
                dependencyName: requirement-webhook
                dataKey: body.requirement_text
              dest: spec.arguments.parameters.3.value
            - src:
                dependencyName: requirement-webhook
                dataKey: body.requester
              dest: spec.arguments.parameters.4.value
```

应用：

```bash
kubectl apply -f sensor-ai-agent-requirement.yaml
```

验证：

```bash
kubectl get eventsource,sensor,eventbus -n argo-events
kubectl get workflows -n argo
```

## 7. 如果需求来自 GitHub Issue

生产里更常见的是 GitHub Issue/PR comment 触发。链路一样，只是 EventSource 从通用 webhook 换成 GitHub event source。

示意配置：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: github-issue-eventsource
  namespace: argo-events
spec:
  github:
    aiAgentIssue:
      repositories:
        - owner: example
          names:
            - demo-service
      webhook:
        endpoint: /github
        port: "12000"
        method: POST
        url: https://argo-events.example.com
      events:
        - issues
      apiToken:
        name: github-webhook-secret
        key: token
      webhookSecret:
        name: github-webhook-secret
        key: secret
      insecure: false
      active: true
      contentType: json
```

然后在 Sensor 里加 filter，只处理带有 `ai-agent` label 的 issue：

```yaml
dependencies:
  - name: github-issue
    eventSourceName: github-issue-eventsource
    eventName: aiAgentIssue
    filters:
      data:
        - path: body.action
          type: string
          value:
            - opened
            - labeled
        - path: body.issue.labels.#.name
          type: string
          value:
            - ai-agent
```

不同 Argo Events 版本对复杂 JSON path/filter 的表现要以实际测试为准。如果 filter 逻辑变复杂，我更推荐让 Sensor 只做基础过滤，进入第一个 `parse-requirement` 容器后再做严格校验。

## 8. Artifact Repository：保存 Agent 产物

上面的模板使用了 artifact。Argo 官方文档也提醒，运行 artifact 示例前需要先配置 artifact repository。常见选择有：

- S3；
- MinIO；
- GCS；
- Azure Blob；
- OSS；
- 自定义 artifact driver。

一个 S3 示例：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: artifact-repositories
  namespace: argo
  annotations:
    workflows.argoproj.io/default-artifact-repository: default-v1
data:
  default-v1: |
    s3:
      bucket: ai-agent-workflow-artifacts
      endpoint: s3.amazonaws.com
      region: us-east-1
      useSDKCreds: true
```

如果用静态 AK/SK：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: s3-credentials
  namespace: argo
type: Opaque
stringData:
  accessKey: replace_me
  secretKey: replace_me
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: artifact-repositories
  namespace: argo
  annotations:
    workflows.argoproj.io/default-artifact-repository: default-v1
data:
  default-v1: |
    s3:
      bucket: ai-agent-workflow-artifacts
      endpoint: minio.minio.svc.cluster.local:9000
      insecure: true
      accessKeySecret:
        name: s3-credentials
        key: accessKey
      secretKeySecret:
        name: s3-credentials
        key: secretKey
```

Artifact 里建议保存：

- 原始需求快照；
- Agent plan；
- 修改后的 workspace 或 patch；
- lint/test/security 报告；
- PR 描述；
- Agent 对自己修改的解释；
- 失败日志。

这些产物后续可以用于审计、复盘、评估模型效果，也可以喂给下一轮 Agent 修复。

## 9. Review 设计：不要让 Agent 自动合并

AI Agent 自动开发最容易踩坑的点，不是能不能写代码，而是“写完以后谁负责质量”。

建议把自动化等级拆成四档：

| 等级 | 行为 | 适用场景 |
| --- | --- | --- |
| L1 | 只生成建议和 patch，不 push | 高风险仓库、初期试点 |
| L2 | 自动 push 分支并创建 PR | 常规业务仓库 |
| L3 | 自动修复测试失败并更新 PR | 测试体系较完备的仓库 |
| L4 | 自动合并低风险变更 | 文档、配置、小范围低风险代码 |

大多数团队应该从 L2 开始：Agent 可以创建 PR，但不能合并。

在 Argo 里，`suspend` 适合作为“流程暂停点”：

```yaml
- name: wait-human-review
  suspend: {}
```

恢复方式：

```bash
argo resume -n argo <workflow-name>
```

如果想和 GitHub Review 更紧密，可以做一个补充 Sensor：

1. 监听 PR approved 事件；
2. 找到对应 Workflow；
3. 调用 Argo CLI/API resume；
4. 继续执行 merge、notify 或 archive。

不过我更推荐第一阶段先手动 resume。自动关联 PR Review 和 Workflow 状态虽然酷，但会增加不少边界条件。

## 10. 一个更轻量的 Steps 版本

如果你刚开始试，不一定要上 DAG。下面是更容易读的 `steps` 版本：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: ai-agent-simple-
  namespace: argo
spec:
  entrypoint: ai-agent-simple
  arguments:
    parameters:
      - name: requirement_text
        value: "给用户列表接口增加 name 模糊搜索"
  templates:
    - name: ai-agent-simple
      steps:
        - - name: plan
            template: run
            arguments:
              parameters:
                - name: command
                  value: "agent plan --text '{{workflow.parameters.requirement_text}}'"
        - - name: code
            template: run
            arguments:
              parameters:
                - name: command
                  value: "agent code"
        - - name: test
            template: run
            arguments:
              parameters:
                - name: command
                  value: "agent test"
        - - name: create-pr
            template: run
            arguments:
              parameters:
                - name: command
                  value: "agent create-pr"
        - - name: wait-review
            template: wait-review

    - name: run
      inputs:
        parameters:
          - name: command
      container:
        image: ghcr.io/example/ai-agent-runner:0.1.0
        command: [sh, -c]
        args: ["{{inputs.parameters.command}}"]

    - name: wait-review
      suspend: {}
```

`steps` 的好处是简单；DAG 的好处是并行和依赖表达更强。AI 自动开发这种场景，一旦你要并行跑 lint/test/security/doc check，DAG 更合适。

## 11. 生产化建议

### 11.1 沙箱隔离

Agent 执行环境要尽量收紧：

- 使用独立 namespace；
- 禁止特权容器；
- 使用只读根文件系统；
- 限制 CPU/内存；
- 限制出网域名；
- 对每个仓库使用不同 token；
- 不挂载 Kubernetes 高权限 ServiceAccount；
- 不允许访问生产 Secret。

### 11.2 Git 权限

Git token 只给这些权限：

- read repository；
- create branch；
- push 到 `ai-agent/*`；
- create/update Pull Request；
- comment on issue/PR。

不要给：

- push main；
- delete branch；
- admin repo；
- manage secrets；
- bypass branch protection。

### 11.3 质量门禁

建议至少有这些检查：

- 代码格式化；
- lint；
- unit test；
- dependency scan；
- secret scan；
- SAST；
- 变更文件数量限制；
- 大文件检查；
- PR 描述必须包含测试结果；
- CODEOWNERS 强制 Review。

### 11.4 Agent 输出结构化

不要只让 Agent 输出自然语言。建议每一步都输出结构化文件：

```json
{
  "requirement_id": "REQ-2026-001",
  "summary": "增加订单状态过滤",
  "changed_files": [
    "src/orders/controller.ts",
    "src/orders/service.ts",
    "test/orders.service.spec.ts"
  ],
  "risk_level": "medium",
  "tests": {
    "command": "pnpm test",
    "status": "passed"
  },
  "requires_human_attention": true
}
```

这样 Argo 后续步骤、通知系统、审计系统都能消费。

### 11.5 失败重试策略

Argo 支持在模板级别配置 `retryStrategy`。AI Agent 场景里建议分类型处理：

- 网络、模型 API 超时：可以重试；
- lint/test 失败：不要盲目重试，应该进入修复步骤或生成失败报告；
- secret scan 失败：直接停止并通知；
- 创建 PR 失败：可重试，但要避免重复 PR。

示例：

```yaml
retryStrategy:
  limit: 2
  retryPolicy: OnError
  backoff:
    duration: "30s"
    factor: 2
    maxDuration: "5m"
```

## 12. 什么时候不建议用 Argo

下面这些情况，不一定适合引入 Argo：

1. **团队没有 Kubernetes 运维能力**  
   Argo 的优势建立在 Kubernetes 基础设施之上。如果只是一个小仓库自动改代码，GitHub Actions 可能更简单。

2. **流程非常短**  
   只有“收到 issue -> 调 API -> 发评论”三个动作，用 serverless function 或普通 worker 就够了。

3. **需要复杂人工表单审批**  
   Argo 的暂停/恢复够用，但不是审批系统。复杂审批应该接入 Jira、飞书、ServiceNow、Backstage 或自研平台。

4. **Agent 需要长时间交互式开发**  
   Argo 更适合批处理式、阶段式任务。强交互 IDE Agent 不适合完全塞进 Workflow。

## 13. 最小落地路径

如果我是从零建设，会按下面顺序推进：

1. **第 1 周：手动触发 Workflow**  
   先不用 Argo Events，只用 `argo submit --from workflowtemplate/...`，验证 Agent 镜像能完成 clone、plan、code、test、create-pr。

2. **第 2 周：接入 Webhook**  
   用 Argo Events 接内部需求系统或 GitHub Issue 事件，完成自动触发。

3. **第 3 周：补齐 Artifact 和 Review 报告**  
   把每一步产物保存到 S3/MinIO，让 PR 描述包含变更摘要、测试结果、风险提示。

4. **第 4 周：加权限和质量门禁**  
   拆 ServiceAccount、Git token、NetworkPolicy、secret scan、CODEOWNERS。

5. **第 5 周以后：让 Agent 处理失败反馈**  
   对测试失败的 PR，触发二次修复 Workflow，但仍然只更新 PR，不自动合并。

## 14. 总结

Argo Workflows 可以满足 AI Agent 自动化开发工作流的核心编排需求，尤其适合 Kubernetes 环境中的团队。它能把“需求进入、Agent 开发、测试扫描、PR 创建、人工 Review、通知收尾”变成可声明、可观测、可重试、可审计的流水线。

但要记住边界：

- Argo 是编排器，不是 Agent；
- Argo Events 负责事件入口，不负责需求理解；
- Agent 可以写代码，但 Review 和合并策略必须由工程制度兜底；
- 生产可用的关键不在 YAML 多复杂，而在权限、沙箱、产物、质量门禁和人工审批设计。

推荐的第一版落地形态是：

```text
GitHub Issue/Webhook
  -> Argo Events
  -> Argo WorkflowTemplate
  -> Agent 生成分支和 PR
  -> 自动测试/扫描
  -> 人工 Review
  -> 手动合并
```

这条链路足够实用，也给后续更高等级的自动化留下空间。

## 参考资料

- [Argo Workflows 官方文档：Workflow Templates](https://argo-workflows.readthedocs.io/en/latest/workflow-templates/)
- [Argo Workflows 官方文档：DAG](https://argo-workflows.readthedocs.io/en/latest/walk-through/dag/)
- [Argo Workflows 官方文档：Artifacts](https://argo-workflows.readthedocs.io/en/latest/walk-through/artifacts/)
- [Argo Workflows 官方文档：Exit handlers](https://argo-workflows.readthedocs.io/en/latest/walk-through/exit-handlers/)
- [Argo Workflows 官方文档：Cron Workflows](https://argo-workflows.readthedocs.io/en/latest/cron-workflows/)
- [Argo Workflows 官方文档：Core Concepts](https://argo-workflows.readthedocs.io/en/release-3.4/workflow-concepts/)
- [Argo Workflows 官方文档：Kubernetes Resources](https://argo-workflows.readthedocs.io/en/release-3.7/walk-through/kubernetes-resources/)
- [Argo Events 官方文档：Argo Workflow Trigger](https://argoproj.github.io/argo-events/sensors/triggers/argo-workflow/)
- [Argo Events 官方文档：EventBus](https://argoproj.github.io/argo-events/eventbus/eventbus/)
- [Argo Workflows GitHub Releases](https://github.com/argoproj/argo-workflows/releases)
- [Argo Events GitHub Releases](https://github.com/argoproj/argo-events/releases)
