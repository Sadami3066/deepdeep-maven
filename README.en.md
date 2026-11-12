# DeepDeep Maven

Public Maven repository for DeepDeep plugin development: it hosts the plugin contract SDK `plugin-api` and the design system `design`, for external `.ddp` plugin projects to reference at compile time.

## Overview

- A **read-only Maven distribution repository** served by GitHub Pages at `https://sadami3066.github.io/deepdeep-maven/`.
- It contains published build artifacts only (AAR, POM, Gradle module metadata, sources and checksums), with no host implementation and no upload endpoint.
- Built for plugin developers: plugins are Kotlin code that implement `DeepDeepPlugin`, compile against `plugin-api`, and are loaded in-process by the host app.
- Plugin template and full developer docs: [DeepDeep-Plugins](https://github.com/Sadami3066/DeepDeep-Plugins)
- Reference implementations: [DeepDeep-Official-Plugins](https://github.com/Sadami3066/DeepDeep-Official-Plugins)

## Repository contents

### Coordinates and versions

| Coordinate | Version | Notes |
| --- | --- | --- |
| `io.github.sadami3066.deepdeep:plugin-api` | `0.8.1` | Plugin contract SDK (AAR), current `PLUGIN_API_LEVEL = 5` |
| `io.github.sadami3066.deepdeep:design` | `0.6.1` | Compose design system shared with the app, versioned independently |

`design` is exposed transitively through the `api()` configuration of `plugin-api`, so plugin projects normally only declare `plugin-api`.

### Directory layout

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

- The Maven group is `io.github.sadami3066.deepdeep`; the code packages remain `io.github.whywood.deepdeep.plugin.api` and `io.github.whywood.deepdeep.design`. Both belong to the same project.
- Artifacts are published from the `:plugin-api` and `:design` modules of the DeepDeep host project. Every version ships a POM, Gradle module metadata, a sources jar, and MD5/SHA-1/SHA-256/SHA-512 checksums.
- The `plugin-api` AAR includes consumer ProGuard rules; the `design` AAR bundles font and icon resources, with their license texts under `assets/licenses/`.

## Quick start

### 1. Configure the repository

Kotlin DSL, preferably in `settings.gradle.kts`:

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

You can also declare it only in a module's `repositories { maven { url = uri("https://sadami3066.github.io/deepdeep-maven") } }`. This repository only serves the two groups above; Compose, coroutines and other dependencies still resolve from `google()` and `mavenCentral()`.

### 2. Declare dependencies

```kotlin
dependencies {
    compileOnly("io.github.sadami3066.deepdeep:plugin-api:0.8.1")
}
```

`design` is already provided transitively by `plugin-api`; if you declare it explicitly, use `compileOnly` as well:

```kotlin
compileOnly("io.github.sadami3066.deepdeep:design:0.6.1")
```

Plugins and the host share the same classes (parent-first). **Any library the host already provides must be declared with `compileOnly`, not `implementation`**: bundling `plugin-api`, Compose or coroutines into the `.ddp` duplicates host classes and causes class conflicts.

### 3. Version alignment

The POM and Gradle module metadata are authoritative:

| Item | Version |
| --- | --- |
| Kotlin (stdlib) | `2.4.21` |
| Compose BOM | `2026.03.00` (imported in the POM; individual Compose versions come from the BOM) |
| kotlinx-coroutines-core | `1.11.0` |
| kotlinx-serialization-json | `1.12.0` |
| compileSdk | `36` (AAR metadata `minCompileSdk=36`) |
| minSdk | `29` |
| JVM target | `17` |

The plugin template currently uses AGP `9.4.1`, Kotlin `2.4.21`, Gradle `9.8.2` and JDK 17; adjust these when the host SDK is upgraded.

### 4. Minimal plugin example

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

The matching `assets/plugin.json`:

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

This example does not use any host service, so `capabilities` is empty; real plugins declare the capabilities they need. See the [plugin development docs](https://github.com/Sadami3066/DeepDeep-Plugins/blob/master/docs/PLUGIN_DEV.md) for manifest fields, capabilities and `.ddp` packaging/signing.

### 5. Start from the official template

```bash
git clone https://github.com/Sadami3066/DeepDeep-Plugins
cp -R DeepDeep-Plugins/template my-plugin
```

The template already points at this repository and pins `pluginApiVersion = "0.8.1"` with the full set of `compileOnly` dependencies. After changing its `namespace` / `applicationId`, the manifest `id` / `version` / `entryClass`, and the package name:

```bash
./gradlew generateSigningKey   # create an Ed25519 signing key on first use; never commit the private key
./gradlew assemblePlugin       # build/plugin/<module>.unsigned.ddp
./gradlew signPlugin           # build/plugin/<module>.ddp
./gradlew verifyPlugin         # verify signature and manifest
```

Copy the `.ddp` to your phone and install it from "Plugins → Import plugin" in the app. See the [template repository](https://github.com/Sadami3066/DeepDeep-Plugins#readme) for the complete flow.

## Usage

### Version selection and upgrades

- Pin `plugin-api` through `pluginApiVersion` (or a literal version) in your plugin project; when the host SDK is upgraded, update it and rebuild.
- The published `plugin-api` version (such as `0.8.1`) and the contract level `PLUGIN_API_LEVEL` (currently `5`) are maintained independently; a version bump does not necessarily change the level.
- `design` is versioned independently (currently `0.6.1`) and is provided transitively by `plugin-api`; do not add it with `implementation`.
- Before upgrading, read [`docs/PLUGIN_DEV.md`](https://github.com/Sadami3066/DeepDeep-Plugins/blob/master/docs/PLUGIN_DEV.md) and the [CHANGELOG](https://github.com/Sadami3066/DeepDeep-Plugins/blob/master/CHANGELOG.md).

### Read-only usage

This repository is distribution only: files under `io/` are published artifacts, so do not edit them or publish into this repository. You may clone it as an offline Maven repository, but keep the original group / artifact / version layout; when using any mirror, verify the SHA-256 checksums shipped with each version.

### Relationship to the app plugin ABI

- `plugin-api` is the single ABI contract between host and plugins. Plugins implement `DeepDeepPlugin` and are loaded by the host in the app process with an in-memory ClassLoader; class resolution is parent-first.
- The host uses the manifest `apiLevel` and the supported range (currently `5..5`) to decide whether to load a plugin; UI has its own `HOST_UI_API_LEVEL = 1` compatibility check.
- **There is no compatibility promise before release**: public types and signatures in `plugin-api` may change or be removed without raising `PLUGIN_API_LEVEL`, so rebuild against the latest SDK when upgrading. After release, the rule is "additive only within the same level; removals or changes require a level bump".
- Compose has no binary ABI promise across versions: if host and plugin Kotlin / Compose versions differ, only that plugin's UI is disabled, not the rest of the plugin.
- The public ABI is pinned by the ABI baseline and golden manifest in the host project; plugins must not reflect over host internals.

## Risks

- **Published artifacts are never deleted**: this repository is append-only, so ship upgrades under a new version and do not rely on old versions being removed or retired.
- **Same-version overwrite and retraction**: republishing under an existing version overwrites the published files and checksums; if repository history is rewritten or GitHub Pages is disabled, versions can disappear. Consumers should pin exact versions and keep checksums.
- **API compatibility and ABI evolution**: before release the SDK changes freely and incompatibly; after release it evolves additively by contract level. Rebuild and regression-test on every version upgrade.
- **GitHub Pages availability**: this repository relies on GitHub Pages for read-only HTTP service and has no SLA. Network restrictions, GitHub outages or cache issues surface as dependency resolution failures; retry with a local Gradle cache or offline mode first.
- **Unofficial mirrors**: the official location is only `https://sadami3066.github.io/deepdeep-maven/` and its GitHub repository. Third-party mirrors may be stale or tampered with; always verify SHA-256 checksums and do not rely on a mirror as the only source.
- **Security note**: plugins run in the same process and under the same UID as the host, without a sandbox; only install plugins you trust.

## Rights and licenses

- The code in `plugin-api` and `design` comes from DeepDeep and is licensed under [Apache-2.0](LICENSE).
- The `design` AAR bundles fonts (Lora, Manrope, Open Runde, Fredoka) under the SIL Open Font License 1.1 and icons (Tabler Icons) under MIT; the full texts are distributed in the AAR's `assets/licenses/`.
- Transitive dependencies (AndroidX Compose, Kotlin, kotlinx, and others) are governed by their own licenses.
- This repository grants no trademark rights to the DeepDeep name or logo.

## Statements and feedback

- This repository is the official, read-only public distribution channel for DeepDeep plugin artifacts. It does not distribute the host app package and does not imply that any third-party plugin has been reviewed or endorsed by the plugin marketplace.
- For dependency resolution, coordinate or version issues, use [DeepDeep-Plugins Issues](https://github.com/Sadami3066/DeepDeep-Plugins/issues) or the [issues of this repository](https://github.com/Sadami3066/deepdeep-maven/issues).
- Please do not report security vulnerabilities in public issues; contact the maintainers privately instead.
