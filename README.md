# ⚡ SKZ V1 Lightstick — BLE Protocol Reverse Engineering & Web Controller

> **Case study:** Reverse engineering the Bluetooth Low Energy protocol of a commercial IoT consumer device (Stray Kids V1 Lightstick) and building a fully functional browser-based controller using the Web Bluetooth API.

<br>

## 📋 Table of Contents

- [Overview](#-overview)
- [Reverse Engineering Methodology](#-reverse-engineering-methodology)
  - [Phase 1 — Static Analysis (APK Decompilation)](#phase-1--static-analysis-apk-decompilation)
  - [Phase 2 — Dynamic Analysis (BLE Connection & Auth Discovery)](#phase-2--dynamic-analysis-ble-connection--auth-discovery)
- [Protocol Specification](#-protocol-specification)
  - [Connection Flow](#connection-flow)
  - [Frame Structure](#frame-structure)
  - [Command Reference](#command-reference)
  - [Color Map](#color-map)
- [Tech Stack](#-tech-stack)
- [Features](#-features)
- [Getting Started](#-getting-started)
- [Deploy](#-deploy)

<br>

## 🔍 Overview

The official Stray Kids V1 lightstick communicates over **Bluetooth Low Energy (BLE)** using a proprietary GATT-based protocol. The official companion app controls color, effects, and synchronization during live concerts — but is limited to Android/iOS.

This project documents the **full reverse engineering process** used to decode that protocol and implements an **open, cross-platform PWA controller** that runs entirely in the browser via the Web Bluetooth API — no native app, no SDK, no dependencies.

**Target Device:** Stray Kids V1 Official Lightstick  
**BLE Service UUID:** `87011111-ffcc-2222-0000-000000008888`  
**BLE Characteristic UUID:** `000092a4-0000-1000-8000-00805f9b34fb`

<br>

## 🔬 Reverse Engineering Methodology

### Phase 1 — Static Analysis (APK Decompilation)

The primary source of protocol knowledge came from decompiling the official companion app. This is where the actual command bytes and color mappings were discovered.

**Tools:** [JADX](https://github.com/skylot/jadx) (DEX → Java decompiler)

**Process:**
1. Obtained the official APK from a mirrored source
2. Decompiled using JADX to recover Java source code
3. Searched for GATT-related classes (`BluetoothGattCallback`, `BluetoothGattCharacteristic`)
4. Identified the custom Service UUID (`87011111-ffcc-2222-0000-000000008888`) and Characteristic UUID (`000092a4-...`) defined as string constants
5. Found **hardcoded color IDs** in the source — the app maps each color to a single-byte constant (`0x01` = Red, `0x03` = Yellow, `0x0A` = Green, `0x14` = Blue)
6. Extracted the frame structure and command type bytes for color control (`0xA5`), power off (`0xA2`), and auth (`0xAD`)

**Key findings from static analysis:**
- Color IDs were **hardcoded constants** in the decompiled source — not derived from any dynamic discovery
- The protocol uses a fixed frame format with `0xFF` as start/end delimiters
- Color commands use a single-byte ID, not RGB values
- The auth command structure was visible, but the actual handshake behavior required live testing to understand

### Phase 2 — Dynamic Analysis (BLE Connection & Auth Discovery)

The decompiled APK revealed the command structure and color mappings, but the **authentication handshake** could only be understood through direct interaction with the device. This is where the actual challenge-response mechanism was figured out.

**Tools:** [nRF Connect](https://www.nordicsemi.com/Products/Development-tools/nrf-connect-for-mobile) (Nordic Semiconductor BLE utility)

**Process:**
1. Connected to the lightstick via nRF Connect
2. Browsed GATT services and located the target service/characteristic using the UUIDs found in Phase 1
3. Subscribed to notifications on the characteristic
4. Manually wrote the Auth Request bytes (`FF AD 03 01 00 00 FF`) identified from the decompiled code
5. Observed the **notification response** — the lightstick replied with a frame containing two challenge bytes (`P1`, `P2`)
6. Wrote the Auth Confirm frame echoing those bytes back (`FF AD 03 02 P1 P2 FF`) — device authenticated successfully
7. After auth, tested the hardcoded color commands from Phase 1 to confirm they worked as expected
8. Tested edge cases: rapid writes, invalid color IDs, writes without auth, reconnection behavior

**Key findings from dynamic analysis:**
- The lightstick sends a **notification** with two challenge bytes (`P1`, `P2`) in response to the Auth Request — this was not obvious from the decompiled code alone
- These bytes must be echoed back in the Auth Confirm frame — this is the handshake
- After authentication, the device accepts direct color/effect commands with no further negotiation
- The device tolerates reconnections without reset after a failed auth
- Invalid color IDs are silently ignored (no error notification)
- Rapid sequential writes can cause GATT errors — a write queue is necessary

<br>

## 📡 Protocol Specification

### Connection Flow

```
┌──────────┐                              ┌────────────┐
│  Client   │                              │ Lightstick  │
│ (Browser) │                              │   (BLE)     │
└─────┬─────┘                              └──────┬──────┘
      │                                           │
      │  1. GATT Connect                          │
      ├──────────────────────────────────────────►│
      │                                           │
      │  2. Discover Service                      │
      │     87011111-ffcc-2222-0000-000000008888  │
      ├──────────────────────────────────────────►│
      │                                           │
      │  3. Subscribe to Notifications            │
      │     on 000092a4-...                       │
      ├──────────────────────────────────────────►│
      │                                           │
      │  4. Write Auth Request                    │
      │     FF AD 03 01 00 00 FF                  │
      ├──────────────────────────────────────────►│
      │                                           │
      │  5. Notification (Challenge)              │
      │     FF AD xx 01 xx P1 P2 xx               │
      │◄──────────────────────────────────────────│
      │                                           │
      │  6. Write Auth Confirm                    │
      │     FF AD 03 02 P1 P2 FF                  │
      ├──────────────────────────────────────────►│
      │                                           │
      │  ✓ Authenticated — ready for commands     │
      │                                           │
      │  7. Write Color / Effect commands         │
      ├──────────────────────────────────────────►│
      │                                           │
```

### Frame Structure

All frames follow a fixed structure with `0xFF` delimiters:

```
┌───────┬──────────┬────────┬──────────────────┬───────┐
│ START │ CMD_TYPE │  LEN   │     PAYLOAD      │  END  │
│ 0xFF  │  1 byte  │ 1 byte │   N bytes        │ 0xFF  │
└───────┴──────────┴────────┴──────────────────┴───────┘
```

| Field | Size | Description |
|-------|------|-------------|
| `START` | 1 byte | Frame start delimiter — always `0xFF` |
| `CMD_TYPE` | 1 byte | Command type identifier (`0xAD` = auth, `0xA5` = color, `0xA2` = off) |
| `LEN` | 1 byte | Payload context/length indicator |
| `PAYLOAD` | Variable | Command-specific data |
| `END` | 1 byte | Frame end delimiter — always `0xFF` |

### Command Reference

| Command | Frame (HEX) | Description |
|---------|-------------|-------------|
| **Auth Request** | `FF AD 03 01 00 00 FF` | Initiates the authentication handshake. Lightstick responds with a notification containing challenge bytes. |
| **Auth Confirm** | `FF AD 03 02 P1 P2 FF` | Completes the handshake. `P1` and `P2` are the challenge bytes received from the notification. |
| **Set Color** | `FF A5 02 CC 02 FF` | Sets a solid color. `CC` is the color ID byte (see table below). Second `02` controls brightness/mode. |
| **Turn Off** | `FF A2 00 FF` | Turns off all LEDs. Shorter frame (4 bytes). |

### Color Map

| Byte (`CC`) | Color | Notes |
|-------------|-------|-------|
| `0x00` | Off | Same effect as Turn Off command |
| `0x01` | 🔴 Red | — |
| `0x03` | 🟡 Yellow | — |
| `0x0A` | 🟢 Green | — |
| `0x14` | 🔵 Blue | — |

> **Note:** Color IDs are not sequential. These were the only IDs confirmed through testing. Additional IDs may exist but were not observed in the decompiled app or traffic captures.

<br>

## 🛠 Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **BLE Communication** | [Web Bluetooth API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Bluetooth_API) | `navigator.bluetooth.requestDevice()` for device discovery, GATT service/characteristic access, notifications |
| **Offline Support** | Progressive Web App (Service Worker) | Cache-first strategy for offline usage at concerts/venues with no connectivity |
| **UI** | Vanilla HTML + CSS | Zero-dependency interface with CSS animations and responsive design |
| **Architecture** | Single-file SPA | Entire app in one `index.html` — no build step, no bundler, instant deploy |

**Why these choices?**
- **Web Bluetooth** enables cross-platform BLE access without native app distribution (no Play Store / App Store needed)
- **PWA** allows fans to install the app once and use it offline at concert venues
- **Zero dependencies** means no `node_modules`, no build pipeline — just deploy static files

<br>

## ✨ Features

| Feature | Description |
|---------|-------------|
| **BLE Auth Handshake** | Automatic challenge-response authentication matching the official app's protocol |
| **Solid Colors** | Red, Yellow, Green, Blue via mapped color ID bytes |
| **Strobe Effects** | Timed flash for any color using interval-based write cycling |
| **Random Flash** | Random color selection from the pool on each strobe tick |
| **Rainbow Mode** | Sequential color cycling through all known IDs |
| **Stray Kids Mode** | Member-themed color presets |
| **Write Queue** | Sequential BLE write queue preventing GATT operation collisions |
| **Installable PWA** | Add to Home Screen on mobile for a native-like experience |

<br>

## 🚀 Getting Started

### Requirements

- Chromium-based browser with Web Bluetooth support (Chrome, Edge, Opera)
- Bluetooth enabled on your device
- Stray Kids V1 Lightstick in pairing mode

### Usage

1. Open `index.html` in a compatible browser (or access the deployed URL)
2. Tap **Conectar Lightstick**
3. Select your lightstick from the browser's Bluetooth device picker
4. Wait for the authentication handshake and sync to complete
5. Use the controls to change colors and effects

<br>

## 📦 Deploy

This is a static PWA — no build step required. Serve the files with any HTTP server:

```bash
npx serve .
```

Or deploy directly to **GitHub Pages**, **Netlify**, **Vercel**, or any static hosting.

<br>

## 📄 License

This project is intended for **educational and research purposes only**. It documents the reverse engineering of a BLE protocol for interoperability and personal use. All trademarks belong to their respective owners.

