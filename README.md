# Android 开发规范与细节要求（LSPosed 模块）

> **用途**：个人 Android LSPosed 模块开发的可复用规范汇总，沉淀自 **DeviceResetSpoofer（DRS）** 与 **RamStatusBar（RSB）** 两个项目。开发新软件、或让 AI 协助开发时，先读本文件，遵循其中的流程、常量、清单与避坑项。
>
> **语言顺序约定**：本仓库与所有项目 README、UI 文案统一为 **英文 → 中文 → 俄文**。

---

## 0. 文档使用说明（AI 与未来的我都先读这段）

- **先读目录**：本文件按「开发流程 → 构建 CI → UI → 更新机制 → 发版 → 避坑 → 新项目清单」组织，按需跳转。
- **版本号可能滞后**：文中的「当前版本」是写文档时的快照。**任何发版动作前，必须先通过 `gh api repos/{owner}/{repo}/contents/app/build.gradle` 读取真实 versionCode / versionName**，禁止凭记忆或本文件写死。
- **不要本地 clone 构建**：个人机器无完整 Android 环境，一律走 GitHub 在线编辑 + GitHub Actions 出包（见 §2、§3）。
- **改完必须复核**：用 GitHub API 改完文件后，**必须重新 GET 该文件验证内容确实已更新**（避免误以为改了）。

---

## 1. 项目总览（当前状态快照）

| 项目 | 全称 | 包名（applicationId） | 源码包名 | 当前版本 | versionCode | minSdk | target/compileSdk |
|---|---|---|---|---|---|---|---|
| DRS | DeviceResetSpoofer | `io.github.gjr787878.devicereset` | 同左 | 2.19 | 39 | 24 | 34 |
| RSB | RamStatusBar | `io.github.gjr787878.ramstatusbar` | `com.example.ramstatusbar` | 1.4.17 | 30 | 26 | 34 |

> 版本号会在每次发版递增；**上表会过时**，以 build.gradle 实时值为准（见 §0）。

### 1.1 仓库与发布映射

| 项目 | 主仓库 | LSPosed 模块仓库 | 主仓库 Release tag | LSPosed tag 格式 | APK 命名 |
|---|---|---|---|---|---|
| DRS | `GJR787878/DeviceResetSpoofer` | `Xposed-Modules-Repo/io.github.gjr787878.devicereset` | `v{version}` | `{versionCode}-{version}` | `DeviceResetSpoofer-v{version}.apk` |
| RSB | `GJR787878/RamStatusBar` | `Xposed-Modules-Repo/io.github.gjr787878.ramstatusbar` | `v{version}` | `{versionCode}-{version}` | `RamStatusBar-v{version}-release.apk` |

### 1.2 技术栈

- **语言**：Java（纯代码构建 UI 为主；DRS 部分页面用 XML 布局）
- **Gradle**：`com.android.application` 插件；DRS/RSB 均为单 app 模块
- **Xposed**：`de.robv.android.xposed:api:82`（`compileOnly`，运行期由 LSPosed 提供）
- **依赖差异**：
  - DRS：`androidx.appcompat`、`material`、`androidx.recyclerview`、本地 `libs/api-82.jar`
  - RSB：`androidx.core`（用于 FileProvider）
- **签名**：debug 与 release 共用同一 release keystore（保证覆盖安装/升级不报签名冲突）；keyAlias 均为 `devicereset`

---

## 2. 开发流程（在线编辑模式）

### 2.1 代码提交原则
- 用 GitHub API（`gh` CLI 或 REST）直接在线提交，不本地 clone、不本地构建。
- 涉及多文件改动时，尽量合并为**一次 commit**（用 Git Data API：blob → tree → commit → update ref）。
- 改动后必须 GET 复核（见 §0）。
- **commit message 用中文标题**。

### 2.2 版本号
- **每次发版必须同时递增 `versionCode` 和 `versionName`**。
- versionCode 不递增 → 安装时显示「重新安装」而非「升级安装」（被判定为降级/同版本）。
- 发版前在 `app/build.gradle` 确认版本号已改对，且**已推送 main 且 CI 构建通过**。

---

## 3. 构建与 CI（GitHub Actions）

两个项目共用同一套 CI 模板（`main` 分支 push 触发；RSB 额外支持 `v*` tag 触发自动发版）。核心步骤与要点：

