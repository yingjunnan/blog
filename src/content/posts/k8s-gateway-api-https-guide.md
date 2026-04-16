---
title: Kubernetes 中使用 Gateway API 暴露服务与 HTTPS 证书绑定实战
published: 2026-04-16
description: 从零梳理 Gateway API 的核心资源、服务暴露流程，以及通过 cert-manager + Let's Encrypt 为 Gateway 绑定 HTTPS 证书的完整步骤与落地案例。
image: ''
tags: [Kubernetes, Gateway API, HTTPS, cert-manager, Envoy Gateway, DevOps]
category: Cloud Platform
draft: false
lang: 'ZH_CN'
---

# Kubernetes 中使用 Gateway API 暴露服务与 HTTPS 证书绑定实战

在 Kubernetes 里做服务暴露，很多人第一反应还是 `Service + Ingress`。但随着业务复杂度上升，传统 Ingress 的几个问题会越来越明显：

- 网关能力强依赖控制器实现，标准不统一；
- 路由、监听、网关基础设施耦合较重；
- 多团队协作时，平台侧和应用侧边界不够清晰；
- 对 TCP、TLS、跨命名空间授权等高级场景表达力有限。

而 `Gateway API` 的目标，就是把这些能力做成更清晰、更标准、更适合平台治理的 Kubernetes 原生 API。

这篇文章我会从实战角度说明：

1. `Gateway API` 到底解决了什么问题；
2. 如何在 Kubernetes 中用 `Gateway + HTTPRoute` 暴露服务；
3. 如何结合 `cert-manager` 给 Gateway 绑定 HTTPS 证书；
4. 给出一套可以直接改造落地的案例；
5. 总结上线和排障时最容易踩坑的点。

本文示例选择：

- Gateway Controller：`Envoy Gateway`
- 证书管理：`cert-manager`
- 公网证书签发：`Let's Encrypt HTTP-01`

这是比较通用的一套组合，适合自建 Kubernetes、测试环境以及大多数需要“标准 Gateway API + 自动签证书”能力的场景。

## 1. Gateway API 是什么

Gateway API 是 Kubernetes SIG Network 推进的一组网络 API 标准，用来替代单体化的 Ingress 表达方式。

它把“流量入口基础设施”和“路由规则”拆成了多个资源：

- `GatewayClass`：由具体网关实现提供的“网关类型”
- `Gateway`：实际的入口实例，定义监听端口、协议、TLS 等
- `HTTPRoute`：HTTP 路由规则，定义域名、路径、Header 与后端服务映射
- `TCPRoute` / `TLSRoute` / `GRPCRoute`：面向更多协议的路由能力
- `ReferenceGrant`：跨命名空间引用授权

和 Ingress 相比，可以把它理解成：

- `Gateway` 更像“负载均衡器 / 网关实例”
- `Route` 更像“挂到这个网关上的流量规则”

这种拆分有两个直接好处：

1. 平台团队可以维护 `Gateway`，控制公网入口、监听端口、证书和暴露范围；
2. 业务团队只维护自己的 `HTTPRoute`，按约定接入，不必争抢一份巨大的 Ingress。

## 2. Gateway API 暴露服务的核心链路

一个最基本的服务暴露链路如下：

```text
Client --> LoadBalancer / Gateway IP --> Gateway Listener --> HTTPRoute --> Service --> Pod
```

其中最关键的两个资源是：

### 2.1 Gateway

`Gateway` 定义入口监听，比如：

- 监听 `80` 端口的 HTTP
- 监听 `443` 端口的 HTTPS
- 指定 `hostname`
- 指定 `tls.certificateRefs`
- 指定哪些命名空间的 Route 可以挂上来

### 2.2 HTTPRoute

`HTTPRoute` 定义流量如何转发，比如：

- `example.com` 转发到 `web-svc`
- `foo.example.com/login` 转发到 `foo-svc`
- 满足某个 Header 时，路由到 canary 服务
- 也可以做 redirect、rewrite、流量拆分等

## 3. 为什么推荐用 Gateway API

在实际项目中，Gateway API 最值得用的地方主要有四个：

### 3.1 角色边界更清晰

- 平台团队负责 `GatewayClass`、`Gateway`
- 应用团队负责 `HTTPRoute`

