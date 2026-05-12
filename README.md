# OpenClaw Agent 实验手册

本手册将一步步指导你在 AgentRuntime 上部署一个专属的 OpenClaw Agent，并通过浏览器体验完整的会话链路与弹性伸缩过程。

---

## 实验目标

通过本实验，你将：

1. 在共享的 AgentRuntime 集群中，部署一个以自己名字命名的独立 Agent（命名空间隔离）。
2. 观察 Agent 在有请求时被拉起、无请求时自动缩容至 0 的完整生命周期。
3. 使用 OpenClaw 前端与你自己的 Agent 实例发起对话。

---

## 前置准备

### Step 1 — 安装必要的命令行工具

请确保你的机器上已经安装以下命令：

- `git`
- `kubectl`
- `helm`

> 💡 **Windows 用户** 请准备一台 Linux 虚拟机（或使用 WSL2），本实验全部基于 Linux/macOS shell 进行。

你可以通过如下命令快速验证：

```bash
git --version
kubectl version --client
helm version
```

### Step 2 — 上报你的出口 IP

控制台平台需要对出口 IP 做白名单放通。请执行：

```bash
curl cip.cc
```

记录输出中的 **IP 字段**，并将该 IP 告知老师，等待老师加入白名单后再继续下一步。

### Step 3 — 绑定控制台域名到本地 hosts

我们需要访问 AgentRuntime 控制台，地址为：

```
agentruntime.console.aliyun.com
```

由于未做公网 DNS，因此需要本地绑定。编辑 `/etc/hosts`（Linux / macOS），在文件**末尾**添加一行：

```
8.136.18.183 agentruntime.console.aliyun.com
```

> macOS / Linux 保存文件可能需要 `sudo`：
>
> ```bash
> sudo vim /etc/hosts
> ```

保存后打开浏览器访问 <http://agentruntime.console.aliyun.com>，使用以下账号登录：

| 字段 | 值 |
| --- | --- |
| 用户名 | `admin` |
| 密码 | `admin123` |

能看到产品界面即代表控制台访问已就绪。

---

## 部署你的 Agent

### Step 4 — 克隆代码仓库

```bash
# 匿名用户
git clone https://github.com/cloudapp-suites/agentrun-openclaw.git

or 
# 登陆用户
git clone git@github.com:cloudapp-suites/agentrun-openclaw.git

```

### Step 5 — 填写 VLLM API Key 并一键部署

#### 5.1 更新 `values-aliyun.yaml` 中的 `vllmApiKey`

部署之前，**必须**先填写 LLM 接入密钥。编辑下面这个文件：

```
examples/values-aliyun.yaml
```

将其中的 `observability.vllmApiKey` 从占位值改成老师下发的真实密钥：

```yaml
observability:
  # ...其他字段保持不变...
  vllmApiKey: "找老师要"      # ← 改成老师下发的 sk-xxxxxxxx
```

> ⚠️ **不要把真实密钥提交到 git**。实验期间本地修改即可，不要 `git push`。

#### 5.2 更新 `agentrun.config` 中的 ClusterLBIP

`config/agentrun.config` 是实验专用的 kubeconfig，默认 `server` 地址中的 IP 是占位符。部署前**必须**将其替换为老师下发的真实 Cluster LoadBalancer IP。

打开文件：

```
config/agentrun.config
```

找到 `clusters[0].cluster.server` 这一行：

```yaml
clusters:
- cluster:
    server: https://<ClusterLBIP>:6443    # ← 将 <ClusterLBIP> 改为老师下发的 IP
```

把 `<ClusterLBIP>` 替换为老师提供的真实 IP（例如 `47.243.x.x`），保留 `https://` 和 `:6443`。

> ⚠️ **不要把真实 IP / 证书提交到 git**。

#### 5.3 执行部署

进入仓库根目录后，加载实验专用的 kubeconfig 并部署你自己的 Agent：

```bash
cd agentrun-openclaw

export KUBECONFIG=./config/agentrun.config

# <your-name> 为你的名字的拼音，老师最终以这个名字判断实验是否完成。
# 示例：thomas / zhangsan / lixiaoming …
./deploy.sh install <your-name>
```

部署成功后，终端将看到类似输出：

```text
$ ./deploy.sh install thomas

[deploy] Command:    install
[deploy] Namespace:  thomas
[deploy] Release:    openclaw-thomas
[deploy] Kubeconfig: /Users/alick/.kube/agentrun.kubeconfig
Release "openclaw-thomas" does not exist. Installing it now.
NAME: openclaw-thomas
LAST DEPLOYED: Tue May 12 14:06:52 2026
NAMESPACE: thomas
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None
```

