---
id: quickstart
title: 快速开始
description: 在已有的 Kubernetes 集群上安装 ModelSphere，打开 Console，认识界面和菜单，并跑通一次从选模型到程序调用的完整流程。
slug: /quickstart
sidebar_position: 2
---

ModelSphere Console 是推理平台的统一入口：在这里部署模型、试用模型、发放 API 密钥，程序再用密钥通过 OpenAI 兼容接口调用模型。

## 前置条件

下面的命令都在 [modelsphere](https://github.com/modelsphere/modelsphere) 仓库的检出目录里执行。本节对应安装指南的 Part C：在已有的 Kubernetes 集群上安装服务栈。裸机准备、创建集群见[安装指南](https://github.com/modelsphere/modelsphere/blob/main/docs/install.md)。

在你执行命令的那台机器上：

| 要求 | 如何得到 |
|---|---|
| 一个 Kubernetes 集群，并且 `kubectl` 指向它 | `kubectl config current-context` |
| helm ≥ 3.8，helmfile ≥ 1.0 | 自行安装；在 helm v4.3.0 + helmfile 1.8.0 上验证过 |
| helm-diff 插件 | `make helm-deps`。每台机器做一次，不能省略：`helmfile apply` 会调用它 |

集群本身需要满足两点，无论安装哪些组件（见安装指南的 [What your cluster needs](https://github.com/modelsphere/modelsphere/blob/main/docs/install.md#what-your-cluster-needs)）：

| 要求 | 具体是什么 |
|---|---|
| **Kubernetes ≥ 1.25** | 各 chart 中最严格的 `kubeVersion`（kube-prometheus-stack）；不装监控时为 1.21 |
| **一个 CNI** | 任意一个都可以，也可以暂时没有：Part B 建出的集群在第 6 步安装 Cilium 之前没有 CNI。自带 CNI 时关掉 Cilium |

之后要部署模型，还需要（见 [Deploying a model](https://github.com/modelsphere/modelsphere/blob/main/docs/deploy-a-model.md#before-you-start)）：

- GPU 节点要提供 `nvidia.com/gpu`：通过你自己的 device plugin，或者设置 `enabled.gpuOperator: true`。
- **模型权重放在每个 GPU 节点的本地磁盘上**，每个模型一个目录，例如 `/mnt/disk0/models/<org>/<model>`。`model.localPath` 是 hostPath，所以不支持共享存储（NFS、CephFS）。

## 安装服务栈

:::warning
这一步会写入集群，默认值是给从零搭建的集群用的。`environments/default.yaml` 会打开十八个组件，其中包括 CNI、GPU operator、调度器和监控栈。集群里已经有的，先在环境文件里关掉——[On a cluster you already have](https://github.com/modelsphere/modelsphere/blob/main/docs/install.md#on-a-cluster-you-already-have) 就是可以照抄的那一块。
:::

可选：先运行 `make helm-bootstrap`，安装 Gateway API CRD 和 PriorityClasses。以下三点同时成立时可以跳过：你用别的方式暴露路由层；自己的 values 没有指定 PriorityClass；没有打开 `cilium.gatewayAPI`（默认关闭；打开后 Cilium 也需要这些 CRD）。这两类对象是集群范围的，每个集群安装一次，不是每个人安装一次。

```bash
make helm-bootstrap   # Gateway API CRDs + PriorityClasses
```

然后复制环境文件并按集群修改，登记到 `helmfile.yaml.gotmpl` 的 `environments:` 下面，再一次性安装整个服务栈。只放进 `environments/` 目录还不够：没有登记的 values 文件，helmfile 不会读。

```bash
# 1. the stack itself, one pass
cp environments/private.yaml.example environments/mycluster.yaml   # edit it,
#    then add it under `environments:` in helmfile.yaml.gotmpl -- a values file
#    nothing registers is a file helmfile never reads:
#      mycluster:
#        values:
#          - environments/default.yaml
#          - environments/mycluster.yaml
make helm-apply ENV=mycluster
```

`helmfile.yaml.gotmpl` 中登记后的样子：

```yaml
environments:
  mycluster:
    values:
      - environments/default.yaml     # the base layer, then your overrides
      - environments/mycluster.yaml
```

环境登记之后，可以先运行 `make helm-diff ENV=mycluster`：它会列出 apply 将要创建的全部内容，什么都不改；确认无误再运行 `make helm-apply ENV=mycluster`。

安装结束后检查：`helm list -A` 里每个启用的 release 都应该是 `deployed`，`kubectl get pods -A` 里除了 `Running`/`Completed` 不应有其他状态。

## 打开 Console

Service 是 `NodePort`，release 起来就能访问，不用先配 Ingress：

```bash
NODE_PORT=$(kubectl -n console get svc console-console -o jsonpath='{.spec.ports[0].nodePort}')
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
echo "http://$NODE_IP:$NODE_PORT/"
```

这也是 chart 自己给出地址的方式，`helm status console -n console` 会把它打印出来。在防火墙后面，做端口转发，再打开 `http://localhost:8080/`：

```bash
kubectl -n console port-forward svc/console-console 8080:8080
```

用 `admin`、密码 `P@88w0rd` 登录。第一次登录会要求设置新密码；默认密码是公开的，强烈建议换掉（见下文「登录」）。如果集群改过命名空间，把上面的 `console` 换成环境文件里的 `namespaces.console`。

## 一次完整的使用流程

```
模型库选模型 -> 部署推理服务 -> 等待就绪 -> Playground 试用 -> 创建 API 密钥 -> 程序调用 /v1
```

| 步骤 | 页面 | 详见 |
|---|---|---|
| 选模型、部署 | 「模型服务 → 模型库」 | [部署模型](deploy.md) |
| 查看部署进度 | 「模型服务 → 推理服务」 | [部署模型](deploy.md) |
| 在线对话 | 「Playground → 对话」 | [Playground](playground.md) |
| 发放密钥 | 「路由 → API 密钥」 | [调用 API](api.md) |
| 管理用户和角色 | 「访问控制 → 用户」 | [访问控制](access-control.md) |

## 登录

- 用管理员发放的用户名和密码登录。安装时自带的管理员账号是 `admin`。
- **首次登录会要求设置新密码**。可以沿用初始密码（会有提示），但建议换一个只有自己知道的密码。
- 之后修改密码：右上角头像 → **修改密码**。新密码至少 8 个字符，页面会实时显示强度。

## 界面

- **左侧菜单**按功能分组。只显示当前账号有权限打开的页面；看不到某个菜单，说明角色没有授予对应权限，请联系管理员。
- **右上角**：
  - 使用文档：随时打开本文档。
  - 语言图标：在中文和英文之间切换。
  - 调色板图标：偏好设置，可选「混合导航 / 经典 / 极简」三种布局。偏好保存在本机浏览器，只影响当前设备。
  - 头像：查看当前身份、修改密码、退出登录。
- **概览**页显示当前会话（用户、分组、角色）和各模块的统计卡片。

## 菜单一览

| 分组 | 菜单 | 用途 |
|---|---|---|
| 模型服务 | 推理服务、模型库、节点、操作记录、站点配置 | 部署和管理推理服务 |
| Playground | 对话 | 在浏览器里直接和模型对话 |
| 路由 | API 密钥 | 为程序发放调用 `/v1` 的密钥 |
| 访问控制 | 用户、登录历史、角色 | 账号与权限管理 |
| 帮助 | 使用文档 | 本文档 |
