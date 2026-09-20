# 通用开发规范与细节要求（参照模板）

> **用途**：个人软件开发的**可复用规范与细节要求**，沉淀自过往 Android 项目的实战经验，泛化为通用模板。
> 开发新项目、或让 AI 协助开发时，把其中的 `{占位符}` 替换为当前项目的实际值，遵循其中的流程、原则、清单与避坑项即可。
> **本文件是"参照"，不含具体项目信息**；具体项目各自的真实数值（版本、包名、仓库等）应在该项目自己的仓库/文档里维护。

---

## 0. 文档使用说明（先读这段）

- **这是通用参照，不是项目文档**：所有 `{xxx}` 占位符在具体项目里都要被替换成真实值。
- **版本号、依赖、配置都会过时**：任何发版/构建动作前，先读取当前项目的真实配置（如 `app/build.gradle`），**禁止凭本文件或记忆写死**。
- **无本地完整环境时**：一律走「在线编辑 + CI 出包」模式（见 §2）。
- **改完必须复核**：用 API 改文件后，必须重新读取确认确实更新（避免误以为改了）。
- **通用原则优先于具体工具**：下面提到 GitHub Actions、Android 等，只是当前技术栈；原则（验签、分支守卫、版本文件、缓存防串等）对任何 CI/平台通用。

---

## 1. 通用开发流程原则

### 1.1 提交与协作
- 改动可追溯：一个功能/修复尽量一次提交，commit message **用中文标题**、写清做了什么。
- 在线编辑时，多文件改动合并成**一次 commit**（用 Git Data API：blob → tree → commit → update ref）。
- 改完**必须重新读取（GET）验证**，尤其用 API 编辑时本地文件可能与线上不同步。

### 1.2 版本号
- 采用语义化版本 `主.次.修订`（如 `1.4.11`）。
- **每次发版必须同时递增 `versionCode`（内部递增号）与 `versionName`（展示版本）**。
- 递增号不递增 → 安装端会判定为「重新安装」而非「升级安装」（同/降版本）。
- 发版前确认版本号已改对、已合并到默认分支且 CI 构建通过。

### 1.3 双通道发布
- 常规发布到**自有仓库 Release**（tag `v{version}`）。
- 若产品投放到**分发/模块仓库**（如 LSPosed 模块仓库），另行发布一份，tag 建议格式 `{versionCode}-{version}`，并上传与主仓库一致的产物。
- 发布后两处都要核对为 `Latest`。

---

## 2. 构建与 CI（无本地环境时）

> 通用模板：`main` 分支 push 触发构建；需要时对 `v*` tag 触发自动发版。

### 2.1 工作流结构
1. **Checkout** → **设置语言运行时（如 JDK）** → **接受 SDK/依赖 license**
2. **解码签名材料**：从 CI secret 还原签名文件，写入工作区
3. **构建**：产出可分发包
4. **验证产物**：验签/校验指纹/跑测试
5. **更新版本文件**：把 `versionName` 写入仓库根 `latest_version.txt` 并推送
6. **上传产物**：上传为 CI artifact 供下载
7. **（可选，tag 触发）**：打包 + 自动创建 Release

### 2.2 关键坑
- **SDK license**：旧版 setup action 可能废弃；改为手动 `mkdir -p $ANDROID_HOME/licenses` 写入 license 字符串，缺失的 platform/build-tools 让构建工具自动下载。
- **验签断言**：构建后用 `apksigner verify --print-certs` 打印证书，**断言指纹等于固定 keystore 的指纹**，不一致即失败。防止签名漂移导致用户无法覆盖安装。
- **debug 也使用 release 签名**：保证调试包与正式包同签名，用户可覆盖安装/升级。
- **版本文件更新步骤加分支守卫**：`if: github.ref == 'refs/heads/main'`——否则**分支/PR 构建里 push main 会报 `refspec main does not match any`，直接失败、无产物**。
- **版本文件更新步骤加 `continue-on-error: true`**：避免并发推送冲突拖垮整个构建。
- 版本文件用 `paths-ignore` 排除：CI 自己提交版本文件的 commit 不要重复触发构建。

### 2.3 签名
- 生成**固定 keystore**，口令默认值写在 `build.gradle` 顶部、CI 用 secret 注入。
- 本地与 CI 用同一 keystore，保证覆盖安装/升级不报签名冲突。

---

## 3. UI 通用规范

