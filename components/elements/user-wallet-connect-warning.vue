<template>
  <div class="flex h-full w-full items-center justify-center text-center">
    <div v-if="cta" class="text-center py-4">
      <p class="text-xs text-gray-300 w-full">
        {{ $t('please_connect_to_your_wallet_cta') }}
      </p>
      <v-button md primary class="mt-2" @click.stop="handleClickOnConnect">
        {{ $t('connect') }}
      </v-button>
    </div>
    <p v-else class="text-xs text-gray-400 py-4">
      {{ $t('please_connect_to_your_wallet') }}
    </p>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'

export default Vue.extend({
  props: {
    cta: {
      required: false,
      type: Boolean,
      default: false
    }
  },

  methods: {
    handleClickOnConnect() {
      this.$root.$emit('wallet-clicked')
    }
  }
})
</script>
