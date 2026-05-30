# 🚀 Fleet GitOps — 全链路交付平台

> **GitOps 全链路交付平台** — 代码 Push → CI 构建 → 镜像扫描 → ArgoCD 同步 → 金丝雀发布 → 自动回滚
>
> Phase A of [K8s 全栈运维平台](https://github.com/290298661-pixel)

[![Status](https://img.shields.io/badge/status-developing-yellow)](https://github.com/290298661-pixel/fleet-gitops)
[![ArgoCD](https://img.shields.io/badge/ArgoCD-v3.4.3-EF7B4D)](https://argoproj.github.io/cd/)
[![K8s](https://img.shields.io/badge/K8s-v1.35-326CE5)](https://kubernetes.io/)

---

## 🎯 一句话描述

**"代码 Push 到 GitHub → 15 分钟内自动到达生产环境，金丝雀发布用我自己的巡检系统判断健康状态，异常自动回滚并收集现场证据。"**

---

## 🏗️ 架构总览

```
                           fleet-gitops (本仓库)
                    ┌──────────────────────────────┐
                    │  bootstrap/root-app.yaml     │  ← 根 Application
                    │  apps/platform/*.yaml         │  ← 三部曲部署定义
                    │  overlays/{dev,staging,prod}  │  ← 多环境差异化
                    │  argocd/analysis-template.yaml│  ← 金丝雀健康检查
                    └──────────────┬───────────────┘
                                   │ ArgoCD 3min 轮询检测变更
                                   ▼
┌──────────────────────────────────────────────────────────────┐
│                       ArgoCD + Argo Rollouts                  │
│                                                               │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────┐   │
│  │ Git Repo │───▶│ Sync & Apply │───▶│ Canary Rollout   │   │
│  │ (真相源)  │    │ (自动同步)    │    │ 20%→50%→100%     │   │
│  └──────────┘    └──────────────┘    └────────┬─────────┘   │
│                                                │              │
│                              ┌─────────────────┘              │
│                              ▼                                │
│                    ┌──────────────────┐                      │
│                    │ AnalysisTemplate │  ← ★ 核心亮点         │
│                    │                  │                      │
│                    │ NHW 节点健康检查  │  自己写的巡检系统     │
│                    │ GFD 分配成功率    │  自己写的 K8s Operator│
│                    │ 扩缩容错误率      │                      │
│                    └────────┬─────────┘                      │
│                             │                                │
│                  任意 Analysis 失败                           │
│                             ▼                                │
│                    ┌──────────────────┐                      │
│                    │ 自动 Rollback    │                      │
│                    │ + kn-diagnose    │  自动诊断收集现场     │
│                    │ + IM 通知        │                      │
│                    └──────────────────┘                      │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔗 在平台中的位置

```
Phase D: LLM 智能运维 ←── 消费 ArgoCD 变更历史做根因分析
    ↑
Phase C: 混沌工程 ←── 验证金丝雀发布 + 回滚链路的韧性
    ↑
Phase B: 统一可观测性 ←── Prometheus 指标 + Grafana Dashboard
    ↑
Phase A: GitOps ←── 你在这里 ★
    ↑
Phase 0: 运维三部曲 ←── 被部署的服务 (NHW + Node Guardian + GFD)
```

---

## 📂 仓库结构

```
fleet-gitops/
├── README.md                          ← 你在这里
├── docs/
│   ├── ARCHITECTURE.md                ← 完整架构设计文档
│   ├── ADR/
│   │   ├── 001-why-argocd.md          ← 为什么选 ArgoCD 而不是 Flux
│   │   ├── 002-kustomize-vs-helm.md   ← 为什么用 Kustomize
│   │   └── 003-canary-strategy.md     ← 金丝雀策略设计决策
│   ├── CI_PIPELINE_DESIGN.md          ← CI/CD 流水线详细设计
│   └── images/                        ← 架构图源文件 + 截图
│
├── bootstrap/
│   └── root-app.yaml                  ← ★ App of Apps 根 Application
│
├── apps/
│   ├── platform/                      ← 运维三部曲
│   │   ├── node-health-watcher/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   ├── configmap.yaml
│   │   │   └── kustomization.yaml
│   │   ├── node-guardian/
│   │   │   ├── daemonset.yaml
│   │   │   └── kustomization.yaml
│   │   └── game-fleet-director/
│   │       ├── controller.yaml
│   │       ├── apiserver.yaml
│   │       ├── crds/
│   │       ├── rbac/
│   │       └── kustomization.yaml
│   └── monitoring/                    ← Phase B 时填充
│       └── .gitkeep
│
├── overlays/
│   ├── dev/                           ← 开发环境 (小副本、debug、无金丝雀)
│   │   ├── kustomization.yaml
│   │   └── patches/
│   ├── staging/                       ← 预发环境 (中等副本、简化 Analysis)
│   │   ├── kustomization.yaml
│   │   └── patches/
│   └── production/                    ← 生产环境 (完整副本、五步金丝雀)
│       ├── kustomization.yaml
│       └── patches/
│
├── argocd/
│   ├── analysis-template.yaml         ← ★ 金丝雀健康检查定义
│   ├── rollouts/                      ← Argo Rollouts 定义
│   │   └── game-fleet-director.yaml
│   └── hooks/
│       └── rollback-diagnose-job.yaml ← 回滚后自动诊断 Job
│
└── .github/
    └── workflows/
        └── validate-gitops.yaml       ← 校验 GitOps 仓库健康状态
```

---

## 🚦 交付流水线全景

```
  Developer Push to GitHub
         │
         ▼
┌─────────────────────────────────────┐
│   GitHub Actions CI (各服务仓库)      │
│                                      │
│   Lint ──▶ Test ──▶ Build ──▶ Scan  │
│                         │            │
│                    Docker Image      │
│                         │            │
│                    Trivy Scan        │
│                    (CRITICAL/HIGH    │
│                     → 阻断)          │
│                         │            │
│                    Push to GHCR      │
│                         │            │
│                    Update Image Tag  │
│                    in fleet-gitops   │
└─────────────────────┬───────────────┘
                      │
                      ▼
┌─────────────────────────────────────┐
│   ArgoCD (3 分钟轮询)                │
│                                      │
│   检测到 fleet-gitops 仓库变更        │
│   → 对比 live 状态 vs desired 状态   │
│   → 自动同步 (auto-sync)            │
│   → 启动 Argo Rollouts (如果配置了)  │
└─────────────────────┬───────────────┘
                      │
                      ▼
┌─────────────────────────────────────┐
│   Argo Rollouts (Canary)             │
│                                      │
│   Step 1: 20% canary pods           │
│   Step 2: AnalysisTemplate          │
│     ├─ NHW 健康分 > 80              │
│     ├─ 分配成功率 > 99%             │
│     ├─ 分配延迟 P99 < 2s            │
│     └─ 熔断器未触发                  │
│   Step 3: 50% → Analysis → 100%     │
│                                      │
│   ❌ 任意 Analysis 失败              │
│   → 自动 Rollback                   │
│   → 触发 rollback-diagnose Job      │
│   → IM 通知                          │
└─────────────────────────────────────┘
```

---

## 🎯 核心设计决策

| 决策 | 选择 | 为什么不选替代方案 |
|------|------|-------------------|
| **GitOps 引擎** | ArgoCD | Web UI 直观展示同步状态，App of Apps 层级清晰，Argo Rollouts 同生态无缝集成 |
| **配置管理** | Kustomize | 比 Helm 更适合 GitOps——不需要模板渲染，patch 就是 overlay，差异一目了然  |
| **容器镜像仓库** | GitHub Container Registry (GHCR) | 免费、与 GitHub Actions 零配置集成、不需要额外账号 |
| **渐进发布** | Argo Rollouts | 比 K8s 原生 rolling update 多了 Analysis 步骤，金丝雀出问题能自动拦截 |
| **镜像扫描** | Trivy | 开源免费、GitHub Actions 有官方 Action、扫描速度快 |
| **镜像更新方式** | Git Push (CI 写回) | 比 Image Updater 自动检测更明确——每次部署都有 git commit 记录，可审计 |

---

## 🔑 核心亮点 — NHW 巡检系统接入金丝雀健康检查

常规的 AnalysisTemplate 只能检查 CPU、内存、错误率等通用指标。本项目的关键设计是将自研的 NHW 节点巡检系统通过 REST API 接入金丝雀 Analysis，实现**节点层面的故障前兆检测**——conntrack 表使用率、磁盘空间、kubelet 状态等在应用层指标恶化之前就暴露问题。

---

## 🚀 快速开始

### 前提

- Kubernetes 集群 (Kind/Minikube/K3s)
- `kubectl` 已配置
- Helm (安装 ArgoCD 用，可选)

### 1. 安装 ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 暴露服务 (NodePort)
kubectl patch svc argocd-server -n argocd -p '{"spec":{"type":"NodePort","ports":[{"port":443,"nodePort":30080}]}}'

# 获取初始密码
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 2. 安装 Argo Rollouts

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

### 3. 部署根 Application

```bash
kubectl apply -f bootstrap/root-app.yaml
```

### 4. 查看同步状态

```bash
# ArgoCD CLI
argocd login localhost:30080 --insecure
argocd app list

# 或浏览器访问
open https://localhost:30080
```

---

## 📋 多环境对比

| 维度 | dev | staging | production |
|------|-----|---------|------------|
| 副本数 | 1 | 2 | 3+ |
| 金丝雀步骤 | 无（直接滚动更新） | 50% → 100% | 20% → 50% → 100% |
| Analysis 检查 | 跳过 | 3 项 | 4 项（全量） |
| 自动同步 | ✅ | ✅ | 手动确认 (`autoPromotionEnabled: false`) |
| 镜像 tag | `latest` / commit SHA | `v*.-rc*` | `v*.*.*` (semver) |
| 日志级别 | debug | info | info |

---

## 📚 文档导航

| 文档 | 适合 |
|------|------|
| [ARCHITECTURE.md](docs/ARCHITECTURE.md) | 深入理解系统设计与架构 |
| [ADR-001 为什么选 ArgoCD](docs/ADR/001-why-argocd.md) | 技术决策记录 |
| [ADR-002 为什么用 Kustomize](docs/ADR/002-kustomize-vs-helm.md) | 配置管理选型 |
| [CI_PIPELINE_DESIGN.md](docs/CI_PIPELINE_DESIGN.md) | CI/CD 流水线设计 |
| [金丝雀策略 ADR](docs/ADR/003-canary-strategy.md) | 发布策略设计 |

---

## 🔗 相关仓库

| 仓库 | 关系 |
|------|------|
| [node-health-watcher](https://github.com/290298661-pixel/node-health-watcher) | 被部署的服务 + AnalysisTemplate 数据源 |
| [node-guardian](https://github.com/290298661-pixel/node-guardian) | 被部署的服务 + 回滚后诊断工具 |
| [game-server-orchestrator](https://github.com/290298661-pixel/game-server-orchestrator) | 被部署的服务 + 金丝雀分析指标来源 |
| [fleet-observability](https://github.com/290298661-pixel/fleet-observability) | Phase B — 本仓库的 monitoring apps 目录为它预留 |
| [fleet-chaos](https://github.com/290298661-pixel/fleet-chaos) | Phase C — 验证本仓库的金丝雀+回滚链路 |
| [fleet-brain](https://github.com/290298661-pixel/fleet-brain) | Phase D — 消费本仓库的 ArgoCD 变更历史 |

---

> **核心理念：不是"我又学了一个工具（ArgoCD）"，而是"这个工具在我的平台里扮演交付层的角色——它让三部曲的部署从手动 kubectl apply 变成了声明式、可审计、自动回滚的 GitOps 流水线"。**
