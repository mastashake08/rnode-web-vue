<script setup>
import { ref, computed, onBeforeUnmount } from 'vue'
import { RNodeController } from '@mastashake08/rnode-web'

const rnode = new RNodeController()

const connectionType = ref(null) // 'serial' | 'ble' | null
const isConnected = computed(() => connectionType.value !== null)
const keyReady = ref(false)
const baudRate = ref(115200)

const frequencyMHz = ref(915.0)
const bandwidthKHz = ref(125)
const spreadingFactor = ref(7)
const codingRate = ref(5)
const txPower = ref(17)

const passphrase = ref('')
const outgoingText = ref('')
const encryptOutgoing = ref(false)

const log = ref([])
function addLog(kind, message) {
  log.value.unshift({ id: Date.now() + Math.random(), time: new Date().toLocaleTimeString(), kind, message })
  if (log.value.length > 200) log.value.pop()
}

rnode.addEventListener('connected', (e) => {
  connectionType.value = e.detail.type
  addLog('status', `Connected via ${e.detail.type}${e.detail.name ? ' (' + e.detail.name + ')' : ''}`)
})
rnode.addEventListener('disconnected', (e) => {
  connectionType.value = null
  addLog('status', `Disconnected${e.detail.unexpected ? ' unexpectedly' : ''}`)
})
rnode.addEventListener('error', (e) => {
  addLog('error', e.detail?.message || String(e.detail))
})
rnode.addEventListener('key-ready', () => {
  keyReady.value = true
  addLog('status', 'Pre-shared key set')
})
rnode.addEventListener('plaintext', (e) => addLog('rx', `Plaintext: ${e.detail.text}`))
rnode.addEventListener('decrypted-text', (e) => addLog('rx', `Decrypted: ${e.detail.text}`))
rnode.addEventListener('decryption-failed', () => addLog('error', 'Failed to decrypt an incoming packet'))
rnode.addEventListener('config-frequency', (e) => addLog('config', `Radio confirmed frequency: ${(e.detail.hz / 1e6).toFixed(3)} MHz`))
rnode.addEventListener('config-bandwidth', (e) => addLog('config', `Radio confirmed bandwidth: ${(e.detail.hz / 1e3).toFixed(1)} kHz`))
rnode.addEventListener('config-sf', (e) => addLog('config', `Radio confirmed spreading factor: ${e.detail.sf}`))
rnode.addEventListener('config-cr', (e) => addLog('config', `Radio confirmed coding rate: ${e.detail.cr}`))
rnode.addEventListener('config-power', (e) => addLog('config', `Radio confirmed TX power: ${e.detail.dbm} dBm`))

async function connectSerial() {
  try {
    await rnode.connectSerial(Number(baudRate.value))
  } catch (err) {
    addLog('error', err.message)
  }
}

async function connectBLE() {
  try {
    await rnode.connectBLE()
  } catch (err) {
    addLog('error', err.message)
  }
}

async function disconnect() {
  try {
    await rnode.disconnect()
  } catch (err) {
    addLog('error', err.message)
  }
}

async function applyTuning() {
  try {
    await rnode.setFrequency(Math.round(Number(frequencyMHz.value) * 1e6))
    await rnode.setBandwidth(Math.round(Number(bandwidthKHz.value) * 1e3))
    await rnode.setSpreadingFactor(Number(spreadingFactor.value))
    await rnode.setCodingRate(Number(codingRate.value))
    await rnode.setTxPower(Number(txPower.value))
    addLog('tx', 'Sent tuning parameters to radio')
  } catch (err) {
    addLog('error', err.message)
  }
}

async function setKey() {
  if (!passphrase.value) return
  try {
    await rnode.setPreSharedKey(passphrase.value)
  } catch (err) {
    addLog('error', err.message)
  }
}

async function sendMessage() {
  if (!outgoingText.value) return
  try {
    if (encryptOutgoing.value) {
      await rnode.sendEncryptedText(outgoingText.value)
      addLog('tx', `Sent encrypted: ${outgoingText.value}`)
    } else {
      await rnode.sendPlaintext(outgoingText.value)
      addLog('tx', `Sent plaintext: ${outgoingText.value}`)
    }
    outgoingText.value = ''
  } catch (err) {
    addLog('error', err.message)
  }
}

onBeforeUnmount(() => {
  if (isConnected.value) rnode.disconnect()
})
</script>

