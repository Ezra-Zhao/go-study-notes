# Kubernetes 核心概念入门笔记

## 1. Kubernetes 是什么

Kubernetes（常缩写为 K8s）是一个开源的容器编排平台，由 Google 在 2014 年开源，目前是云原生领域的事实标准。它的核心定位可以用一句话概括：**自动化容器的部署、扩缩容与管理**。

要理解它为什么存在，可以先看一个对比。容器技术（比如 Docker）解决了"应用在哪里都能一样运行"的问题，但当容器数量从几个变成几百、几千个，分布在多台机器上时，靠人工运维就会失控：谁来决定每个容器跑在哪台机器上？流量高峰时怎么自动加机器？某个容器挂了谁来重启？发布新版本时怎么做到不停机？Kubernetes 就是为解决这一类问题而设计的。

它提供的主要能力包括：

- **服务发现与负载均衡**：给一组容器一个稳定的访问入口，自动把请求分发到健康的实例上。
- **自动扩缩容**：根据 CPU / 内存等指标自动增减副本数，流量来了加、走了减。
- **滚动更新与回滚**：新版本分批替换旧版本，出问题可以一键回到上一个稳定版本。
- **故障自愈**：容器挂了自动重启，节点挂了自动把容器调度到健康节点上。
- **配置与密钥管理**：配置、密码、证书和镜像分开存放，修改配置不用重新打包镜像。
- **存储编排**：自动挂载外部存储，让有状态应用也能跑在集群里。

一句话总结：容器负责"装好一个应用"，Kubernetes 负责"让成千上万个容器可靠地跑起来"。

## 2. 核心概念：Pod

**Pod 是 Kubernetes 里最小的调度和管理单元**，是你要打交道的最基本概念。

- 一个 Pod 里可以跑一个或多个容器，这些容器总是被一起调度到同一台节点上。
- 同一个 Pod 内的容器共享网络（同一个 IP 地址，通过 localhost 互相访问）和存储卷。
- Pod 是临时的：它可以被销毁、重建，IP 地址也可能变化，所以**不要直接依赖 Pod 的 IP 做服务访问**，这是后面 Service 存在的理由。
- 实际使用中很少直接创建 Pod，通常由 Deployment 等更高层控制器来管理。

打个比方：如果容器是一间间独立的房间，Pod 就是把它们圈在一起的一个小院子，院子里的房间共用一个大门（IP）和水电（存储）。

## 3. 核心概念：Deployment

**Deployment 是最常用的应用部署控制器**，用来声明式地管理无状态应用。

它的工作方式是"声明期望状态"：你告诉它"我要 3 个 nginx 副本、镜像用 1.25 版"，它就负责把集群的实际状态收敛到这个期望——副本不够就创建，多了就删掉，挂了就重建。

关键能力：

- **副本管理**：指定 replicas 数量，控制器持续保证这个数量的 Pod 在运行。
- **滚动更新**：改镜像版本后，逐个用新 Pod 替换旧 Pod，服务不中断。
- **回滚**：更新出问题时，用 `kubectl rollout undo` 回到上一个版本。
- **暂停与恢复**：可以暂停更新、检查无误后再继续。

一般规律：凡是"同一份代码跑多个一模一样的副本"的 Web 服务、API 服务，都用 Deployment 来管。

## 4. 核心概念：Service

Pod 的 IP 会变，直接访问 Pod 不可靠。**Service 就是给一组 Pod 提供一个稳定访问入口的抽象**。

它的原理很简单：Service 通过标签选择器（selector）选中一批 Pod，给这批 Pod 一个固定的虚拟 IP（ClusterIP）和 DNS 名称，请求打到 Service 上，它再转发给后端健康的 Pod，天然带负载均衡。后端某个 Pod 挂了，Service 会自动把它从转发列表里摘掉。

常见的 Service 类型：