### 3.1 全局主题统一
- 若产品走深色风格，**必须整体验深色**：所有页面深色底，所有系统弹窗（日期、时间、列表、选择器等）统一深色，**不得出现默认浅色弹窗**。
- 实现：定义全局 `AppTheme`（如基于 Material 深色主题），挂到 Manifest 的 `<application android:theme>`，并显式指定 `alertDialogTheme` / `datePickerDialogTheme` / `timePickerDialogTheme` 为深色弹窗主题。

### 3.2 统一控件风格
- **弹窗圆角与按钮圆角一致**（同一项目内统一一个 radius 值）。
- 按钮与弹窗底部间距 ≥ 20dp（不贴底）。
- 弹窗窗口背景用 `GradientDrawable` 设置；自定义 View 根布局设透明背景。
- 统一封装可复用控件（如玻璃/胶囊按钮类），**新弹窗、新按钮一律复用，不另起样式**。

### 3.3 多语言
- 文案必须多语齐全（如 中文/英文/俄文），用统一的多语言辅助函数 + 偏好存储存语言键。
- **对外文档/主页介绍语言顺序统一**（如 英文 → 中文 → 俄文）。

### 3.4 多尺寸 / 平板适配（核心原则：重排，非拉伸）
- **不要简单拉伸**——大屏下按钮布局、导航形态都要重排。
- 断点判定（Android）：`smallestScreenWidthDp >= 600` 视为平板。
- 平板常见做法：
  - **导航**：底部导航可改为**屏幕左侧悬浮胶囊**（垂直居中、约半屏高、不铺满），主内容区让出导航宽度
  - **内容宽度上限**：设最大内容宽（如 760dp），避免长文本整行拉伸
  - **按钮网格化**：竖排的按钮组在大屏改为**横排一行 / 网格**（等宽 + weight），关键分组保持成行
  - 长内容列表/网格：大屏用多列布局
- 两种实现路径：XML 项目用 `layout-sw600dp` 覆盖默认布局；纯代码 UI 项目在代码里按断点分支设置方向/权重/宽度。

### 3.5 可访问性与易用性
- 触控目标不小于 44×44dp。
- 状态反馈即时（点击有视觉变化），可点击元素有明确焦点态。

### 3.6 选项与开关：一律毛玻璃胶囊化（硬性要求）

> **原则**：任何"带可选状态"的控件——单选选项、独立开关、操作按钮——**都必须用 `GlassButtonDrawable` 毛玻璃胶囊作背景**。禁止裸文字、禁止原生 `Switch`/`ToggleButton`、禁止 `RadioButton` 自带圆圈、禁止系统默认按钮样式。优先用 §8 组件库的 `GlassCapsuleButton` / `GlassRadioButton`；单文件拷贝场景才手写 Drawable。

**胶囊两态的视觉规范（所有选项/开关共用一套）：**

| 状态 | 描边 | 填充 | 文字 |
|---|---|---|---|
| 未选中（关） | 1dp，淡白 `0x40FFFFFF` | 通透玻璃底（约 20%~70% 深灰 `#1C1C1E`，按组件默认） | 白 `0xFFFFFFFF` |
| 选中（开） | 2dp，纯白 `0xFFFFFFFF` | 填充加深（约 35%） | 强调蓝 `0xFF0A84FF` |
| 按压中 | —— | 填充轻微提亮，保留反馈 | —— |

**A. 单选选项（互斥，如"仅时间 / 时间+内存 / 仅内存"）：**
- 容器用 `RadioGroup`；每个选项是 `RadioButton`，必须 `setButtonDrawable(null)` 去掉原生圆圈。
- 背景 `GlassButtonDrawable(圆角dp, 1dp, false)`。
- 点击后**遍历同组所有项**重算选中态：选中项 `setGlassSelected(true)` + 文字蓝，其余 `setGlassSelected(false)` + 文字白。
- 竖排 `MATCH_PARENT`、上下间距约 12dp；平板横排用 `weight=1` 等宽 + 左右 6dp 间距（见 §3.7 平板）。

**B. 开始/关闭开关（独立布尔，如"各 Hook 项独立开/关"）——实现方法：**
- **不用原生 Switch**，用一行可点击项（`LinearLayout` 或按钮），背景就是毛玻璃胶囊。
- 点击整行触发 `toggle(index)`，状态机固定三步：
  1. `cur = readState(index)`（从 SharedPreferences / 配置文件读当前值）
  2. `next = !cur; writeState(index, next)`（立即持久化）
  3. 更新 UI：`glass.setGlassSelected(next)`（开=蓝描边，关=淡描边）+ 该项文字 `setSelected ? 蓝 : 白`
