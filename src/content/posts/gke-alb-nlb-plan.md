---
title: GKE 中配置 ALB 与 NLB 暴露服务方案
published: 2026-03-05
description: 在 GKE 中通过 ALB（L7）和 NLB（L4）暴露服务的选型、配置示例、上线流程与排障清单。
image: ''
tags: [GCP, GKE, Kubernetes, ALB, NLB, LoadBalancer]
category: Cloud Platform
draft: false
lang: 'ZH_CN'
---

# GKE 中配置 ALB 与 NLB 暴露服务方案

## 1. 目标与术语

本文给出一套可直接落地的方案，用于在 GKE 中暴露服务：

- **ALB（L7）**：处理 HTTP/HTTPS，支持基于 Host/Path 的路由、TLS 终止、WAF/CDN 等能力。
- **NLB（L4）**：处理 TCP/UDP，适合数据库、MQ、gRPC/TCP、游戏长连接等场景。

在 GKE 中通常对应为：

- **ALB**：`Ingress`（`gce` / `gce-internal`）或 `Gateway API`（`gke-l7-*` GatewayClass）
- **NLB**：`Service type: LoadBalancer`（external/internal passthrough Network Load Balancer）

## 2. 选型建议（先定方向）

| 维度 | ALB（L7） | NLB（L4） |
|---|---|---|
| 协议 | HTTP/HTTPS/HTTP2 | TCP/UDP |
| 路由能力 | Host/Path/Header 等七层路由 | 仅四层转发 |
| TLS 终止 | 支持（证书在 LB） | 一般由后端自己处理 |
| 典型场景 | Web/API 网关、多服务统一入口 | Redis/MySQL/MQ、纯 TCP 服务 |
| GKE 资源 | Ingress 或 Gateway + HTTPRoute | Service(type=LoadBalancer) |

结论：

- 面向 Web/API 流量，优先 **ALB**。
- 面向非 HTTP 协议或只要四层转发，优先 **NLB**。

## 3. 前置条件

1. 已有 GKE 集群（Standard/Autopilot 均可）。
2. 已部署业务 `Deployment`。
3. 需要固定 IP 时，提前预留静态 IP（ALB 外网通常是 global，NLB 通常是 regional）。
4. 共享 VPC 场景要确认防火墙和 IAM 权限。

## 4. 方案 A：ALB 暴露 HTTP/HTTPS

### 4.1 路径 A1：使用 Ingress（兼容面广）

> 关键点：GKE Ingress 仍使用 `kubernetes.io/ingress.class` 注解，不用 `spec.ingressClassName`。

#### Step 1) 服务（Ingress 后端）

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
  annotations:
    cloud.google.com/neg: '{"ingress": true}'
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
```

#### Step 2) 外网 ALB Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-external-ingress
  annotations:
    kubernetes.io/ingress.class: "gce"
    kubernetes.io/ingress.global-static-ip-name: "web-alb-ip"
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /*
        pathType: ImplementationSpecific
        backend:
          service:
            name: web-svc
            port:
              number: 80
```

#### Step 3) 内网 ALB Ingress（仅内网访问）

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-internal-ingress
  annotations:
    kubernetes.io/ingress.class: "gce-internal"
    kubernetes.io/ingress.regional-static-ip-name: "web-ilb-ip"
spec:
  defaultBackend:
    service:
      name: web-svc
      port:
        number: 80
```

### 4.2 路径 A2：使用 Gateway API（新项目推荐）

如果你希望后续做更清晰的流量治理（Route 拆分、团队边界、策略扩展），推荐用 Gateway API。

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
spec:
  gatewayClassName: gke-l7-global-external-managed
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
spec:
  parentRefs:
  - name: web-gateway
  hostnames:
  - "app.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web-svc
      port: 80
```

内网 ALB 时，将 `gatewayClassName` 改为 `gke-l7-rilb`。

## 5. 方案 B：NLB 暴露 TCP/UDP

### 5.1 外网 NLB（External passthrough）

> 建议在 Service 创建时就带上 `cloud.google.com/l4-rbs: "enabled"`，创建 backend service-based NLB。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: tcp-api-external
  annotations:
    cloud.google.com/l4-rbs: "enabled"
    networking.gke.io/load-balancer-ip-addresses: "tcp-api-nlb-ip"
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  selector:
    app: tcp-api
  ports:
  - name: tcp-443
    protocol: TCP
    port: 443
    targetPort: 8443
```

说明：

- `externalTrafficPolicy: Local` 可保留客户端源 IP。
- 若集群版本较老或不使用新注解，可退回 `spec.loadBalancerIP`。

### 5.2 内网 NLB（Internal passthrough）

```yaml
apiVersion: v1
kind: Service
metadata:
  name: tcp-api-internal
  annotations:
    networking.gke.io/load-balancer-type: "Internal"
    networking.gke.io/load-balancer-ip-addresses: "tcp-api-ilb-ip"
spec:
  type: LoadBalancer
  selector:
    app: tcp-api
  ports:
  - name: tcp-5432
    protocol: TCP
    port: 5432
    targetPort: 5432
```

兼容说明：

- 新版 GKE 推荐 `networking.gke.io/load-balancer-type: "Internal"`。
- 老版本也常见 `cloud.google.com/load-balancer-type: "Internal"`。

## 6. 推荐上线流程（可直接执行）

1. 预留静态 IP（按 external/global 或 regional/internal 区分）。
2. 先部署 `Deployment + Service`，确认 Pod 健康检查通过。
3. 部署 Ingress/Gateway（ALB）或 LoadBalancer Service（NLB）。
4. 等待 IP 分配并做连通验证。
5. 配置 DNS（A/AAAA 记录）并灰度切流。
6. 观察 15~30 分钟再全量切换。

## 7. 验证与排障命令

```bash
# ALB
kubectl get ingress -A
kubectl describe ingress web-external-ingress

# Gateway API
kubectl get gateway,httproute -A
kubectl describe gateway web-gateway

# NLB
kubectl get svc -A
kubectl describe svc tcp-api-external

# 事件排查
kubectl get events -A --sort-by=.lastTimestamp | tail -n 50
```

常见问题：

1. **Ingress 一直不出 IP**：检查是否使用了正确 class 注解（`gce` / `gce-internal`）。
2. **NLB 端口不通**：确认防火墙规则、后端 Pod 就绪、`targetPort` 是否匹配容器端口。
3. **加了 l4-rbs 注解但没生效**：若 Service 已创建为旧类型，通常需要重建 Service。
4. **静态 IP 绑定失败**：检查 IP 的作用域（global/regional）是否和 LB 类型匹配。

## 8. 生产建议

- ALB 场景优先开启 HTTPS，结合证书自动化与最小暴露面。
- NLB 场景尽量使用固定 IP，避免变更导致上游白名单失效。
- 配置可观测性：LB 日志、应用指标、SLO 告警（5xx、延迟、连接失败率）。
- 对关键流量启用灰度发布和回滚预案。

## 9. 参考文档（官方）

- [GKE Ingress（ALB）](https://cloud.google.com/kubernetes-engine/docs/how-to/load-balance-ingress)
- [GKE Ingress 概念与限制](https://cloud.google.com/kubernetes-engine/docs/concepts/ingress)
- [GKE Gateway API](https://cloud.google.com/kubernetes-engine/docs/concepts/gateway-api)
- [GKE LoadBalancer Service 概念](https://cloud.google.com/kubernetes-engine/docs/concepts/service-load-balancer)
- [GKE LoadBalancer Service 参数](https://cloud.google.com/kubernetes-engine/docs/concepts/service-load-balancer-parameters)
