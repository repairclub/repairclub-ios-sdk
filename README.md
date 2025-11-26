# RepairClubSDK

**RepairClubSDK** is a Swift Package Manager–ready framework that provides an easy-to-use interface (`RepairClubManager`) for discovering, connecting to, and interacting with Repair Club built devices over BLE. It exposes APIs for scanning devices, connecting/disconnecting, subscribing to events (VIN, vehicle data, live data, diagnostics), performing scans, retrieving DTC descriptions, and firmware updates.

> **Note:** This SDK is only for connecting to Repair Club built devices. It does not include any UI components; it focuses purely on BLE communication and vehicle data handling.

---

## Installation

Add **RepairClubSDK** to your project via Swift Package Manager:

1. In Xcode, go to **File ▶ Swift Packages ▶ Add Package Dependency...**  
2. Enter your repository URL (e.g. `https://github.com/repairclub/repairclub-ios-sdk.git`) and click **Next**.  
3. Choose a version (e.g. “Up to Next Major” or a specific tag) and click **Next**.  
4. Select the `RepairClubSDK` package product. Under **Add to Target**, verify your app or framework target is checked. Click **Finish**.  

Xcode will fetch the prebuilt `RepairClubSDK.xcframework` and link it automatically. If you open **YourApp ▶ General ▶ Frameworks, Libraries, and Embedded Content**, you should see `RepairClubSDK.xcframework` listed with “Embed & Sign.”

---

## Quick Start

```swift
import RepairClubSDK

// 1. Configure the SDK (usually at app launch)
let manager = RepairClubManager.shared
manager.configureSDK(
    tokenString:    "YOUR_AUTH_TOKEN",
    appName:        "MyApp",
    appVersion:     "1.0.0",
    userID:         "user@example.com"
)

// 2. Scan for devices
manager.returnDevices { result in
    switch result {
    case .success(let devices):
        print("Found devices:", devices)
        // You can pick a CBPeripheral from `devices` to connect
    case .failure(let error):
        print("Scan error:", error)
    }
}

// 3. Connect to a selected device
// Assume `peripheral` is a CBPeripheral from the `returnDevices` callback
manager.connectToDevice(peripheral: peripheral) { connectionEntry, stage, state in
    print("Connection stage:", stage, "State:", state)
    if state == .connected {
        // Now you’re connected—e.g., request VIN or start scans
    }
}

// 4. Subscribe to VIN updates
manager.registerVIN { result in
    switch result {
    case .success(let vin):
        print("Received VIN:", vin)
    case .failure(let error):
        print("VIN error:", error)
    }
}

// 5. Subscribe to vehicle data
manager.registerVehicle { result in
    switch result {
    case .success(let vehicleEntry):
        print("Vehicle details:", vehicleEntry ?? "None")
    case .failure(let error):
        print("Vehicle error:", error)
    }
}

// 6. Start a DTC (trouble-code) scan
manager.startTroubleCodeScan(advancedScan: false) { update in
    switch update {
    case .scanStarted(let info, let scanEntry):
        print("Scan started:", info, scanEntry)
    case .progressUpdate(let percent):
        print("Scan progress:", percent)
    case .modulesUpdate(let modules, let scanEntry):
        print("Modules updated:", modules, scanEntry)
    case .scanSucceeded(let modules, let scanEntry, _):
        print("Scan succeeded:", modules)
    case .scanFailed(let modules, let scanEntry, let errors):
        print("Scan failed with errors:", errors)
    default:
        break
    }
}

// 7. Subscribe to live data streams
manager.subscribeToLiveStream { result in
    switch result {
    case .success(let liveDataDict):
        print("Live data entries:", liveDataDict)
    case .failure(let error):
        print("Live data error:", error)
    }
}

```

---

## Main API: `RepairClubManager`

`RepairClubManager.shared` is the singleton entry-point for all SDK functionality. Below is a summary of commonly used methods:

### Configuration
- `configureSDK(tokenString:appName:appVersion:userID:)`  
  Initialize SDK with authentication token, app name/version, and user ID.
- `configureSdkUser(userID:)`  
  Reconfigure with a different user ID if needed.

### Device Discovery & Connection
- `returnDevices(_:)`  
  Scan for in-range Repair Club devices. Returns `[DeviceItem]`.
- `stopScanningForDevices()`  
  Manually stop BLE scanning.
- `connectToDevice(peripheral:_:)`  
  Connect to a chosen `CBPeripheral`.
- `disconnectFromDevice()` → `(Bool, Error?)`  
  Disconnect from currently connected device.
- `reconnectToPeripheral(with:completionHandler:)`  
  Reconnect by known `UUID`.

### Connection State Callbacks
- `subscribeToDisconnections(_:)`  
  Invoked when the BLE link is lost.
