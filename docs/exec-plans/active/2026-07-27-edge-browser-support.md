# Edge 浏览器支持

## 目标

让 `open-browser-use` 的 profile 发现、native messaging host 安装、
External Extensions 注册（best-effort）、`setup` / `setup beta` 打开浏览器
逻辑都能识别并支持 Microsoft Edge（Chromium 内核），复用现有
`--browser <selector>` 机制,不新增平行命令。

## 范围

- 包含：
  - `supportedBrowserProfileRoots()` 新增 `edge` browser id（macOS/Windows/
    Linux 三平台 user-data root）。
  - native messaging host manifest 安装路径按 browser id 参数化，覆盖
    macOS（已走 `browserRootForInstallSelector`，只需补 root）、
    Windows（registry key vendor/product 前缀改为按 browser id 查表）、
    Linux（当前完全硬编码 Chrome 路径，改为按 selector 派生）。
  - External Extensions 安装路径同样按 browser id 参数化（macOS/Windows/
    Linux），并在 Edge 场景下的文案/返回值中明确标注"best-effort，不保证
    生效"，因为 Edge 的 `update_url` 生态与 Chrome Web Store 不同源。
  - macOS 上 `open -a "Google Chrome"` 三处硬编码调用改为按 browser
    selector 映射 app 名称（`edge` -> `Microsoft Edge`），并让
    `setupChrome` / `newSetupBetaCommand` 把当前 `--browser` 值传下去。
  - `open-browser-use setup beta --browser edge` 端到端可用（unpacked +
    zip 拖拽安装路径，不依赖任何商店基础设施）。
  - `docs/ARCHITECTURE.md`、`skills/open-browser-use/SKILL.md`、
    `skills/open-browser-use/references/installation.md` 补充 Edge 相关
    selector 示例和平台说明。
  - Go 单测覆盖新增 Edge 分支（参考现有 chrome-beta 测试模式）。
- 不包含：
  - 不做 Microsoft Edge Add-ons 商店的官方发布流程（平行于
    `chrome-web-store-publish.yml` 的 Edge 发布链路留作后续 plan）。
  - 不实现 `setup --browser edge`（非 beta）的"保证生效"承诺；只 best-effort
    写 External Extensions 注册,并在输出中明确告知用户大概率需要手动在
    `edge://extensions` 启用开发者模式加载 beta ZIP。
  - 不处理 Edge 的 profile 迁移/多实例（Edge Beta/Dev/Canary channel）；
    第一版只支持 Edge Stable,与当前 Chrome-only-stable-plus-beta 的覆盖
    深度保持一致的取舍。
  - 不新增 BitBrowser 之外的第三方多实例浏览器。

## 背景

- 相关文档：
  - `docs/ARCHITECTURE.md`（Chrome route 拓扑、multi-browser 段落）。
  - `docs/exec-plans/active/2026-05-14-multi-browser-selection.md`（已完成的
    Chrome Stable/Beta/BitBrowser 多浏览器基础设施,Edge 复用同一套
    `browserProfileRoot` / selector 匹配框架）。
  - `docs/exec-plans/active/2026-05-12-chrome-profile-selection.md`
    第23行明确把 Edge 排除在当时范围外,这份 plan 是其后续。
- 相关代码路径（`cmd/open-browser-use/main.go`）：
  - `supportedBrowserProfileRoots()`（约1520-1571行）：browser id ->
    user-data root 的唯一权威映射表。
  - `installNativeManifestForBrowser()`（约303-343行）、
    `defaultNativeHostManifestPathForBrowser()`（约2880-2904行）：native
    messaging manifest 安装路径,Windows 分支硬编码
    `HKCU\Software\Google\Chrome\NativeMessagingHosts\...`。
  - `installChromeExternalExtensionForBrowser()`（约1051-1085行）、
    `defaultChromeExternalExtensionPathForBrowser()`（约2806-2831行）：
    External Extensions 安装,Windows 分支硬编码
    `HKCU\Software\Google\Chrome\Extensions\...` 且有 selector 守卫拒绝
    非 chrome selector。
  - `openFile()`（3314行,当前未被调用）、`openChromeExtensionsPage()`
    （3350行）、`openChromeWebStorePage()`（3368行）：macOS 分支硬编码
    `open -a "Google Chrome"`,均不接受 browserSelector 参数。
  - `browserRootForInstallSelector()`（2906-2942行）、
    `browserSelectorMatches()`（2493-2511行）：通用 selector 匹配框架,
    Edge 只要在 root 表里注册即可复用,无需改动。
