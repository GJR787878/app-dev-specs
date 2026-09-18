# 开发规范与细节要求

> DeviceResetSpoofer & RamStatusBar 两个 LSPosed 模块的开发、发版、UI 规范汇总。

---

## 一、开发流程

### 代码提交
- **直接用 GitHub API 在线编辑提交**，不本地 clone 构建
- 依赖 CI（GitHub Actions）出包
- 改完代码后**必须重新 GET 验证文件确实改了**（本地文件可能不同步）
- 多个文件改动合并成一次 commit（用 Git Data API：blob → tree → commit → ref）

### 版本号
- 每次发版 **versionCode 和 versionName 都要递增**
- 发版前在 GitHub 上**确认 build.gradle 里的版本号已正确更新**
- versionCode 不递增会导致安装时显示"重新安装"而非"升级安装"
- 当前版本：
  - DRS: v2.19 (versionCode 39)
  - RSB: v1.4.17 (versionCode 30)

---

## 二、UI 规范

### 弹窗风格
- **所有弹窗圆角与按钮圆角一致**：
  - DRS: 圆角 24dp
  - RSB: 圆角 28dp
- **按钮和弹窗底部间距 ≥ 20dp**（不能贴底）
- 弹窗背景色：
  - DRS: `#2B2B2B`
  - RSB: `#1C1C1E`
- 用 `GradientDrawable` 设置弹窗窗口背景（`dialog.getWindow().setBackgroundDrawable()`）
- 自定义 View 根布局设为透明背景（`ColorDrawable.TRANSPARENT`）

### 玻璃按钮
- 使用 `GlassButtonDrawable(radius, border, false)`
- DRS: radius 24dp, border 1dp
- RSB: radius 28dp, border 1dp
- 按钮文字大小 14sp

> **GlassButtonDrawable 来源**：
> - DRS: `GJR787878/DeviceResetSpoofer` → `app/src/main/java/io/github/gjr787878/devicereset/GlassButtonDrawable.java`
> - RSB: `GJR787878/RamStatusBar` → `app/src/main/java/com/example/ramstatusbar/GlassButtonDrawable.java`
> - 两个仓库各自维护一份，新弹窗/按钮样式统一复用此类

### 多语言
- 三语：中文（zh）、英文（en）、俄文（ru）
- **README 语言顺序：英文 → 中文 → 俄文**
- App 内所有文案都要有三语版本

---

## 三、内置更新下载机制

### 弹窗结构
**"发现新版本"弹窗**（检测到更新时）：
- 标题 + 描述
- 下方左右两个玻璃按钮：取消 / 下载
- 圆角背景 + 玻璃按钮，与 App 其他弹窗风格一致

**"正在下载"进度弹窗**（点击下载后）：
- 标题：⬇️ 正在下载 v{version}
- 横向 ProgressBar + 百分比文字
- 下方左右两个玻璃按钮：取消 / 仓库主页
  - 取消：停止下载、删除临时文件、关闭弹窗
  - 仓库主页：取消下载、跳转 GitHub 仓库页面
- 下载完成自动关闭弹窗并拉起系统安装界面

### 下载 URL 顺序（重要！）
- **代理优先，直连兜底**（国内直连 GitHub 会超时）
- 连接超时 **5 秒**（失败快速切下一个源）
- 镜像列表：
  1. ghfast.top
  2. gh-proxy.com
  3. ghproxy.net
  4. gh.llkk.cc
  5. mirror.ghproxy.com
  6. github.moeyy.xyz
  7. ghproxy.cc
  8. 直连 github.com（兜底）

### 安装
- 需要 `REQUEST_INSTALL_PACKAGES` 权限（Android 8+ 必需）
- 用 FileProvider 提供 content:// URI
- file_paths.xml 配置：
  - `external-files-path name="downloads" path="Download/"`
  - `cache-path name="cache" path="."`
- FileProvider authority = `{applicationId}.fileprovider`
- 安装 Intent：`ACTION_VIEW` + `application/vnd.android.package-archive` + `FLAG_GRANT_READ_URI_PERMISSION` + `FLAG_ACTIVITY_NEW_TASK`

---

## 四、更新检测机制

### 多通道检测（提高成功率）
1. **GitHub API**：`api.github.com/repos/{repo}/releases/latest`
2. **页面 302**：`github.com/{repo}/releases/latest` 重定向提取 tag
3. **CDN/代理读 latest_version.txt**（国内主要通道）：
   - jsDelivr（cdn/fastly/gcore 三个节点）
   - ghfast.top、gh-proxy.com、ghproxy.net 等代理读 raw.githubusercontent.com
   - **URL 末尾加 `?t={timestamp}` 防 CDN 缓存**
   - 多源取最大版本号（compareVersions）
4. **IP 直连兜底**：绕过 DNS 污染，用 GitHub Anycast IP + Host 头 + TLS SNI

### latest_version.txt
- 仓库根目录的纯文本文件，内容就是版本号（如 `2.19`）
- **每次发版必须更新**（CI 自动更新或手动通过 API 更新）
- CI 工作流中此步骤要设 `continue-on-error: true`（避免推送冲突导致整个构建失败）

---

## 五、发版流程

### DRS (DeviceResetSpoofer)
1. 改代码 → push main → CI 构建成功
2. 手动创建 Release（tag: `v{version}`）
3. 上传 APK 资产：`DeviceResetSpoofer-v{version}.apk`
4. 发布到 LSPosed 模块仓库（tag: `{versionCode}-{version}`），上传同名 APK
5. 更新 `latest_version.txt` 为新版本号

### RSB (RamStatusBar)
1. 改代码 → push main → CI 构建成功
2. 打 tag `v{version}` → CI 自动构建并发 Release
3. 发布到 LSPosed 模块仓库（tag: `{versionCode}-{version}`），上传 `RamStatusBar-v{version}-release.apk`
4. 更新 `latest_version.txt` 为新版本号

### LSPosed 模块仓库
- 组织：`Xposed-Modules-Repo`
- DRS 仓库：`io.github.gjr787878.devicereset`
- RSB 仓库：`io.github.gjr787878.ramstatusbar`
- tag 格式：`{versionCode}-{version}`（如 `39-2.19`、`30-1.4.17`）

---

## 六、仓库信息

| 项目 | 主仓库 | LSPosed 模块仓库 |
|---|---|---|
| DeviceResetSpoofer | GJR787878/DeviceResetSpoofer | Xposed-Modules-Repo/io.github.gjr787878.devicereset |
| RamStatusBar | GJR787878/RamStatusBar | Xposed-Modules-Repo/io.github.gjr787878.ramstatusbar |

### 包名
- DRS: `io.github.gjr787878.devicereset`
- RSB: `io.github.gjr787878.ramstatusbar`（applicationId），源码包名 `com.example.ramstatusbar`

---

## 七、常见坑

1. **版本号没递增** → 安装显示"重新安装"
2. **缺 REQUEST_INSTALL_PACKAGES** → 下载完成后不弹安装界面
3. **直连 GitHub 优先** → 国内白等 10 秒超时才切代理
4. **latest_version.txt 没更新** → 检测不到新版本（CDN 返回旧版本号）
5. **CDN 缓存** → jsDelivr 缓存旧版本号，URL 加 `?t=timestamp` 防缓存
6. **FileProvider authority 不匹配** → 安装时崩溃
7. **CI latest_version.txt 推送冲突** → 步骤设 `continue-on-error: true`
8. **本地文件不同步** → 用 API 改完要 GET 验证
