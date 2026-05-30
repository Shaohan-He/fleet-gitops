# CI/CD 流水线设计

> 定义三个三部曲仓库的 GitHub Actions CI 模板，以及镜像更新策略。

---

## 流水线架构

```
每个服务仓库 (.github/workflows/ci.yml)
┌──────────────────────────────────────────────────────┐
│                    CI Pipeline                        │
│                                                       │
│  Push to Main / Tag v*                                │
│       │                                               │
│       ▼                                               │
│  ┌─────────┐                                         │
│  │  Lint   │  ← ruff / golangci-lint / shellcheck    │
│  └────┬────┘                                         │
│       │ ✅                                            │
│       ▼                                               │
│  ┌─────────┐                                         │
│  │  Test   │  ← pytest / go test / bats              │
│  └────┬────┘                                         │
│       │ ✅                                            │
│       ▼                                               │
│  ┌──────────────┐                                    │
│  │  Build Image  │  ← docker build                   │
│  └──────┬───────┘                                    │
│         │                                             │
│         ▼                                             │
│  ┌──────────────┐                                    │
│  │  Trivy Scan   │  ← CRITICAL/HIGH → 阻断           │
│  └──────┬───────┘                                    │
│         │ ✅                                          │
│         ▼                                             │
│  ┌──────────────┐                                    │
│  │  Push to GHCR │  ← ghcr.io/290298661-pixel/<repo> │
│  └──────┬───────┘                                    │
│         │                                             │
│         ▼                                             │
│  ┌────────────────────────┐                          │
│  │  Update fleet-gitops    │                          │
│  │  ├─ clone fleet-gitops  │                          │
│  │  ├─ kustomize edit      │                          │
│  │  │   set image tag      │                          │
│  │  ├─ git commit + push   │                          │
│  │  └─ 触发 ArgoCD 同步    │                          │
│  └────────────────────────┘                          │
│                                                       │
└──────────────────────────────────────────────────────┘
```

---

## 各语言 CI 配置

### 1. Game Fleet Director (Go)

```yaml
name: CI + GitOps

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
      - name: golangci-lint
        run: |
          go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
          golangci-lint run ./...
      - name: go test
        run: go test -v -race ./...

  build-and-deploy:
    needs: lint-and-test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - name: Set image tag
        id: meta
        run: |
          TAG=${GITHUB_REF#refs/tags/}
          TAG=${TAG:-$(git rev-parse --short HEAD)}
          echo "tag=$TAG" >> $GITHUB_OUTPUT

      - name: Build Docker image
        run: |
          docker build -t $REGISTRY/$IMAGE_NAME:${{ steps.meta.outputs.tag }} .

      - name: Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.tag }}"
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Push image
        run: docker push $REGISTRY/$IMAGE_NAME:${{ steps.meta.outputs.tag }}

      - name: Update GitOps repo
        env:
          GITOPS_TOKEN: ${{ secrets.GITOPS_TOKEN }}
        run: |
          git clone https://x-access-token:${GITOPS_TOKEN}@github.com/290298661-pixel/fleet-gitops.git
          cd fleet-gitops
          cd overlays/production
          kustomize edit set image \
            $REGISTRY/$IMAGE_NAME=$REGISTRY/$IMAGE_NAME:${{ steps.meta.outputs.tag }}
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git commit -am "bump: $IMAGE_NAME to ${{ steps.meta.outputs.tag }}"
          git push
```

### 2. Node Health Watcher (Python)

```yaml
name: CI + GitOps

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - name: ruff lint
        run: |
          pip install ruff
          ruff check .
      - name: pytest
        run: |
          pip install -e ".[dev]"
          pytest -v --cov

  build-and-deploy:
    needs: lint-and-test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - name: Set image tag
        id: meta
        run: |
          TAG=${GITHUB_REF#refs/tags/}
          TAG=${TAG:-$(git rev-parse --short HEAD)}
          echo "tag=$TAG" >> $GITHUB_OUTPUT

      - name: Build Docker image
        run: docker build -t $REGISTRY/$IMAGE_NAME:${{ steps.meta.outputs.tag }} .

      - name: Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.tag }}"
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Push image
        run: docker push $REGISTRY/$IMAGE_NAME:${{ steps.meta.outputs.tag }}

      - name: Update GitOps repo
        env:
          GITOPS_TOKEN: ${{ secrets.GITOPS_TOKEN }}
        run: |
          git clone https://x-access-token:${GITOPS_TOKEN}@github.com/290298661-pixel/fleet-gitops.git
          cd fleet-gitops/overlays/production
          kustomize edit set image \
            $REGISTRY/$IMAGE_NAME=$REGISTRY/$IMAGE_NAME:${{ steps.meta.outputs.tag }}
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git commit -am "bump: $IMAGE_NAME to ${{ steps.meta.outputs.tag }}"
          git push
```

### 3. Node Guardian (Bash)

```yaml
name: CI + GitOps

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: ShellCheck
        run: |
          sudo apt-get install -y shellcheck
          shellcheck bin/*
      - name: BATS test
        run: |
          sudo apt-get install -y bats
          bats tests/

  build-and-deploy:
    needs: lint-and-test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - name: Set image tag
        id: meta
        run: |
          TAG=${GITHUB_REF#refs/tags/}
          TAG=${TAG:-$(git rev-parse --short HEAD)}
          echo "tag=$TAG" >> $GITHUB_OUTPUT

      - name: Build Docker image
        run: docker build -t $REGISTRY/$IMAGE_NAME:${{ steps.meta.outputs.tag }} .

      - name: Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.tag }}"
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Push image
        run: docker push $REGISTRY/$IMAGE_NAME:${{ steps.meta.outputs.tag }}

      - name: Update GitOps repo
        env:
          GITOPS_TOKEN: ${{ secrets.GITOPS_TOKEN }}
        run: |
          git clone https://x-access-token:${GITOPS_TOKEN}@github.com/290298661-pixel/fleet-gitops.git
          cd fleet-gitops/overlays/production
          kustomize edit set image \
            $REGISTRY/$IMAGE_NAME=$REGISTRY/$IMAGE_NAME:${{ steps.meta.outputs.tag }}
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git commit -am "bump: $IMAGE_NAME to ${{ steps.meta.outputs.tag }}"
          git push
```

---

## 镜像 tag 策略

| 触发条件 | Tag 格式 | 示例 | 用途 |
|---------|---------|------|------|
| `git push` to main | commit SHA 前 7 位 | `a3f2b1c` | dev/staging 环境 |
| git tag `v*` | SemVer | `v0.2.0` | production 环境 |
| PR (不构建镜像) | 仅 lint + test | - | 质量门禁 |

## 镜像仓库设计

```
ghcr.io/290298661-pixel/
├── node-health-watcher:latest
├── node-health-watcher:v0.1.0
├── node-health-watcher:a3f2b1c
├── node-guardian:latest
├── node-guardian:v0.1.0
├── game-fleet-director-controller:latest
├── game-fleet-director-controller:v0.2.0
└── game-fleet-director-apiserver:latest
```

**为什么用 GHCR 而不是阿里云 ACR？**
- 免费（public repos 无限制）
- 与 GitHub Actions 零配置集成（`secrets.GITHUB_TOKEN` 自动可用）
- 不需要额外注册账号
- GFD 现有镜像在阿里云 ACR → 迁移到 GHCR 作为 Phase A 的一部分