> 关键点：Helm Release 名字和 Kubernetes Namespace 都会被自动派生为 `openclaw-<your-name>` / `<your-name>`。

### Step 6 — 查看 Agent 资源

```bash
kubectl get agent -n <your-name>
```

正常情况下的输出如下：

```text
$ kubectl get agent -n thomas
NAME              TYPE      LATESTREVISION                           DESIRED   READY   STATUS      REASON   URL                                                     AGE
openclaw-thomas   managed   openclaw-thomas-ephemeral-1-3baa08390b   0         0       Succeeded            https://latest-openclaw-thomas.thomas.ai.agentrun.com   4m28s
```

请**记下 `URL` 字段**，下一步会用到其中的 host 部分。

### Step 7 — 绑定 Agent 的访问域名到 hosts

将 Step 6 输出的 URL host 再写入本地 `/etc/hosts`。以 `thomas` 为例：

```
8.136.106.88 latest-openclaw-thomas.thomas.ai.agentrun.com
```

请把 `latest-openclaw-thomas.thomas.ai.agentrun.com` 替换为**你实际拿到的 host**。

---

## 体验 OpenClaw

### Step 8 — 打开页面并观察 Pod 弹性

在浏览器中访问 Step 7 中绑定的域名（例如 <http://latest-openclaw-thomas.thomas.ai.agentrun.com>），页面即为你专属的 OpenClaw。

此时打开另一个终端窗口，持续观察 Pod：

```bash
watch -n 2 kubectl get pod -n <your-name>
# 或者反复执行
kubectl get pod -n <your-name>
```

你会看到当浏览器访问到达时，一个新的 Pod 从 0 被拉起；随着你和页面交互，Pod 会保持 Running。

### Step 9 — 配置 Gateway Token 并发起对话

1. 在 OpenClaw 页面中切换到 **Overview** 标签。
2. 在 `Gateway Token` 输入框中填入：

   ```
   cb019cddf41fc030315c46daa1f97761197c930ccb00c790
   ```

3. 保存后切回 **Chat**，即可发起对话。

### Step 10 — 观察无流量时的缩容

停止与页面的交互，等待约 **3 分钟**，继续观察：

```bash
kubectl get pod -n <your-name>
```

你会看到 Pod 数量从 `1 → 0`，即 Agent 在无流量时被自动缩容。

此时**继续在浏览器中发起访问**，Pod 会被重新拉起（0 → 1），会话恢复可用。这就是 Serverless Agent 的冷热切换效果。

---

## 清理

实验完成后，可以卸载自己的 Agent：

```bash
./deploy.sh uninstall <your-name>
```

> 命名空间**不会**随 `uninstall` 一并删除（避免误删同学的资源）。如需彻底回收：
>
> ```bash
> kubectl delete ns <your-name>
> ```

---

## 常用调试命令

| 目的 | 命令 |
| --- | --- |
| 查看部署的 Agent | `kubectl get agent -n <your-name>` |
| 查看当前 Pod 列表 | `kubectl get pod -n <your-name>` |
| 查看 Pod 日志 | `kubectl logs -n <your-name> <pod-name>` |
| 查看 Helm release 状态 | `./deploy.sh status <your-name>` |
| 预览将被渲染的 Helm 模板 | `./deploy.sh template <your-name>` |
| 查看 kubeconfig 连接的集群 | `kubectl cluster-info` |

---

## 常见问题

**Q1. 执行 `./deploy.sh install` 报 `Error: INSTALLATION FAILED`？**

- 检查 `KUBECONFIG` 是否已正确导出：`echo $KUBECONFIG`
- 检查出口 IP 是否已被老师加入白名单
- 确认 `<your-name>` 中不包含大写字母、下划线或其他非法字符（仅允许小写字母、数字和 `-`）

**Q2. 浏览器打不开 `latest-openclaw-<your-name>.<your-name>.ai.agentrun.com`？**

- 确认已在 `/etc/hosts` 中加入 `8.136.106.88 <上述域名>`
- 执行 `ping latest-openclaw-<your-name>.<your-name>.ai.agentrun.com` 验证解析
- 如果 `kubectl get agent` 里 URL 的 host 与你绑定的不一致，以实际输出为准

**Q3. Pod 一直是 0，且访问页面没有拉起？**

- 检查 `kubectl get agent -n <your-name>` 的 `STATUS` 是否为 `Succeeded`
- 打开浏览器开发者工具，确认请求是否真的抵达 host（未走缓存 / 未被 hosts 拦截到别处）

---

祝实验顺利 🎉
