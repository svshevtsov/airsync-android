# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AirSync is an Android application that enables seamless synchronization between Android devices and Mac/PC computers. The app handles clipboard sync, notification forwarding, call history sync, media control, and file transfer over a local network using WebSocket connections.

- **Language:** Kotlin
- **Min SDK:** 30 (Android 11)
- **Target SDK:** 36
- **UI Framework:** Jetpack Compose
- **Architecture:** Clean Architecture + MVVM

## Build & Development Commands

### Building
```bash
# Build debug APK
./gradlew assembleDebug

# Build release APK
./gradlew assembleRelease

# Install debug APK to connected device
./gradlew installDebug
```

### Testing
```bash
# Run unit tests
./gradlew test

# Run instrumented tests (requires connected device/emulator)
./gradlew connectedAndroidTest

# Run specific test
./gradlew test --tests com.sameerasw.airsync.ExampleUnitTest
```

### Linting & Code Quality
```bash
# Run lint checks
./gradlew lint

# Generate lint report (check app/build/reports/lint-results.html)
./gradlew lintDebug
```

### Cleaning
```bash
# Clean build artifacts
./gradlew clean
```

## Architecture

### Package Structure

The codebase follows Clean Architecture principles with clear separation of concerns:

```
app/src/main/java/com/sameerasw/airsync/
├── domain/              # Business logic layer
│   ├── model/          # Data models & domain entities
│   └── repository/     # Repository interfaces
├── data/               # Data persistence & implementations
│   ├── local/          # DataStore implementation (DataStoreManager)
│   └── repository/     # Repository implementations
├── presentation/       # UI layer
│   ├── viewmodel/      # ViewModels (primarily AirSyncViewModel)
│   ├── ui/
│   │   ├── activities/ # Activity components (MainActivity, QRScannerActivity, etc.)
│   │   ├── screens/    # Composable screens
│   │   └── components/ # Reusable UI components
│   └── ui/theme/       # Theme, colors, typography
├── service/            # Android system services
├── utils/              # Utility helpers (35+ utility classes)
├── smartspacer/        # Smartspacer integration
└── widget/             # Home screen widget provider
```

### Key Architectural Patterns

**Network-Aware Device Persistence:**
- Stores device connections mapped to WiFi networks
- Enables automatic device switching when changing networks
- Keys stored as `network_device_<name>_*` and `network_connections_<name>` in DataStore
- JSON format: `{ "ourWifiIp": "clientIp", ... }`

**Singleton Pattern for System Integration:**
- `WebSocketUtil` - Global WebSocket connection manager
- `MacDeviceStatusManager` - Mac device status tracking
- `SyncManager` - Orchestrates sync operations
- `NetworkMonitor` - WiFi network change detection

**Multi-Listener Pattern:**
- WebSocket uses multiple listener types to prevent tight coupling:
  - `connectionStatusListeners` - UI updates
  - Manual connect listeners - Cancel auto-reconnect logic
  - Custom message handlers via `WebSocketMessageHandler`

### Core Components

**WebSocket Connection (`WebSocketUtil`):**
- OkHttp3-based persistent connection
- 30-second ping intervals
- Auto-reconnect with exponential backoff
- Network-aware reconnection (maintains connections per WiFi network)
- 7-second handshake timeout to detect auth failures
- States: `isConnected`, `isConnecting`, `isSocketOpen`, `handshakeCompleted`

**Notification Handling (`MediaNotificationListener`):**
- Extends `NotificationListenerService`
- Real-time media tracking (title, artist, album art)
- Now playing status reporting to Mac
- Like status caching for Spotify/YouTube Music
- Integrates with SyncManager

**QR Code Scanning:**
- ML Kit Vision barcode scanner with CameraX
- Deep link format: `airsync://host:port?name=X&plus=Y&key=Z`
- Validates local network IPs only
- Launches MainActivity with parsed connection params

**Call History & Monitoring:**
- `CallMonitorService` - Optional foreground service (only active when connected)
- `CallReceiver` - BroadcastReceiver for phone state changes
- `CallLogObserver` - Content observer for call log updates
- `CallSyncClient` - Syncs missed/recent calls via WebSocket
- Requires permissions: READ_PHONE_STATE, READ_CALL_LOG, READ_CONTACTS

**Network Discovery:**
- `AdbMdnsDiscovery` - Discovers ADB wireless services via mDNS
- Auto-initialized on app startup
- `NetworkMonitor` - Observes WiFi IP address changes for auto-reconnect

### Data Persistence