### 3.1 工作流结构（`build.yml`）
1. **Checkout** → **JDK 17**（temurin）→ **接受 Android SDK license**（见下）
2. **Decode keystore**：从 secret `DRS_KEYSTORE_B64` base64 解码出 keystore 到仓库根 `keystore/`，并写 `DRS_KEYSTORE_PATH` 到 `$GITHUB_ENV`
3. **gradlew 可执行**：RSB 若缺 gradlew 用 `gradle wrapper --gradle-version 8.4` 生成
4. **Build**：`./gradlew assembleDebug --stacktrace`（DRS 带 Gradle cache 步骤）
5. **Verify APK signature**：用 `apksigner verify --print-certs` 打印证书，**断言指纹等于固定值**
   `d21cd192f0b2e4c0cd81b52db9b8b5e2fe62376fd87b13e374560f124ad7f843`，不一致即 `exit 1` 失败
6. **Update latest_version.txt**：解析 build.gradle 的 versionName 写入仓库根 `latest_version.txt` 并 push main
7. **Upload APK artifact**（DRS：`DeviceResetSpoofer-debug`；RSB：`RamStatusBar-debug-apk`）
8. **仅 RSB（tag 触发）**：打包 ZIP + `softprops/action-gh-release@v2` 自动建 Release

### 3.2 Android SDK license 处理（重要坑）
`android-actions/setup-android@v3` 已停止维护、与新 cmdline-tools 不兼容（报 `Failed to find package 'tools'`）。改用：
```bash
mkdir -p "$ANDROID_HOME/licenses"
printf "8933bad161af4178b1185d1a37fbf41ea5269c55\nd56f5187479451eabf01fb78af6dfcb131a6481e\n24333f8a63b6825ea9c5514f83c2829b004d1fee\n" \
  > "$ANDROID_HOME/licenses/android-sdk-license"
```
缺失的 platform / build-tools 由 Gradle 自动下载。

### 3.3 latest_version.txt 步骤的守卫（关键坑）
- 该步骤必须加 **`if: github.ref == 'refs/heads/main'`**：
  - 否则在**分支 / PR 构建**里 `git push origin main` 会因本地无 main ref 报 `refspec main does not match any`，导致整个任务失败、无 APK 产物。
- 加 **`continue-on-error: true`**：避免推送冲突（多人/CI 并发改 latest_version.txt）时拖垮构建。
- `paths-ignore: ['latest_version.txt']`：CI 自己提交 latest_version.txt 时不重复触发构建。

---

## 4. UI 规范（可复用）

### 4.1 全局暗色主题
- **应用必须整体暗色**：所有页面黑底，所有系统弹窗（日期、时间、时区、颜色选择、AlertDialog）统一暗色，不得出现默认浅色弹窗。
- 实现：`res/values/styles.xml` 定义 `AppTheme`（parent `android:Theme.Material`），并在 Manifest `<application android:theme="@style/AppTheme">` 挂载。
  - 关键 item：`windowBackground=black`、`colorAccent=#0A84FF`、`colorPrimary=#1C1C1E`、`colorPrimaryDark=#000`、`textColorPrimary=#FFFFFF`
  - 显式指定 `alertDialogTheme` / `datePickerDialogTheme` / `timePickerDialogTheme` 为暗色弹窗主题（parent `android:Theme.Material.Dialog.Alert`）
- RSB 弹窗背景色 `#1C1C1E`，DRS `#2B2B2B`（`GradientDrawable` 设置，见 4.3）。

### 4.2 玻璃按钮（GlassButtonDrawable）
- 统一用 `GlassButtonDrawable(radius, border, false)`：
  - DRS：radius **24dp**，border 1dp
  - RSB：radius **28dp**，border 1dp
- 按钮文字大小 **14sp**。
- 类位置（两仓库各维护一份，样式同源）：
  - DRS：`DeviceResetSpoofer/app/src/main/java/io/github/gjr787878/devicereset/GlassButtonDrawable.java`
  - RSB：`RamStatusBar/app/src/main/java/com/example/ramstatusbar/GlassButtonDrawable.java`
- 新弹窗 / 新按钮一律复用此类，不另起样式。

### 4.3 弹窗规范
- **弹窗圆角与按钮圆角一致**（DRS 24dp / RSB 28dp）。
- 按钮与弹窗底部**间距 ≥ 20dp**（不贴底）。
- 弹窗窗口背景用 `GradientDrawable` 设置：`dialog.getWindow().setBackgroundDrawable(gd)`；自定义 View 根布局设透明 `ColorDrawable.TRANSPARENT`。

### 4.4 多语言（三语）
- 文案必须三语齐全：**中文（zh）/ 英文（en）/ 俄文（ru）**。
- App 内用 `lang(zh, en, ru)` 辅助函数 + SharedPreferences 存 `language` 键。
- **README / GitHub 主页介绍顺序：英文 → 中文 → 俄文**。

### 4.5 平板适配（sw600dp 断点）
> 原则：**不是简单拉伸**，按钮布局、导航形态都要重排。

