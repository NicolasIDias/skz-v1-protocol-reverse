# ⚡ SKZ V1 Lightstick — Protocol Reverse & Controller

Web-based BLE controller for the **Stray Kids V1 Lightstick**, built using the Web Bluetooth API.

> This project reverse-engineers the BLE protocol used by the official SKZ V1 lightstick and provides a fully functional PWA controller that runs directly in the browser.

## Features

- **BLE Connection & Authentication** — Automatic PIN handshake with the lightstick
- **Solid Colors** — Red, Yellow, Green, Blue
- **Flash Effects** — Strobe any color with adjustable timing
- **Random Flash** — Random color strobe effect
- **Rainbow Mode** — Cycle through all colors
- **Stray Kids Mode** — Member-themed color presets
- **PWA Support** — Installable as a standalone app on mobile

## Requirements

- A Chromium-based browser with Web Bluetooth support (Chrome, Edge, Opera)
- Bluetooth enabled on your device
- A Stray Kids V1 Lightstick in pairing mode

## Usage

1. Open `index.html` in a compatible browser (or deploy it)
2. Tap **Conectar Lightstick**
3. Select your lightstick from the Bluetooth device picker
4. Wait for the synchronization to complete
5. Control your lightstick!

## Protocol Overview

| Command | Bytes | Description |
|---------|-------|-------------|
| Auth Request | `FF AD 03 01 00 00 FF` | Request PIN from lightstick |
| Auth Confirm | `FF AD 03 02 P1 P2 FF` | Confirm PIN (P1, P2 from device) |
| Set Color | `FF A5 02 CC 02 FF` | Set color (CC = color ID) |
| Turn Off | `FF A2 00 FF` | Turn off all LEDs |

### Known Color IDs

| ID | Color |
|----|-------|
| `0x01` | Red |
| `0x03` | Yellow |
| `0x0A` | Green |
| `0x14` | Blue |

## Deploy

This is a static PWA — just serve the files with any HTTP server:

```bash
npx serve .
```

Or deploy to GitHub Pages, Netlify, Vercel, etc.

