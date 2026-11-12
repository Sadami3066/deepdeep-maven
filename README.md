# DeepDeep Maven

DeepDeep 插件开发的公开 Maven 仓库：托管插件契约 `plugin-api` 与设计系统 `design`，供外部 `.ddp` 插件工程在编译期引用。

## 概览

- 这是一个**只读的 Maven 分发仓库**，通过 GitHub Pages 提供：`https://sadami3066.github.io/deepdeep-maven/`
- 仓库只存放已发布的构建产物（AAR、POM、Gradle module metadata、sources 与校验文件），不包含宿主实现，也不接受上传。
- 面向插件开发者：插件是 Kotlin 代码，实现 `DeepDeepPlugin`，用 `plugin-api` 编译，由宿主 App 在进程内加载。
- 插件模板与完整开发文档：[DeepDeep-Plugins](https://github.com/Sadami3066/DeepDeep-Plugins)
- 官方插件参考实现：[DeepDeep-Official-Plugins](https://github.com/Sadami3066/DeepDeep-Official-Plugins)

## 仓库内容

### 坐标与版本

| 坐标 | 版本 | 说明 |
| --- | --- | --- |
| `io.github.sadami3066.deepdeep:plugin-api` | `0.8.1` | 插件契约 SDK（AAR），当前 `PLUGIN_API_LEVEL = 5` |
| `io.github.sadami3066.deepdeep:design` | `0.6.1` | 与 App 同源的 Compose 设计系统，版本独立维护 |

`design` 通过 `plugin-api` 的 `api()` 传递，插件工程通常只需声明 `plugin-api`。

### 目录结构

```
io/github/sadami3066/deepdeep/
├── plugin-api/
│   ├── maven-metadata.xml
│   └── 0.8.1/
│       ├── plugin-api-0.8.1.aar
│       ├── plugin-api-0.8.1.pom
│       ├── plugin-api-0.8.1.module
│       ├── plugin-api-0.8.1-sources.jar
│       └── *.md5 / *.sha1 / *.sha256 / *.sha512
└── design/
    ├── maven-metadata.xml
    └── 0.6.1/
        ├── design-0.6.1.aar
        ├── design-0.6.1.pom
        ├── design-0.6.1.module
        ├── design-0.6.1-sources.jar
        └── *.md5 / *.sha1 / *.sha256 / *.sha512
```

- Maven group 是 `io.github.sadami3066.deepdeep`；代码包名沿用 `io.github.whywood.deepdeep.plugin.api` 与 `io.github.whywood.deepdeep.design`，两者属于同一项目。
- 制品来自 DeepDeep 宿主工程的 `:plugin-api` 与 `:design` 模块；每个版本附 POM、Gradle module metadata、sources jar 与 MD5/SHA-1/SHA-256/SHA-512 校验和。
- `plugin-api` 的 AAR 自带 consumer ProGuard 规则；`design` 的 AAR 包含字体与图标资源，其许可文本随 `assets/licenses/` 分发。

## 快速开始

### 1. 配置仓库

Kotlin DSL，推荐在 `settings.gradle.kts` 中集中声明：

```kotlin
dependencyResolutionManagement {
    repositories {
        maven {
            url = uri("https://sadami3066.github.io/deepdeep-maven")
            content { includeGroup("io.github.sadami3066.deepdeep") }
        }
        google()
        mavenCentral()
    }
}
```

也可以只在模块的 `repositories { maven { url = uri("https://sadami3066.github.io/deepdeep-maven") } }` 中声明。本仓库只提供上面两个 group 的 AAR；Compose、协程等依赖仍从 `google()` / `mavenCentral()` 解析。

### 2. 声明依赖

```kotlin
dependencies {
    compileOnly("io.github.sadami3066.deepdeep:plugin-api:0.8.1")
}
```

`design` 已由 `plugin-api` 传递；需要显式写坐标时同样使用 `compileOnly`：

```kotlin
compileOnly("io.github.sadami3066.deepdeep:design:0.6.1")
```

插件与宿主共享同一份类（parent-first）。**宿主已提供的库一律用 `compileOnly`，不要用 `implementation`**：把 `plugin-api`、Compose 或协程打包进 `.ddp` 会与宿主重复并触发类冲突。

### 3. 版本对齐

以 POM 与 Gradle module metadata 为准：

| 项 | 版本 |
| --- | --- |
| Kotlin（stdlib） | `2.4.21` |
| Compose BOM | `2026.03.00`（POM 中 import，具体 Compose 版本由 BOM 决定） |
| kotlinx-coroutines-core | `1.11.0` |
| kotlinx-serialization-json | `1.12.0` |
| compileSdk | `36`（AAR 元数据 `minCompileSdk=36`） |
| minSdk | `29` |
| JVM target | `17` |

插件模板当前使用 AGP `9.4.1`、Kotlin `2.4.21`、Gradle `9.8.2` 与 JDK 17；宿主升级 SDK 后需同步调整。

### 4. 最小插件示例

```kotlin
package com.example.myplugin

import io.github.whywood.deepdeep.plugin.api.Contributions
import io.github.whywood.deepdeep.plugin.api.DeepDeepPlugin
import io.github.whywood.deepdeep.plugin.api.PluginAvailability
import io.github.whywood.deepdeep.plugin.api.PluginHost
import io.github.whywood.deepdeep.plugin.api.PluginTool
import io.github.whywood.deepdeep.plugin.api.PluginToolResult
import kotlinx.serialization.json.buildJsonObject

class MyPlugin : DeepDeepPlugin {
    override suspend fun availability(host: PluginHost): PluginAvailability =
        PluginAvailability.Available

    override fun contributions(host: PluginHost): Contributions = Contributions(
        tools = listOf(
            PluginTool(
                name = "hello",
                label = "Hello",
                description = "Return a greeting.",
                parameters = buildJsonObject {},
                execute = { PluginToolResult.text("hello from MyPlugin") },
            ),
        ),
    )
}
```

对应 `assets/plugin.json`：

```json
{
  "id": "com.example.myplugin",
  "version": "1.0.0",
  "entryClass": "com.example.myplugin.MyPlugin",
  "name": { "default": "My Plugin", "zh": "我的插件" },
  "summary": { "default": "Minimal DeepDeep plugin" },
  "capabilities": []
}
```

示例不访问宿主服务，因此 `capabilities` 为空数组；实际插件按需声明所需能力。清单字段、能力声明与 `.ddp` 打包签名见 [插件开发文档](https://github.com/Sadami3066/DeepDeep-Plugins/blob/master/docs/PLUGIN_DEV.md)。

### 5. 从官方模板开始

```bash
git clone https://github.com/Sadami3066/DeepDeep-Plugins
cp -R DeepDeep-Plugins/template my-plugin
```

模板已配置好本仓库地址、`pluginApiVersion = "0.8.1"` 与全套 `compileOnly` 依赖。改成自己的 `namespace` / `applicationId`、清单 `id` / `version` / `entryClass` 与包名后：

```bash
./gradlew generateSigningKey   # 首次生成 Ed25519 签名密钥，私钥不入库
./gradlew assemblePlugin       # build/plugin/<module>.unsigned.ddp
./gradlew signPlugin           # build/plugin/<module>.ddp
./gradlew verifyPlugin         # 校验签名与清单
```

把 `.ddp` 传到手机，在 App 的「插件 → 导入插件」中安装。完整流程见[模板仓库](https://github.com/Sadami3066/DeepDeep-Plugins#readme)。

## 使用方式

### 版本选择与升级

- 插件工程通过 `pluginApiVersion`（或直接写版本号）锁定 `plugin-api`；宿主 SDK 升级时同步改版本并重新编译。
- `plugin-api` 的发布版本（如 `0.8.1`）与契约等级 `PLUGIN_API_LEVEL`（当前 `5`）独立维护，版本号变化不必然改变等级。
- `design` 版本独立（当前 `0.6.1`），由 `plugin-api` 传递，不要单独用 `implementation` 引入。
- 升级前查看 [`docs/PLUGIN_DEV.md`](https://github.com/Sadami3066/DeepDeep-Plugins/blob/master/docs/PLUGIN_DEV.md) 与 [CHANGELOG](https://github.com/Sadami3066/DeepDeep-Plugins/blob/master/CHANGELOG.md)。

### 只读使用

本仓库只做分发：`io/` 下的文件是已发布产物，请勿手改或在本仓库上发布。可以 clone 到本地作为离线 Maven 仓库使用，但请保留原 group / artifact / version 路径；使用任何镜像时核对随版本发布的 SHA-256。

### 与 App 插件 ABI 的关系

- `plugin-api` 是宿主与插件之间的唯一 ABI 契约。插件实现 `DeepDeepPlugin`，由宿主在 App 进程内用内存 ClassLoader 装载，类解析 parent-first。
- 宿主按清单 `apiLevel` 与支持范围（当前 `5..5`）决定是否装载插件；UI 另有 `HOST_UI_API_LEVEL = 1` 的兼容判定。
- **发布前没有兼容承诺**：`plugin-api` 的公开类型与签名可能随版本删改而不提升 `PLUGIN_API_LEVEL`，升级后需用最新 SDK 重新编译；发布后才启用「同一等级只做加法，删改必须提升等级」的规则。
- Compose 没有跨版本二进制 ABI 承诺：宿主与插件的 Kotlin / Compose 版本不一致时，只会禁用该插件的 UI，不影响插件其余部分。
- 公开 ABI 由宿主工程中的 ABI 基线与 golden 清单固定；插件不要反射宿主内部实现。

## 风险

- **公开制品不可删除**：本仓库是追加式分发，升级请用新版本号，不要依赖旧版本被删除或下线。
- **同版本覆盖与撤回**：以相同版本号重复发布时，已发布文件与校验和会被覆盖；仓库历史被改写或 Pages 关闭时版本也可能消失。依赖方应锁定精确版本并保存校验和。
- **API 兼容性与 ABI 演进**：发布前 SDK 可随时做不兼容变更；发布后按契约等级做加法。跨版本升级请重新编译并跑宿主回归。
- **GitHub Pages 可用性**：本仓库依赖 GitHub Pages 提供只读 HTTP 服务，不承诺 SLA；网络受限、GitHub 故障或缓存异常都会表现为依赖解析失败。可先用本地 Gradle 缓存或离线模式重试。
- **非官方镜像**：官方地址只有 `https://sadami3066.github.io/deepdeep-maven/` 与其对应的 GitHub 仓库；第三方镜像可能过期或被篡改，使用前务必核对 SHA-256，不建议作为唯一来源。
- **安全提示**：插件与宿主同进程、同 UID，没有沙箱；只导入你信任的插件。

## 权益与许可

- `plugin-api` 与 `design` 的代码来自 DeepDeep，采用 [Apache-2.0](LICENSE)。
- `design` AAR 内含的字体（Lora、Manrope、Open Runde、Fredoka）为 SIL Open Font License 1.1，图标（Tabler Icons）为 MIT；完整文本随 AAR 的 `assets/licenses/` 分发。
- 传递依赖（AndroidX Compose、Kotlin、kotlinx 等）遵循各自的许可证。
- 本仓库不授予 DeepDeep 名称、图标等商标权利。

## 声明与反馈

- 本仓库是 DeepDeep 插件制品的官方公开分发渠道，只读；不提供宿主 App 安装包，也不代表插件市场对第三方的审核背书。
- 依赖解析、坐标或版本问题请在 [DeepDeep-Plugins Issues](https://github.com/Sadami3066/DeepDeep-Plugins/issues) 或[本仓库 Issues](https://github.com/Sadami3066/deepdeep-maven/issues) 反馈。
- 安全漏洞请勿公开提交 issue，请通过私下渠道联系维护者。
