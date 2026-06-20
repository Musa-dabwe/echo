## Project Overview
**Echo** is a feature-rich Android music player application designed with an extensible architecture. It consists of two main modules:
- **app**: The primary Android application module, containing the UI (Fragments, ViewModels), dependency injection (Koin), and core logic for playback, downloads, and extension management.
- **common**: A Kotlin Multiplatform (KMP) library module that defines the interfaces and shared data structures used by both the app and its external extensions.

The project uses modern Android technologies including **Media3** for audio playback, **Room** for persistence, **Coil 3** for image loading, and **Paging 3** for list management. It follows a modular design where extensions (Music, Lyrics, etc.) can be dynamically loaded as APKs or installed packages.

## Build Failure Analysis
The build fails during the configuration phase with the following error:
```
Process 'command 'git'' finished with non-zero exit value 128
```
- **Exact Location**: `app/build.gradle.kts` at line 97 (within the `execute` helper function).
- **Root Cause**: The build script attempts to eagerly resolve the current Git commit hash and commit count to generate the `versionName` and `versionCode`. It uses `providers.exec { ... }.standardOutput.asText.get()`.
- **Why Exit Code 128 occurs**: Git returns 128 when it is executed outside of a valid Git repository (e.g., in a downloaded ZIP archive), when Git is not installed in the environment, or when the repository has no commits. Because the `.get()` call is eager and the default behavior of `providers.exec` is to fail on non-zero exit values, the entire Gradle configuration phase crashes.

## Fix Recommendations
To resolve this, the `execute` function should be made robust by ignoring exit values and providing a fallback.

### Before (Fragile):
```kotlin
fun execute(vararg command: String): String = providers.exec {
    commandLine(*command)
}.standardOutput.asText.get().trim()
```

### After (Robust):
```kotlin
fun execute(vararg command: String): String = providers.exec {
    commandLine(*command)
    isIgnoreExitValue = true
}.standardOutput.asText.map { it.trim() }.getOrElse("unknown")
```

*Note: For `gitCount`, which is converted to an integer, the fallback should be a numeric string:*
```kotlin
val gitHash = execute("git", "rev-parse", "HEAD").take(7)
val gitCountString = execute("git", "rev-list", "--count", "HEAD")
val gitCount = if (gitCountString == "unknown") 0 else gitCountString.toInt()
```

## Code Quality Notes
- **Hardcoded Versions in `common`**: The `common/build.gradle.kts` file has hardcoded versions for `mavenPublishing` and `dokka` (e.g., `1.0.0`). These should ideally be managed via `libs.versions.toml` for consistency with the `app` module.
- **Jetifier Enabled**: `android.enableJetifier=true` is enabled in `gradle.properties`. Given the project uses modern AndroidX and Media3 libraries, Jetifier is likely no longer necessary and slows down build times.
- **Bleeding Edge Tooling**: The project uses **AGP 8.12.2**, **Kotlin 2.2.10**, and **compileSdk 36**. These are extremely recent (preview/canary) versions. While innovative, they may introduce instability or compatibility issues with certain plugins.
- **Unused GMS Plugins**: In `app/build.gradle.kts`, GMS and Crashlytics plugins are applied conditionally based on `google-services.json`. This is a good practice, but the `alias` declarations at the top are always present.

## Dependencies & Risks
- **Dependency Versioning**: Some libraries in `libs.versions.toml` use alpha/beta versions (e.g., `material = "1.14.0-alpha04"`). This poses a risk of breaking changes during updates.
- **Maven Central Publishing**: The `common` module is configured to publish to Maven Central. Ensure that the `SCM` and `developer` information in `common/build.gradle.kts` is updated to reflect the actual project ownership if it's a fork or template.
- **Large JVM Args**: `org.gradle.jvmargs=-Xmx2048M` is relatively low for a modern Android project with KSP and KMP. If the build runs out of memory, this should be increased to `4G`.
