# Fleet GitOps — 架构设计文档

> 深入设计 GitOps 交付平台的系统架构、组件交互、数据流和故障模式处理。

---

## 目录

1. [设计哲学](#设计哲学)
2. [系统架构](#系统架构)
3. [核心组件](#核心组件)
4. [数据流与交互时序](#数据流与交互时序)
5. [App of Apps 设计](#app-of-apps-设计)
6. [金丝雀发布策略](#金丝雀发布策略)
7. [多环境管理](#多环境管理)
8. [故障模式与处理](#故障模式与处理)
9. [安全设计](#安全设计)

---

## 设计哲学

### 三条核心原则

1. **Git 是唯一真相源（Single Source of Truth）**
   - 不是 "kubectl apply -f deploy/" —— 那是命令式操作，不可审计
   - 所有 K8s 资源定义（Deployment、Service、ConfigMap、Rollout）都在 Git 中
   - ArgoCD 持续对比 Git 状态 vs 集群实际状态，偏差自动修复（self-healing）

2. **声明式 > 命令式**
   - 运维人员不直接 kubectl apply。改动的唯一方式是 commit + push
   - 回滚 = git revert + push。不需要"知道上次 apply 了什么"
   - 审计日志 = git log。谁在什么时候改了什么、为什么改

3. **渐进式交付（Progressive Delivery）**
   - 不是"全量替换"——那是赌博
   - 20% canary → 用指标验证 → 50% → 再验证 → 100%
   - 任何一步失败自动回滚，最小化影响面

### 设计层次

GitOps 不仅仅是 "YAML 存 Git"。本平台将 GitOps 分为三个递进的层次：

1. **声明式部署** — 所有 K8s 资源定义在 Git 中，`kubectl apply` 被 commit + push 替代
2. **自动同步** — ArgoCD 持续对比 Git 状态与集群实际状态，自动修复偏差（self-healing）
3. **数据驱动发布** — 金丝雀能否放量由实时健康指标决定（NHW 巡检 + Prometheus 指标），而非固定时间窗口

---

## 系统架构

### 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Fleet GitOps — 全链路交付平台                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    开发工作流                                      │   │
│  │                                                                   │   │
│  │  ┌─────────┐    ┌──────────┐    ┌──────────────────┐            │   │
│  │  │ 写代码    │───▶│ git push │───▶│ GitHub Actions CI │            │   │
│  │  │ (三部曲)  │    │ (trigger)│    │                  │            │   │
│  │  └─────────┘    └──────────┘    │ Lint + Test      │            │   │
│  │                                  │ Build Image      │            │   │
│  │                                  │ Trivy Scan       │            │   │
│  │                                  │ Push to GHCR     │            │   │
│  │                                  │ Update GitOps    │            │   │
│  │                                  └────────┬─────────┘            │   │
│  └───────────────────────────────────────────┼──────────────────────┘   │
│                                               │                         │
│                                               ▼                         │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     fleet-gitops (本仓库)                          │   │
│  │                                                                   │   │
│  │  ┌────────────┐   ┌───────────┐   ┌──────────┐   ┌──────────┐  │   │
│  │  │ bootstrap/  │   │ apps/     │   │ overlays/ │   │ argocd/  │  │   │
│  │  │ root-app    │   │ platform/ │   │ dev/      │   │ analysis │  │   │
│  │  │             │   │ monitoring│   │ staging/  │   │ rollouts │  │   │
│  │  └────────────┘   └───────────┘   │ prod/     │   └──────────┘  │   │
│  │                                    └──────────┘                   │   │
│  └──────────────────────────────┬───────────────────────────────────┘   │
│                                  │                                       │
│                                  ▼                                       │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     ArgoCD (K8s 集群内)                            │   │
│  │                                                                   │   │
│  │  ┌──────────────┐   ┌──────────────┐   ┌──────────────────┐    │   │
│  │  │ Repo Server   │   │ API Server   │   │ App Controller   │    │   │
│  │  │ (Git 拉取)    │   │ (Web UI+API) │   │ (协调循环)        │    │   │
│  │  └──────┬───────┘   └──────┬───────┘   └────────┬─────────┘    │   │
│  │         │                  │                     │               │   │
│  │         │    每 3 分钟     │    对比集群状态      │               │   │
│  │         │    检测变更 ─────┼──▶ 与 Git 定义 ─────▶               │   │
│  │         │                  │    有差异 = OutOfSync               │   │
│  │         │                  │    auto-sync 自动修复               │   │
│  └─────────┼──────────────────┼─────────────────────┼──────────────┘   │
│            │                  │                     │                   │
│            ▼                  ▼                     ▼                   │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                Argo Rollouts (渐进发布引擎)                         │   │
│  │                                                                   │   │
│  │  ┌──────────────────────────────────────────────────────────┐   │   │
│  │  │ Canary Steps:                                              │   │   │
│  │  │   setWeight: 20  ──▶ pause: 120s ──▶ AnalysisTemplate     │   │   │
│  │  │   setWeight: 50  ──▶ pause: 120s ──▶ AnalysisTemplate     │   │   │
│  │  │   setWeight: 100                                          │   │   │
│  │  └──────────────────────────────────────────────────────────┘   │   │
│  │                              │                                    │   │
│  │                              ▼                                    │   │
│  │  ┌──────────────────────────────────────────────────────────┐   │   │
│  │  │ AnalysisTemplate — 四项健康检查                              │   │   │
│  │  │                                                            │   │   │
│  │  │  1. NHW 节点健康分 > 80          (web metric)               │   │   │
│  │  │  2. 分配成功率 > 99%             (prometheus metric)       │   │   │
│  │  │  3. 分配延迟 P99 < 2s            (prometheus metric)       │   │   │
│  │  │  4. 熔断器未触发                  (prometheus metric)       │   │   │
│  │  │                                                            │   │   │
│  │  │  任意项 failureLimit 次失败 → 自动 Rollback                 │   │   │
│  │  └──────────────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 核心组件

### 1. ArgoCD — GitOps 引擎

**角色：** 集群状态的"恒温器"

```
工作原理:
  Git Repo (desired state) ──对比──▶ K8s Cluster (live state)
                                       │
                              一致?    │    不一致?
                               │       │
                           什么都不做    │
                                       ▼
                                 自动同步 / 告警
```

**配置要点：**
- `syncPolicy.automated.prune: true` — 删除 Git 中移除的资源
- `syncPolicy.automated.selfHeal: true` — 手动修改集群 3 分钟后自动回退
- `syncPolicy.syncOptions.ApplyOutOfSyncOnly=true` — 只同步有差异的资源

**为什么不用 Flux？**
- ArgoCD 提供原生 Web UI，集群状态可视化即时
- Argo Rollouts 同一生态，AnalysisTemplate 原生集成
- 支持多集群管理，UI 一目了然

### 2. Argo Rollouts — 渐进发布引擎

**角色：** 发布过程的"交通管制员"

**Rollout 替代 Deployment 的关键差异：**

| | Deployment | Rollout |
|---|---|---|
| 发布方式 | 滚动更新（全量逐步替换） | 金丝雀（分步放量 + 分析） |
| 决策依据 | "新 Pod Ready 了就行" | "Analysis 指标通过了才放量" |
| 回滚 | 手动 kubectl rollout undo | 自动回滚 + 可触发 hook |
| 流量控制 | 无 | setWeight 精确控制百分比 |
| 多版本并存 | 不支持 | 支持（canary + stable 同时存在） |

### 3. AnalysisTemplate — 发布质量门禁

**这是整个平台最有价值的设计——自己写的巡检系统成了金丝雀发布的判定标准。**

```
分析维度:
  ┌─────────────────────────────────────────┐
  │  1. NHW 节点健康检查 (web metric)        │
  │     来源: NHW /api/v1/node-health       │
  │     判定: healthy_ratio >= 0.8          │
  │     目的: 节点层面是否有故障前兆？        │
  │     ★ 自己写的巡检系统！                 │
  ├─────────────────────────────────────────┤
  │  2. 游戏服分配成功率 (prometheus metric)  │
  │     来源: GFD gfd_allocation_total      │
  │     判定: success_rate >= 0.99          │
  │     目的: 核心业务是否正常？              │
  ├─────────────────────────────────────────┤
  │  3. 分配延迟 P99 (prometheus metric)     │
  │     来源: GFD gfd_allocation_latency    │
  │     判定: P99 < 2.0s                   │
  │     目的: 用户体验是否劣化？              │
  ├─────────────────────────────────────────┤
  │  4. 熔断器状态 (prometheus metric)       │
  │     来源: GFD gfd_circuit_breaker       │
  │     判定: triggered == 0               │
  │     目的: 系统是否在自我保护模式？        │
  └─────────────────────────────────────────┘
```

**为什么是这四项？**
- 指标 1（NHW）覆盖基础设施层——节点级别的问题
- 指标 2-3（分配成功率+延迟）覆盖应用层——用户体验
- 指标 4（熔断器）覆盖控制层——系统自我保护状态
- 三层交叉验证：如果节点有问题但应用没问题 → 可能是单节点故障已规避
- 如果节点正常但应用反常 → 可能是应用层 bug

### 4. Kustomize — 配置管理

**为什么不用 Helm？**

| 对比维度 | Helm | Kustomize |
|---------|------|-----------|
| 范式 | 模板渲染（Go template） | Patch 叠加（overlay） |
| Git 可见性 | 渲染后的 YAML 不在 Git 中 | 最终 YAML 可由 `kustomize build` 生成 |
| 团队学习成本 | 需要学 template 语法 | 只需要理解 patch 语义 |
| ArgoCD 集成 | 支持 | 原生支持，不需要额外插件 |
| 适合场景 | 通用 chart 分发给第三方 | 自己维护、多环境差异化 |

**选择 Kustomize 的理由：**

> "Helm 的模板语法在简单场景下过度设计。我的三个服务、三个环境，差异就是副本数、镜像 tag、是否开启金丝雀——用 Kustomize 的 overlay patch 就够：dev 环境 patch 副本数为 1，production 环境 patch 开启完整的 Analysis 步骤。每个 overlay 的差异一个文件就能看到，不需要在 if/else 模板里找。"

---

## 数据流与交互时序

### 完整发布流程（以 GFD Controller 一次发版为例）

```
时刻 T+0:  开发者修改 GFD Controller 代码，git push
时刻 T+1:  GitHub Actions CI 触发
           ├─ ruff/golangci-lint 代码检查 (1 min)
           ├─ pytest/go test 测试 (2 min)
           ├─ docker build 构建镜像 (3 min)
           ├─ trivy scan 安全扫描 (1 min)
           ├─ docker push to GHCR (2 min)
           └─ git clone fleet-gitops → 更新镜像 tag → git push (30s)
           总计: ~10 min

时刻 T+10: fleet-gitops 仓库有新 commit (image tag 变更)

时刻 T+13: ArgoCD 每 3 分钟轮询检测到变更
           ├─ Repo Server 拉取最新 Git 状态
           ├─ App Controller 对比 live vs desired
           ├─ 发现差异: Rollout.spec.template 的镜像变了
           └─ 触发 sync (auto-sync)

时刻 T+13: Argo Rollouts 开始金丝雀
           ├─ setWeight: 20 → 创建 1 个 canary pod (原 2 个 stable)
           ├─ pause: 120s (给指标稳定时间)
           ├─ AnalysisTemplate 运行:
           │   ├─ NHW 健康分: 95/100 ✅
           │   ├─ 分配成功率: 99.8% ✅
           │   ├─ 分配延迟 P99: 1.2s ✅
           │   └─ 熔断器: 0 ✅
           ├─ setWeight: 50 → 扩到 2 canary
           ├─ pause: 120s → AnalysisTemplate (同上)
           ├─ setWeight: 100 → 全量切换
           └─ 旧 stable pod 被回收
           总计: ~6 min

时刻 T+19: 发布完成。从 push 到全量上线 ≈ 19 分钟

如果 Analysis 失败:
  T+13 → 20% canary → Analysis 失败 (如分配成功率 97% < 99%)
  → 立即 Rollback → 删除 canary pod → stable 不变
  → 触发 rollback-diagnose Job (kn-diagnose 收集现场)
  → IM 推送告警
  影响面: 0 用户 (20% canary 失败后自动回退，stable 始终在运行)
```

---

## App of Apps 设计

### 设计模式

```
root-app (bootstrap/root-app.yaml)
  │
  ├──▶ platform-apps (Application)
  │     │
  │     ├──▶ node-health-watcher (Application)
  │     │     ├── Deployment (CronJob 模式)
  │     │     ├── Service
  │     │     └── ConfigMap
  │     │
  │     ├──▶ node-guardian (Application)
  │     │     └── DaemonSet
  │     │
  │     └──▶ game-fleet-director (Application)
  │           ├── Deployment (Controller)
  │           ├── Deployment (API Server)
  │           ├── CRDs
  │           ├── Service
  │           └── Rollout (替代 Deployment 做金丝雀)
  │
  └──▶ monitoring-apps (Application) ← Phase B 填充
        ├── prometheus
        ├── grafana
        └── loki
```

### 为什么用 App of Apps？

1. **一键部署整个平台：** `kubectl apply -f bootstrap/root-app.yaml`
2. **独立管理：** 修改 NHW 的部署不影响 GFD
3. **权限隔离：** 未来可以为不同团队分配不同子 App 的权限
4. **部署层级清晰：** 父子关系在 ArgoCD UI 中以树状结构展示

### root-app.yaml 设计

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Shaohan-He/fleet-gitops.git
    targetRevision: main
    path: apps/
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## 金丝雀发布策略

### 发布步骤设计

| 步骤 | 权重 | 暂停 | 分析 | 设计意图 |
|------|------|------|------|---------|
| Step 1 | 20% | 120s | 4 项全量检查 | 最小爆炸半径——出问题只影响 20% 流量 |
| Step 2 | 50% | 120s | 4 项全量检查 | 确认 20% 不是"运气好"，扩大样本 |
| Step 3 | 100% | - | - | 全量切换 |

**为什么是 20% → 50% → 100% 而不是直接 10% → 100%？**
> "20% 是一个足够小的爆炸半径（5 个 pod 只影响 1 个），但样本又够大（能暴露大部分问题）。50% 是中间验证——如果 20% 阶段的指标只是'暂时好'（比如刚启动还没收到真实流量），50% 阶段就暴露了。这比 10%→100% 多了一个安全网。"

### 各环境差异

| 环境 | 金丝雀策略 | 理由 |
|------|-----------|------|
| **dev** | 无（直接滚动更新） | 开发环境追求速度，出问题影响 = 0 |
| **staging** | 50% → 100%（简化） | 验证流程可用即可 |
| **production** | 20% → 50% → 100%（完整） | 安全第一 |

---

## 多环境管理

### Kustomize Overlay 策略

```
base/ (共享配置)
  ├── deployment.yaml     ← 基础 Deployment 结构
  ├── service.yaml        ← Service 定义
  └── kustomization.yaml  ← 公共资源引用

overlays/dev/
  ├── kustomization.yaml  ← 引用 base + patches
  └── patches/
      └── replicas.yaml   ← replicas: 1, logLevel: debug

overlays/staging/
  ├── kustomization.yaml
  └── patches/
      └── replicas.yaml   ← replicas: 2, canary: 50%→100%

overlays/production/
  ├── kustomization.yaml
  └── patches/
      └── replicas.yaml   ← replicas: 3, canary: 20%→50%→100%
```

### 环境一致性保证

| 维度 | 策略 |
|------|------|
| 基础 YAML | 所有环境共享 `base/` |
| 差异化 | 通过 `patches/` 中的 strategic merge patch |
| 不可变资源名 | 同一套 Service name、ConfigMap name 跨环境 |
| 新增环境 | 复制最近的 overlay 目录，改几个值即可 |

---

## 故障模式与处理

### 1. ArgoCD 本身挂了

```
影响: 新部署被阻塞，已有服务不受影响
原因: ArgoCD 只负责同步，不参与运行时
恢复: ArgoCD 自身是 Deployment → kubelet 自动重启
```

### 2. 金丝雀分析误判

```
场景: Prometheus 短暂不可用 → Analysis 查询超时 → 判定失败 → 错误回滚
防护:
  1. failureLimit: 2 (需要连续 2 次失败才判定)
  2. interval: 30s (给 Prometheus 恢复时间)
  3. 四项检查独立——一个挂了不影响其他
```

### 3. GitOps 仓库被错误修改

```
场景: CI 错误推送了错误的镜像 tag
检测: ArgoCD UI 显示 OutOfSync → 人工介入
恢复: git revert + push → ArgoCD 自动同步回正确版本
审计: git log 精确到谁、什么时间、改了什么
```

### 4. 镜像仓库不可用

```
场景: GHCR 挂了
影响: 新 Pod 无法拉取镜像（kubelet 用 IfNotPresent 策略，已有 Pod 不受影响）
缓解: 多镜像仓库备份（GHCR + 阿里云 ACR）
```

---

## 安全设计

### 镜像安全

```
CI Pipeline:
  docker build → Trivy Scan → 检查结果
                                 │
                    CRITICAL/HIGH? │
                    ┌──────────────┴──────────┐
                    ▼                         ▼
                   是                        否
              ┌──────────┐            ┌──────────┐
              │ CI 失败   │            │ Push 镜像 │
              │ 阻断发布   │            │ 继续流程  │
              └──────────┘            └──────────┘
```

### GitOps 仓库保护

- `main` 分支受保护（需要 PR review）
- CI 更新镜像 tag 使用专用 GitHub Token（最小权限：只写 fleet-gitops）
- 所有变更通过 git log 可审计

### Secret 管理（Phase A 简化方案）

- ConfigMap 存非敏感配置（公开）
- Secret 存敏感数据（ArgoCD 密码、IM webhook URL）
- 使用 K8s Secret + Sealed Secrets（或 External Secrets，后续迭代）

---

## 未来演进

| 阶段 | 计划 | 依赖 |
|------|------|------|
| Phase B | 部署 Prometheus + Grafana + Loki 通过此 GitOps 流水线 | Phase A 完成 |
| Phase B | Grafana Dashboard 展示部署频率/回滚率（DORA metrics） | Phase B 完成 |
| Phase C | 混沌实验验证金丝雀+回滚链路的韧性 | Phase A + B 完成 |
| Phase D | LLM Agent 通过 `get_recent_deployments()` 获取变更历史做根因分析 | Phase A 完成 |