- `registerDeviceConnectionState(changes:)`  
  Observes `DeviceConnectionState` changes (`.connecting`, `.connected`, `.disconnected`, etc.).
- `registerVehicleConnectionState(changes:)`  
  Observes `VehicleConnectionState` changes (e.g. `.vehicleAttached`, `.vehicleReady`).
- `registerBluetoothStatus(changes:)`  
  Observes `BluetoothStatus` (`.poweredOn`, `.poweredOff`, `.unauthorized`, etc.).

### Vehicle / VIN
- `requestVIN()` → `Result<String, Error>`  
  Synchronous attempt to return last-known VIN; fails if none available.
- `registerVIN(callback:)`  
  Asynchronously delivers new VIN when received.
- `registerVehicle(callback:)`  
  Asynchronously delivers a `VehicleEntry` once vehicle details arrive.
- `supplyVehicleInformation(vin:vehicleEntry:)`  
  Supply a manual VIN or `VehicleEntry` when prompted by vehicle info callback.
- `getCurrentVehicleEntry()` → `VehicleEntry?`  
  Get the currently connected `VehicleEntry`.
- `setCurrentVehicleToA(manual:)`  
  Force‐set a manual `VehicleEntry` when in simulation or manual mode.

### Trouble Code Scans
- `startTroubleCodeScan(advancedScan:progressHandler:)`  
  Kick off a generic or advanced DTC scan. Progress via `ScanProgressUpdate`.
- `stopTroubleCodeScan()`  
  Stops an ongoing DTC scan.
- `startExtendedTroubleCodeScan(progressHandler:)`  
  Start extended scan (returns extra data).
- `pvcop(input:progressHandler:)`  
  Programmatic scan for GM/Ford/FCA CAN protocols by integer code (e.g. `111`, `129`).

### Freeze Frame
- `requestFreezeFrameFor(code:scanEntry:callback:)`  
  Get full freeze‐frame monitors for a given trouble code.
- `requestFreezeFrameValuesFor(code:scanEntry:valueCompletionHandler:)`  
  Get only freeze‐frame PID values (raw dictionary).

### Live Data (Mode 01)
- `requestAllLiveDataSources()` → `Result<[LiveDataSource], Error>`  
  List of every possible Mode 01 PID supported.
- `requestAvailableLiveDataSources()` → `Result<[LiveDataSource], Error>`  
  Currently available PIDs that the vehicle reports.
- `startLiveDataStream(with:)` → `Result<Bool, Error>`  
  Begin streaming live data (max 8 PIDs at once).
- `endLiveDataStream()` → `Bool`  
  Stop live data streaming.
- `subscribeToLiveStream(_:)`  
  Receive periodic `[String: LiveFeedDatedEntry]` updates.
- `subscribeToLiveDataRanges(_:)`  
  Receive `(min: [String: Double], max: [String: Double])` for live data ranges.
- `currentLiveFeedCSV(countToKeep:)` → `Result<String, Error>`  
  Get CSV‐formatted snapshot of current live feed.
- `returnLiveFeedCSV(from:countToKeep:)` → `Result<String, Error>`  
  Build CSV for a given set of `LiveFeedDatedEntry`.

### Monitors (Mode 06 & Readiness)
- `subscribeToMonitors(_:)`  
  Get periodic `[ValueMonitor]` readiness monitors (with looping).
- `subscribeToMode06Monitors(_:)`  
  Get `[ValueMonitor]` for Mode 06 live monitors.
- `subscribeToMode06SupportedMonitors(_:)`  
  Get `[String]` of supported Mode 06 PIDs.
- `requestReadinessMonitors(shouldLoop:reqType:)`  
  Send direct readiness requests (11-bit, 29-bit, or internal loops).
- `requestMode06Monitors(shouldLoop:)`  
  Request Mode 06 monitors (looped or one-shot).
- `startRequestMonitorsTimer()` / `endRequestMonitorsTimer()`  
  Control the built-in 10 sec timer for readiness looping.
- `registerMilChange(callback:)` → subscriptions for MIL status changes.
- `requestMilStatus()` → `Result<Bool, Error>` for current MIL bit.

### DTC Descriptions & Vehicle APIs
- `fetchDTCDescription(for:on:)` async/await → `Result<DTCResult?, Error>`  
  Async/await fetch from network for a DTC code description.
- `fetchDTCDescription(for:on:completion:)`  
  Completion‐handler version for the same.
- `requestVinDecode(for:)` async throws → `VINResult`  
  Async decode a VIN via network services.
- `requestVinDetailDecode(for:completionHandler:)`  
  Completion version for detailed VIN decode.