**判定**：`getResources().getConfiguration().smallestScreenWidthDp >= 600` 即为平板。

**RSB（纯代码 UI）平板规则**：
- **导航栏**：从底部改为**屏幕左侧悬浮胶囊**（参考 Play 商店侧栏）：
  - 垂直居中，高度约为**屏幕一半**，**不铺满**；`gravity = LEFT | CENTER_VERTICAL`，`leftMargin ≈ 20dp`
  - 胶囊宽约 **72dp**（不宜过宽）；内容区左侧让出 `≈ 110dp`（`contentContainer.leftMargin`）
- **内容宽度上限**：平板内容宽上限 **760dp**（`contentWidthPx = min(screenW - 导航留白 - 48*2, 760dp)`），避免长文本整行拉伸。
- **按钮横排**（对比手机竖排）：
  - 配置页 3 个模式单选按钮 → 横排一行（RadioGroup 设 `HORIZONTAL`，按钮 `width=0, weight=1`，左右留隙 6dp）
  - 设置页「背景颜色/时间设置/设备信息」3 个操作按钮 → 横排一行
  - 设置页语言单选（中文/English/Русский）→ 横排一行
  - 时间设置页「自动同步/选择时区/自定义时间」→ 横排一行
  - 颜色设置页「选择颜色/透明/返回」→ 横排一行
  - 主页「检查更新」按钮平板**居中固定宽度**（如 360dp），不整行拉伸

**DRS（XML 布局）平板规则**：
- 新增 `res/layout-sw600dp/` 同名布局覆盖默认布局：
  - `activity_config.xml`：8 个玻璃胶囊按钮改 **2 列网格**（Android ID|广告ID / IMEI|Build / MAC|GSF / 运营商|显示当前伪装值）
  - `activity_main.xml`：3 个操作按钮一行 + 2×2 玻璃卡片（含红色警告卡）
- Java 端（`AppPickerActivity`）：`smallestScreenWidthDp≥600` 时用 `GridLayoutManager` 双列。

---

## 5. 内置更新下载机制

### 5.1 「发现新版本」弹窗
- 结构：标题 + 描述；下方左右两个玻璃按钮 **取消 / 下载**。
- 圆角背景 + 玻璃按钮，与 App 其他弹窗风格一致。

### 5.2 「正在下载」进度弹窗
- 标题：`⬇️ 正在下载 v{version}`
- 横向 ProgressBar + 百分比文字。
- 下方两个玻璃按钮：
  - **取消**：停止下载、删除临时文件、关闭弹窗
  - **仓库主页**：取消下载、跳转 GitHub 仓库页面
- 下载完成 → 自动关闭弹窗 → 拉起系统安装界面。

### 5.3 下载 URL 顺序（重要！）
**代理优先、直连兜底**（国内直连 GitHub 会超时）。**连接超时 5 秒**，失败快速切下一个源。镜像列表顺序：
1. `ghfast.top`
2. `gh-proxy.com`
3. `ghproxy.net`
4. `gh.llkk.cc`
5. `mirror.ghproxy.com`
6. `github.moeyy.xyz`
7. `ghproxy.cc`
8. 直连 `github.com`（兜底）

### 5.4 安装（Android 8+）
- 权限：`REQUEST_INSTALL_PACKAGES`（Manifest）。
- 用 `androidx.core.content.FileProvider` 提供 `content://` URI。
- `res/xml/file_paths.xml`：
  ```xml
  <external-files-path name="downloads" path="Download/"/>
  <cache-path name="cache" path="."/>
  ```
- authority = `{applicationId}.fileprovider`（如 `io.github.gjr787878.ramstatusbar.fileprovider`），Manifest 中 `<provider>` 声明。
- 安装 Intent：`ACTION_VIEW` + `application/vnd.android.package-archive` + `FLAG_GRANT_READ_URI_PERMISSION` + `FLAG_ACTIVITY_NEW_TASK`。

---

## 6. 更新检测机制

### 6.1 多通道检测（提高成功率）
1. **GitHub API**：`api.github.com/repos/{repo}/releases/latest`
2. **页面 302**：`github.com/{repo}/releases/latest` 重定向提取 tag
3. **CDN / 代理读 `latest_version.txt`**（国内主要通道）：
   - jsDelivr（`cdn` / `fastly` / `gcore` 三个节点）
   - `ghfast.top`、`gh-proxy.com`、`ghproxy.net` 等代理读 raw.githubusercontent.com
   - **URL 末尾加 `?t={timestamp}` 防 CDN 缓存**
4. **IP 直连兜底**：绕过 DNS 污染，用 GitHub Anycast IP + Host 头 + TLS SNI

