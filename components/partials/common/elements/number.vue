<template>
  <div>
    <span class="font-mono leading-5 inline-block">
      <div class="flex">
        <span class="">{{ prefix || '' }}{{ formattedNumber[0] }}</span>
        <span v-if="formattedNumber[1]" class="opacity-25">{{
          formattedNumber[1] || ''
        }}</span>
        {{ suffix || '' }}
      </div>
    </span>
    <slot name="addon"></slot>
  </div>
</template>

<script lang="ts">
import { BigNumberInBase } from '@injectivelabs/utils'
import Vue, { PropType } from 'vue'
import { UI_DEFAULT_DISPLAY_DECIMALS } from '~/app/utils/constants'
import { getDecimalsFromNumber } from '~/app/utils/helpers'

export default Vue.extend({
  props: {
    decimals: {
      required: false,
      default: UI_DEFAULT_DISPLAY_DECIMALS,
      type: Number
    },

    useNumberDecimals: {
      required: false,
      default: false,
      type: Boolean
    },

    prefix: {
      required: false,
      type: String,
      default: ''
    },

    suffix: {
      required: false,
      type: String,
      default: ''
    },

    number: {
      required: true,
      type: Object as PropType<BigNumberInBase>
    },

    dontGroupValues: {
      required: false,
      default: false,
      type: Boolean
    }
  },

  computed: {
    formattedNumber(): string[] {
      const { dontGroupValues, number, useNumberDecimals, decimals } = this

      if (number.eq(0)) {
        return [number.toFixed(0)]
      }

      const actualDecimals = useNumberDecimals
        ? getDecimalsFromNumber(number.toNumber())
        : decimals
      const formattedNumber = number.toFormat(
        actualDecimals,
        BigNumberInBase.ROUND_DOWN
      )

      if (dontGroupValues) {
        return [formattedNumber]
      }

      const match = formattedNumber.match(/^-?([\d,]+)((\.)(\d+?\d+?)(0*))?$/)
      const groups = !match
        ? formattedNumber
          ? [formattedNumber]
          : []
        : match[2]
        ? [`${match[1]}${match[3]}${match[4]}`, match[5]]
        : [`${match[1]}`]

      return groups
    }
  }
})
</script>