- `requestVehicleID(for:completionHandler:)`  
  Fetch NHTSA vehicle ID for a given `VehicleEntry`.
- `requestSafetyRatingsFor(id:completionHandler:)`  
  Get NHTSA safety ratings by vehicle ID.
- `requestMakeList(_:)`  
  Get `[MakeResult]` from network.
- `requestModels(for:completionHandler:)`  
  Get `[ModelResult]` for a specific make ID.
- `requestRecalls(for:completionHandler:)`  
  Fetch `VehicleRecallResults` for a given `VehicleEntry`.

### Services (Advanced Features)
- `fetchAvailableServices()` → `[ServiceAdvanced]?`  
  List available advanced services on the device.
- `startService(service:progressHandler:)`  
  Start a service (e.g., OEM-specific calibration, programming). Progress via `OperationProgressUpdate`.
- `clearGenericCodes(completionHandler:)`  
  Clear all generic (Mode 03) codes.
- `clearAllCodes(progressHandler:)`  
  Clear ALL codes (generic + advanced), with progress updates.

### Firmware Updates
- `subscribeToFirmwareProgress(_:)`  
  Subscribe to ongoing firmware update progress (a `FirmwareProgress` struct).
- `startDeviceFirmwareUpdate(to:reqReleaseLevel:progressCallback:completionCallback:)`  
  Start a versioned firmware update, returning `Double` progress and a completion.
- `cancelDeviceFirmwareUpdate(completionCallback:)`  
  Cancel an in-progress update.
- `startDeviceFirmwareUpdate(reqVersion:reqReleaseLevel:)`  
  Another form of start (no callbacks).
- `stopDeviceFirmwareUpdate()`  
  Stop and (optionally) disconnect.
- `subscribeToFirmwareVersionChanges(reqReleaseLevel:completionCallback:)`  
  Monitor version info changes (`currentVersion`, `newestVersionAvailable`).
- `getFirmwareVersions(reqReleaseLevel:completion:)`  
  Return `(current, alpha, beta, prod)` version strings.
- `getFirmwareVersionsWithNotes(reqReleaseLevel:completion:)`  
  Return `FirmwareVersionInfo` structs.
- `getDeviceFirmwareVersion()` → `Result<String?, Error>`  
  Get current device firmware.
- `getNewestAvailableFirmwareVersion()` → `Result<String?, Error>`  
  Get newest available firmware.

### Retry & Misc
- `retryDecodingTheVIN()` / `retryTheVehicleConfig()`  
  Manual retry entry points when a previous step‐in‐chain failed.
- `retryBusSync(completion:)`  
  Manual retry for bus sync.
- `clearKeyCahe()` (async)  
  Clear stored encryption keys.

### Settings & Configuration
- `configureSDK(with:)`  
  Provide an `SDKConfiguration` object for settings (dev server, debugger, etc.).
- `returnDevInfo(with:completionHandler:)`  
  Fetch individual developer info from `SettingsSDKManager`.
- Flags & toggles:
  - `isVinUnavailable()`, `isUseToastEnabled()`, `isUseImperialUnitsEnabled()`, `isDeveloperModeEnabled()`, `isTesterModeEnabled()`, `isUseWrapperEnabled()`
  - `useDevServerValue()`, `getSampleVehicleOnSim()`
- Setters:
  - `setVinUnavailable(_:)`, `setUseToast(_:)`, `setUseImperialUnits(_:)`,
  - `setUseWrapper(_:)`, `setUseDevServer(_:)`, `setSampleVehicleOnSim(_:)`,
  - `setProcess(with:)`, `sendAdditionalDataRequests()`
- Cache clearing:
  - `clearCache(type:)`

---

## Example

```swift
import RepairClubSDK

class MyViewModel: ObservableObject {
    @Published var foundDevices: [DeviceItem] = []
    @Published var connectionState: DeviceConnectionState = .disconnected

    init() {
        let sdk = RepairClubManager.shared
        sdk.configureSDK(
            tokenString: "abc123xyz",    // Obtain from Repair Club, LLC
            appName: "YourApp",
            appVersion: "1.2.3",
            userID: "some.user@example.com"
        )

        // Listen for device connection state
        sdk.registerDeviceConnectionState { state in
            DispatchQueue.main.async {
                self.connectionState = state
            }
        }

        // Scan for devices
        sdk.returnDevices { result in
            switch result {
            case .success(let devices):
                self.foundDevices = devices
            case .failure(let error):
                print("Scan error:", error)
            }
        }
    }

    func connect(to item: DeviceItem) {
        if let peripheral = item.peripheral {
            RepairClubManager.shared.connectToDevice(peripheral: peripheral) { _, _, state in
                print("Connected state:", state)
            }
        }
    }
}
```

---