多源取**最大版本号**（`compareVersions`）。

### 6.2 latest_version.txt
- 仓库根目录纯文本，内容即版本号（如 `2.19`）。
- **每次发版必须更新**（CI 自动更新或手动 API 更新）。
- CI 工作流中更新步骤设 `continue-on-error: true` + `if: github.ref == 'refs/heads/main'`（见 §3.3）。

---

## 7. 发版流程

### 7.1 DRS（手动发版）
1. 改代码 → push main → CI 构建成功
2. 手动创建 Release（tag `v{version}`）
3. 上传 APK：`DeviceResetSpoofer-v{version}.apk`
4. 发布到 LSPosed 模块仓库（tag `{versionCode}-{version}`），上传同名 APK
5. 更新 `latest_version.txt` 为新版本号

### 7.2 RSB（tag 自动发版）
1. 改代码 → push main → CI 构建成功
2. 打 tag `v{version}` → CI 自动构建并创建 Release（打包 ZIP + 发布正文 `RELEASE_BODY.md`）
3. 发布到 LSPosed 模块仓库（tag `{versionCode}-{version}`），上传 `RamStatusBar-v{version}-release.apk`
4. 更新 `latest_version.txt` 为新版本号

### 7.3 发版后核对清单
- [ ] 主仓库 `releases/latest` 已是新 tag
- [ ] LSPosed 模块仓库 `releases/latest` 已是 `{versionCode}-{version}`
- [ ] `latest_version.txt` 内容 == versionName
- [ ] 双仓库 APK 资产大小一致 / 可下载
- [ ] APK 验签指纹匹配（`apksigner verify --print-certs`）

---

## 8. 常见坑（故障排查表）

| # | 症状 | 原因 | 解决 |
|---|---|---|---|
| 1 | 安装显示「重新安装」 | versionCode 未递增 | 发版前递增 versionCode + versionName |
| 2 | 下载完不弹安装界面 | 缺 `REQUEST_INSTALL_PACKAGES` | Manifest 加权限 |
| 3 | 国内下载白等很久 | 直连 GitHub 优先、超时太长 | 代理优先 + 超时 5s + 直连兜底 |
| 4 | 检测不到新版本 | `latest_version.txt` 没更新 | 发版必须更新该文件 |
| 5 | CDN 返回旧版本 | jsDelivr 等缓存 | URL 加 `?t={timestamp}` |
| 6 | 安装时崩溃 | FileProvider authority 不匹配 | authority = `{applicationId}.fileprovider`，与 Manifest 一致 |
| 7 | 分支/PR 构建失败无产物 | latest_version.txt 步骤 push main 失败 | 该步骤加 `if: github.ref == 'refs/heads/main'` |
| 8 | CI 更新版本文件冲突 | 并发推送 | 该步骤加 `continue-on-error: true` |
| 9 | 本地文件与线上不同步 | 误以为本地已改 | 用 API 改完必须 GET 复核 |
| 10 | 系统弹窗白底，与暗色 UI 不统一 | 应用无暗色主题 | 设 `AppTheme` + 暗色弹窗主题（§4.1） |
| 11 | 平板布局被简单拉伸、不协调 | 无断点适配 | 用 sw600dp + 按钮横排 + 左侧胶囊导航（§4.5） |

---

## 9. 新项目初始化清单（下一个 App 照此检查）

- [ ] **Manifest**：加 `INTERNET` + `REQUEST_INSTALL_PACKAGES`；声明 Xposed meta-data（module / description / minversion / scope）；`theme=AppTheme`
- [ ] **styles.xml**：暗色 `AppTheme` + 暗色弹窗主题（§4.1）
- [ ] **build.gradle**：统一签名（debug 用 release keystore）；compileSdk/targetSdk 34；版本号起步
- [ ] **GlassButtonDrawable**：从 DRS/RSB 复用，统一按钮/弹窗圆角
- [ ] **三语文案**：所有 UI 字符串 zh/en/ru 三语；README 顺序 英→中→俄
- [ ] **CI workflow**：沿用模板；latest_version.txt 步骤加 main 守卫 + continue-on-error
- [ ] **更新机制**：镜像下载顺序 + FileProvider + file_paths.xml（§5、§6）
- [ ] **平板适配**：判定 sw600dp；导航/按钮重排，不做简单拉伸（§4.5）
- [ ] **发布**：先出 APK 审核 → 合并 → 双仓库发布 + 更新 latest_version.txt + 核对清单（§7.3）

---

> 维护提示：本文件随 DRS / RSB 的演进持续补充。新增踩坑、新规范时，保持「结论先行 + 表格 + 文件路径 + 常量」的结构，便于 AI 与后续快速读取执行。