这比所有人都去改一个 Ingress 清晰得多。

### 3.2 表达能力更强

除了普通 HTTP 暴露，还能更自然地表达：

- TLS 终止
- TLS Passthrough
- TCP/UDP/gRPC 暴露
- Header / Query / Method 级路由
- 跨命名空间授权

### 3.3 更标准，不被单一实现绑死

Ingress 在不同控制器之间差异很大，很多高级能力都得靠 annotation。Gateway API 把很多常用能力上升成了标准字段，迁移和治理会更稳。

### 3.4 更适合多租户平台

通过 `allowedRoutes`、`parentRefs`、`ReferenceGrant` 等机制，可以更明确地控制：

- 哪些业务命名空间可以接入某个网关
- 哪些对象可以跨命名空间引用
- 证书 Secret 能不能被别的 namespace 使用

## 4. 前置条件

开始之前，请先准备好以下环境：

1. 一个可正常访问的 Kubernetes 集群；
2. 本地已安装并配置 `kubectl`、`helm`；
3. 有一个可解析到公网 LB IP 的域名，例如 `demo.example.com`；
4. 集群出方向可以访问 Let's Encrypt；
5. 你选择的 Gateway Controller 已支持 Gateway API v1。

注意两点：

- `Gateway API` 只是标准 CRD，不自带实际数据面，必须安装控制器，例如 Envoy Gateway、NGINX Gateway Fabric、Istio、云厂商 Gateway 实现等。
- 如果你使用的是云厂商托管 Gateway，`gatewayClassName` 和证书绑定方式可能略有差异。本文示例使用的是比较通用的 `Envoy Gateway + cert-manager`。

## 5. 安装 Gateway API 与 Envoy Gateway

### 5.1 安装 Gateway API CRDs

```bash
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.1/standard-install.yaml
```

验证：

```bash
kubectl get crd | grep gateway.networking.k8s.io
```

你应该能看到类似下面这些资源：

- `gatewayclasses.gateway.networking.k8s.io`
- `gateways.gateway.networking.k8s.io`
- `httproutes.gateway.networking.k8s.io`
- `referencegrants.gateway.networking.k8s.io`

### 5.2 安装 Envoy Gateway

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.5.4 \
  -n envoy-gateway-system \
  --create-namespace
```

验证控制器状态：

```bash
kubectl get pods -n envoy-gateway-system
kubectl get gatewayclass
```

通常会看到一个由 Envoy Gateway 管理的 `GatewayClass`，其控制器名为：

```text
gateway.envoyproxy.io/gatewayclass-controller
```

## 6. 部署一个示例应用

下面创建一个简单的 Web 应用，稍后通过 Gateway API 暴露出去。

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: demo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginxdemos/hello:plain-text
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-svc
  namespace: demo-app
spec:
  selector:
    app: web
  ports:
  - name: http
    port: 80
    targetPort: 80
```

执行：

```bash
kubectl apply -f app.yaml
kubectl get pod,svc -n demo-app
```

## 7. 通过 Gateway + HTTPRoute 暴露 HTTP 服务

这里我建议把网关资源放到一个独立命名空间，比如 `infra-gateway`，这样更贴近生产治理习惯。

### 7.1 创建 Gateway

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: infra-gateway
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: public-gateway
  namespace: infra-gateway
spec:
  gatewayClassName: envoy-gateway
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All
```

应用：

```bash
kubectl apply -f gateway-http.yaml
kubectl get gateway -n infra-gateway
kubectl get gateway public-gateway -n infra-gateway -o yaml
```

重点观察状态字段：

- `Accepted=True`
- `Programmed=True`
- `status.addresses`

拿到入口地址：

```bash
export GATEWAY_IP=$(kubectl get gateway public-gateway -n infra-gateway -o jsonpath='{.status.addresses[0].value}')
echo $GATEWAY_IP
```

### 7.2 创建 HTTPRoute

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
  namespace: demo-app
spec:
  parentRefs:
  - name: public-gateway
    namespace: infra-gateway
    sectionName: http
  hostnames:
  - demo.example.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web-svc
      port: 80
```

应用：

```bash
kubectl apply -f httproute.yaml
kubectl get httproute -n demo-app
kubectl get httproute web-route -n demo-app -o yaml
```

