<template>
  <div v-if="market" class="h-8 w-full">
    <table class="table">
      <thead xs>
        <tr>
          <th class="w-1/3 text-right">
            <span>{{ $t('price') }}</span>
            <span
              class="font-semibold text-white uppercase inline-block lg:hidden xl:inline-block"
            >
              {{ market.quoteToken.symbol }}
            </span>
          </th>
          <th class="w-1/3 text-right">
            <span>{{ $t('amount') }}</span>
            <span
              class="font-semibold text-white uppercase inline-block lg:hidden xl:inline-block"
            >
              {{ market.baseToken.symbol }}
            </span>
          </th>
          <th class="w-1/3 text-right">
            <span>{{ $t('time') }}</span>
          </th>
        </tr>
      </thead>
    </table>
  </div>
</template>

<script lang="ts">
import Vue, { PropType } from 'vue'
import { UiDerivativeMarket, UiSpotMarket } from '~/types'

export default Vue.extend({
  props: {
    market: {
      required: true,
      type: Object as PropType<UiDerivativeMarket | UiSpotMarket>
    }
  }
})
</script>
