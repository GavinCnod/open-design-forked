# GitHub Actions 工作流说明

## 临时停用方式

如果你不想删除或修改现有 workflow 文件，可以把它们从 `.github/workflows/` 临时移动到仓库中的其他目录。GitHub 只会识别 `.github/workflows/` 目录下的 `.yml` / `.yaml` 工作流文件，因此移出该目录后，这些 Actions 就不会继续触发。

推荐做法：

- 新建备份目录，例如 `.github/workflows.disabled/`
- 将当前所有 workflow 文件从 `.github/workflows/` 移动到 `.github/workflows.disabled/`
- 需要恢复时，再原样移回 `.github/workflows/`

建议注意：

- 这是“停用”而不是“删除”，文件内容本身不需要改
- 如果仓库有依赖这些 workflow 的说明文档或团队流程，最好同步通知
- 如果只是短期停用，这种方式比逐个修改 `on:` 或手动在 GitHub 页面禁用更容易回滚

## 工作流汇总

### `ci`

- 文件：`.github/workflows/ci.yml`
- 作用：主仓库通用 CI，校验整个 workspace 的安装、类型检查、测试与构建是否正常
- 主要执行内容：
  - 安装依赖
  - 预构建部分类型声明
  - 执行 workspace `typecheck`
  - 检查 residual JS
  - 执行 `pnpm test`
  - 执行 workspace `build`
- 触发方式：
  - `pull_request`
  - `push` 到 `main`
  - `workflow_dispatch`

### `release-beta`

- 文件：`.github/workflows/release-beta.yml`
- 作用：执行 Beta 发布流程，构建并发布多平台安装产物
- 主要执行内容：
  - 生成 beta 版本元数据
  - 构建 mac arm64、Windows x64、Linux x64 产物
  - 生成校验文件与更新器 feed 文件
  - 上传构建产物
  - 创建或更新 GitHub prerelease
  - 更新 beta channel feed
- 触发方式：
  - `workflow_dispatch`
- 额外输入：
  - `signed`：控制是否生成签名/公证的 mac 构建

### `release-stable`

- 文件：`.github/workflows/release-stable.yml`
- 作用：执行正式版发布流程，先校验，再构建并发布稳定版产物
- 主要执行内容：
  - 生成 stable 版本元数据
  - 执行发布前校验
  - 构建 mac arm64、Windows x64、Linux x64 产物
  - 上传构建产物
  - 创建 draft release
  - 上传资产并发布 stable release
  - 失败时执行 release/tag 清理
- 触发方式：
  - `workflow_dispatch`
- 额外输入：
  - `mac_signed`：控制是否生成签名/公证的 mac 构建

### `landing-page-ci`

- 文件：`.github/workflows/landing-page-ci.yml`
- 作用：专门校验 `apps/landing-page`，避免每次都跑全仓库主 CI
- 主要执行内容：
  - 安装依赖
  - 对 landing page 执行 `typecheck`
  - 构建 landing page
  - 校验生成 HTML 中没有外部 JavaScript
  - 校验 Cloudflare 图片缩放 URL 是否符合预期
- 触发方式：
  - `pull_request`
    - 受 `paths` 限制，仅在以下路径变化时触发：
      - `.github/workflows/landing-page-ci.yml`
      - `.github/workflows/landing-page.yml`
      - `apps/landing-page/**`
      - `package.json`
      - `pnpm-lock.yaml`
      - `pnpm-workspace.yaml`
  - `push` 到 `main`
    - 同样受上述 `paths` 限制
  - `workflow_dispatch`

### `landing-page-deploy`

- 文件：`.github/workflows/landing-page-deploy.yml`
- 作用：构建并部署 landing page 到 Cloudflare Pages
- 主要执行内容：
  - 安装依赖
  - 执行 `typecheck`
  - 构建 landing page
  - 校验 HTML 中没有不允许的外部 JS
  - 校验 Cloudflare 图片 URL
  - 部署 `apps/landing-page/out` 到 Cloudflare Pages
- 触发方式：
  - `push` 到 `main`
    - 受 `paths` 限制，仅在以下路径变化时触发：
      - `.github/workflows/landing-page-deploy.yml`
      - `.github/workflows/landing-page-ci.yml`
      - `apps/landing-page/**`
      - `package.json`
      - `pnpm-lock.yaml`
      - `pnpm-workspace.yaml`
  - `workflow_dispatch`

### `github-metrics`

- 文件：`.github/workflows/metrics.yml`
- 作用：生成仓库 GitHub metrics SVG，并在内容变化时通过 PR 更新仓库文件
- 主要执行内容：
  - 使用 `lowlighter/metrics` 生成 `docs/assets/github-metrics.svg`
  - 当数据变化时，自动创建或更新 PR
- 触发方式：
  - `schedule`
    - 每天 UTC `00:15`
  - `workflow_dispatch`
  - `push` 到 `main`
    - 仅当 `.github/workflows/metrics.yml` 变化时触发

### `refresh-contributors-wall`

- 文件：`.github/workflows/refresh-contributors-wall.yml`
- 作用：定时刷新 README 中 contributors wall 的 `cache_bust` 日期，并自动创建更新 PR
- 主要执行内容：
  - 更新 `README*.md` 中的 `cache_bust=YYYY-MM-DD`
  - 使用 `peter-evans/create-pull-request` 自动创建 PR
- 触发方式：
  - `schedule`
    - 每天 UTC `01:00`
  - `workflow_dispatch`

## 按触发方式汇总

### `pull_request`

- `ci`
- `landing-page-ci`

### `push` 到 `main`

- `ci`
- `landing-page-ci`
- `landing-page-deploy`
- `github-metrics`（仅 workflow 文件变更时）

### `workflow_dispatch`

- `ci`
- `release-beta`
- `release-stable`
- `landing-page-ci`
- `landing-page-deploy`
- `github-metrics`
- `refresh-contributors-wall`

### `schedule`

- `github-metrics`
- `refresh-contributors-wall`