- **进入页面必须 `loadConfig()`**：onCreate 时遍历**全部**开关项，按持久化值逐组恢复 `setGlassSelected(...)` 与文字色，否则旋转/重进会回到默认关态。
- 一个开关 = 一个独立 `GlassButtonDrawable` 实例（用数组/字段持有），切换时只 `setGlassSelected`  invalidate，不重建背景对象。

**C. 操作按钮（选择颜色 / 透明 / 返回 / 确认 / 取消等）：**
- 一律 `createGlassButtonBg(density)` = `GlassButtonDrawable(圆角dp, 1dp, false)`，文字白、`setAllCaps(false)`，内边距约 左右 24dp / 上下 14dp。
- 多按钮并排时等宽 `weight=1`，不各写死宽度。

> 圆角取值全项目统一：DRS 24dp、RSB 28dp（见 §8）；同一张界面里不混用两种圆角。

### 3.7 二级界面（子页面）的美术风格与逻辑

> 二级界面 = 从主界面/标签页点进去的独立设置页（如"背景颜色""时间设置""应用选择器"）。**必须与主界面同一套美术语言，禁止另起风格。**

**形态与跳转：**
- 独立 `Activity`，主界面 `startActivity(new Intent(主界面.this, XxxSettingsActivity.class))`。
- 返回不依赖系统默认返回键样式，页面底部自绘一个毛玻璃胶囊「返回」按钮，点击 `finish()`。
- 不用 `onActivityResult` 回传结果：二级页自己持有并直接写配置，主界面 `onResume()` 重读即可。

**美术风格（逐项照做）：**
- 背景纯黑 `0xFF000000`，与主界面一致。
- 根布局 padding：**顶部 48dp**（给系统状态栏让位）、左右 24dp、底部 32dp。
- 标题 20sp 白色；描述/副标 14sp 灰色 `0xFFCCCCCC`；间距约 16~24dp。
- 页面内**所有**按钮、取色块、操作项都套毛玻璃胶囊（§3.6），与主界面同一圆角值。
- 不出现浅色弹窗、不出现系统默认控件样式（深色主题见 §3.1）。

**逻辑：**
- **进入即初始化**：`onCreate` 从持久层读当前值，还原标题、预览、所有开关/选项态。
- **改动即时生效并持久化**：需 root 的写 `/data/local/tmp/` 配置文件；普通项写 SharedPreferences。写成功/失败都 `Toast` 反馈（失败提示检查 root 权限）。
- 有"效果预览"的，先给实时预览区，再放"确认/应用"按钮。
- **平板（`smallestScreenWidthDp >= 600`）**：竖排的操作按钮组改为横排一行，`weight=1` 等宽 + 左右 4~6dp 间距，不整行拉伸（与 §3.4 一致）。
- 三语文案齐全，全部走统一多语言函数。

---

## 4. 通用功能实现要点（以 Android 为例）

> 本节是**「检测 → 下载 → 安装」完整流程**，新项目直接照此实现；各通道 URL 用 `{占位符}` 换成实际值。

### 4.1 内置更新：触发与整体流程
- **触发时机**：应用启动后**异步检测**（不阻塞首屏）；另可在设置页提供「检查更新」手动入口。
- **整体流程（顺序执行）**：
  1. 多通道检测版本（见 §4.2）→ 得到线上最大 `versionName`
  2. 与本地 `versionCode` 比较，有新版 → 弹「发现新版本」窗（标题 + 描述 + 取消 / 下载）
  3. 点下载 → 弹「下载进度」窗（进度条 + 百分比 + 取消 / 跳转仓库主页）
  4. 下载完成 → 校验产物 → 自动拉起安装界面（§4.4）
  5. 安装被拒 / 失败 → 回退：跳转仓库 Releases 页或浏览器下载（§4.4 失败回退）

### 4.2 检测通道（优先级 + URL 形态）
> **代理优先、直连兜底**；每条通道设**短超时（如 5 秒）**，超时立即切下一条，不白等。多源取**最大版本号**，比较后再提示。

