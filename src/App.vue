<script setup>
import { ref, computed, onBeforeMount, markRaw } from 'vue'
import { ethers } from 'ethers'
import { find, forEach, set, map, every } from 'lodash'
import Anima from './assets/contract/Anima.json'

// Get constructor inputs
const constructor = find(Anima.abi, ['type', 'constructor'])

// Define refs
const provider = ref(null)
const signer = ref(null)
const address = ref(null)
const form = ref(false)
const allowSubmit = computed(() => {
  if (!form.value) {
    return false
  } else {
    const values = map(form.value, 'value')
    return every(values, Boolean)
  }
})

// Connect wallet
const connectWallet = async () => {
  if (!window.ethereum) {
    window.alert("You don't have MetaMask installed on your browser!")
    return false
  }

  provider.value = markRaw(new ethers.providers.Web3Provider(window.ethereum))
  await provider.value.send("eth_requestAccounts", [])
  signer.value = provider.value.getSigner()
  address.value = await signer.value.getAddress()
  subscribe()
}

// Subscribe to chain events
const subscribe = () => {
  const _provider = provider.value.provider

  _provider.on('accountsChanged', (accounts) => {
    if (accounts.length)
      address.value = accounts[0]
    else 
      disconnect()
  })

  _provider.on('chainChanged', () => {
    window.location.reload()
  })

  _provider.on('disconnect', (code, reason) => {
    console.log(code, reason)
    disconnect()
  })
}

// Disconnect wallet
const disconnect = () => {
  signer.value = null
  address.value = null
}

// Get short address
const shortAddress = (address) => {
  const start = address.substring(0, 6)
  const end = address.substring(address.length - 4, address.length)
  return `${start}...${end}`
}

// Deploy smart contract
const deployContract = async () => {
  // Get form values
  const values = map(form.value, 'value')

  // Deploy the contract
  const factory = new ethers.ContractFactory(Anima.abi, Anima.bytecode.object, signer.value)
  const contract = await factory.deploy(...values)
  await contract.deployed()
  console.log(`Deployment successful! Contract Address: ${contract.address}`)
}

// Populate form inputs
onBeforeMount(() => {
  if (constructor) {
    let inputs = {}
    forEach(constructor.inputs, (o) => {
      set(inputs, o.name, {
        value: null,
        type: o.type
      })
    })
    form.value = inputs
  }
})
</script>

<template>
  <div class="w-full h-screen flex items-center justify-center bg-gray-800">
    <div class="max-w-full w-80 rounded-md border p-4">
      <div v-if="!address">
        <button
          type="button"
          class="w-full py-2 px-3 rounded text-black bg-white
          hover:bg-blue-500 hover:text-white"
          @click="connectWallet"
        >
          Connect Wallet
        </button>
      </div>
      <form v-else-if="form" @submit.prevent="deployContract">
        <div class="mb-3 text-center">
          Connected: {{ shortAddress(address) }}
        </div>
        <div v-for="input, key in form" :key="key" class="mb-3">
          <input type="text" v-model="form[key].value" class="form-input" :placeholder="key">
        </div>
        <div>
          <button
            type="submit"
            class="w-full py-2 px-3 rounded text-black bg-white
            hover:bg-blue-500 hover:text-white disabled:pointer-events-none"
            :disabled="!allowSubmit"
          >
            Deploy Contract
          </button>
        </div>
      </form>
    </div>
  </div>
</template>

<style>
@import './assets/scss/app.scss';
</style>
