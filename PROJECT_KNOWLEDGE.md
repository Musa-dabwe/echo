# PROJECT_KNOWLEDGE.md

## 1. PROJECT OVERVIEW
- **Description:** Echo is a feature-rich, extension-based music player for Android, designed with a focus on a clean UI and modular content sourcing. It allows users to stream and download music from various providers via dynamically loaded extensions.
- **Platform/Language:** Android / Kotlin / Kotlin Multiplatform (KMP).
- **Minimum SDK:** Android SDK 24 (Android 7.0).
- **Current Status:** Development (shifting towards Multiplatform/Compose).
- **Build Requirements:** JDK 17, Gradle 8+, Git (for versioning).

---

## 2. REPOSITORY STRUCTURE
```
app/
  src/main/
    java/dev.brahmkshatriya.echo/
      MainActivity.kt       # Primary Activity, manages navigation and theme
      MainApplication.kt    # App entry point, initializes Koin and Coil
      di/                   # Dependency Injection modules (Koin)
      download/             # Download management, Room DB, and WorkManager tasks
      extensions/           # Extension loading, management, and Built-in extensions
      playback/             # Media3 integration, PlayerService, and session management
      ui/                   # UI components (Fragments, ViewModels) grouped by feature
      utils/                # Helper classes for UI, Coroutines, Serialization, etc.
      widget/               # Home screen widgets (Horizontal, Vertical, Circle)
    res/                    # XML layouts, drawables, strings, and themes
  build.gradle.kts          # Module-level build configuration for Android app
common/
  src/commonMain/kotlin/
    dev.brahmkshatriya.echo.common/
      Extension.kt          # Base classes for all extension types
      clients/              # Interface definitions for Music, Lyrics, Tracker, etc.
      helpers/              # Pagination, exceptions, and injection helpers
      models/               # Data classes (Track, Album, Artist, Shelf, Feed)
      providers/            # Provider interfaces for app-to-extension communication
  build.gradle.kts          # KMP build config, handles Maven publishing
gradle/
  libs.versions.toml        # Centralized dependency management
```

---

## 3. ARCHITECTURE & DESIGN PATTERNS
- **Overall Pattern:** MVVM (Model-View-ViewModel) with a Fragment-based UI architecture.
- **Layer Breakdown:**
  - **UI Layer:** Fragments and ViewModels using ViewBinding and Flow.
  - **Domain/Service Layer:** `ExtensionLoader`, `Downloader`, and `PlayerService` orchestrate core logic.
  - **Data Layer:** Room databases (`DownloadDatabase`, `ExtensionDatabase`) and `UnifiedExtension` for data aggregation.
- **Design Decisions:**
  - **Plugin Architecture:** Content is decoupled from the app via an Extension system. Extensions are loaded as DEX files from APKs (local or installed).
  - **Unified Provider:** `UnifiedExtension` acts as a facade, aggregating data from multiple extensions into a single interface for the UI.
- **Dependency Injection:** Koin (defined in `DI.kt`).
- **Concurrency Model:** Kotlin Coroutines and StateFlow/SharedFlow for reactive data streams.

---

## 4. CORE COMPONENTS

**MainActivity**
- **Path:** `app/src/main/java/dev/brahmkshatriya/echo/MainActivity.kt`
- **Role:** Hosts the main UI and handles global navigation and theme changes.
- **Key methods:** `onCreate`, `getAppTheme`, `applyUiChanges`.
- **Inputs:** UI events, preference changes.
- **Outputs:** Fragment transactions, theme application.
- **Dependencies:** `UiViewModel`, `ExtensionLoader`.

**MainApplication**
- **Path:** `app/src/main/java/dev/brahmkshatriya/echo/MainApplication.kt`
- **Role:** Application entry point for initialization.
- **Key methods:** `onKoinStartup`, `newImageLoader`, `applyLocale`.
- **Dependencies:** Koin, Coil, AppShortcuts.