- 已知约束：
  - Edge 沿用 Chromium 的 `chrome-extension://` origin scheme 和 CRX id
    算法,manifest 的 `allowed_origins` 字段和 host name 命名规则不需要
    为 Edge 单独改动。
  - Edge 官方文档要求 native messaging registry key 使用
    `HKCU\Software\Microsoft\Edge\NativeMessagingHosts\<host_name>`
    （vendor/product 前缀与 Chrome 不同）。
  - Edge 官方文档的 external extension 机制：Windows 走
    `HKCU\Software\Microsoft\Edge\Extensions\<id>`；Linux 走 JSON manifest
    文件 `/opt/microsoft/msedge/extensions/<id>/manifest.json`（格式与
    Chrome 的纯 registry 或纯目录路径不同,Linux 尤其要注意这不是目录路径
    而是每个扩展一个 manifest 文件）。
  - Edge 在 2025 年底已经收紧或移除"允许从其他商店安装扩展"的用户开关,
    `external_update_url` 指向 Chrome Web Store 的做法在 Edge 上不保证
    生效;这是本 plan 把 `setup --browser edge` 明确降级为 best-effort、
    并把 `setup beta --browser edge` 作为主推路径的原因。
  - macOS 上 Edge 应用名为 `Microsoft Edge`（`/Applications/Microsoft
    Edge.app`,已在本机确认存在）。

## 风险

- 风险：Windows registry key 的 vendor/product 前缀如果查表逻辑写错,会
  同时影响现有 Chrome/Chrome Beta 行为（回归风险）。
  - 缓解：新增一个纯函数把 `browserID -> (vendor, product)` 映射独立出来,
    先用现有 chrome/chrome-beta 测试覆盖回归,再加 edge 分支。
- 风险：Linux 分支重构涉及三处硬编码路径同时改动（manifest 路径、
  external extension 路径、且当前 Linux 完全不支持 chrome-beta/
  bitbrowser),范围扩大容易牵连出无关改动。
  - 缓解：只按 selector 派生 Chrome 自身路径与新增 Edge 路径,不在这个
    plan 里补齐 Linux 的 chrome-beta/bitbrowser 支持（记录到
    `docs/exec-plans/tech-debt-tracker.md`）。
- 风险：无法在本机验证 Windows 注册表行为和 Linux 路径的真实生效情况
  （当前开发环境是 macOS）。
  - 缓解：Windows/Linux 分支的改动只做路径生成的单元测试验证,不声称做过
    真机验证;在 plan 的验证方式里如实注明。
- 风险：Edge 的 External Extensions 自动安装大概率不生效,如果文案不清楚
  会误导用户以为跟 Chrome 一样能自动装上。
  - 缓解：`setup --browser edge` 的输出增加明确提示,建议改用
    `setup beta --browser edge`。

## 里程碑

1. Go 后端：`supportedBrowserProfileRoots()` 新增 Edge root（三平台）。
2. Go 后端：native messaging manifest 安装路径参数化（Windows registry
   vendor 查表 + Linux selector 派生),覆盖 Edge。
3. Go 后端：External Extensions 安装路径参数化,覆盖 Edge,并调整
   `setup` 在 Edge 场景下的输出文案。
4. Go 后端：`openChromeExtensionsPage()` / `openChromeWebStorePage()` 按
   selector 映射 macOS app 名称,调用方传入当前 `--browser` 值。
5. 测试：为以上每处新增单元测试。
6. 文档：更新 `docs/ARCHITECTURE.md`、`skills/open-browser-use/SKILL.md`、
   `skills/open-browser-use/references/installation.md`。
7. 验证与收尾：跑 Go 测试套件,本机（macOS）跑一次 Edge 实机
   `setup beta --browser edge` 验证,记录 history。

## 验证方式

### 命令（自动化）

- `go test ./cmd/open-browser-use/...` 必须包含新增的 Edge 相关用例,且
  现有 chrome/chrome-beta/bitbrowser 用例保持通过（防回归）。
- 仓库级 `scripts/ci.sh` 照常通过。

### 手工 test case（本机 macOS,已确认装有 Microsoft Edge.app）

1. `go run ./cmd/open-browser-use profiles --json` → 期望输出包含
   `browser: "edge"` 的条目（前提：Edge 至少启动过一次,存在
   `~/Library/Application Support/Microsoft Edge/Default`）。
2. `go run ./cmd/open-browser-use install-manifest --browser edge --path
   <built-binary>` → 期望写入
   `~/Library/Application Support/Microsoft Edge/NativeMessagingHosts/com.ifuryst.open_browser_use.extension.json`。
3. `go run ./cmd/open-browser-use setup beta --browser edge --no-open` →
   期望生成 keyed unpacked 目录 + ZIP,注册 Edge native manifest 成功。
4. 手动在 Edge 里打开 `edge://extensions`,启用开发者模式,加载第3步生成
   的 unpacked 目录 → 期望扩展加载成功且 extension id 与 Chrome 侧
   unpacked 一致。
5. `go run ./cmd/open-browser-use info --browser edge --timeout 5s` →
   期望连接到 Edge 侧 host。
6. Windows/Linux 分支（registry key 生成、Linux 路径生成）：只做单元
   测试层验证,不具备本机真机验证条件,在 history 中如实注明。

### 观测检查

- `setup --browser edge` 的输出必须明确提示"External Extensions 自动
  安装在 Edge 上不保证生效,建议使用 setup beta"。

