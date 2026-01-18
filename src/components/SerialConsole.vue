<template>
  <div>
    <h2>Web Serial Console</h2>

    <label>
      通信速度:
      <select v-model="baudRate">
        <option value="9600">9600</option>
        <option value="19200">19200</option>
        <option value="38400">38400</option>
        <option value="57600">57600</option>
        <option value="115200">115200</option>
      </select>
    </label>
    <button @click="connect">Connect</button>
    <button @click="disconnect" :disabled="!port">Disconnect</button>

    <div>
      <textarea v-model="received" rows="10" cols="50" readonly></textarea>
    </div>

    <input v-model="sendText" placeholder="Send text" />
    <button @click="send" :disabled="!writer">Send</button>

    <div style="margin-top: 1em;">
      <strong>ログ:</strong>
      <div style="white-space: pre-line; border: 1px solid #ccc; padding: 0.5em; min-height: 2em;">{{ log }}</div>
    </div>

    <div v-if="portInfo" style="margin-top: 1em;">
      <strong>接続中ポート情報:</strong>
      <div style="border: 1px solid #ccc; padding: 0.5em;">
        {{ portInfo }}
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const port = ref(null)
const reader = ref(null)
const writer = ref(null)
const received = ref('')
const sendText = ref('')
const log = ref('')
const baudRate = ref(115200)
const portInfo = ref('')

async function connect() {
  try {
    port.value = await navigator.serial.requestPort()
    await port.value.open({ baudRate: Number(baudRate.value) })
    // ポート情報取得
    if (port.value.getInfo) {
      const info = port.value.getInfo()
      // USBデバイスの場合
      if (info.usbVendorId && info.usbProductId) {
        portInfo.value = `USB Vendor ID: ${info.usbVendorId}, Product ID: ${info.usbProductId}`
      } else {
        portInfo.value = JSON.stringify(info)
      }
    } else {
      portInfo.value = '情報取得不可'
    }

    const textDecoder = new TextDecoderStream()
    const readableStreamClosed = port.value.readable.pipeTo(textDecoder.writable)
    reader.value = textDecoder.readable.getReader()

    const textEncoder = new TextEncoderStream()
    const writableStreamClosed = textEncoder.readable.pipeTo(port.value.writable)
    writer.value = textEncoder.writable.getWriter()

    log.value = '接続に成功しました'
    readLoop()
  } catch (e) {
    log.value = '接続エラー: ' + e
    console.error('Connection error:', e)
  }
}

async function readLoop() {
  while (true) {
    const { value, done } = await reader.value.read()
    if (done) break
    received.value += value
  }
}

async function send() {
  if (!writer.value) return
  await writer.value.write(sendText.value + '\n')
}

async function disconnect() {
  let errorMsg = ''
  if (reader.value) {
    await reader.value.cancel()
    try {
      reader.value.releaseLock()
    } catch (e) {
      errorMsg += '\nreader releaseLock error: ' + e
    }
  }
  if (writer.value) {
    await writer.value.close()
    try {
      writer.value.releaseLock()
    } catch (e) {
      errorMsg += '\nwriter releaseLock error: ' + e
    }
  }
  if (port.value) {
    try {
      await port.value.close()
    } catch (e) {
      errorMsg += '\nport close error: ' + e
    }
  }
  portInfo.value = ''
  log.value = errorMsg ? '切断時エラー:\n' + errorMsg : '切断処理中（ストリーム終了待ち）'
}
</script>

<style>
textarea {
  width: 100%;
}
</style>
