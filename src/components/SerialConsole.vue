<template>
  <div>
    <h2>Web Serial Console</h2>

    <button @click="connect">Connect</button>
    <button @click="disconnect" :disabled="!port">Disconnect</button>

    <div>
      <textarea v-model="received" rows="10" cols="50" readonly></textarea>
    </div>

    <input v-model="sendText" placeholder="Send text" />
    <button @click="send" :disabled="!writer">Send</button>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const port = ref(null)
const reader = ref(null)
const writer = ref(null)
const received = ref('')
const sendText = ref('')

async function connect() {
  try {
    port.value = await navigator.serial.requestPort()
    await port.value.open({ baudRate: 115200 })

    const textDecoder = new TextDecoderStream()
    const readableStreamClosed = port.value.readable.pipeTo(textDecoder.writable)
    reader.value = textDecoder.readable.getReader()

    const textEncoder = new TextEncoderStream()
    const writableStreamClosed = textEncoder.readable.pipeTo(port.value.writable)
    writer.value = textEncoder.writable.getWriter()

    readLoop()
  } catch (e) {
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
  try {
    if (reader.value) {
      await reader.value.cancel()
      reader.value.releaseLock()
    }
    if (writer.value) {
      await writer.value.close()
      writer.value.releaseLock()
    }
    if (port.value) {
      await port.value.close()
    }
  } catch (e) {
    console.error('Disconnect error:', e)
  }
}
</script>

<style>
textarea {
  width: 100%;
}
</style>
