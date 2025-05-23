<script setup>
import HelloWorld from './components/HelloWorld.vue'
import { reactive } from 'vue'

import AES from 'crypto-js/aes';
import { enc } from 'crypto-js';

const message = 'Hello, world!';
const privateKey = '1234567890';

// Encrypt
const encrypted = AES.encrypt(message, privateKey).toString();

// Decrypt
const decrypted = AES.decrypt(encrypted, privateKey).toString(enc.Utf8);


const createNode = ({
  name,
  to,
  toSecret, 
  fromSecret
} = {}) => {

  const sendMessage = (message) => {
    const encrypted = AES.encrypt(message, toSecret).toString();
    return {
      to,
      message: encrypted
    }
  }

  const receiveMessage = (message) => {
    const decrypted = AES.decrypt(message, fromSecret).toString(enc.Utf8);
    return decrypted;
  }

  return {
    name,
    to,
    toSecret, 
    fromSecret,
    sendMessage,
    receiveMessage
  }
}

const step = reactive({
  encrypted: encrypted,
  decrypted: decrypted
})
</script>

<template>
  <div>
    <a href="https://vite.dev" target="_blank">
      <img src="/vite.svg" class="logo" alt="Vite logo" />
    </a>
    <a href="https://vuejs.org/" target="_blank">
      <img src="./assets/vue.svg" class="logo vue" alt="Vue logo" />
    </a>
  </div>
  <HelloWorld msg="Vite + Vue" />
  {{ step }}
</template>

<style scoped>
.logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
  transition: filter 300ms;
}
.logo:hover {
  filter: drop-shadow(0 0 2em #646cffaa);
}
.logo.vue:hover {
  filter: drop-shadow(0 0 2em #42b883aa);
}
</style>