## 进度记录

- [x] M1 `supportedBrowserProfileRoots()` 新增 Edge root
- [x] M2 native messaging manifest 路径参数化（Windows + Linux）
- [x] M3 External Extensions 路径参数化 + setup 文案调整
- [x] M4 macOS `open -a` app 名称按 selector 映射
- [x] M5 单元测试
- [x] M6 文档更新
- [x] M7 实机验证（macOS）与 history 记录

## 实机验证记录

- `go run ./cmd/open-browser-use profiles --json`：本机实际检测到
  `edge:Default`（用户机器上此前已通过 unpacked 方式装好插件），证明 Edge
  root 发现和 profile 扫描逻辑在真实 Edge 安装上可用。
- `install-manifest --browser edge --path <binary>`：写入
  `~/Library/Application Support/Microsoft Edge/NativeMessagingHosts/com.ifuryst.open_browser_use.extension.json`，
  内容中 `path` 字段指向全局稳定 native host link，`allowed_origins` 为
  keyed extension id 对应的 `chrome-extension://` origin，与 Chrome 侧格式
  一致。
- `setup beta --browser edge`：生成的 keyed unpacked 扩展 id
  `pnbmoicbkopffjjgfgfglopechaiemkp` 与用户 Edge 中实际已安装的扩展 id
  完全一致，证明 keyed unpacked 生成逻辑在 Chrome/Edge 间具有确定性和
  一致性。
- `setup --browser edge`：输出正确包含 best-effort 提示文案，并指向
  `setup beta --browser edge` 作为替代路径。
- `info --browser edge --timeout 5s`：成功连接到 Edge 侧已运行的 host，
  返回真实的 `extensionInstanceId` 等 metadata，证明 selector 路由和 socket
  归属反查在 Edge 上和 Chrome/Chrome Beta 行为一致。
- Linux 分支（`defaultNativeHostManifestPathForBrowser`、
  `defaultChromeExternalExtensionPathForBrowser` 的 linux case）：额外在
  `golang:1.25` Docker 容器（真实 Linux 内核/文件系统）中重新运行了相关单
  元测试并全部通过，比单纯在 macOS 上跑字符串断言更有说服力，但**不等于
  在真实 Linux 桌面环境装有 Chrome/Edge 的情况下做过端到端验证**。
- Windows 分支（`windowsRegistryBrowserVendorProduct` 查表函数、registry
  key 写入路径）：**没有任何本机或容器可用的 Windows 环境**，只做了纯函数
  单元测试（覆盖 `""`/`chrome`/`chrome-beta`/`edge`/display name/未知
  selector 报错等用例），未验证 `reg add` 真实写入注册表后 Edge 是否真的
  识别到 native messaging host 或 external extension。这是本次交付已知的
  验证空白，按 plan「风险」章节约定如实记录，而非声称已验证。

## 决策记录

- 2026-07-27：`setup --browser edge`（非 beta,依赖 Chrome Web Store
  External Extensions 自动安装机制）保留但降级为 best-effort,不追求
  "保证生效"；主推 `setup beta --browser edge`（unpacked + ZIP 拖拽,不
  依赖任何商店基础设施）。理由：Edge 的扩展安装生态已与 Chrome Web
  Store 脱钩（2025 年底收紧"允许从其他商店安装"选项）,继续假装 Edge 能
  和 Chrome 一样自动安装会误导用户；但保留该路径的价值在于:一旦 Edge
  策略允许,或企业环境通过策略开启了该选项,用户不需要额外命令即可享受
  同样的体验。
- 2026-07-27：第一版只支持 Edge Stable,不支持 Edge Beta/Dev/Canary
  channel。理由：与当前 Chrome 覆盖深度（Stable + Beta,无 Dev/Canary）
  保持一致的取舍标准,后续如有需求再单独立 plan。
- 2026-07-27：不在这个 plan 里补齐 Linux 的 chrome-beta/bitbrowser 支持
  缺口,即使改动 Linux native messaging manifest 路径生成函数时会经过
  那段代码。理由：保持这个 plan 的范围聚焦在"新增 Edge",避免把两个独立
  问题（Edge 支持 vs Linux 多浏览器基础设施滞后）混在一次改动里。
- 2026-07-27：`browserSelectorMatches(selector, info)` 对空字符串 selector
  恒返回 `true`（语义是"未显式选择 browser 时匹配任意浏览器"）。这意味着
  任何新增的"判断当前 selector 是否明确指向 Edge"逻辑（Linux 路径派生、
  External Extensions 路径、`setupChrome` 的 best-effort 提示、macOS app
  名称映射）都必须先把空字符串归一化为默认值 `"chrome"`,再传入
  `browserSelectorMatches`,否则未传 `--browser` 的默认场景会被误判为
  Edge。实现过程中三处独立代码都在 first pass 时漏掉这一步,均在自查时
  发现并改正。记录为决策/坑点,供后续新增其他浏览器 selector 分支时参考,
  避免重复踩坑。
