<template>
  <div
    v-if="summary && market"
    class="p-4 bg-gray-700 rounded-xl flex flex-col flex-wrap text-xs min-w-2xs"
  >
    <div class="flex justify-between items-center mb-2">
      <span class="font-bold">{{ $t('average_price') }}:</span>
      <span>≈ {{ averagePrice }}</span>
    </div>
    <div class="flex justify-between items-center mb-2">
      <span class="font-bold">
        {{ $t('total_volume_in_quote', { symbol: market.baseToken.symbol }) }}:
      </span>
      <span>
        <v-number
          :decimals="
            market
              ? market.quantityDecimals
              : UI_DEFAULT_AMOUNT_DISPLAY_DECIMALS
          "
          :number="totalVolume"
          dont-group-values
        />
      </span>
    </div>
    <div class="flex justify-between items-center">
      <span class="font-bold">
        {{ $t('total_volume_in_base', { symbol: market.quoteToken.symbol }) }}:
      </span>
      <span class="flex items-center">
        <v-number
          :decimals="
            market ? market.priceDecimals : UI_DEFAULT_PRICE_DISPLAY_DECIMALS
          "
          :number="totalPrice"
          dont-group-values
        />
      </span>
    </div>
  </div>
</template>
<script lang="ts">
import Vue, { PropType } from 'vue'
import { BigNumberInBase } from '@injectivelabs/utils'
import { UiDerivativeMarket, UiSpotMarket, UiOrderbookSummary } from '~/types'
import {
  UI_DEFAULT_AMOUNT_DISPLAY_DECIMALS,
  UI_DEFAULT_PRICE_DISPLAY_DECIMALS,
  UI_DEFAULT_AGGREGATION_DECIMALS
} from '~/app/utils/constants'

export default Vue.extend({
  props: {
    summary: {
      type: Object as PropType<UiOrderbookSummary>,
      default: null
    },

    market: {
      type: Object as PropType<UiDerivativeMarket | UiSpotMarket>,
      default: null
    }
  },

  data() {
    return {
      UI_DEFAULT_AMOUNT_DISPLAY_DECIMALS,
      UI_DEFAULT_PRICE_DISPLAY_DECIMALS,
      UI_DEFAULT_AGGREGATION_DECIMALS
    }
  },

  computed: {
    totalVolume(): BigNumberInBase {
      const { summary } = this

      return summary.quantity
    },

    totalPrice(): BigNumberInBase {
      const { summary } = this

      return summary.total
    },

    averagePrice(): string {
      const { totalPrice, totalVolume, market } = this

      return totalPrice
        .dividedBy(totalVolume)
        .toFormat(market?.priceDecimals || UI_DEFAULT_AGGREGATION_DECIMALS)
    }
  }
})
</script>
