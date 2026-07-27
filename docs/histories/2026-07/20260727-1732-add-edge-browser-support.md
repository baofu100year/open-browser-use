## [2026-07-27 17:32] | Task: 为 open-browser-use 添加 Microsoft Edge 浏览器支持

### 🤖 Execution Context

- **Agent ID**: `Claude Code`
- **Base Model**: `Sonnet 5`
- **Runtime**: `Claude Code CLI`

### 📥 User Query

> 当前项目对于 edge 浏览器支持的不太好，帮我支持 edge 浏览器。经 AskUserQuestion 确认取舍：`setup --browser edge`（依赖 Chrome Web Store External Extensions 机制）保留但降级为 best-effort 并明确标注非官方保证；`setup beta --browser edge`（unpacked + ZIP 手动加载，不依赖任何商店基础设施）作为主推路径。

### 🛠 Changes Overview

**Scope:** `cmd/open-browser-use`, `docs/ARCHITECTURE.md`, `skills/open-browser-use`

**Key Actions:**

- **[Profile 发现]**: `supportedBrowserProfileRoots()` 新增 `edge` browser id 的 macOS/Windows/Linux user-data root，复用既有 `browserSelectorMatches` selector 匹配框架。
- **[Native messaging manifest 路径参数化]**: 新增纯函数 `windowsRegistryBrowserVendorProduct(browserSelector)`，把 `HKCU\Software\<vendor>\<product>\NativeMessagingHosts\...` 的 vendor/product 前缀按 browser id 查表（`chrome`/`chrome-beta` → `Google\Chrome`，`edge` → `Microsoft\Edge`），`installNativeManifestForBrowser` 的 Windows 分支改为调用它。Linux 分支（`defaultNativeHostManifestPathForBrowser`）从硬编码 Chrome 路径改为按 selector 派生 `~/.config/google-chrome` 或 `~/.config/microsoft-edge`。
- **[External Extensions 路径参数化]**: `installChromeExternalExtensionForBrowser` 与 `defaultChromeExternalExtensionPathForBrowser` 的 Windows 分支同样改为调用 `windowsRegistryBrowserVendorProduct` 查表（移除原先"拒绝非 chrome selector"的守卫）；Linux 分支按 selector 派生 `/opt/google/chrome/extensions` 或 `/opt/microsoft/msedge/extensions`。
- **[setup best-effort 文案]**: `setupResult` 新增 `BestEffortNote` 字段，`setupChrome` 在 selector 指向 Edge 时生成提示（Edge 已收紧"允许从其他商店安装扩展"，建议改用 `setup beta --browser edge`），`renderStoreSetupResult` 输出该提示。
- **[macOS app 名称映射]**: 新增纯函数 `macOSBrowserAppName(browserSelector)`（chrome → "Google Chrome"，chrome-beta → "Google Chrome Beta"，edge → "Microsoft Edge"），`openChromeExtensionsPage` / `openChromeWebStorePage` 改为接受 `browserSelector` 参数并调用它，替换原先三处硬编码的 `open -a "Google Chrome"`。
- **[CLI 帮助文案]**: `setup`/`setup beta`/`install-manifest` 的 `--browser` flag 说明更新为包含 `edge` 及 best-effort 提示。
- **[测试]**: 新增 8 个测试函数覆盖 Edge 分支（`TestCobraInstallManifestSupportsEdge`、`TestDefaultNativeHostManifestPathForBrowserLinuxSelectsEdge`、`TestDefaultChromeExternalExtensionPathForBrowserLinuxSelectsEdge`、`TestWindowsRegistryBrowserVendorProduct`、`TestMacOSBrowserAppName`、`TestListInstalledChromeProfilesIncludesEdge`、`TestCobraSetupBrowserEdgeMentionsBestEffort` 等），并额外在 `golang:1.25` Docker 容器中重跑 Linux 分支相关测试确认真实 Linux 环境下可编译运行。
- **[文档]**: 更新 `docs/ARCHITECTURE.md` 的 multi-browser 段落、`skills/open-browser-use/SKILL.md` 的 selector 列表、`skills/open-browser-use/references/installation.md` 的 Set Up A Browser / Platform Notes 章节，补充 Edge 相关说明。
- **[exec-plan]**: 新增 `docs/exec-plans/active/2026-07-27-edge-browser-support.md`，记录目标/范围/风险/里程碑/决策记录/实机验证记录。

### 🧠 Design Intent (Why)

Edge 是 Chromium 内核，`chrome-extension://` origin scheme 和 CRX id 算法与 Chrome 通用，因此 native messaging manifest 结构不需要为 Edge 单独改动，真正需要改的是"安装路径"这一层——Windows registry 的 vendor/product 前缀、Linux 的 XDG 风格路径此前都硬编码指向 Chrome。改动的核心是把这些路径生成函数从"只认识 chrome/chrome-beta"扩展为"按 browser selector 查表"，复用已有的 `browserSelectorMatches` 框架而不新增平行代码路径。

Edge 在 2025 年底收紧了"允许从其他商店安装扩展"的开关，导致 Chrome Web Store 式的 External Extensions 自动安装在 Edge 上不保证生效。与其掩盖这一差异让用户误以为 Edge 和 Chrome 体验一致，选择保留 `setup --browser edge` 路径（面向未来 Edge 策略放开或企业策略允许的场景）但明确标注 best-effort，同时把不依赖任何商店基础设施的 `setup beta --browser edge`（unpacked + ZIP 手动加载）作为主推路径。

实现中三次独立触发同一个陷阱：`browserSelectorMatches` 对空字符串 selector 恒返回 `true`（语义是"未显式选择时匹配任意浏览器"），任何"判断当前 selector 是否明确指向 Edge"的新逻辑都必须先把空字符串归一化为默认值 `"chrome"`，否则默认场景会被误判为 Edge。已记录进 exec-plan 决策记录，供后续新增其他浏览器分支时参考。

### 📁 Files Modified

- `cmd/open-browser-use/main.go`
- `cmd/open-browser-use/main_test.go`
- `docs/ARCHITECTURE.md`
- `skills/open-browser-use/SKILL.md`
- `skills/open-browser-use/references/installation.md`
- `docs/exec-plans/active/2026-07-27-edge-browser-support.md`（新增）

### ⚠️ 已知局限

- Windows 分支（registry vendor/product 查表、`reg add` 写入路径）只有纯函数单元测试覆盖，没有任何 Windows 机器或容器可用于验证 `reg add` 真实写入注册表后 Edge 是否识别到 native messaging host / external extension。
- Linux 分支在 `golang:1.25` Docker 容器（真实 Linux 内核/文件系统）中验证过路径生成逻辑的编译与单测通过，但不等于在装有真实 Chrome/Edge 的 Linux 桌面环境下做过端到端验证。
- macOS 手动验证覆盖 5 项 case（profile 发现、native manifest 安装、`setup beta --browser edge`、`setup --browser edge` 文案、`info --browser edge` 连接），均在本机真实已安装的 Microsoft Edge.app 上验证通过。