如果 `HTTPRoute` 成功挂到 `Gateway`，通常会在状态里看到：

- `Accepted=True`
- `ResolvedRefs=True`

### 7.3 验证 HTTP 暴露

如果域名还没有解析，可以先通过 `Host` 头验证：

```bash
curl -H 'Host: demo.example.com' http://${GATEWAY_IP}/
```

如果已经解析好了 DNS，则可以直接：

```bash
curl http://demo.example.com/
```

到这里，`Gateway API` 的基础服务暴露就完成了。

## 8. 给 Gateway 绑定 HTTPS 证书的两种常见思路

在 Gateway API 中做 HTTPS，核心是给 `Gateway listener` 增加 TLS 配置：

```yaml
listeners:
- name: https
  protocol: HTTPS
  port: 443
  hostname: demo.example.com
  tls:
    mode: Terminate
    certificateRefs:
    - kind: Secret
      group: ""
      name: demo-example-com-tls
```

这里的 `certificateRefs` 指向 Kubernetes Secret，Gateway Controller 会读取这个 Secret 中的证书和私钥来完成 TLS 终止。

常见绑定方式有两种：

### 8.1 手工创建 TLS Secret

适合内网、离线环境或已有企业 CA 的场景：

```bash
kubectl -n infra-gateway create secret tls demo-example-com-tls \
  --cert=tls.crt \
  --key=tls.key
```

然后在 Gateway 的 HTTPS listener 里引用该 Secret 即可。

### 8.2 使用 cert-manager 自动签发和续期

这更适合长期运行的生产环境。

典型流程是：

1. 在 `Gateway` 上增加 cert-manager annotation；
2. cert-manager 识别到 HTTPS listener；
3. 根据 `hostname + certificateRefs.secretName` 自动创建 `Certificate`；
4. 签发完成后把证书写入目标 Secret；
5. Gateway Controller 读取 Secret，完成 HTTPS 加载。

本文下面采用的就是这套方式。

## 9. 安装 cert-manager，并开启 Gateway API 支持

### 9.1 安装 cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true \
  --set config.apiVersion="controller.config.cert-manager.io/v1alpha1" \
  --set config.kind="ControllerConfiguration" \
  --set config.enableGatewayAPI=true
```

验证：

```bash
kubectl get pods -n cert-manager
kubectl wait --for=condition=Available deployment -n cert-manager --all --timeout=180s
```

如果你的 Gateway API CRD 是在 cert-manager 之后安装的，建议重启一下 cert-manager：

```bash
kubectl rollout restart deployment cert-manager -n cert-manager
```

## 10. 创建 Let's Encrypt ClusterIssuer

先用 `staging` 环境验证，成功后再切到生产环境，避免因为调试频繁触发速率限制。

### 10.1 Staging Issuer

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: you@example.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging-account-key
    solvers:
    - http01:
        gatewayHTTPRoute:
          parentRefs:
          - group: gateway.networking.k8s.io
            kind: Gateway
            name: public-gateway
            namespace: infra-gateway
```

应用：

```bash
kubectl apply -f clusterissuer-staging.yaml
kubectl get clusterissuer letsencrypt-staging
```

### 10.2 Production Issuer

确认流程完全打通后，再切换成正式签发：

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: you@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - http01:
        gatewayHTTPRoute:
          parentRefs:
          - group: gateway.networking.k8s.io
            kind: Gateway
            name: public-gateway
            namespace: infra-gateway
```

## 11. 给 Gateway 增加 HTTPS Listener 并让 cert-manager 自动签发证书

这里是最关键的一步。

### 11.1 更新 Gateway

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: public-gateway
  namespace: infra-gateway
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging
spec:
  gatewayClassName: envoy-gateway
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All
  - name: https
    protocol: HTTPS
    port: 443
    hostname: demo.example.com
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        group: ""
        name: demo-example-com-tls
    allowedRoutes:
      namespaces:
        from: All
```

应用：

```bash
kubectl apply -f gateway-https.yaml
```

### 11.2 更新 HTTPRoute，让业务挂到 HTTPS Listener

最简单的方式是把 `parentRefs.sectionName` 指到 `https`，也可以同时挂到 `http` 与 `https` 两个 listener。

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
  namespace: demo-app
