---
name: my-dev-specs
description: 我的通用 Android/LSPosed 模块开发规范，包含 UI 毛玻璃胶囊标准、平板适配、CI 签名流程、LSPosed 仓库发布踩坑、GlassButtons 组件库引用等全套开发要求。开发新模块或修改 UI 时必须先读。
---

# 我的开发手册

这是个人 Android/LSPosed 模块开发规范，开发任何项目时严格执行。

## 使用方式（强制）

### 首次开发 app 时（硬规则）
**必须完整读完手册全文（约 500 行）再动手。** 执行：
```bash
curl -s "https://raw.githubusercontent.com/GJR787878/app-dev-specs/main/README.md" | tee /tmp/dev-specs.md
wc -l /tmp/dev-specs.md  # 确认总行数
# 然后完整读取，输出：已通读 N 行，覆盖 8/8 章节
```

**禁止只读开头就动手。** 写完代码后必须逐条核对 §7 新项目初始化清单。

### 修改已有项目 / 修 bug
可以只拉取 §3.6 + §6 常见坑表。

回复用户时先提示："正在读取开发手册..."

## 核心规范速查（缓存）

### 工具链版本（钉死）
| 项 | 值 |
|---|---|
| AGP | **8.5.2** |
| Gradle | **8.9**（wrapper） |
| JDK | 17 |
| minSdk / compileSdk | 26 / 34 |
| 圆角 | 24dp（DRS）或 28dp（RSB），项目内统一 |

### UI（§3.6 硬性要求）
- 所有选项/开关必须用毛玻璃胶囊，**禁止原生 Switch**
- 未选中：1dp 淡白 `0x40FFFFFF` 边框 + 通透玻璃底 + 白文字
- 选中：2dp 纯白 `0xFFFFFFFF` 边框 + 填充加深 35% + 蓝文字 `#0A84FF`
- 根布局 padding：顶 48dp、左右 24dp、底 32dp
- **禁止手写简化版玻璃 Drawable**，必须用 GlassButtons 完整 5 层实现

### 平板适配（§3.4 硬性要求）
- 断点：`smallestScreenWidthDp >= 600` 视为平板
- 平板：导航改**左侧竖排悬浮胶囊**（垂直居中、半屏高），内容区 `paddingLeft` 让出约 120dp
- 手机：底部横排导航
- **不能简单拉伸**，必须重排

### 布局踩坑（§6）
- 导航栏浮底时，**padding 必须设置在内容 LinearLayout 上，不是 ScrollView**
- 选中项高亮圆角必须和外层导航栏一致，不能用全圆角
- 复制 Java 文件后必须批量改包名，grep 校验无残留

### GlassButtons 组件库（§8）
- **JitPack 依赖**（推荐）：`implementation 'com.github.GJR787878:GlassButtons:v1.0.4'`
- 根 build.gradle 加 `maven { url 'https://jitpack.io' }`
- 组件：`GlassCapsuleButton` / `GlassRadioButton` / `GlassNavBar` / `GlassButtonDrawable` / `GlassButtonStyle`

### LSPosed 发布（§6 常见坑）
- tag 格式：`{versionCode}-{versionName}`
- 仓库必须有：SUMMARY、SCOPE、SOURCE_URL、ADDITIONAL_AUTHORS、latest_version.txt
- description 不能为空
- 普通 push 不触发索引，必须重新打 tag 或重建 release

## 手册位置

- GitHub 仓库：https://github.com/GJR787878/app-dev-specs
- 完整规范见 README.md，触发时自动拉取最新版
