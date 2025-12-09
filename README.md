# expo-audio (Crash Resiliency Fork)

This is a fork of the official [expo-audio](https://github.com/expo/expo/tree/main/packages/expo-audio) package with **crash resiliency patches** for persistent recording storage.

## Changes from Upstream

This fork contains **2 critical modifications** to ensure audio recordings survive app crashes, OS kills, and cache purges:

### 1. Android: Persistent Storage Location
**File:** `android/src/main/java/expo/modules/audio/AudioRecorder.kt`

```kotlin
// Upstream: Uses cacheDir (can be cleared by OS)
val directory = File(context.cacheDir, "Audio")

// This fork: Uses filesDir (persistent app storage)
val directory = File(context.filesDir, "recordings").apply { mkdirs() }
```

### 2. iOS: Persistent Storage Location
**File:** `ios/AudioRecorder.swift`

```swift
// Upstream: Uses cachesDirectory (can be purged by iOS)
guard let cachesDir = appContext?.fileSystem?.cachesDirectory else { ... }

// This fork: Uses documentDirectory (persistent, backed up)
guard let docsDir = appContext?.fileSystem?.documentDirectory else { ... }
let recordingsDir = URL(fileURLWithPath: docsDir).appendingPathComponent("recordings")
```

## Why These Changes?

- **Cache directories** (`cacheDir` on Android, `cachesDirectory` on iOS) can be cleared by the OS at any time:
  - When device storage is low
  - When app is backgrounded/terminated
  - During system maintenance

- **Persistent directories** (`filesDir` on Android, `documentDirectory` on iOS):
  - Survive app crashes and restarts
  - Are not subject to automatic cache clearing
  - On iOS, can be backed up to iCloud

These changes are essential for offline-friendly recording workflows where recordings must survive unexpected app terminations.

## Keeping Up to Date

To sync with upstream expo-audio releases:

```bash
# Add upstream remote (one-time)
git remote add upstream https://github.com/expo/expo.git

# Fetch latest from expo monorepo
git fetch upstream main

# Check for changes in packages/expo-audio and manually merge
```

## Installation

```json
{
  "dependencies": {
    "expo-audio": "github:zcharef/expo-audio"
  }
}
```

## License

MIT - Same as the original expo-audio package.
