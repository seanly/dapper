# Fork 仓库发布 Release 与 Homebrew Tap 指南

本指南说明如何在你的 dapper fork 上通过 GitHub Actions 发布 Release，并通过自建 Homebrew tap 提供 `brew install` 安装。

## 一、整体流程

1. **GitHub Actions**：在 fork 里推送 `v*` tag 时，自动构建多平台二进制、打 GitHub Release，并**更新你的 Homebrew tap 仓库**。
2. **Homebrew Tap**：一个独立的 GitHub 仓库（如 `你的用户名/homebrew-tap`），存放 Formula；用户通过 `brew tap 你的用户名/tap` 和 `brew install 你的用户名/tap/dapper` 安装。

配置已按「当前仓库 owner」写好，fork 后无需改仓库名，只需准备好 Secret 和 tap 仓库即可。

---

## 二、GitHub 配置

### 2.1 所需 Secrets

在 **fork 的仓库** Settings → Secrets and variables → Actions 中配置：

| Secret | 说明 | 必填 |
|--------|------|------|
| `PUSH_GITHUB_TOKEN` | GitHub PAT（Personal Access Token），需勾选 `repo`、`write:packages`；用于推送到 **tap 仓库** 和登录 ghcr.io | 是 |

- **GITHUB_TOKEN** 由 Actions 自动注入，只能操作当前仓库（创建 Release、上传资产）。
- 推送到 **tap 仓库** 必须使用 PAT（`PUSH_GITHUB_TOKEN`），因为 tap 是另一个仓库。

### 2.2 创建 PAT（PUSH_GITHUB_TOKEN）

1. GitHub → Settings → Developer settings → Personal access tokens → Generate new token (classic)。
2. 勾选：`repo`（完整）、`write:packages`（如需推送到 ghcr.io）。
3. 将生成的 token 存为 fork 仓库的 Secret：`PUSH_GITHUB_TOKEN`。

---

## 三、创建 Homebrew Tap 仓库

Tap 命名约定为 `homebrew-tap`，对应 brew 命令中的 `tap` 名为 `你的用户名/tap`。

### 3.1 创建仓库

1. 在 GitHub 新建仓库，名称：**homebrew-tap**。
2. 可见性选 Public，可不勾选「Add a README」（GoReleaser 会推送 Formula）。

### 3.2 可选：先手动建一个 Formula（便于测试 tap）

在 **homebrew-tap** 仓库中创建：

- 目录：**Formula**
- 文件：**Formula/dapper.rb**

内容示例（把 `YOUR_GITHUB_USERNAME` 换成你的 GitHub 用户名，`x.x.x` 换成当前版本号）：

```ruby
# Formula/dapper.rb
class Dapper < Formula
  desc "Docker build wrapper (Dapper)"
  homepage "https://github.com/YOUR_GITHUB_USERNAME/dapper"
  version "x.x.x"

  on_macos do
    on_intel do
      url "https://github.com/YOUR_GITHUB_USERNAME/dapper/releases/download/v#{version}/dapper_#{version}_darwin_amd64.tar.gz"
      sha256 "REPLACE_WITH_REAL_SHA256"
    end
    on_arm do
      url "https://github.com/YOUR_GITHUB_USERNAME/dapper/releases/download/v#{version}/dapper_#{version}_darwin_arm64.tar.gz"
      sha256 "REPLACE_WITH_REAL_SHA256"
    end
  end

  on_linux do
    on_intel do
      url "https://github.com/YOUR_GITHUB_USERNAME/dapper/releases/download/v#{version}/dapper_#{version}_linux_amd64.tar.gz"
      sha256 "REPLACE_WITH_REAL_SHA256"
    end
    on_arm do
      url "https://github.com/YOUR_GITHUB_USERNAME/dapper/releases/download/v#{version}/dapper_#{version}_linux_arm64.tar.gz"
      sha256 "REPLACE_WITH_REAL_SHA256"
    end
  end

  def install
    bin.install "dapper"
  end

  test do
    system "#{bin}/dapper", "--version"
  end
end
```

首次发布后，从 GitHub Release 下载对应 tar.gz，在本地执行 `shasum -a 256 文件名` 得到 sha256 替换上述占位。**之后每次用 tag 发布，GoReleaser 会自动更新该 Formula。**

---

## 四、本地验证 GoReleaser

在推 tag 触发 CI 之前，可在本机用 **snapshot 模式** 跑一遍 GoReleaser（只构建、不发布）：

```bash
# 安装 GoReleaser（可选）
brew install goreleaser/tap/goreleaser

cd /path/to/dapper
goreleaser release --snapshot --skip publish --rm-dist
```

产物在 `dist/` 下，可检查二进制和压缩包是否正常。

---

## 五、发布 Release 的步骤

1. 在 fork 仓库中确认代码已提交并推送到默认分支（如 `main` 或 `master`）。
2. 打 tag 并推送，例如：
   ```bash
   git tag v1.0.0
   git push origin v1.0.0
   ```
3. 打开 fork 仓库的 **Actions** 页，查看 **release** workflow 运行。
4. 完成后：
   - **Releases** 页会出现该版本的 Release 和二进制资产；
   - 你的 **homebrew-tap** 仓库中 `Formula/dapper.rb` 会被 GoReleaser 自动更新（若已存在则覆盖）。

---

## 六、用户安装方式（你的 tap）

用户安装你 fork 的 dapper：

```bash
brew tap 你的用户名/tap
brew install 你的用户名/tap/dapper
```

例如 GitHub 用户名为 `seanly`：

```bash
brew tap seanly/tap
brew install seanly/tap/dapper
```

---

## 七、本仓库已做的配置说明

- **`.github/workflows/build.yaml`**
  - 触发器：`pull_request` / `push` 到 `main` 或 `master`。
  - 使用 `Makefile.ci` 执行 `go vet` 和 `go build`（无单元测试时可仅做构建与静态检查）。

- **`.github/workflows/release.yaml`**
  - 触发器：`push` 到 tag `v*`。
  - 使用 `PUSH_GITHUB_TOKEN` 登录 ghcr.io（若将来启用 Docker 构建）。
  - 将 `PUSH_GITHUB_TOKEN` 传给 GoReleaser 作为 `HOMEBREW_TAP_GITHUB_TOKEN`，用于推送到你的 homebrew-tap。

- **`.goreleaser.yml`**
  - `brews.tap.owner`：使用 `GITHUB_REPOSITORY_OWNER`（Actions 注入），即当前仓库 owner。
  - `brews.tap.name`：`homebrew-tap`（对应 `brew tap 用户名/tap`）。
  - 构建使用 go modules（不 vendoring），依赖由 go.mod/go.sum 管理。

因此你**不需要**在仓库里改任何用户名，只要：

1. 在 fork 里配置好 `PUSH_GITHUB_TOKEN`，
2. 创建并准备好 **homebrew-tap** 仓库（可先空仓库，或按 3.2 先建 Formula），
3. 推送 `v*` tag，

即可在 fork 的 Release 中发布，并通过 `brew tap 你的用户名/tap && brew install 你的用户名/tap/dapper` 安装。
