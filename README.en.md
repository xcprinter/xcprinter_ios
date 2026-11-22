# Bluetooth Driver for Receipt (Label) Printers

Key features:
* Device scanning
* Device connection and disconnection
* Print data transmission
* Printer MTU and Credits flow control

## The XCFramework contains the following slices:

- iOS device: `arm64`
- iOS Simulator: `arm64`, `x86_64`

## How to Use

### Import the SDK Files

* Copy the files: copy the `XcPrinterBluetooth` directory into your Xcode project (including Package.swift);
* Import via Git submodule: `git submodule add git@github.com:xcprinter/xcprinter_ios.git XcPrinterBluetooth`;

### Configure in Xcode

* In Xcode, choose `File > Add Package Dependencies… > Add Local…`
* Select the `XCPrinterBluetooth` directory and add the `XCPrinterBluetooth` library to your App target.

### Import in Code

```swift
import XCPrinterBluetooth
```

### Configure Bluetooth Permissions

The App integrating the SDK must add the following to its own `Info.plist`:

```xml
<key>NSBluetoothAlwaysUsageDescription</key>
<string>Bluetooth access is required to discover and connect to printers for printing</string>
```

The permission description can be adjusted to match your App's actual use case.

### Create an Instance

```swift
import XCPrinterBluetooth

let bluetooth = BluetoothManager()
```

Keep a strong reference to this instance and use the same instance for scanning, connecting, sending, and disconnecting. Different instances manage their own CoreBluetooth sessions, so do not create a new instance for each operation.

## Scan for Devices

`scan()` returns an `AsyncStream<DeviceInfo>`. Each scan lasts 5 seconds and emits an event when a device is discovered:

```swift
let scanTask = Task {
  for await device in bluetooth.scan() {
    print("Name: \(device.name ?? "Unknown")")
    print("Address: \(device.address)")
  }
}
```

`DeviceInfo` properties:

| Property | Type | Description |
| --- | --- | --- |
| `name` | `String?` | Device broadcast name |
| `address` | `String` | The device UUID string assigned by iOS, used for connecting and disconnecting |

To stop consuming scan events, cancel the corresponding Task:

```swift
scanTask.cancel()
```

## Connect to a Device

Use the `address` from a scan result:

```swift
let connected = await bluetooth.connect(to: device.address)

if connected {
  print("Connected successfully")
} else {
  print("Connection failed")
}
```

It is recommended to complete device scanning before initiating a connection. If the specified device cannot be found, the current implementation restarts scanning and returns `false`.

A successful connection result means the system Bluetooth link has been established; the SDK will continue to initialize the FF00 service, FF02/FF03 characteristics, and notifications in the background.

## Send Data

The API accepts `[UInt8]`:

```swift
let bytes: [UInt8] = [0x1B, 0x40]
let result = await bluetooth.send(bytes: bytes)

if result.success {
  print("Data written to the Bluetooth channel")
} else {
  print("Send failed: \(result.error ?? "Unknown error")")
}
```

If you already have `Data`, convert it like this:

```swift
let result = await bluetooth.send(bytes: Array(data))
```

The SDK automatically splits data according to the MTU and sends packets serially using Credits flow control. `success == true` means all data from this call has been submitted to the Bluetooth write channel; it does not mean the printer has finished printing.

When no device is connected, the following is returned:

```swift
SendResultData(success: false, error: "未连接设备")
```

If the connection drops, an in-progress send returns a failure result, and the caller decides whether to retry.

## Disconnect

```swift
let disconnected = await bluetooth.disconnect(to: device.address)
```

If the device no longer exists or is not connected, the current implementation returns `true`.

## Observe Bluetooth Readiness Events

When Bluetooth becomes available, `readyEvents` emits `true`:

```swift
let readyTask = Task {
    for await isReady in bluetooth.readyEvents {
        print("Bluetooth ready: \(isReady)")
    }
}
```

## Observe Unexpected Disconnection Events

Disconnections not caused by an explicit call to `disconnect` are reported through `disconnectionEvents`:

```swift
let disconnectionTask = Task {
    for await disconnected in bluetooth.disconnectionEvents {
        print("Device disconnected: \(disconnected)")
    }
}
```

When you no longer need to listen, cancel the corresponding Tasks:

```swift
readyTask.cancel()
disconnectionTask.cancel()
```

## Query Known Device Status

If you already have a device UUID, you can query the device currently recorded by the system:

```swift
if let uuid = UUID(uuidString: device.address) {
  let peripherals = await bluetooth.getConnectedPeripherals(
    identifiers: [uuid]
  )

  if let peripheral = peripherals.first {
    print("Connection state: \(peripheral.state.rawValue)")
  }
}
```

This method returns a CoreBluetooth `CBPeripheral`; import CoreBluetooth in the calling file if needed:

```swift
import CoreBluetooth
```
