# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```sh
npm install        # install dependencies
npm run dev         # start Vite dev server with HMR
npm run build        # type-check-free production build to dist/
npm run preview       # serve the production build locally
```

There is no test suite, linter, or type checker configured in this project (plain JS, no ESLint/Vitest config present). Do not invent commands for these — if you add tests or linting, add the corresponding npm script and document it here.

## Architecture

This is a **single-page Vue 3 + Vite app** that is essentially a thin UI wrapper around one third-party class: `RNodeController` from [`@mastashake08/rnode-web`](https://github.com/mastashake08/rnode-web) (installed in `node_modules/@mastashake08/rnode-web`, source is a single `index.js`).

- **Almost all app logic lives in [src/App.vue](src/App.vue)** as one component — there is no router, store, or component tree. `main.js` just mounts `App.vue`.
- `RNodeController` extends `EventTarget` and is instantiated once at module scope in `App.vue`'s `<script setup>`. All UI state (`connectionType`, `keyReady`, log entries, etc.) is driven by listening to its events, not by inspecting its internal state directly.

### RNodeController transport model

The controller talks to physical RNode hardware over one of two transports, chosen at connect time and mutually exclusive per instance:
- **Web Serial** (`connectSerial(baudRate, flowControl)`) — USB, KISS frames written directly to the serial port.
- **Web Bluetooth** (`connectBLE(options)`) — Nordic UART Service (NUS) GATT characteristics, KISS frames chunked into 20-byte MTU-safe writes.

Both transports feed the same internal KISS frame parser/encoder (`_parseIncomingBytes` / `_writeKissFrame`), so app code never needs to know which transport is active — it just calls the same public methods (`sendPlaintext`, `setFrequency`, etc.) and the controller routes bytes to whichever transport is connected.

**Constraint:** Web Serial/Web Bluetooth only work in Chromium-based browsers, only over a secure context (`https://` or `localhost`), and connection must be initiated from a direct user gesture (e.g. a button click) — this is why `connectSerial`/`connectBLE` in `App.vue` are wired straight to button `@click` handlers rather than called programmatically.

### Event-driven data flow

`App.vue` never polls the controller; it registers listeners once and reacts to `CustomEvent`s dispatched by the controller as KISS frames arrive:

| Event | Meaning |
|---|---|
| `connected` / `disconnected` | Transport-level connection state |
| `key-ready` | AES-GCM key derived after `setPreSharedKey()` |
| `plaintext` / `decrypted-text` | Incoming data frame, decoded (unencrypted vs. successfully decrypted) |
| `decryption-failed` | Incoming data frame failed AES-GCM decryption |
| `config-frequency` / `config-bandwidth` / `config-sf` / `config-cr` / `config-power` | Radio echoed back a confirmed tuning parameter |
| `error` | Transport or hardware-level error |

All of these are appended to the on-screen `log` array in `App.vue` rather than using `console.log` — when debugging behavior, check the Log panel's event kind (`status`/`error`/`rx`/`tx`/`config`) rather than the browser console.

### Encryption

Encryption is opt-in and per-message: `setPreSharedKey(passphrase)` derives an AES-GCM key (SHA-256 of the passphrase, or a raw 32-byte key) via Web Crypto. Once set, `sendEncryptedText()` prepends a random 12-byte IV to each ciphertext; `sendPlaintext()`/`sendRaw()` bypass encryption entirely regardless of whether a key is set. Encrypted send is only exposed in the UI once `key-ready` has fired.

### Tuning parameters

Tuning setters (`setFrequency`, `setBandwidth`, `setSpreadingFactor`, `setCodingRate`, `setTxPower`) all write KISS command frames to the radio; they do not return a value. Confirmation of an applied setting only arrives asynchronously via the corresponding `config-*` event echoed back from hardware — don't assume a tuning call succeeded just because the `await` resolved (that only means the frame was written to the transport).

## Deployment

`.github/workflows/deploy.yml` builds and deploys `dist/` to GitHub Pages on every push to `main` (or manual dispatch), using `actions/upload-pages-artifact` + `actions/deploy-pages`. `vite.config.js` sets `base: './'` (relative paths) specifically so the built app works when served from a GitHub Pages project subpath without needing the repo name hardcoded.