spec:
  parentRefs:
  - name: public-gateway
    namespace: infra-gateway
    sectionName: https
  hostnames:
  - demo.example.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web-svc
      port: 80
```

### 11.3 验证 cert-manager 是否自动创建证书对象

```bash
kubectl get certificate -A
kubectl get certificaterequest -A
kubectl get order,challenge -A
kubectl describe gateway public-gateway -n infra-gateway
kubectl describe certificate demo-example-com-tls -n infra-gateway
kubectl get secret demo-example-com-tls -n infra-gateway
```

如果一切正常，cert-manager 会自动创建名为 `demo-example-com-tls` 的 `Certificate` 和同名 Secret。

## 12. DNS 与 ACME HTTP-01 校验注意事项

在 Let's Encrypt 正式签发之前，下面几件事必须满足：

1. 域名 `demo.example.com` 已解析到 Gateway 的公网 IP；
2. 外网可以访问你的 `80` 端口；
3. cert-manager 可以在校验时临时创建用于 ACME challenge 的 `HTTPRoute`；
4. Gateway 对这个 HTTP challenge 流量可达。

检查公网地址：

```bash
kubectl get gateway public-gateway -n infra-gateway -o jsonpath='{.status.addresses[0].value}'
```

检查 DNS：

```bash
dig +short demo.example.com
```

两者必须一致，或者域名必须正确指向该 LB 地址。

## 13. 案例：为 demo.example.com 开启 HTTPS 暴露

下面把整套案例串起来看一遍。

### Step 1：应用服务上线

- `demo-app` namespace 中运行 `web` Deployment
- `web-svc` 暴露 80 端口

### Step 2：创建公网 Gateway

- `infra-gateway/public-gateway`
- 暴露 `80` 和 `443`
- 允许所有 namespace 的 Route 接入

### Step 3：创建 HTTPRoute

- `demo-app/web-route`
- 绑定到 `public-gateway`
- `hostnames: [demo.example.com]`
- 后端指向 `web-svc:80`

### Step 4：配置 cert-manager

- 安装 cert-manager
- 开启 `enableGatewayAPI=true`
- 创建 `letsencrypt-staging` ClusterIssuer

### Step 5：在 Gateway 上声明 TLS

- 给 `Gateway` 加 `cert-manager.io/cluster-issuer` annotation
- 新增 `https` listener
- `tls.mode: Terminate`
- `certificateRefs.name: demo-example-com-tls`

### Step 6：等待签发完成

```bash
kubectl get certificate -n infra-gateway
kubectl get secret -n infra-gateway demo-example-com-tls
kubectl get gateway -n infra-gateway public-gateway -o yaml
```

### Step 7：验证 HTTPS

如果 DNS 已生效：

```bash
curl -Iv https://demo.example.com/
```

如果还没做正式解析，也可以先本地强制指定 Host 进行验证：

```bash
curl -Iv --resolve demo.example.com:443:${GATEWAY_IP} https://demo.example.com/
```

正常情况下你会看到：

- 证书链由 Let's Encrypt 签发；
- `HTTP/1.1 200` 或 `HTTP/2 200`；
- 后端返回 `nginxdemos/hello` 页面内容。

## 14. 可选优化：把 HTTP 强制跳转到 HTTPS

很多生产环境会要求 80 端口只做跳转。

可以额外创建一个绑定到 `http` listener 的 `HTTPRoute`：

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: http-to-https-redirect
  namespace: demo-app
spec:
  parentRefs:
  - name: public-gateway
    namespace: infra-gateway
    sectionName: http
  hostnames:
  - demo.example.com
  rules:
  - filters:
    - type: RequestRedirect
      requestRedirect:
        scheme: https
        statusCode: 301
```

应用后，访问：

```bash
curl -I http://demo.example.com/
```

应返回 `301` 或 `302` 跳转到 HTTPS 地址。

## 15. 生产实践中的几个关键细节

### 15.1 Secret 通常要和 Gateway 在同一命名空间

证书 Secret 是给 Gateway listener 用的，而 cert-manager 基于 Gateway annotation 自动创建证书时，默认也是在 Gateway 所在 namespace 里生成 Secret。

所以一个简单、稳定的实践是：

- `Gateway` 放在 `infra-gateway`
- 证书 Secret 也放在 `infra-gateway`
- 业务 `HTTPRoute` 可以在各自 namespace