**ExtensionLoader**
- **Path:** `app/src/main/java/dev/brahmkshatriya/echo/extensions/ExtensionLoader.kt`
- **Role:** Discovers, loads, and manages the lifecycle of extensions.
- **Key methods:** `setPermGranted`, `setupMusicExtension`, `mapped`.
- **State it owns:** Lists of loaded `music`, `tracker`, `lyrics`, and `misc` extensions.
- **Dependencies:** `App`, `CombinedRepository`, `ExtensionDatabase`.

**UnifiedExtension**
- **Path:** `app/src/main/java/dev/brahmkshatriya/echo/extensions/builtin/unified/UnifiedExtension.kt`
- **Role:** Aggregates functionality from all enabled extensions into a single MusicExtension implementation.
- **Key methods:** `loadHomeFeed`, `search`, `loadTrack`, `loadPlaylist`.
- **Gotchas:** Acts as a proxy; failures in one extension can affect the unified view if not handled.

**PlayerService**
- **Path:** `app/src/main/java/dev/brahmkshatriya/echo/playback/PlayerService.kt`
- **Role:** Media3-based background service for audio playback.
- **Key methods:** `onGetSession`, `getController`.
- **Dependencies:** `ExoPlayer`, `MediaSession`, `PlayerState`.

**PlayerState**
- **Path:** `app/src/main/java/dev/brahmkshatriya/echo/playback/PlayerState.kt`
- **Role:** Holds the reactive state of the current playback session.
- **State it owns:** `current` track, `radio` state, `session` ID.

**Downloader**
- **Path:** `app/src/main/java/dev/brahmkshatriya/echo/download/Downloader.kt`
- **Role:** Manages track downloads using WorkManager.
- **Key methods:** `enqueue`, `pause`, `resume`, `delete`.
- **Dependencies:** `DownloadDatabase`, `DownloadWorker`.

**Extension** (Shared)
- **Path:** `common/src/commonMain/kotlin/dev/brahmkshatriya/echo/common/Extension.kt`
- **Role:** Base class defining the structure of Music, Tracker, Lyrics, and Misc extensions.

---

## 5. DATA FLOW DIAGRAMS (text-based)

### Primary Happy Path: Playback Flow
```
User selects Track in UI
  → PlayerViewModel.playCommand()
  → PlayerService.setMediaItem(mediaItem)
  → StreamableMediaSource.prepareSource()
  → StreamableLoader.load(mediaItem)
  → Extension(id).loadStreamableMedia(track)
  → ExoPlayer starts playback
  → TrackingListener.onTrackChanged()
  → Extension(id).onTrackChanged(details)
```

### Search Flow
```
User enters query in Search Bar
  → SearchViewModel.queryFlow updates
  → SearchViewModel.quickSearch(extensionId, query)
  → Extension(id).quickSearch(query)
  → Result<List<QuickSearchItem>> returned
  → UI updates with suggestions
```

### Download Flow
```
User taps Download button
  → PlayerViewModel.download()
  → Downloader.enqueue(track)
  → DownloadDatabase inserts DownloadEntity
  → WorkManager triggers DownloadWorker
  → DownloadWorker calls Extension(id).loadStreamableMedia()
  → Bytes saved to internal storage
  → DownloadDatabase updates status to "FullyDownloaded"
```

---

## 6. EXTERNAL DEPENDENCIES & INTEGRATIONS

| Name | Version | Purpose | Where Used |
|------|---------|---------|-----------|
| Media3 | 1.8.0 | Playback & MediaSession | `PlayerService`, `playback/` |
| Room | 2.7.2 | Local Persistence | `download/db`, `extensions/db` |
| Koin | 4.1.0 | Dependency Injection | `di/DI.kt`, `MainApplication` |
| Coil | 3.3.0 | Image Loading | `MainApplication`, UI Adapters |
| OkHttp | 5.1.0 | HTTP Client | `common` (shared by extensions) |
| Kotlinx Serialization | 1.9.0 | JSON Parsing | Models, Database entities |
| Paging | 3.3.6 | List Pagination | `PagedData`, UI ViewModels |

