# expo-audio (Crash Resiliency Fork)

This is a fork of the official [expo-audio](https://github.com/expo/expo/tree/main/packages/expo-audio) package with **crash resiliency patches** for persistent recording storage.

## Changes from Upstream

This fork contains **3 modifications** — 2 for crash resiliency (persistent recording storage) and 1 for recording interruption notifications:

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

### 3. Recording Interruption Notification (2026-02-05)

Added native local notifications on both iOS and Android when a background audio recording is interrupted (phone call, alarm, Siri, etc.). The user receives an immediate notification to resume recording.

#### iOS (`ios/AudioModule.swift`)
- Schedules a `UNNotificationRequest` from `handleInterruptionBegan()` when a recorder is actively recording and the app is not in the foreground
- Uses `beginBackgroundTask` to secure execution before iOS suspends the app
- 3-second debounce via `UNTimeIntervalNotificationTrigger` to avoid false positives on short interruptions (e.g., Siri)
- Cancels pending notification in `resumeInterruptedPlayers()` if interruption ends quickly
- Notification identifier: `"recording-interrupted"`

#### Android (`android/.../AudioModule.kt` + `android/.../service/AudioRecordingService.kt`)
- Added `showInterruptionNotification()` and `cancelInterruptionNotification()` to `AudioRecordingService`
- Triggers notification from `audioFocusChangeListener` on `AUDIOFOCUS_LOSS` and `AUDIOFOCUS_LOSS_TRANSIENT` when any recorder is active
- Cancels notification on `AUDIOFOCUS_GAIN`
- Notification-only: recorder behavior is NOT modified (no pause/resume on focus loss)
- Uses separate notification ID (2002) from the foreground service notification (2001)
- Same channel: `expo_audio_recording_channel`

#### Requires
- iOS: Notification permission must be requested at the app level (via notifee or UNUserNotificationCenter)
- Android: `POST_NOTIFICATIONS` permission in AndroidManifest (standard for Android 13+)

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
