<template>
  <div class="flex flex-col">
    <div
      v-if="title || $slots['title'] || $slots['title-context']"
      class="flex justify-between items-center mb-6"
    >
      <h3 class="text-xl font-bold text-gray-200">
        <slot name="title">{{ title }}</slot>
      </h3>
      <slot name="title-context" />
    </div>
    <slot name="context" class="flex justify-between items-center w-full" />
    <v-card class="relative flex-1">
      <div class="v-panel-content">
        <slot ref="content" />
      </div>
    </v-card>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'

export default Vue.extend({
  props: {
    title: {
      required: false,
      default: '',
      type: String
    },

    portalName: {
      required: false,
      default: '',
      type: String
    }
  }
})
</script>
