# RNode Console

A simple browser-based control panel for tuning and communicating with [RNode](https://github.com/markqvist/RNode_Firmware) LoRa hardware, built with Vue 3 + Vite on top of the [`@mastashake08/rnode-web`](https://github.com/mastashake08/rnode-web) package.

It talks to the radio directly from the browser over Web Serial (USB) or Web Bluetooth (BLE) using the KISS/TNC protocol — no native drivers, Python, or backend server required.

## Features

- **Connect** to an RNode over USB (Web Serial) or BLE (Web Bluetooth, Nordic UART Service)
- **Tune** the radio: frequency, bandwidth, spreading factor, coding rate, and TX power
- **Encrypt** messages with a pre-shared passphrase (AES-GCM via Web Crypto)
- **Send** plaintext or encrypted text messages over the air
- **Live log** of connection state, sent/received messages, radio config confirmations, and errors

## Requirements

- A Chromium-based browser (Chrome, Edge, Brave, etc.) — Web Serial and Web Bluetooth are not supported in Firefox or Safari
- Served over `https://` or `localhost` (both Web Serial and Web Bluetooth require a secure context)
- An RNode-compatible device connected via USB or paired over BLE

## Project Setup

```sh
npm install
```

### Compile and Hot-Reload for Development

```sh
npm run dev
```

### Compile and Minify for Production

```sh
npm run build
```

### Preview the Production Build Locally

```sh
npm run preview
```

## Usage

1. Click **Connect via USB (Serial)** or **Connect via Bluetooth** and select your RNode device from the browser prompt (this must be triggered by a user click, not run automatically).
2. Adjust the tuning parameters and click **Apply Tuning** to push them to the radio.
3. Optionally set a pre-shared passphrase under **Encryption** to enable encrypted messaging.
4. Type a message under **Send Message**, toggle **Encrypt** if desired, and click **Send**.
5. Watch the **Log** panel for connection events, transmitted/received messages, and any errors.

## Deployment

Pushes to `main` automatically build and deploy this app to GitHub Pages via [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml).

To enable it on a new repository:

1. In the repo settings, go to **Pages** and set the source to **GitHub Actions**.
2. Push to `main` (or run the workflow manually via **Actions → Deploy to GitHub Pages → Run workflow**).
3. The site will be published at `https://<user>.github.io/<repo>/`.

The Vite config uses a relative `base: './'` so the build works correctly regardless of the repository name or subpath it's served from.

## Recommended IDE Setup

[VS Code](https://code.visualstudio.com/) + [Vue (Official)](https://marketplace.visualstudio.com/items?itemName=Vue.volar) (and disable Vetur).