| 优先级 | 通道 | URL 形态（`{占位符}`） | 适用 |
|---|---|---|---|
| 1 | 上游 API | `https://api.github.com/repos/{owner}/{repo}/releases/latest`（读 `tag_name`） | 海外/能直连 |
| 2 | 代理/镜像 API | `https://{proxy}/api.github.com/repos/{owner}/{repo}/releases/latest` | 国内主通道 |
| 3 | 版本文件（CDN/代理） | `https://{proxy}/repos/{owner}/{repo}/latest_version.txt?t={ts}` | **国内主要通道**，读纯文本版本号 |
| 4 | 页面 302 重定向 | `https://github.com/{owner}/{repo}/releases/latest`（跟随重定向取 tag） | 无 API 可用时 |
| 5 | IP 直连兜底 | 版本文件 `raw.githubusercontent.com` 的 IP 直连 | 绕过 DNS 污染 |

- 通道 1/2 属「API 型」，通道 3/5 属「版本文件型」；两者都取到后以**版本文件型为准**并做一次交叉校验，避免 API 返回被污染。

### 4.3 下载与镜像策略
- **下载 URL** = 对应 release 的 asset 链接；**代理优先、直连兜底**。
- 镜像列表**顺序化**（`[proxy1, proxy2, 直连]`），直连放**最后**兜底。
- 连接超时短（5 秒）快速切源；用 HTTP 流式读 `Content-Length` / 已读字节做进度。
- **CDN 缓存**：URL 末尾加 `?t={timestamp}` 防返回旧版本/旧产物。

### 4.4 安装（Android 8+）
- 权限：`REQUEST_INSTALL_PACKAGES`。
- 用 `androidx.core.content.FileProvider` 提供 `content://` URI。
- `res/xml/file_paths.xml` 声明下载目录与缓存目录。
- FileProvider authority = `{applicationId}.fileprovider`，并在 Manifest 声明 `<provider>`。
- 安装 Intent：`ACTION_VIEW` + `application/vnd.android.package-archive` + `FLAG_GRANT_READ_URI_PERMISSION` + `FLAG_ACTIVITY_NEW_TASK`。
- **失败回退**：`try/catch` 捕获 `ActivityNotFoundException` / 安装被拒 → 用 `ACTION_VIEW` 打开 `https://github.com/{owner}/{repo}/releases/latest`（或代理页面），让用户浏览器下载。

### 4.5 版本文件（latest_version.txt）
- **位置**：仓库根目录纯文本，**内容即版本号**（如 `1.4.17`）。
- **每次发版必须更新**（CI 自动更新或发版手动），否则检测不到新版本。
- **多仓库同步**：自有仓库与分发/模块仓库各自的 `latest_version.txt` 都要更新。

---

## 5. 发布后核对清单（通用）

- [ ] 自有仓库 `releases/latest` 已是新 tag
- [ ] 分发/模块仓库 `releases/latest` 已是新 tag（如 `{versionCode}-{version}`）
- [ ] **所有**发布仓库的 `latest_version.txt` 内容 == versionName（§4.5）
- [ ] 各仓库产物大小一致 / 可下载
- [ ] 产物验签指纹匹配
- [ ] 升级检测各通道（§4.2）实测均能取到新版本，且页面「检查更新」可弹出新版
- [ ] 变更说明 / 发布正文已写好（含中文）

---

## 6. 常见坑（通用故障排查表）

| # | 症状 | 原因 | 解决 |
|---|---|---|---|
| 1 | 安装显示「重新安装」 | versionCode 未递增 | 发版前递增 versionCode + versionName |
| 2 | 装完后不弹安装界面 | 缺安装权限 | Manifest 加 `REQUEST_INSTALL_PACKAGES` |
| 3 | 下载慢/白等很久 | 直连优先、超时太长 | 代理优先 + 短超时 + 直连兜底 |
| 4 | 检测不到新版本 | 版本文件没更新 | 发版必须更新 `latest_version.txt` |
| 5 | CDN 返回旧版本 | 缓存 | URL 加 `?t={timestamp}` |
| 6 | 安装时崩溃 | FileProvider authority 不匹配 | authority = `{applicationId}.fileprovider`，与 Manifest 一致 |
| 7 | 分支/PR 构建失败无产物 | 版本文件步骤 push 默认分支失败 | 该步骤加 `if: github.ref == 'refs/heads/{默认分支}'` |
| 8 | CI 更新版本文件冲突 | 并发推送 | 该步骤加 `continue-on-error: true` |
| 9 | 本地文件与线上不同步 | 误以为本地已改 | 用 API 改完必须 GET 复核 |
| 10 | 系统弹窗白底，与深色 UI 不统一 | 无全局深色主题 | 设 `AppTheme` + 深色弹窗主题 |
| 11 | 大屏布局被简单拉伸、不协调 | 无断点适配 | 断点重排 + 按钮网格化 + 导航形态切换 |
| 12 | LSPosed 仓库 Latest 不更新 | tag 格式不对 | tag 必须是 `{versionCode}-{versionName}`，不是 `v{versionName}` |
| 13 | LSPosed 仓库有 release 但下载不到 APK | release 没上传 asset | 创建 release 时必须同时上传 APK 附件 |
| 14 | LSPosed 仓库有新 release 但不显示为 Latest | 没设 make_latest | 用 API PATCH `releases/{id}` 设 `make_latest=true` |
| 15 | LSPosed 更新检测不到新版本 | 缺 latest_version.txt | 仓库根目录放 `latest_version.txt`，内容为纯版本号 |
| 16 | 自己仓库 release 没 APK asset | release 是空 release | 从 CI artifacts 下载 APK，或发布时直接附带 |

