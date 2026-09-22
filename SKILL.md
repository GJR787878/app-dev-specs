---
name: my-dev-specs
description: 我的通用 Android/LSPosed 模块开发规范，包含 UI 毛玻璃胶囊标准、平板适配、CI 签名流程、LSPosed 仓库发布踩坑、GlassButtons 组件库引用等全套开发要求。开发新模块或修改 UI 时必须先读。
---

# 我的开发手册

这是个人 Android/LSPosed 模块开发规范，开发任何项目时严格执行。

## 使用方式（强制，三层保障）

### 第一层：开发前强制通读（硬规则，不可跳过）

**首次开发新 app 时，必须完整读完手册全文再动手。** 执行步骤：

```bash
# 1. 拉最新手册
curl -s "https://raw.githubusercontent.com/GJR787878/app-dev-specs/main/README.md" | tee /tmp/dev-specs.md

# 2. 统计总行数
wc -l /tmp/dev-specs.md

# 3. 完整读取全部内容（不能只读前 100 行就动手）
# 4. 输出确认：已通读 N 行，覆盖 10/10 章节
```

**违反后果：** 如果跳过通读直接动手，写完后必须重写，因为大概率会漏掉平板适配、Gradle 版本、弹窗胶囊等硬性要求。

### 第二层：写完后逐条核对（硬规则，不可跳过）

写完代码、推送构建前，必须对照 §0.5 速查表逐条检查，输出检查清单：

```
✅ AGP 版本：8.5.2（符合）
✅ Gradle 版本：8.9（符合）
✅ minSdk：31（符合）
✅ 所有开关用 GlassCapsuleButton（符合）
✅ 所有弹窗单选用 GlassRadioButton（符合）
✅ 平板适配代码已写（符合）
✅ 导航栏磨砂渐变参数正确（符合）
✅ gradle.properties 已创建（符合）
✅ GlassButtons import 已加（符合）
...（逐条对照）
```

**任何一条不符合，必须修正后再推送。**

### 第三层：每个关键决策引用手册章节号

写代码时，每个关键 UI 组件、每个参数，都要在注释里写对应的手册章节号：

```java
// §3.6 毛玻璃胶囊：所有开关必须用 GlassCapsuleButton
GlassCapsuleButton btn = new GlassCapsuleButton(this);

// §3.6.1 导航栏磨砂渐变：上白下透
GradientDrawable navBg = new GradientDrawable(
    GradientDrawable.Orientation.TOP_BOTTOM,
    new int[]{0xF06A6A72, 0x882C2C2E});
```

**禁止凭印象写代码。** 如果某个参数不确定，先去手册查，不要猜。

### 修改已有项目 / 修 bug
可以只拉取 §3.6 + §6 常见坑表，但必须输出"已读取 §3.6 + §6 共 N 行"。

回复用户时先提示："正在读取开发手册..."

## 核心规范速查（缓存）

### 工具链版本（钉死）
| 项 | 值 |
|---|---|
| AGP | **8.5.2** |
| Gradle | **8.9**（wrapper） |
| JDK | 17 |
| minSdk / compileSdk | **31**（Android 12+，以后只开发这个及以上） / 34 |
| 圆角 | 24dp（DRS）或 28dp（RSB），项目内统一 |

### UI（§3.6 硬性要求）
- 所有选项/开关必须用毛玻璃胶囊，**禁止原生 Switch**
- 未选中：1dp 淡白 `0x40FFFFFF` 边框 + 通透玻璃底 + 白文字
- 选中：2dp 纯白 `0xFFFFFFFF` 边框 + 填充加深 35% + 蓝文字 `#0A84FF`
- 根布局 padding：顶 48dp、左右 24dp、底 32dp
- **禁止手写简化版玻璃 Drawable**，必须用 GlassButtons 完整 5 层实现
- 弹窗内单选选项必须用 `GlassRadioButton`，不能用原生 RadioButton

### 平板适配（§3.4 硬性要求）
- 断点：`smallestScreenWidthDp >= 600` 视为平板
- 平板：导航改**左侧竖排悬浮胶囊**（垂直居中、半屏高），内容区 `paddingLeft` 让出约 120dp
- 手机：底部横排导航
- **不能简单拉伸**，必须重排

### 导航栏磨砂（§3.6.1 硬性要求）
- 渐变方向：上白下透（`TOP_BOTTOM`）
- 顶部色：`0xF06A6A72`（94% 不透明，泛白磨砂）
- 底部色：`0x882C2C2E`（53% 不透明，更透明）
- 边框：1dp 淡白 `0x55FFFFFF`
- **禁止**：纯透明白、纯黑色、`setBackgroundBlurRadius()`（会闪退）

### 布局踩坑（§6）
- 导航栏浮底时，**padding 必须设置在内容 LinearLayout 上，不是 ScrollView**
- 选中项高亮圆角必须和外层导航栏一致，不能用全圆角
- 复制 Java 文件后必须批量改包名，grep 校验无残留
- 新项目必须创建 `gradle.properties`，写 `android.useAndroidX=true`
- 用了 GlassButtons 依赖后，所有 Java 文件必须 `import com.gjr.glassbutton.*;`

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