---

## 7. CONFIGURATION & ENVIRONMENT
- **SharedPreferences:** Used for app settings (Theme, AMOLED mode, Language, Color, Last Extension).
- **BuildConfig Fields:** `VERSION_NAME`, `BUILD_TYPE`.
- **Environment:** No specific ENV variables; uses `google-services.json` for Firebase (optional).
- **Extension Settings:** Each extension gets its own `SharedPreferences` isolated by its ID.

---

## 8. DATABASE & PERSISTENCE
- **DownloadDatabase:** Tracks download tasks and downloaded media metadata.
  - `DownloadEntity`: `id`, `trackId`, `task` (download status), `finalFile` path.
  - `ContextEntity`: Stores metadata of the context (Album/Playlist) from which a download originated.
- **ExtensionDatabase:** Manages extension state and user accounts.
  - `ExtensionEntity`: `id`, `type`, `enabled`.
  - `UserEntity`: Stores login data for extensions.
  - `CurrentUser`: Tracks the logged-in user per extension.
- **UnifiedDatabase:** (Internal to UnifiedExtension)
  - `PlaylistEntity`: Local playlists created by the user.
  - `SavedEntity`: Media items saved to the user's library.

---

## 9. KNOWN ISSUES, TODOS & TECH DEBT
- [ ] `app/.../playback/AndroidAutoCallback.kt:403` — Missing implementation for `toMediaItems`.
- [ ] `app/.../playback/AndroidAutoCallback.kt:414` — `TODO()` in `getFeed` (will crash if called).
- [ ] `app/.../playback/listener/AudioFocusListener.kt:63` — Fix needed to support playback during calls.
- [ ] Build script fragility: Relies on `git` command being present to resolve versioning; crashes if `.git` is missing (mitigated by check).

---

## 10. CRITICAL RULES & CONSTRAINTS
- **Thread Safety:** Never call extension methods from the Main Thread. Always use `Dispatchers.IO`.
- **Extension Isolation:** Extensions should only communicate with the app via the `common` interfaces.
- **DEX Loading:** Extension classes are loaded dynamically. Ensure `Metadata` IDs are unique and consistent.
- **Playback Control:** Use `PlayerController` (from Media3) rather than interacting with `ExoPlayer` directly from the UI.

---

## 11. TESTING
- **Automated Tests:** None currently present in the codebase.
- **Manual Procedures:** Verification requires installing extensions (built-in or external) and testing search/playback/download flows.

---

## 12. BUILD & RELEASE
- **Commands:** `./gradlew assembleRelease`, `./gradlew publishToMavenLocal` (for common).
- **Build Types:**
  - `release`: Minified and obfuscated.
  - `nightly`: Suffix `.nightly`, unique app name.
  - `stable`: Standard release build.
- **Versioning:** Derived from Git commit count and hash.

---

## 13. RECENT CHANGES & EVOLUTION
- **Media3 Migration:** Transitioned from legacy ExoPlayer/MediaSession to Media3 for better session management.
- **Coil 3:** Upgraded to the latest Kotlin Multiplatform version of Coil.
- **KMP Shift:** Moving common logic and models to the `common` module to support future Desktop/Web versions.
- **Unified Extension:** Introduced to simplify UI logic by providing a single point of access to all content.

---

## 14. GLOSSARY

| Term | Meaning |
|------|---------|
| Extension | A separate APK/module providing content (Music, Lyrics, Trackers). |
| Shelf | A horizontal or vertical row of items in the UI (e.g., "Recently Played"). |
| Feed | A collection of Shelves or Tabs representing a complete screen. |
| Unified | The built-in extension that combines all other enabled extensions. |
| PagedData | A custom wrapper for paginated content from extensions. |
| Client | Interface (e.g., `TrackClient`, `SearchClient`) implemented by extensions. |
