# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

```bash
# Build debug APK
./gradlew assembleDebug

# Build release APK (with ProGuard minification and resource shrinking)
./gradlew assembleRelease

# Install debug build to connected device
./gradlew installDebug

# Clean build
./gradlew clean

# Run tests (if available)
./gradlew test
```

## Project Overview

Chora is an Android music player that streams from Subsonic/Navidrome servers or plays local files. It supports Android Auto, internet radio, synced/unsynced lyrics, and multiple Navidrome libraries.

**Important Context**: This is the author's first Kotlin project. The codebase has some inconsistencies and is acknowledged as "not well-organized." When making changes, prioritize improving code quality and consistency while implementing features.

## Architecture

### Clean Architecture with MVVM

```
┌─────────────────────────────────────────────────────────────┐
│                    Presentation Layer                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │   Compose    │  │  ViewModels  │  │   Navigation     │  │
│  │     UI       │◄─┤   (Hilt)     │  │   (Nested Nav)   │  │
│  └──────────────┘  └──────┬───────┘  └──────────────────┘  │
└────────────────────────────┼────────────────────────────────┘
                             │
┌────────────────────────────┼────────────────────────────────┐
│                    Domain Layer                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Repositories │◄─┤  Managers    │  │   DataStore      │  │
│  │  (Interface) │  │ (Business)   │  │   (Settings)     │  │
│  └──────┬───────┘  └──────────────┘  └──────────────────┘  │
└─────────┼──────────────────────────────────────────────────┘
          │
┌─────────┼──────────────────────────────────────────────────┐
│                      Data Layer                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  DataSource  │  │   Ktor       │  │   MediaStore     │  │
│  │   (Local,    │  │   Client     │  │   (Local Files)  │  │
│  │  Navidrome)  │  └──────────────┘  └──────────────────┘  │
│  └──────────────┘                                            │
└─────────────────────────────────────────────────────────────┘
```

### Key Technologies

- **UI**: Jetpack Compose with Material 3
- **Navigation**: Jetpack Navigation Compose with nested graphs
- **DI**: Hilt (all ViewModels use `@HiltViewModel`, repositories use `@Inject`)
- **Media**: Media3 (ExoPlayer + MediaLibraryService for Android Auto)
- **Async**: Kotlin Coroutines + StateFlow/Flow
- **Networking**: Ktor client
- **Persistence**: DataStore for preferences

## Package Structure

```
com.craftworks.music/
├── MainActivity.kt              # Single activity entry point
├── NavGraph.kt                  # Navigation setup with nested graphs
├── ChoraApplication.kt          # Hilt application class
│
├── ui/
│   ├── screens/                 # Compose screens (one per destination)
│   │   ├── HomeScreen.kt
│   │   ├── AlbumScreen.kt
│   │   ├── ArtistScreen.kt
│   │   ├── PlaylistScreen.kt
│   │   ├── SongsScreen.kt
│   │   ├── RadioScreen.kt
│   │   ├── SettingScreen.kt
│   │   └── settings/            # Settings sub-screens
│   ├── viewmodels/              # Screen ViewModels
│   ├── elements/                # Reusable UI components
│   ├── playing/                 # Now playing screen components
│   └── theme/                   # Material 3 theme
│
├── data/
│   ├── model/                   # Data models + Screen.kt (routes)
│   ├── repository/              # Repository implementations
│   └── datasource/              # Data sources (local, navidrome, lrclib)
│
├── providers/                   # Music providers (LocalProvider, NavidromeProvider)
├── managers/                    # Business logic managers
│   └── settings/                # Settings managers (DataStore wrappers)
└── player/
    └── ChoraMediaLibraryService.kt  # Media3 service for Android Auto
```

## Navigation Architecture

- **Single Activity** with Compose navigation
- **Nested navigation graphs** for artists, playlists, and settings (defined in NavGraph.kt)
- **Screen routes** defined in `data/model/Screen.kt` as sealed class
- **ViewModel scoping**: ViewModels scoped to `main_graph` parent entry for shared state across screens

### Navigation Graphs
- `main_graph` - Root graph containing all top-level destinations
- `artists_graph` - Artists list → Artist details
- `playlists_graph` - Playlists list → Playlist details
- `settings_graph` - Settings → Appearance/Providers/Playback

## Media Player Architecture

- **Player**: Media3 ExoPlayer managed by `ChoraMediaLibraryService`
- **Service**: `MediaLibraryService` for Android Auto compatibility
- **Controller**: `rememberManagedMediaController()` composable for UI access
- **Metadata**: `MediaMetadata` flows from player through UI state

## Data Flow Patterns

### Repository Pattern
Each data type has a repository that abstracts multiple sources:
- **Local files** via MediaStore
- **Navidrome server** via Ktor HTTP client
- **LrcLib** for lyrics via Ktor

### State Management
- **ViewModels**: Expose `StateFlow`/`Flow` for reactive state
- **UI**: `collectAsStateWithLifecycle()` for Compose integration
- **Settings**: `DataStore` with Flow-based managers (e.g., `AppearanceSettingsManager`)

## Important Conventions

1. **Hilt DI**: All ViewModels marked with `@HiltViewModel`, dependencies with `@Inject`
2. **Coroutines**: Use viewModelScope for ViewModels, rememberCoroutineScope in Composable
3. **Screen routes**: Add new routes to `Screen.kt` sealed class
4. **Navigation**: Update `NavGraph.kt` when adding new screens
5. **Bottom nav**: Customize via `AppearanceSettingsManager.bottomNavItemsFlow`
6. **Media items**: Use `MediaItem` and `MediaMetadata` from Media3
7. **URL encoding**: URLs in navigation arguments are URL-encoded, decode with `URLDecoder.decode()`

## Multi-Provider Support

The app supports multiple music providers simultaneously:
- **LocalProvider**: Device's local music files via MediaStore
- **NavidromeProvider**: Navidrome/Subsonic server via REST API

Each provider has a manager class that handles authentication, data fetching, and caching.

## Android TV Support

- TV mode detected via `Configuration.UI_MODE_TYPE_TELEVISION`
- Landscape orientation shows dedicated now-playing screen
- NavigationRail (instead of BottomBar) on TV/landscape
- Focus navigation optimizations required for TV

## Known Issues

- In Android Auto, radios do not set metadata correctly
- Local DB for offline mode is WIP
