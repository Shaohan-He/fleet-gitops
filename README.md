# Fleet GitOps

> Kubernetes GitOps delivery repository based on Argo CD, Argo Rollouts, and Kustomize.

[![Status](https://img.shields.io/badge/status-developing-yellow)](https://github.com/Shaohan-He/fleet-gitops)
[![ArgoCD](https://img.shields.io/badge/ArgoCD-v3.4.3-EF7B4D)](https://argoproj.github.io/cd/)
[![K8s](https://img.shields.io/badge/K8s-v1.35-326CE5)](https://kubernetes.io/)

## 概述

Fleet GitOps 用于管理 Kubernetes 应用的声明式部署配置。仓库以 Git 作为期望状态来源，通过 Argo CD 同步到集群，并使用 Kustomize 管理不同环境的差异。

当前重点是：

- 维护 Argo CD App of Apps 入口。
- 管理 `dev`、`staging`、`production` 环境 overlays。
- 为发布流程预留 Argo Rollouts 和 AnalysisTemplate 配置。
- 在 Git 中保留部署变更记录，便于审计和回滚。

本仓库只负责交付配置，不承载业务服务代码。

## 架构

```text
Git repository
    |
    v
Argo CD root application
    |
    +-- apps/platform
    +-- apps/monitoring
    |
    v
Kustomize overlays
    |
    +-- dev
    +-- staging
    +-- production
    |
    v
Kubernetes cluster
```

## 目录结构

```text
fleet-gitops/
├── bootstrap/
│   └── root-app.yaml                  # Argo CD 根 Application
├── apps/
│   ├── platform/                      # 平台组件部署清单
│   └── monitoring/                    # 可观测性组件部署清单预留目录
├── overlays/
│   ├── dev/                           # 开发环境差异配置
│   ├── staging/                       # 预发环境差异配置
│   └── production/                    # 生产环境差异配置
├── argocd/
│   ├── analysis-template.yaml         # Rollouts 分析模板
│   ├── rollouts/                      # Rollout 资源定义
│   └── hooks/                         # 回滚或诊断相关 Hook
├── docs/
│   ├── ARCHITECTURE.md
│   ├── CI_PIPELINE_DESIGN.md
│   └── ADR/
└── .github/workflows/
    └── validate-gitops.yaml
```

## 快速开始

### 前提条件

- Kubernetes 集群
- `kubectl`
- Helm
- Argo CD CLI（可选）

### 安装 Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 安装 Argo Rollouts

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

### 部署根 Application

```bash
kubectl apply -f bootstrap/root-app.yaml
```

### 查看同步状态

```bash
argocd app list
kubectl get applications -n argocd
```

## 环境策略

| 环境 | 用途 | 同步策略 | 发布策略 |
| --- | --- | --- | --- |
| `dev` | 本地或开发集群验证 | 可自动同步 | 简化配置，便于调试 |
| `staging` | 发布前验证 | 自动或手动同步 | 可启用基础分析检查 |
| `production` | 生产部署 | 建议手动确认关键变更 | 渐进发布和回滚策略 |

## 设计取舍

| 主题 | 当前选择 | 说明 |
| --- | --- | --- |
| GitOps 引擎 | Argo CD | 生态成熟，支持 App of Apps，适合声明式同步 |
| 配置管理 | Kustomize | 适合环境差异管理，避免模板复杂度 |
| 渐进发布 | Argo Rollouts | 提供 Canary、Analysis、Rollback 等能力 |
| 配置审计 | Git commit | 每次部署配置变更都有历史记录 |

## 验证

```bash
# 检查 Kustomize 渲染
kubectl kustomize overlays/dev
kubectl kustomize overlays/staging
kubectl kustomize overlays/production

# 检查 Argo CD Application
kubectl apply --dry-run=client -f bootstrap/root-app.yaml
```

## 文档

| 文档 | 内容 |
| --- | --- |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | 架构说明 |
| [docs/CI_PIPELINE_DESIGN.md](docs/CI_PIPELINE_DESIGN.md) | CI/CD 流程设计 |
| [docs/ADR/001-why-argocd.md](docs/ADR/001-why-argocd.md) | Argo CD 选型记录 |
| [docs/ADR/002-kustomize-vs-helm.md](docs/ADR/002-kustomize-vs-helm.md) | Kustomize 与 Helm 取舍 |
| [docs/ADR/003-canary-strategy.md](docs/ADR/003-canary-strategy.md) | Canary 策略说明 |

## 相关项目

| 仓库 | 关系 |
| --- | --- |
| [fleet-observability](https://github.com/Shaohan-He/fleet-observability) | 可观测性配置与仪表盘 |
| [node-health-watcher](https://github.com/Shaohan-He/node-health-watcher) | 节点巡检与告警 |
| [node-guardian](https://github.com/Shaohan-He/node-guardian) | 节点诊断与维护工具 |
| [k8s-healing-agent](https://github.com/Shaohan-He/k8s-healing-agent) | 告警驱动的 Kubernetes 修复实验项目 |

## License

MIT
