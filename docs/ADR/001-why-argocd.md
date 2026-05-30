# ADR-001: 选择 ArgoCD 作为 GitOps 引擎

- **状态：** 已采纳
- **日期：** 2026-05-30
- **决策者：** 何少涵

---

## 背景

需要选择一个 GitOps 引擎来管理 K8s 集群中的应用部署。候选方案：ArgoCD 和 Flux CD。

## 决策

**选择 ArgoCD。**

## 理由

### 1. Web UI 可视化运维（权重：高）

ArgoCD 的 Web UI 提供直观的可视化运维界面：
- 应用同步状态（Synced / OutOfSync）
- 资源树（Deployment → ReplicaSet → Pod 的层级关系）
- 金丝雀发布进度（Argo Rollouts 集成）
- 多集群管理面板

Flux 主要依赖 CLI (`flux get` / `flux logs`)，不提供原生 Web 界面。

### 2. 生态整合（权重：高）

Argo Rollouts 是 Argo 生态的一部分，与 ArgoCD 无缝集成：
- AnalysisTemplate 可以直接作为 ArgoCD Application 的子资源
- Rollout 的同步状态在 ArgoCD UI 中可见

Flux 的渐进发布方案是 Flagger，虽然也能用，但与 Flux 的整合不如 ArgoCD + Argo Rollouts 紧密。

### 3. App of Apps 模式（权重：中）

```
root-app → platform-apps → nhw / guardian / gfd
         → monitoring-apps → prometheus / grafana / loki
```

一个 `kubectl apply -f bootstrap/root-app.yaml` 部署一切。子 App 独立管理、独立同步、独立回滚。

### 4. 社区与文档（权重：低）

ArgoCD 的 CNCF 毕业时间更早（2022 年 12 月），社区规模更大，中文资料更多。

## 后果

### 正面
- Web UI 提供直观的集群状态可视化
- Argo Rollouts AnalysisTemplate 实现数据驱动的渐进发布
- App of Apps 架构支持层级化管理

### 负面
- 比 Flux 资源占用稍高（~1GB vs ~500MB 内存）
- 需要额外安装 Argo Rollouts controller

### 风险缓解
- Kind 集群 8GB 内存足够运行 ArgoCD + Argo Rollouts + 三部曲
- Argo Rollouts 安装是一个 `kubectl apply`，复杂度可接受

## 替代方案评估

### Flux CD
- **优点：** 更轻量、CLI 功能强、与 Kustomize 原生集成更好
- **不选原因：** 无原生 Web UI（需要 Weave GitOps 插件），渐进发布需额外集成 Flagger

### Jenkins X
- **不选原因：** 太重、学习曲线陡、不适合 1 人项目