- **ClusterIP**（默认）：只在集群内部可访问，适合微服务之间的内部调用。
- **NodePort**：在每个节点上开一个固定端口，集群外部可以通过"节点 IP + 端口"访问，适合测试和小规模对外暴露。
- **LoadBalancer**：在公有云上自动申请一个云厂商的负载均衡器，把流量引进来，适合生产环境对外服务。

记忆要点：Pod 负责跑，Deployment 负责管，Service 负责让外面找得到。

## 5. 基本工作流程

一个应用从写好到在 Kubernetes 里跑起来，典型的流程是这样的：

1. **写 YAML 描述文件**：用 YAML 写清楚"我要什么"——Deployment 描述应用的副本数和镜像，Service 描述怎么暴露访问。
2. **提交给集群**：`kubectl apply -f xxx.yaml`，把描述文件发给 API Server。
3. **调度**：Scheduler 看到有新的 Pod 需要创建，根据资源、亲和性等规则选一台合适的节点。
4. **拉起容器**：节点上的 kubelet 收到指令，拉取镜像、启动容器，Pod 开始运行。
5. **持续运维**：Deployment 的控制器不停地做"期望状态 vs 实际状态"的对账，副本挂了就补；Service 则把流量稳定地导向健康的 Pod。
6. **更新发布**：改镜像版本再 apply 一次，Deployment 自动做滚动更新；出问题就回滚。

整个过程你几乎不用登录到具体机器上操作，全靠声明式的描述文件和命令行驱动。

## 6. kubectl 常用命令

kubectl 是和集群交互的命令行工具，下面是入门阶段最常用的命令：

```bash
# 查看资源
kubectl get pods                 # 列出当前命名空间的 Pod
kubectl get pods -A              # 列出所有命名空间的 Pod
kubectl get deployments          # 列出 Deployment
kubectl get services             # 列出 Service

# 查看详情
kubectl describe pod <pod名>     # 看 Pod 的详细信息（排错第一步）
kubectl logs <pod名>             # 看容器日志
kubectl logs <pod名> -f          # 实时跟随日志

# 部署与变更
kubectl apply -f app.yaml        # 按 YAML 创建或更新资源
kubectl delete -f app.yaml       # 按 YAML 删除资源
kubectl delete pod <pod名>       # 删除单个 Pod（会被控制器重建）

# 进入容器
kubectl exec -it <pod名> -- /bin/sh   # 进入 Pod 里的容器排查

# 扩缩容与发布
kubectl scale deployment <名称> --replicas=5   # 手动调整副本数
kubectl rollout status deployment <名称>        # 查看滚动更新进度
kubectl rollout undo deployment <名称>         # 回滚到上一个版本
```

排错小习惯：Pod 起不来时，先 `describe` 看事件（Events），再 `logs` 看容器输出，大部分问题在这两步就能定位。

## 7. 完整示例：Deployment + Service

下面是一个最简但完整可用的例子：部署 3 个 nginx 副本，并用 ClusterIP 类型的 Service 在集群内部暴露 80 端口。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-nginx
  labels:
    app: demo-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo-nginx
  template:
    metadata:
      labels:
        app: demo-nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: demo-nginx-svc
spec:
  type: ClusterIP
  selector:
    app: demo-nginx
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
```

使用方法：

```bash
kubectl apply -f demo.yaml
kubectl get pods -l app=demo-nginx
kubectl get svc demo-nginx-svc
```

几点说明：`selector.matchLabels` 必须和 Pod 模板的 `labels` 一致，Service 的 `selector` 也必须能选中这些 Pod，标签是把三者串起来的纽带；`resources` 里 requests 是调度依据、limits 是上限，新手建议两个都写，养成习惯。

## 8. 小结

- Kubernetes = 容器编排平台，解决"大量容器如何可靠运行"的问题。
- Pod 是最小调度单元，一组共享网络和存储的容器。
- Deployment 管部署：副本数、滚动更新、回滚。
- Service 管访问：给 Pod 一个稳定的入口和负载均衡。
- 日常操作围绕"写 YAML → kubectl apply → 观察 → 更新/回滚"这个闭环。

本文为学习笔记（编纂），用自己的话重写；
