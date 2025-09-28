---
title: gke-apiproxy
published: 2025-08-08
description: ''
image: './GKE.webp'
tags: [GCP, Kubernetes, GKE]
category: Cloud Platform
draft: false 
lang: 'ZH_CN'
---


# 制作镜像与部署代理（来自 Google Cloud 文档摘录与整理）

> 本文件整理自 Google Cloud 文档（归档页面）“Creating GKE private clusters with network proxies for controller access” 的 **Creating the Docker image** 和 **Deploying the image and Service** 两个部分，便于笔记与直接复用。
> 原始页面： <https://cloud.google.com/kubernetes-engine/docs/archive/creating-kubernetes-engine-private-clusters-with-net-proxies>

---

## 目标（Objectives）

- Create a GKE private cluster with no external access.
- Create and deploy a Docker image to run the proxy.
- Create a Kubernetes Service to access the proxy.
- Test access to the proxy.

---

## 一、制作 Docker 镜像（Creating the Docker image）

使用下面步骤构建一个名为 `k8s-api-proxy` 的 Kubernetes API 代理镜像（作为到 Kubernetes API server 的正向代理）。

1. 在 Cloud Shell 中创建目录并进入：

```bash
mkdir k8s-api-proxy && cd k8s-api-proxy
```

2. 创建 `Dockerfile`。下面示例以 Alpine 为基础，安装 `privoxy`、`curl`、`jq`，并添加配置文件与启动脚本，暴露端口 8118：

```dockerfile
FROM alpine
RUN apk add -U curl privoxy jq && \
    mv /etc/privoxy/templates /etc/privoxy-templates && \
    rm -rf /var/cache/apk/* /etc/privoxy/* && \
    mv /etc/privoxy-templates /etc/privoxy/templates
ADD --chown=privoxy:privoxy config /etc/privoxy/
ADD --chown=privoxy:privoxy k8s-only.action /etc/privoxy/
ADD --chown=privoxy:privoxy k8s-rewrite-internal.filter /etc/privoxy/
ADD k8s-api-proxy.sh /
EXPOSE 8118/tcp
ENTRYPOINT ["./k8s-api-proxy.sh"]
```

3. 在 `k8s-api-proxy` 目录中创建 `config` 文件，内容示例：

```text
#config directory
confdir /etc/privoxy
# Allow Kubernetes API access only
actionsfile /etc/privoxy/k8s-only.action
# Rewrite https://CLUSTER_IP to https://kubernetes.default
filterfile /etc/privoxy/k8s-rewrite-internal.filter
# Don't show the pod name in errors
hostname k8s-privoxy
# Bind to all interfaces, port :8118
listen-address  :8118
# User cannot click-through a block
enforce-blocks 1
# Allow more than one outbound connection
tolerate-pipelining 1
```

4. 创建 `k8s-only.action`（运行时会由脚本替换 `CLUSTER_IP`）：

```text
# Block everything...
{+block{Not Kubernetes}}
/

# ... except the internal k8s endpoint, which you rewrite (see
# k8s-rewrite-internal.filter).
{+client-header-filter{k8s-rewrite-internal} -block{Kubernetes}}
CLUSTER_IP/
```

5. 创建 `k8s-rewrite-internal.filter`（运行时会由脚本替换 `CLUSTER_IP`）：

```text
CLIENT-HEADER-FILTER: k8s-rewrite-internal
# Rewrite https://CLUSTER_IP to https://kubernetes.default
s@(CONNECT) CLUSTER_IP:443(HTTP/\d\.\d)@$1 kubernetes.default:443 $2@ig
```

6. 创建启动脚本 `k8s-api-proxy.sh`，脚本会读取 ServiceAccount token 获取集群内部 API 地址并替换 `CLUSTER_IP`，然后以非守护进程方式启动 privoxy：