### 15.2 hostname 不能为空

对于 cert-manager 基于 Gateway 自动签发证书的场景，listener 的 `hostname` 非常重要。没有 hostname，cert-manager 无法正确生成证书的 `dnsNames`。

### 15.3 TLS 模式要用 `Terminate`

如果你希望在 Gateway 上终止 HTTPS，那么 listener 必须类似这样：

```yaml
tls:
  mode: Terminate
```

如果使用 `Passthrough`，则 TLS 不会在 Gateway 终止，而是原样透传给后端，此时证书处理逻辑完全不同。

### 15.4 先用 staging，再切 production

这是强烈建议。

因为 DNS 未生效、80 端口不通、Route 未绑定成功、Gateway 未就绪等问题，在首次调试中都非常常见。先用 staging 可以避免过快触发 Let's Encrypt 限流。

### 15.5 云厂商实现可能有自己的证书集成方式

如果你使用的是 GKE、AKS、EKS 上的托管 Gateway / Load Balancer 产品，通常会有更云原生的证书接入方式，例如：

- 直接引用云证书资源
- 使用厂商管理证书
- 与私有 CA / 证书服务集成

这时 Gateway API 的路由方式仍然成立，但具体证书绑定字段、注解或控制器行为需要看对应实现文档。

## 16. 常见故障排查

### 16.1 Gateway 没有地址

检查：

```bash
kubectl get gateway -A
kubectl describe gateway public-gateway -n infra-gateway
kubectl get pods -n envoy-gateway-system
```

常见原因：

- Gateway Controller 没安装好；
- `gatewayClassName` 写错；
- 云资源创建失败；
- LB 权限不足。

### 16.2 HTTPRoute 没有绑定成功

检查：

```bash
kubectl describe httproute web-route -n demo-app
```

重点看：

- `Accepted`
- `ResolvedRefs`
- `ParentRefs`

常见原因：

- `parentRefs.name` 或 namespace 写错；
- listener 不允许该 namespace 的 Route 接入；
- 后端 Service 名称或端口错误。

### 16.3 证书一直不签发

检查：

```bash
kubectl get certificate,certificaterequest,order,challenge -A
kubectl describe certificate demo-example-com-tls -n infra-gateway
kubectl logs deploy/cert-manager -n cert-manager
```

常见原因：

- `enableGatewayAPI=true` 没开启；
- Gateway API CRD 安装顺序不对；
- 域名未解析到 Gateway 地址；
- 80 端口未对外开放；
- listener 没有 `hostname`；
- `certificateRefs.name` 未配置；
- Gateway 与 Secret namespace 不符合控制器要求。

### 16.4 HTTPS 可以通，但证书不对

检查：

```bash
openssl s_client -connect demo.example.com:443 -servername demo.example.com
```

重点确认：

- 服务出的证书 CN / SAN 是否包含目标域名；
- 是否命中了错误 listener；
- 是否还在用旧 Secret；
- 是否存在多个证书与 hostname 冲突。

## 17. 总结

如果你现在还在用单一 Ingress 做所有服务暴露，那么在下面这些场景里，Gateway API 值得优先考虑：

- 多团队共享入口；
- 要做更清晰的平台治理；
- 需要更强的七层路由表达；
- 希望把 HTTPS、流量规则、命名空间授权拆开治理；
- 后续还要支持 TCP、gRPC、TLS Passthrough 等扩展能力。

落地时，可以记住下面这条最小链路：

1. 安装 Gateway API CRD；
2. 安装 Gateway Controller；
3. 创建 `Gateway`；
4. 创建 `HTTPRoute`；
5. 安装 cert-manager 并开启 Gateway API 支持；
6. 在 `Gateway` 上声明 HTTPS listener 与 `certificateRefs`；
7. 配置 `ClusterIssuer`；
8. 做 DNS 解析和证书验证；
9. 最后加上 HTTP -> HTTPS redirect 与监控告警。

如果你只是想先跑通，我建议直接按本文的 `Envoy Gateway + cert-manager + Let's Encrypt staging` 方案先做一遍。它足够标准、足够通用，而且后续切换到生产证书也很顺滑。

等这套跑通之后，再结合你的云平台实现、WAF、CDN、私有 CA 或多环境发布策略继续增强，会轻松很多。