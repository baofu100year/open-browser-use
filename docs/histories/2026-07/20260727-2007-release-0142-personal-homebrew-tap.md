## [2026-07-27 20:07] | Task: 发版 0.1.42 并为个人 fork 搭建独立 Homebrew tap

### 🤖 Execution Context

- **Agent ID**: `Claude Code`
- **Base Model**: `Sonnet 5`
- **Runtime**: `Claude Code CLI`

### 📥 User Query

> Edge 浏览器支持完成后，用户要求 commit + push，并希望通过 `brew tap` 这个仓库（`baofu100year/open-browser-use` fork）装上带 Edge 支持的新版二进制。确认沿用上游 `iFurySt/open-browser-use` 的方式：独立 tap 仓库 + CI 自动渲染 formula + 下载预编译 tarball，而不是 build-from-source 或手动一次性发布。

### 🛠 Changes Overview

**Scope:** `cmd/open-browser-use`, `.github/workflows`, `scripts`, 各 SDK/extension package 版本号, `docs`

**Key Actions:**

- **[个人 tap 基础设施]**: 新建 `baofu100year/homebrew-open-browser-use` 仓库作为个人 Homebrew tap；在 `baofu100year/open-browser-use` 配置 `HOMEBREW_TAP_TOKEN` secret（复用当前已授权 Contents:RW 的 fine-grained PAT，验证过对新 tap 仓库也有写权限）。
- **[修复 fork 无法复用的硬编码]**: `scripts/render-homebrew-formula.sh` 原来把 `homepage` 和下载 URL 硬编码指向 `iFurySt/open-browser-use`，导致即使 CI 在 fork 上跑通，渲染出的 formula 也会去上游仓库的 release 找 tarball（下载 404 或 sha256 不符）。改为新增第 6 个参数 `github-repository`，由 `.github/workflows/homebrew-publish.yml` 传入 `${GITHUB_REPOSITORY}`，使渲染结果始终指向实际运行 CI 的仓库。
- **[tap 目标仓库切换]**: `.github/workflows/homebrew-publish.yml` 的 `HOMEBREW_TAP_REPOSITORY` 从 `iFurySt/homebrew-open-browser-use` 改为 `baofu100year/homebrew-open-browser-use`。
- **[版本号提升]**: `cmd/open-browser-use/main.go` 的 `const version`、`apps/chrome-extension/manifest.json`、`packages/open-browser-use-cli/package.json`、`packages/browser-client-rewrite/package.json`、`packages/open-browser-use-js/package.json`、`packages/browser-use-protocol/package.json`、`packages/open-browser-use-python/pyproject.toml` 从 `0.1.41` 统一提升到 `0.1.42`。`main_test.go` 中的版本断言均通过符号引用 `version` 常量，未需要改动。
- **[发布记录]**: `docs/releases/feature-release-notes.md` 新增 2026-07-27 行，记录 Microsoft Edge 浏览器支持随 `0.1.42` 发布。

### 🧠 Design Intent (Why)

用户的 fork 需要一条独立于上游的发布通道，才能把 fork 里已经实现的 Edge 支持真正装到本地 Homebrew。直接复用上游 `release.yml`/`homebrew-publish.yml` 的思路（tag push 触发、CI 生成预编译 tarball、渲染 formula 推到独立 tap 仓库）成本最低——两个 workflow 本身不需要新增逻辑，只需要让"目标 tap 仓库"和"formula 里引用的下载源仓库"都从硬编码改成指向 fork 自己。`render-homebrew-formula.sh` 的硬编码是这次发现的唯一实质性代码问题：如果不修，CI 会"看起来成功"（渲染、push formula 都不报错），但用户 `brew install` 时会静默拿到上游的二进制或直接因 sha256 不匹配装不上，属于容易被忽略的隐性 bug，因此单独修掉而不是绕过。

`CHROME_EXTENSION_PRIVATE_KEY`、`CWS_*` 等上游发布链路用到的 secret 在 fork 上均未配置；确认 `package-chrome-extension-crx.mjs` 在私钥缺失时会自动生成临时签名密钥、Chrome Web Store 提交步骤本身有 `if` 条件保护不会在缺 secret 时被触发，因此不需要额外处理即可让 `release.yml` 在 fork 上跑通。

### 📁 Files Modified

- `cmd/open-browser-use/main.go`
- `apps/chrome-extension/manifest.json`
- `packages/open-browser-use-cli/package.json`
- `packages/browser-client-rewrite/package.json`
- `packages/open-browser-use-js/package.json`
- `packages/browser-use-protocol/package.json`
- `packages/open-browser-use-python/pyproject.toml`
- `docs/releases/feature-release-notes.md`
- `.github/workflows/homebrew-publish.yml`
- `scripts/render-homebrew-formula.sh`

### ⚠️ 已知局限 / 后续动作

- 尚未验证 tag push 触发后 CI 实际跑通的结果（release.yml 产出 4 个 CLI tarball、homebrew-publish.yml 成功渲染并推送 formula 到 `baofu100year/homebrew-open-browser-use`）——这是下一步要做的事。
- `npm-publish.yml`、`pypi-publish.yml` 会被同一个 tag push 一起触发，fork 上没有配置对应发布凭证，预期会失败；这是已知且无害的行为（不影响 Homebrew 链路），未做额外处理去抑制这两个 workflow。