```bash
#!/bin/sh
set -o errexit
set -o pipefail
set -o nounset

# 获取集群内部 IP
export TOKEN=$(cat /run/secrets/kubernetes.io/serviceaccount/token)
INTERNAL_IP=$(curl -H "Authorization: Bearer $TOKEN" -k -SsL https://kubernetes.default/api |
    jq -r '.serverAddressByClientCIDRs[0].serverAddress')

# 替换 CLUSTER_IP
sed -i "s/CLUSTER_IP/${INTERNAL_IP}/g" /etc/privoxy/k8s-rewrite-internal.filter
sed -i "s/CLUSTER_IP/${INTERNAL_IP}/g" /etc/privoxy/k8s-only.action

# 启动 Privoxy（非守护进程模式）
privoxy --no-daemon /etc/privoxy/config
```

7. 赋予脚本执行权限并构建/推送镜像：

```bash
chmod +x k8s-api-proxy.sh

docker build -t gcr.io/$PROJECT_ID/k8s-api-proxy:0.1 .
docker push gcr.io/$PROJECT_ID/k8s-api-proxy:0.1
```

> **备注**：文档中说明为了教程简便，没有为 Kubernetes API server 或代理配置/验证证书。生产环境请正确安装与验证 TLS 证书。

---

## 二、部署镜像与 Service（Deploying the image and Service）

1. 在 Cloud Shell 中 SSH 登录到之前创建的 client VM：

```bash
gcloud compute ssh proxy-temp
```

2. 安装 `kubectl`：

```bash
sudo apt-get install kubectl
```

3. 将项目 ID 保存为环境变量：

```bash
export PROJECT_ID=$(gcloud config list --format="value(core.project)")
```

4. 获取集群凭证（使用内部 IP）：

```bash
gcloud container clusters get-credentials frobnitz \
  --zone us-central1-c --internal-ip
```

5. 创建一个 Deployment 来运行你构建的镜像：

```bash
kubectl run k8s-api-proxy \
  --image=gcr.io/$PROJECT_ID/k8s-api-proxy:0.1 \
  --port=8118
```

6. 创建 `ilb.yaml`（Internal Load Balancer Service）：

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    run: k8s-api-proxy
  name: k8s-api-proxy
  namespace: default
  annotations:
    cloud.google.com/load-balancer-type: "Internal"
spec:
  ports:
  - port: 8118
    protocol: TCP
    targetPort: 8118
  selector:
    run: k8s-api-proxy
  type: LoadBalancer
```

7. 部署 Internal LB Service：

```bash
kubectl create -f ilb.yaml
```

8. 检查 Service 并等待分配 IP：

```bash
kubectl get service/k8s-api-proxy
```

示例输出（当看到 External IP 时代理已准备好）：

```
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
k8s-api-proxy   LoadBalancer   10.24.13.129   10.24.24.3    8118:30282/TCP   2m
```

9. 保存 ILB IP 至环境变量：

```bash
export LB_IP=$(kubectl get service/k8s-api-proxy -o jsonpath='{.status.loadBalancer.ingress[].ip}')
```

10. 保存集群 Controller 的私有端点 IP：

```bash
export CONTROLLER_IP=$(gcloud container clusters describe frobnitz \
  --zone=us-central1-c \
  --format="get(privateClusterConfig.privateEndpoint)")
```

11. 验证通过代理访问 Kubernetes API：

```bash
curl -k -x $LB_IP:8118 https://$CONTROLLER_IP/version
```

示例返回（你的输出会不同）：

```json
{
  "major": "1",
  "minor": "15+",
  "gitVersion": "v1.15.11-gke.5",
  ...
}
```

12. 若需要让 `kubectl` 命令都走该代理，可临时设置环境变量（完成后记得 `unset https_proxy`）：

```bash
export https_proxy=$LB_IP:8118
kubectl get pods
```

---

## 三、清理（简要）

```bash
# 删除 GKE 集群（示例）
gcloud container clusters delete frobnitz
```

---

## 参考来源

- Google Cloud — *Creating GKE private clusters with network proxies for controller access*（归档页面）  
  <https://cloud.google.com/kubernetes-engine/docs/archive/creating-kubernetes-engine-private-clusters-with-net-proxies>
