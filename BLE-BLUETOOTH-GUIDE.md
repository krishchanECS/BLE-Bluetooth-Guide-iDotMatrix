# BLE/Bluetooth Concepts Guide for iDotMatrix Development

A comprehensive guide to understanding Bluetooth Low Energy (BLE) concepts essential for working with iDotMatrix projects on ESP32.

---

## Table of Contents

1. [BLE Architecture Overview](#1-ble-architecture-overview)
2. [Services & Characteristics](#2-services--characteristics-the-core-structure)
3. [Characteristic Properties](#3-characteristic-properties-what-operations-are-allowed)
4. [BLE Advertising](#4-ble-advertising-how-devices-get-discovered)
5. [Connection Lifecycle](#5-connection-lifecycle)
6. [Communication Patterns](#6-read-vs-write-vs-notify-communication-patterns)
7. [Packet Structure & Framing](#7-packet-structure--framing)
8. [MTU (Maximum Transmission Unit)](#8-mtu-maximum-transmission-unit)
9. [GATT Descriptors](#9-gatt-descriptors-metadata-about-characteristics)
10. [Threading & Synchronization](#10-threading--synchronization)
11. [Common BLE Issues](#11-common-ble-issues-to-watch-for)
12. [Protocol Layers](#12-protocol-layers-in-idotmatrix)
13. [Quick Reference APIs](#quick-reference-esp32-ble-apis-used-in-idotmatrix)

---

## 1. BLE Architecture Overview

### BLE Roles

BLE devices operate in one of two roles:

- **Central/Client** — Initiates connections and makes requests (like the iDotMatrix app on your phone)
- **Peripheral/Server** — Advertises and waits for connections (like the ESP32 emulator)

In **iDotMatrix ESP32 Emulator**, the ESP32 acts as a **peripheral/server** that the official app (central) connects to.

This is different from most public iDotMatrix projects, where the ESP32 acts as a client controlling original hardware.

---

## 2. Services & Characteristics (The Core Structure)

Think of BLE communication like a filing system:

```
Service (UUID)
├── Characteristic (UUID)
│   ├── Properties (WRITE, READ, NOTIFY, etc.)
│   └── Value (actual data)
└── Characteristic
    └── ...
```

### UUID (Universally Unique Identifier)

Each service and characteristic has a unique 128-bit identifier:
- **Standard UUIDs**: `0000xxxx-0000-1000-8000-00805f9b34fb` (Bluetooth SIG defined)
- **Custom UUIDs**: Arbitrary UUIDs for proprietary protocols (like iDotMatrix)

### In iDotMatrix: Two Services

| Service | UUID | Purpose |
|---------|------|---------|
| **FA Service** | `000000fa-0000-1000-8000-00805f9b34fb` | Main command/response channel |
| **AE Service** | `0000ae00-0000-1000-8000-00805f9b34fb` | Secondary channel |

### Characteristics within FA Service

| Characteristic | UUID | Direction | Type | Use |
|---|---|---|---|---|
| **FA02** | `0000fa02-0000-1000-8000-00805f9b34fb` | App → Device | WRITE | App sends commands to device |
| **FA03** | `0000fa03-0000-1000-8000-00805f9b34fb` | Device → App | NOTIFY/READ | Device sends responses back |

### Characteristics within AE Service

| Characteristic | UUID | Direction | Type | Use |
|---|---|---|---|---|
| **AE01** | `0000ae01-0000-1000-8000-00805f9b34fb` | App → Device | WRITE | Secondary write channel |
| **AE02** | `0000ae02-0000-1000-8000-00805f9b34fb` | Device → App | NOTIFY | Secondary notifications |

---

## 3. Characteristic Properties (What Operations Are Allowed)

Properties define what operations a characteristic supports:

```cpp
BLECharacteristic::PROPERTY_WRITE      // Client can write values to this
BLECharacteristic::PROPERTY_WRITE_NR   // Client can write without response
BLECharacteristic::PROPERTY_READ       // Client can read current value
BLECharacteristic::PROPERTY_NOTIFY     // Server can notify client of changes
BLECharacteristic::PROPERTY_INDICATE   // Server sends notifications with confirmation
```

### Example from iDotMatrix Code

```cpp
// FA02: Accept write commands from app
fa02 = fas->createCharacteristic(FA02_UUID, 
  BLECharacteristic::PROPERTY_WRITE | BLECharacteristic::PROPERTY_WRITE_NR);
fa02->setCallbacks(new FA02Callbacks());

// FA03: Send notifications back to app
fa03 = fas->createCharacteristic(FA03_UUID, 
  BLECharacteristic::PROPERTY_READ | BLECharacteristic::PROPERTY_NOTIFY);
fa03->addDescriptor(new BLE2902());  // Enable notification capability
```

### Property Breakdown

- **WRITE**: Response is sent back (slower, more reliable)
- **WRITE_NR**: No response expected (faster, "fire and forget")
- **NOTIFY**: Unreliable one-way update to client
- **INDICATE**: Reliable one-way update (requires client confirmation)

---

## 4. BLE Advertising (How Devices Get Discovered)

Before a connection, the ESP32 broadcasts advertisement packets saying **"I exist, connect to me!"** This includes:

- Device name
- Service UUIDs
- Manufacturer data (firmware version, device type, etc.)
- TX power level

### Advertisement Packet Anatomy

```
┌──────────────┐
│ Flags        │ BLE mode (discoverable, BR/EDR support, etc.)
├──────────────┤
│ Device Name  │ "iDotMatrix-ESP32"
├──────────────┤
│ Service List │ FA Service UUID
├──────────────┤
│ Mfg Data     │ Manufacturer-specific data (version, type)
└──────────────┘
```

### iDotMatrix Advertising Example

```cpp
BLEDevice::init(DEVICE_NAME);
BLEAdvertising *adv = BLEDevice::getAdvertising();

BLEAdvertisementData ad;
ad.setFlags(ESP_BLE_ADV_FLAG_GEN_DISC | ESP_BLE_ADV_FLAG_BREDR_NOT_SPT);
ad.setName(DEVICE_NAME);
ad.setCompleteServices(BLEUUID(FA_SERVICE_UUID));

// Manufacturer data: version and device type
const char mb[] = {0x54, 0x52, 0x00, 0x70, 
                   (char)IDOTMATRIX_SCREEN_TYPE,  // 0x01 = 16x16, 0x03 = 32x32, 0x04 = 64x64
                   (char)FW_RELEASE_MAJOR,         // e.g., 0
                   (char)FW_RELEASE_MINOR};        // e.g., 4
ad.setManufacturerData(String(mb, sizeof(mb)));

adv->setAdvertisementData(ad);
adv->start();
```

The app scans for this signal and identifies:
- Device name and availability
- Resolution type (16×16, 32×32, or 64×64)
- Firmware version

---

## 5. Connection Lifecycle

### State Flow Diagram

```
┌──────────────────────────────────────┐
│ 1. Advertising State                 │
│ ├─ Device broadcasts availability    │
│ └─ Listening for connection requests │
└──────────┬───────────────────────────┘
           │
           │ App (central) scans and finds device
           │
           ↓
┌──────────────────────────────────────┐
│ 2. Connection Requested              │
│ └─ App initiates BLE connection      │
└──────────┬───────────────────────────┘
           │
           │ BLE link layer negotiates connection parameters
           │ (connection interval, latency, timeout)
           │
           ↓
┌──────────────────────────────────────┐
│ 3. Connected State                   │
│ ├─ Services/Characteristics visible  │
│ ├─ Ready to exchange data            │
│ └─ onConnect() callback fires        │
└──────────┬───────────────────────────┘
           │
           │ App sends commands, device sends responses
           │ (connection maintained with periodic packets)
           │
           ↓
┌──────────────────────────────────────┐
│ 4. Disconnected                      │
│ ├─ Connection lost or explicitly     │
│ │  closed by app or device           │
│ └─ onDisconnect() callback fires     │
└──────────┬───────────────────────────┘
           │
           │ Device resumes advertising
           │
           ↓
┌──────────────────────────────────────┐
│ 5. Advertising Again                 │
└──────────────────────────────────────┘
```

### iDotMatrix Connection Callbacks

```cpp
class ServerCallbacks : public BLEServerCallbacks {
  void onConnect(BLEServer*) override {
    deviceConnected = true;           // Track connection state
    screenOn = true;                  // Wake up display
    setStatusLed(true);               // Show connection status
    refreshMatrix();                  // Update display
    triggerConnectionBuzzer();        // Audio feedback
    pendingDeviceInfoPush = true;     // Send device info
    deviceInfoPushAt = millis() + 1200;
  }
  
  void onDisconnect(BLEServer*) override {
    deviceConnected = false;
    packetReceived = packetExpected = 0;  // Reset packet state
    resetBulkTransfer(true);               // Abort any pending transfers
    
    // Schedule advertising restart (after 300ms to avoid BLE stack issues)
    pendingAdvertisingRestart = true;
    advertisingRestartAt = millis() + 300;
  }
};

// Usage
BLEServer *server = BLEDevice::createServer();
server->setCallbacks(new ServerCallbacks());
```

### Connection Parameters

When connected, BLE negotiates:

- **Connection Interval**: How often to exchange packets (e.g., 7.5 ms - 4 seconds)
- **Slave Latency**: How many connection events the peripheral can skip
- **Supervision Timeout**: When to declare connection lost (e.g., 4 seconds)

iDotMatrix doesn't explicitly control these, but they affect:
- Responsiveness (shorter interval = faster response)
- Power consumption (longer interval = lower power)

---

## 6. Read vs. Write vs. Notify (Communication Patterns)

### WRITE (Request-Response)

**Direction**: Client → Server  
**Use Case**: App sends commands to device

The app writes data to a characteristic and optionally waits for a response.

```cpp
// Device side: Handle incoming write
class FA02Callbacks : public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic *c) override {
    String value = c->getValue();
    if(!value.length()) return;
    
    // Lock access to shared state
    if(!lockRuntimeState()) return;
    
    // Process the command
    processFA02Write((const uint8_t*)value.c_str(), value.length());
    
    unlockRuntimeState();
  }
};
```

**iDotMatrix Examples**:
- Setting brightness: `05 00 04 80 PERCENT`
- Sending a GIF: Multi-packet bulk transfer
- Setting alarm: `CMD=00 SUB=80 [alarm_data...]`

### NOTIFY (Unsolicited Updates)

**Direction**: Server → Client  
**Use Case**: Device sends data to app without being asked

The device pushes updates to the app. The app has subscribed to notifications via CCCD (see section 9).

```cpp
// Device side: Send notification
void sendFA03(const uint8_t *data, size_t len) {
  if (!deviceConnected || !fa03) return;
  
  fa03->setValue(data, len);       // Set the new value
  fa03->notify();                  // Push to app
  
  #if DEBUG_SERIAL
    Serial.print("TX FA03 [");
    Serial.print(len);
    Serial.print("]: ");
    dumpHex(data, len);
  #endif
}
```

**Reliability**: Notifications are **unreliable** — if the app misses a packet, there's no automatic retry.

**iDotMatrix Examples**:
- Sending ACK: `05 00 CMD SUB STATUS`
- Device info response: `09 00 01 80 VERSION_MAJOR VERSION_MINOR ...`
- Countdown completion: `05 00 08 80 03`

### READ (Pull Data)

**Direction**: Client → Server  
**Use Case**: App requests current value

Less commonly used in iDotMatrix (mostly WRITE/NOTIFY), but the characteristic always has a readable value.

```cpp
// Device side: Set readable value
fa03->setValue(data, len);  // This value can be read by app anytime
```

### Comparison Table

| Pattern | Direction | Reliability | Speed | Use |
|---------|-----------|-------------|-------|-----|
| **WRITE** | Client → Server | Yes (ACK) | Medium | Commands |
| **NOTIFY** | Server → Client | No (best effort) | Fast | Responses, events |
| **READ** | Client → Server (pull) | Yes | Slow (requires app poll) | Status queries |
| **INDICATE** | Server → Client | Yes (with confirmation) | Slow | Critical data |

---

## 7. Packet Structure & Framing

iDotMatrix uses a **length-prefixed packet format** for all data:

### Packet Header

```
Byte 0-1:  [LEN_LO] [LEN_HI]   (total packet length, little-endian uint16)
Byte 2+:   [PAYLOAD...]        (actual command/data)
```

### Examples

**Device Info Query**:
```
04 00  →  Length = 0x0004 (4 bytes)
09 80  →  CMD=09, SUB=80
```

**Device Info Response**:
```
09 00  →  Length = 0x0009 (9 bytes)
01 80  →  CMD=01, SUB=80
1A     →  Release Major = 0x1A (26, for version 26.x)
04     →  Release Minor = 0x04 (4, for version x.4)
01     →  Reserved
04     →  Screen Type = 0x04 (64x64)
00     →  Reserved
```

**Brightness Command**:
```
05 00  →  Length = 0x0005 (5 bytes)
04 80  →  CMD=04 (brightness), SUB=80
50     →  Brightness = 80% (decimal)
```

### Why Length-Prefixed?

BLE may fragment large packets across multiple writes (MTU = 20 bytes). The device needs to know:
1. When the complete packet has arrived
2. If there's more data coming

```cpp
void processFA02Write(const uint8_t *data, size_t len) {
  // On first write: extract declared packet length
  if(packetReceived == 0) {
    packetExpected = (uint16_t)data[0] | ((uint16_t)data[1] << 8);
  }
  
  // Accumulate bytes
  memcpy(packetBuffer + packetReceived, data, len);
  packetReceived += len;
  
  // When complete, process
  if(packetReceived >= packetExpected) {
    processFA02Packet(packetBuffer, packetExpected);
    packetReceived = packetExpected = 0;
  }
}
```

### Multi-Byte Fields: Little-Endian

All multi-byte values are stored in **little-endian** format (least significant byte first):

```
uint16 = 0x1234  →  bytes: [0x34, 0x12]
uint32 = 0x12345678  →  bytes: [0x78, 0x56, 0x34, 0x12]
```

Example from protocol:
```
Brightness CRC: 0xABCDEF12
Transmitted as: [0x12, 0xEF, 0xCD, 0xAB]
```

---

## 8. MTU (Maximum Transmission Unit)

### BLE MTU Limits

- **Default MTU**: 20 bytes (payload) + 3 bytes (header) = **23 bytes total**
- **Maximum MTU**: Typically 244-512 bytes (depends on hardware and negotiation)

### Problem: Large Transfers

Sending a 64×64 GIF (>50 KB) in 20-byte chunks would require thousands of writes!

### iDotMatrix Solutions

#### 1. Packet Fragmentation (Transparent to App)

The device's BLE stack automatically fragments large `setValue()` calls across multiple ATT writes.

#### 2. Bulk Transfer Protocol

For very large data (GIFs, Device Assets), use a multi-packet protocol with:
- Chunk size: 4096 bytes
- CRC32 validation per chunk
- Timeout protection: 30 seconds

```cpp
#define MAX_PACKET_SIZE     8192      // Max single logical packet
#define MAX_TEXT_PAYLOAD    4096      // Max TEXT data
#define BULK_TRANSFER_TIMEOUT_MS  30000UL  // Abort if no activity
#define PACKET_REASSEMBLY_TIMEOUT_MS  5000UL  // Discard incomplete packets
```

### Bulk Transfer Handshake

```
┌─────────────────────────┐
│ App sends bulk header   │
│ (type=GIF, size, CRC)   │
└────────┬────────────────┘
         │
         ↓
┌─────────────────────────┐
│ Device ACKs: 0x01       │
│ (continue sending)      │
└────────┬────────────────┘
         │
         ↓ (repeated for each 4KB chunk)
┌─────────────────────────┐
│ App sends chunk 1..N    │
└────────┬────────────────┘
         │
         ↓
┌─────────────────────────┐
│ Device validates CRC    │
│ and ACKs: 0x01 or 0x03  │
│ (0x03 = transfer done)  │
└─────────────────────────┘
```

---

## 9. GATT Descriptors (Metadata About Characteristics)

**Descriptors** are metadata attributes attached to characteristics. The most important is **CCCD** (Client Characteristic Configuration Descriptor).

### CCCD (0x2902)

Enables the app to **subscribe/unsubscribe** from notifications:

```cpp
// Device side: Make FA03 notifiable
fa03 = fas->createCharacteristic(FA03_UUID,
  BLECharacteristic::PROPERTY_READ | BLECharacteristic::PROPERTY_NOTIFY);
fa03->addDescriptor(new BLE2902());  // Add CCCD descriptor
```

**What happens**:
1. App connects to device
2. App reads CCCD value for FA03 → learns notifications are available
3. App writes 0x01 to CCCD → subscribes to notifications
4. Device calls `fa03->notify()` → data sent to app
5. If app writes 0x00 to CCCD → unsubscribes

Without CCCD, the device has no way to know if the app is listening!

### Common Descriptors

| Descriptor | UUID | Purpose |
|---|---|---|
| **CCCD** | `0x2902` | Enable/disable notifications for this characteristic |
| **CUDD** | `0x2901` | User description (human-readable name) |
| **CPF** | `0x2904` | Characteristic presentation format (data type, units) |

---

## 10. Threading & Synchronization

### The Problem

In ESP32 with FreeRTOS:
- **BLE callbacks** run on **Core 0** (or a dedicated BLE task)
- **Arduino `loop()`** runs on **Core 1**

Both may access shared state simultaneously → **race condition** → data corruption!

### Example Race Condition

```cpp
// Core 0 (BLE callback)
void onWrite(BLECharacteristic *c) {
  brightness = c->getValue()[0];  // Read from BLE
}

// Core 1 (main loop)
void loop() {
  FastLED.setBrightness(brightness);  // Use brightness
  FastLED.show();
}
// What if onWrite fires between read and write?
```

### iDotMatrix Solution: Mutex Lock

```cpp
static SemaphoreHandle_t runtimeStateMutex = nullptr;

bool lockRuntimeState() {
  if (!runtimeStateMutex) return false;
  return xSemaphoreTake(runtimeStateMutex, pdMS_TO_TICKS(50)) == pdTRUE;
}

void unlockRuntimeState() {
  if (runtimeStateMutex) xSemaphoreGive(runtimeStateMutex);
}

// In setup
runtimeStateMutex = xSemaphoreCreateMutex();

// In BLE callback
class FA02Callbacks : public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic *c) override {
    if(!lockRuntimeState()) return;  // ← Acquire lock
    processFA02Write((const uint8_t*)c->getValue().c_str(), ...);
    unlockRuntimeState();  // ← Release lock
  }
};

// In main loop
void loop() {
  if(!lockRuntimeState()) return;
  
  // ... safely access/modify shared state ...
  
  unlockRuntimeState();
  delay(5);
}
```

### Why Mutex, Not SpinLock?

- **SpinLock** (disables interrupts): Unsafe for blocking operations (filesystem, network, etc.)
- **Mutex** (FreeRTOS semaphore): Allows context switching; safe for any operation

iDotMatrix uses a **mutex** because it performs blocking I/O (LittleFS, AnimatedGIF decode).

---

## 11. Common BLE Issues to Watch For

| Issue | Root Cause | Prevention | iDotMatrix Solution |
|-------|-----------|------------|-------------------|
| **Packets lost** | MTU fragmentation, timing | Acknowledge receipts | Bulk transfer CRC32 |
| **Corrupted state** | Race condition (callbacks vs. loop) | Synchronization primitives | Mutex lock on shared state |
| **Connection drops** | Radio interference, timeout, power | Stable connection params | Reconnect advertising |
| **Stalled transfers** | Network congestion, app crash | Inactivity timeout | 30-second bulk timeout |
| **Memory leaks** | Decoder running during transfer | Separate RX/PLAY files | LittleFS isolation |
| **Slow response** | Large MTU negotiation overhead | Request/response batching | Async notifications |
| **Notification miss** | Unreliable delivery | Use INDICATE for critical data | Non-critical updates via NOTIFY |
| **CCCD not set** | App didn't subscribe | Verify CCCD in app | Device checks `deviceConnected` flag |

---

## 12. Protocol Layers in iDotMatrix

```
┌──────────────────────────────────────────────────────┐
│ Layer 7: Application                                 │
│ iDotMatrix Commands                                  │
│ (brightness, GIF, alarm, clock, scoreboard, etc.)    │
└────────┬─────────────────────────────────────────────┘
         │
┌────────┴─────────────────────────────────────────────┐
│ Layer 6: Logical Packet                              │
│ Length-prefixed reassembly                           │
│ Multi-write accumulation (timeout: 5 sec)            │
└────────┬─────────────────────────────────────────────┘
         │
┌────────┴─────────────────────────────────────────────┐
│ Layer 5: Bulk Transfer (Large Data)                  │
│ Multi-packet protocol with CRC32                     │
│ (timeout: 30 sec, chunk: 4096 bytes)                 │
└────────┬─────────────────────────────────────────────┘
         │
┌────────┴─────────────────────────────────────────────┐
│ Layer 4: BLE GATT                                    │
│ Services (FA, AE)                                    │
│ Characteristics (FA02, FA03, AE01, AE02)             │
│ Properties (WRITE, READ, NOTIFY)                     │
│ Descriptors (CCCD)                                   │
└────────┬─────────────────────────────────────────────┘
         │
┌────────┴─────────────────────────────────────────────┐
│ Layer 3: BLE Link Layer                              │
│ Connection management                                │
│ Encryption/authentication (optional)                 │
│ PDU fragmentation (MTU negotiation)                   │
└────────┬─────────────────────────────────────────────┘
         │
┌────────┴─────────────────────────────────────────────┐
│ Layer 2: Radio Physical Layer                        │
│ 2.4 GHz ISM band (same as WiFi)                      │
│ Adaptive frequency hopping                           │
│ Power level control                                  │
└────────┬─────────────────────────────────────────────┘
         │
┌────────┴─────────────────────────────────────────────┐
│ Layer 1: Hardware                                    │
│ ESP32 BLE radio                                      │
│ Antenna, RF frontend                                 │
└──────────────────────────────────────────────────────┘
```

### Data Flow Example: Setting Brightness

```
App (Central)                          Device (Peripheral / ESP32)
     │                                        │
     │ User taps brightness slider (80%)     │
     │                                        │
     │──── WRITE FA02────────────────────────>│
     │   [05 00 04 80 50]                     │
     │   (CMD=04, brightness=80%)             │
     │                                        │
     │                                    onWrite callback
     │                                    (Core 0, BLE task)
     │                                        │
     │                                    lockRuntimeState()
     │                                        │
     │                                    processFA02Write()
     │                                    parseCommand()
     │                                    brightness=80%
     │                                        │
     │                                    unlockRuntimeState()
     │                                        │
     │                              <─── NOTIFY FA03 ───
     │   [05 00 04 80 01]                    │
     │   (ACK: command accepted)             │
     │                                        │
     │              (in main loop, Core 1)  │
     │              lockRuntimeState()       │
     │              FastLED.setBrightness()  │
     │              FastLED.show()           │
     │              (LEDs brighten to 80%)   │
```

---

## Quick Reference: ESP32 BLE APIs Used in iDotMatrix

### Initialization

```cpp
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>

// Initialize BLE with device name
BLEDevice::init(DEVICE_NAME);

// Create server
BLEServer *server = BLEDevice::createServer();
server->setCallbacks(new ServerCallbacks());
```

### Create Service & Characteristics

```cpp
// Create service
BLEService *service = server->createService(SERVICE_UUID);

// Create characteristic with properties
BLECharacteristic *characteristic = service->createCharacteristic(
  CHAR_UUID,
  BLECharacteristic::PROPERTY_WRITE | 
  BLECharacteristic::PROPERTY_WRITE_NR
);

// Add CCCD descriptor for notifications
characteristic->addDescriptor(new BLE2902());

// Set callbacks for writes
characteristic->setCallbacks(new MyCallbacks());

// Start service
service->start();
```

### Notifications (Server → Client)

```cpp
void sendNotification(BLECharacteristic *characteristic, 
                     const uint8_t *data, size_t len) {
  if (!deviceConnected || !characteristic) return;
  
  characteristic->setValue(data, len);
  characteristic->notify();
}
```

### Write Callbacks

```cpp
class MyCallbacks : public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic *c) override {
    String value = c->getValue();
    
    // Process the write
    const uint8_t *data = (const uint8_t*)value.c_str();
    size_t len = value.length();
    
    // ... handle command ...
  }
};
```

### Advertising

```cpp
BLEAdvertising *advertising = BLEDevice::getAdvertising();

BLEAdvertisementData adData;
adData.setFlags(ESP_BLE_ADV_FLAG_GEN_DISC | ESP_BLE_ADV_FLAG_BREDR_NOT_SPT);
adData.setName(DEVICE_NAME);
adData.setCompleteServices(BLEUUID(SERVICE_UUID));

advertising->setAdvertisementData(adData);
advertising->start();
```

### Connection State

```cpp
class ServerCallbacks : public BLEServerCallbacks {
  void onConnect(BLEServer*) override {
    // Device just connected
    deviceConnected = true;
  }
  
  void onDisconnect(BLEServer*) override {
    // Device disconnected
    deviceConnected = false;
    // Schedule advertising restart
    pendingAdvertisingRestart = true;
  }
};
```

### Common Check

```cpp
// Always check if client is connected before notifying
if (!deviceConnected || !characteristic) return;

characteristic->setValue(data, len);
characteristic->notify();
```

---

## Summary

### Key Takeaways

1. **BLE is client-server**: App (central) connects to ESP32 (peripheral)

2. **Services & Characteristics organize data**:
   - Service = folder
   - Characteristic = file with properties (WRITE, READ, NOTIFY)

3. **Three communication patterns**:
   - **WRITE**: App → Device (commands)
   - **NOTIFY**: Device → App (responses, unsolicited)
   - **READ**: App pulls data (rarely used)

4. **Packets are length-prefixed**:
   - First 2 bytes = total packet length (little-endian)
   - Allows reassembly of fragmented data

5. **Large transfers use bulk protocol**:
   - Multi-packet with CRC32 validation
   - Timeout protection (30 seconds)

6. **Threading is critical**:
   - Use mutex to protect shared state
   - BLE callbacks run on different task than main loop

7. **Always check connection state**:
   - `if (!deviceConnected)` before notifying
   - Handle onConnect/onDisconnect properly

### For iDotMatrix Specifically

- Two main services: **FA** (primary) and **AE** (secondary)
- Most data flows via **FA02** (write) and **FA03** (notify)
- Advertising broadcasts device type (16×16, 32×32, 64×64)
- GIFs and large assets use 4KB bulk transfers
- All multi-byte values are **little-endian**
- Use `lockRuntimeState()` / `unlockRuntimeState()` to access shared state safely

---

## Resources

- [Bluetooth SIG Official Specs](https://www.bluetooth.com/specifications/specs/)
- [ESP32 BLE API Reference](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/bluetooth/esp_gatt_defs.html)
- [iDotMatrix ESP32 Emulator GitHub](https://github.com/piggei/IDotMatrix-ESP32-Emulator)
- [iDotMatrix PROTOCOL.md](https://github.com/piggei/IDotMatrix-ESP32-Emulator/blob/main/PROTOCOL.md)

---

**Document Version**: 1.0  
**Last Updated**: 2026-09-12  
**Source**: iDotMatrix ESP32 Emulator reverse-engineering notes and official protocol documentation
