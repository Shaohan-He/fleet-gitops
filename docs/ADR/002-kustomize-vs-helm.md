# ADR-002: 选择 Kustomize 作为配置管理工具

- **状态：** 已采纳
- **日期：** 2026-05-30
- **决策者：** 何少涵

---

## 背景

GitOps 仓库需要管理多环境（dev/staging/production）的 K8s 配置。候选方案：Helm 和 Kustomize。

## 决策

**选择 Kustomize。**

## 理由

### 1. Git 可见性（权重：高）

```bash
# Kustomize: overlay 的差异一目了然
$ diff overlays/dev/patches/replicas.yaml overlays/production/patches/replicas.yaml
- replicas: 1
+ replicas: 3
```

Helm 的模板渲染结果不在 Git 中，需要 `helm template` 才能看到最终 YAML。对于 GitOps 范式（Git 是唯一真相源），这不符合原则。

### 2. 学习成本（权重：高）

Kustomize 只需要理解两个概念：
- `resources` — 要部署什么
- `patches` — 在不同环境改什么

Helm 需要学 Go template 语法（`{{ .Values }}`、`{{- if }}`、`{{- range }}`）、内置对象（`.Release`、`.Chart`）、函数（`include`、`tpl`）等。

### 3. 与 ArgoCD 集成（权重：中）

ArgoCD 原生支持 Kustomize，不需要额外配置。在 Application 的 `source` 中指定 `directory` 即可。

对于 Helm，ArgoCD 需要额外的 `helm` 配置块。虽然都是原生支持，但 Kustomize 更"无感"。

### 4. 项目规模适合（权重：中）

当前管理 3 个服务 × 3 个环境 = 9 个变体。Kustomize 的 overlay 模式在这个规模下最合适。

Helm 的优势在于"一个 chart 分发给多个团队"（如 bitnami/charts），但我的项目只有一个运维团队（我自己），不需要 chart 分发能力。

## 后果

### 正面
- `kubectl kustomize overlays/production/` 即时渲染最终 YAML
- patch 文件 = 环境差异文件，代码审查时一眼看出改了什么
- 不需要维护 `values.yaml` 的嵌套结构

### 负面
- GFD 已经用了阿里云 ACR 镜像地址，Kustomize 修改 image tag 需要用 `kustomize edit set image`
- Kustomize 的 strategic merge patch 在复杂场景下可能不符合直觉

### 风险缓解
- GFD 的 `deploy/kustomization.yaml` 已经支持 `images:` 字段配置，直接复用
- CI 中用 `kustomize edit set image` 命令更新镜像 tag

## 替代方案评估

### Helm
- **优点：** 生态系统成熟、社区 chart 丰富、适合复杂模板
- **不选原因：** 对 3 个服务来说过度设计、模板渲染结果不在 Git 中、学习成本高

### 纯 kubectl apply + envsubst
- **不选原因：** 缺乏结构化 diff、不可审计、太原始