---

## 7. 新项目初始化模板清单

> 用 `{占位符}` 替换为实际值，逐项打勾。

- [ ] **工程骨架**：`build.gradle` 配置 `applicationId`、`minSdk`、`targetSdk/compileSdk`、起始版本号；统一签名（debug 用 release keystore）
- [ ] **Manifest**：网络权限、安装权限；全局深色主题挂载；声明需要的 provider / meta-data
- [ ] **主题**：全局 `AppTheme` + 深色弹窗主题
- [ ] **统一控件**：复用的按钮/弹窗控件类 + 统一圆角/间距常量
- [ ] **多语言**：所有 UI 字符串多语齐全 + 语言偏好；对外文档语言顺序统一
- [ ] **CI workflow**：沿用模板；验签断言、版本文件步骤加分支守卫 + continue-on-error
- [ ] **更新机制**：多通道检测 + 下载镜像顺序 + 安装（FileProvider/权限/Intent）
- [ ] **平板/大屏适配**：断点判定 + 导航/按钮重排，不做简单拉伸
- [ ] **发布**：先出包审核 → 合并 → 双通道发布 + 更新版本文件 + 核对清单（§5）

---

## 8. 可复用组件库（引用，单点维护）

> 规范的落地常伴随可复用组件。**独立组件库保持独立仓库维护，不合并进本文档**（本文档只放规范、不放源码）。用到的组件在此登记引用，开新项目时从这里找。

### 玻璃拟态按钮组件库 GlassButtons
- **仓库**：https://github.com/GJR787878/GlassButtons
- **调用入口（直达）**：
  - README / 使用说明：https://github.com/GJR787878/GlassButtons#readme
  - 源码（组件目录）：`glassbutton/src/main/java/com/gjr/glassbutton/`
  - 最新 Release / demo APK：https://github.com/GJR787878/GlassButtons/releases/latest
  - 历史版本：https://github.com/GJR787878/GlassButtons/releases
- **定位**：苹果风格毛玻璃半透明按钮组件库（抽自 RamStatusBar），纯 Java + Android framework，无第三方依赖。
- **组件**：`GlassCapsuleButton` / `GlassRadioButton` / `GlassNavBar` / `GlassButtonDrawable` / `GlassButtonStyle`。
- **圆角规范对应**：组件默认 28dp；接入时按项目设定——**DRS 用 24dp、RSB 用 28dp**（构造参数或 `app:glassCornerRadius`）。
- **两种接入方式**：
  1. **Library 模块依赖**：拷贝 `glassbutton/` → `settings.gradle` 加 `include ':glassbutton'` → app 依赖 `implementation project(':glassbutton')`
  2. **单文件拷贝（零依赖）**：直接拷贝 `GlassButtonDrawable/GlassButtonStyle/GlassCapsuleButton/GlassRadioButton/GlassNavBar` 五个 `.java` 到项目（XML 调用还需 `attrs.xml`）
- **平板导航复用**：`GlassNavBar` 既可做底部导航，也可在平板作为**左侧悬浮胶囊导航**（垂直居中、约半屏高），与 §3.4 平板规范配套。
- 开新项目：把该组件库作为玻璃风格 UI 的**唯一来源**，新按钮/导航一律用它，不另起样式。

---

> 维护提示：本文件是**通用参照模板**，随实战经验持续沉淀。新增经验时保持「结论先行 + 表格 + 占位符 + 文件路径」的结构，便于后续项目与 AI 快速读取执行。