**DataStore (Primary Storage):**
- Type: Preferences DataStore
- File: `airsync_settings`
- Manages all app settings, feature toggles, and state
- Key categories:
  - Connection settings (IP, port, device name, network mappings)
  - Feature toggles (clipboard, notifications, auto-reconnect, etc.)
  - Mac status (battery, music for widget)
  - Call sync preferences
  - Notification app whitelist
- Export/import via JSON (`exportAllDataToJson()`, `importAllDataFromJson()`)

**No Room Database:**
- Room dependencies exist but aren't actively used
- All persistence handled through DataStore (lightweight approach)
- Clipboard history kept in-memory only (cleared on disconnect)

### Activities & Screens

**MainActivity:**
- Entry point with Compose-based UI
- Handles deep links (`airsync://` scheme)
- Manages splash screen animation
- Permission requests (notifications, calls, contacts)
- Auto-initializes mDNS discovery
- Notes Role integration for content capture

**QRScannerActivity:**
- CameraX + ML Kit barcode scanning
- Returns QR data for device connection

**ShareActivity:**
- Handles share intents from other apps

**PermissionsActivity:**
- Dedicated permissions request UI

**UI Navigation:**
- NavHost with route-based routing
- Pager-based tabs: Connection, Clipboard, Settings
- Card-based component decomposition

### ViewModel

**AirSyncViewModel:**
- Single centralized ViewModel managing all app state
- `UiState` data class tracks:
  - WebSocket connection status
  - Device info (name, IP, port)
  - Feature toggle states
  - Clipboard history
  - Permission states
- Handles:
  - WebSocket lifecycle
  - Network change monitoring
  - Auto-reconnect logic
  - Clipboard sync
  - Settings export/import

### System Integrations

**Quick Settings Tile (`AirSyncTileService`):**
- Toggle connection on click
- Long-press opens QR scanner when disconnected

**Widget (`AirSyncWidgetProvider`):**
- Home screen widget displaying Mac battery and music status
- Reads from DataStore for Mac device status

**Smartspacer Integration:**
- `AirSyncDeviceTarget` provider
- Optional "show when disconnected" toggle
- Notified of connection status changes

**Content Capture (Android 14+):**
- Notes Role integration
- Floating window screenshot capability
- Stylus mode detection

## Important Development Notes

### Permissions Model
- Runtime permissions via ActivityResultContracts
- Feature toggles for permission-dependent functionality
- Graceful degradation when permissions denied
- Call monitoring only active during active connection

### Connection Flow
1. User scans QR code OR manually enters connection details
2. App validates local network IP and port
3. WebSocket connection established with handshake timeout
4. On successful handshake, connection saved to network-aware storage
5. Auto-reconnect enabled for network changes

### Handshake Timeout Strategy
- 7-second timeout after WebSocket opens
- Detects authentication failures before marking connection successful
- Triggers auth failure dialog in UI if timeout occurs

### Multi-Network Support
- Device connections stored per WiFi network
- Automatically switches to correct device when network changes
- Timestamp-based "most recent device" selection
- Falls back to legacy storage for backwards compatibility

### Utilities Overview (35+ Classes)
Key utility classes handle specialized operations:
- **System:** PermissionUtil, NotificationUtil, HapticUtil, KeyguardHelper
- **Communication:** WebSocketMessageHandler, JsonUtil, CryptoUtil, FileTransferProtocol
- **Mac Integration:** MacDeviceStatusManager, MacMediaPlayerService, MediaControlUtil
- **Sync:** ClipboardSyncManager, ClipboardUtil, ContactLookupHelper, WallpaperUtil, AppIconUtil
- **Device:** DeviceInfoUtil, SyncManager

## Contributing

- Make PRs to `main` or `dev` branch
- Follow existing Kotlin code style
- Respect clean architecture boundaries (domain → data → presentation)
- Test on physical device when possible (notification listener, call monitoring require real device)
- License: Mozilla Public License 2.0 + Non-Commercial Use Clause

## Key Dependencies

- **UI:** Jetpack Compose, Material3
- **Network:** OkHttp3 (WebSocket)
- **Data:** DataStore Preferences, Room (declared but unused)
- **Camera/QR:** CameraX, ML Kit Barcode Scanning
- **Media:** MediaSessionManager
- **JSON:** Gson, org.json
- **Coroutines:** kotlinx-coroutines-android
- **Phone:** libphonenumber (number normalization)
- **Smartspacer:** Kieronquinn SDK

## Version Catalog

Dependencies managed via `gradle/libs.versions.toml`:
- AGP: 8.12.3
- Kotlin: 2.0.21
- Compose BOM: 2024.09.00