<template>
  <div class="app">
    <header>
      <h1>RNode Console</h1>
      <span class="status" :class="{ on: isConnected }">
        {{ isConnected ? `Connected (${connectionType})` : 'Disconnected' }}
      </span>
    </header>

    <section class="panel">
      <h2>Connection</h2>
      <div class="row">
        <label>Baud rate
          <input type="number" v-model="baudRate" :disabled="isConnected" />
        </label>
        <button @click="connectSerial" :disabled="isConnected">Connect via USB (Serial)</button>
        <button @click="connectBLE" :disabled="isConnected">Connect via Bluetooth</button>
        <button @click="disconnect" :disabled="!isConnected" class="danger">Disconnect</button>
      </div>
    </section>

    <section class="panel">
      <h2>Tune Radio</h2>
      <div class="grid">
        <label>Frequency (MHz)
          <input type="number" step="0.001" v-model="frequencyMHz" />
        </label>
        <label>Bandwidth (kHz)
          <input type="number" step="0.1" v-model="bandwidthKHz" />
        </label>
        <label>Spreading Factor
          <input type="number" min="6" max="12" v-model="spreadingFactor" />
        </label>
        <label>Coding Rate (denominator)
          <input type="number" min="5" max="8" v-model="codingRate" />
        </label>
        <label>TX Power (dBm)
          <input type="number" min="0" max="22" v-model="txPower" />
        </label>
      </div>
      <button @click="applyTuning" :disabled="!isConnected">Apply Tuning</button>
    </section>

    <section class="panel">
      <h2>Encryption</h2>
      <div class="row">
        <input type="password" v-model="passphrase" placeholder="Pre-shared passphrase" />
        <button @click="setKey" :disabled="!passphrase">Set Key</button>
        <span class="status" :class="{ on: keyReady }">{{ keyReady ? 'Key ready' : 'No key set' }}</span>
      </div>
    </section>

    <section class="panel">
      <h2>Send Message</h2>
      <div class="row">
        <input
          type="text"
          v-model="outgoingText"
          placeholder="Message text"
          @keyup.enter="sendMessage"
          :disabled="!isConnected"
        />
        <label class="checkbox">
          <input type="checkbox" v-model="encryptOutgoing" :disabled="!keyReady" />
          Encrypt
        </label>
        <button @click="sendMessage" :disabled="!isConnected || !outgoingText">Send</button>
      </div>
    </section>

    <section class="panel log-panel">
      <h2>Log</h2>
      <ul class="log">
        <li v-for="entry in log" :key="entry.id" :class="entry.kind">
          <span class="time">{{ entry.time }}</span>
          <span class="message">{{ entry.message }}</span>
        </li>
      </ul>
    </section>
  </div>
</template>

<style scoped>
.app {
  max-width: 720px;
  margin: 0 auto;
  padding: 1.5rem;
  font-family: system-ui, sans-serif;
  color: #1a1a1a;
}
header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 1rem;
}
h1 {
  font-size: 1.4rem;
  margin: 0;
}
.status {
  padding: 0.2rem 0.6rem;
  border-radius: 999px;
  background: #e0e0e0;
  font-size: 0.85rem;
}
.status.on {
  background: #c8f0c8;
  color: #14532d;
}
.panel {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 1rem;
  margin-bottom: 1rem;
}
.panel h2 {
  font-size: 1rem;
  margin: 0 0 0.75rem 0;
}
.row {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 0.5rem;
}
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
  gap: 0.75rem;
  margin-bottom: 0.75rem;
}
label {
  display: flex;
  flex-direction: column;
  font-size: 0.8rem;
  gap: 0.25rem;
}
.checkbox {
  flex-direction: row;
  align-items: center;
  gap: 0.35rem;
}
input[type="text"],
input[type="password"],
input[type="number"] {
  padding: 0.4rem;
  border: 1px solid #ccc;
  border-radius: 4px;
}
button {
  padding: 0.45rem 0.9rem;
  border: none;
  border-radius: 4px;
  background: #2563eb;
  color: white;
  cursor: pointer;
}
button:disabled {
  background: #a0aec0;
  cursor: not-allowed;
}
button.danger {
  background: #dc2626;
}
.log-panel {
  max-height: 260px;
  overflow-y: auto;
}
.log {
  list-style: none;
  margin: 0;
  padding: 0;
  font-family: ui-monospace, monospace;
  font-size: 0.8rem;
}
.log li {
  display: flex;
  gap: 0.5rem;
  padding: 0.2rem 0;
  border-bottom: 1px solid #eee;
}
.log li.error { color: #dc2626; }
.log li.rx { color: #0f766e; }
.log li.tx { color: #1d4ed8; }
.log li.config { color: #92400e; }
.time {
  color: #888;
  flex-shrink: 0;
}
</style>
