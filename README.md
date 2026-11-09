# DeepDeep Maven

Public Maven repository for DeepDeep plugin development artifacts, served by GitHub Pages.

## Coordinates

```kotlin
repositories {
    maven { url = uri("https://sadami3066.github.io/deepdeep-maven") }
}

dependencies {
    compileOnly("io.github.sadami3066.deepdeep:plugin-api:0.7.0")
}
```

`plugin-api` exposes the DeepDeep plugin contract SDK. `design` is the shared design system and comes in transitively through `plugin-api`.

Artifacts are published from the DeepDeep host app with `./scripts/gradlew-locked :plugin-api:publishAllPublicationsToPagesRepository :design:publishAllPublicationsToPagesRepository` and committed here.
