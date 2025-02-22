<template>
  <div class="flex flex-col flex-wrap overflow-y-hidden w-full">
    <div
      ref="sellOrders"
      class="overflow-y-scroll overflow-x-hidden w-full orderbook-half-h"
    >
      <div class="flex h-full w-full">
        <ul
          class="list-order-book w-full mt-auto"
          @mouseenter="autoScrollSellsLocked = true"
          @mouseleave="autoScrollSellsLocked = false"
        >
          <v-record
            v-for="(sell, index) in sellsWithDepth"
            :key="`order-book-sell-${index}`"
            :ref="`order-book-sell-${index}`"
            :class="{
              active: sellHoverPosition !== null && index >= sellHoverPosition
            }"
            :position="index"
            :aggregation="aggregation"
            :type="DerivativeOrderSide.Sell"
            :user-orders="sellUserOrderPrices"
            :record="sell"
            @update:active-position="handleSellOrderHover"
          ></v-record>
        </ul>
      </div>
    </div>
    <div
      v-if="market"
      class="orderbook-middle-h bg-gray-900 flex flex-col items-center justify-center border-t border-b"
    >
      <div class="w-full flex items-center justify-center">
        <v-icon-arrow
          v-if="
            [Change.Increase, Change.Decrease].includes(lastTradedPriceChange)
          "
          class="transform w-3 h-3 lg:w-4 lg:h-4 4xl:w-5 4xl:h-5"
          :class="{
            'text-red-500 -rotate-90':
              lastTradedPriceChange === Change.Decrease,
            'text-aqua-500 rotate-90': lastTradedPriceChange === Change.Increase
          }"
        />
        <span
          :class="{
            'text-red-500': lastTradedPriceChange === Change.Decrease,
            'text-aqua-500': lastTradedPriceChange !== Change.Decrease
          }"
          class="font-bold font-mono text-base lg:text-lg 4xl:text-xl"
        >
          {{ lastTradedPriceToFormat }}
        </span>
      </div>
    </div>
    <ul
      ref="buyOrders"
      class="list-order-book overflow-y-scroll overflow-x-hidden w-full orderbook-half-h rounded-b-lg"
      @mouseenter="autoScrollBuysLocked = true"
      @mouseleave="autoScrollBuysLocked = false"
    >
      <v-record
        v-for="(buy, index) in buysWithDepth"
        :key="`order-book-buy-${index}`"
        :ref="`order-book-buy-${index}`"
        :class="{
          active: buyHoverPosition !== null && index <= buyHoverPosition
        }"
        :position="index"
        :aggregation="aggregation"
        :type="DerivativeOrderSide.Buy"
        :user-orders="buyUserOrderPrices"
        :record="buy"
        @update:active-position="handleBuyOrderHover"
      ></v-record>
    </ul>

    <!-- orderbook summary popup -->
    <div ref="orderbookSummary" class="orderbook-summary">
      <SummaryPopup :market="market" :summary="orderBookSummary" />
    </div>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'
import { BigNumberInBase, BigNumberInWei } from '@injectivelabs/utils'
import { createPopper, Instance } from '@popperjs/core'
import Record from './record.vue'
import SummaryPopup from '~/components/partials/common/orderbook/summary-popup.vue'
import {
  getAggregationPrice,
  computeOrderbookSummary
} from '~/app/services/derivatives'
import {
  UI_DEFAULT_PRICE_DISPLAY_DECIMALS,
  ZERO_IN_BASE
} from '~/app/utils/constants'
import {
  UiDerivativeTrade,
  UiDerivativeMarket,
  UiDerivativeLimitOrder,
  UiPriceLevel,
  UiDerivativeOrderbook,
  TradeDirection,
  DerivativeOrderSide,
  UiOrderbookPriceLevel,
  UiOrderbookSummary,
  Change
} from '~/types'

export default Vue.extend({
  components: {
    SummaryPopup,
    'v-record': Record
  },

  props: {
    aggregation: {
      type: Number,
      required: true
    }
  },

  data() {
    return {
      Change,
      TradeDirection,
      DerivativeOrderSide,
      autoScrollSellsLocked: false,
      autoScrollBuysLocked: false,
      buyHoverPosition: null as number | null,
      sellHoverPosition: null as number | null,
      popper: {} as Instance,
      popperOption: {
        placement: 'left',
        modifiers: [
          {
            name: 'preventOverflow',
            options: {
              mainAxis: false
            }
          }
        ]
      } as Object
    }
  },

  computed: {
    trades(): UiDerivativeTrade[] {
      return this.$accessor.derivatives.trades
    },

    subaccountOrders(): UiDerivativeLimitOrder[] {
      return this.$accessor.derivatives.subaccountOrders
    },

    market(): UiDerivativeMarket | undefined {
      return this.$accessor.derivatives.market
    },

    orderbook(): UiDerivativeOrderbook | undefined {
      return this.$accessor.derivatives.orderbook
    },

    lastTradedPrice(): BigNumberInBase {
      return this.$accessor.derivatives.lastTradedPrice
    },

    lastTradedPriceChange(): Change {
      return this.$accessor.derivatives.lastTradedPriceChange
    },

    lastTradedPriceToFormat(): string {
      const { market, lastTradedPrice } = this

      if (!market) {
        return lastTradedPrice.toFormat(UI_DEFAULT_PRICE_DISPLAY_DECIMALS)
      }

      return lastTradedPrice.toFormat(market.priceDecimals)
    },

    buys(): UiPriceLevel[] {
      const { orderbook } = this

      if (!orderbook) {
        return []
      }

      return orderbook.buys
    },

    sells(): UiPriceLevel[] {
      const { orderbook } = this

      if (!orderbook) {
        return []
      }

      return orderbook.sells
    },

    buyUserOrderPrices(): string[] {
      const { subaccountOrders } = this

      return subaccountOrders.reduce((records, { orderSide, price }) => {
        return orderSide === DerivativeOrderSide.Buy
          ? [...records, price]
          : records
      }, [] as string[])
    },

    sellUserOrderPrices(): string[] {
      const { subaccountOrders } = this

      return subaccountOrders.reduce((records, { orderSide, price }) => {
        return orderSide === DerivativeOrderSide.Sell
          ? [...records, price]
          : records
      }, [] as string[])
    },

    buysTotalNotional(): BigNumberInBase {
      const { buys, market } = this

      if (!market) {
        return ZERO_IN_BASE
      }

      return buys.reduce((total, buy) => {
        return total.plus(
          new BigNumberInWei(buy.quantity)
            .times(buy.price)
            .toBase(market.quoteToken.decimals)
        )
      }, ZERO_IN_BASE)
    },

    sellsTotalNotional(): BigNumberInBase {
      const { sells, market } = this

      if (!market) {
        return ZERO_IN_BASE
      }

      return sells.reduce((total, sell) => {
        return total.plus(
          new BigNumberInWei(sell.quantity)
            .times(sell.price)
            .toBase(market.quoteToken.decimals)
        )
      }, ZERO_IN_BASE)
    },

    aggregatedBuyOrders(): UiOrderbookPriceLevel[] {
      const { aggregation, buys, market } = this

      if (!market) {
        return []
      }

      const orders = {} as Record<string, any>
      buys.forEach((record: UiPriceLevel) => {
        const price = new BigNumberInBase(
          new BigNumberInWei(record.price).toBase(market.quoteToken.decimals)
        )

        const aggregatedPrice = getAggregationPrice({
          price,
          aggregation,
          isBuy: true
        })
        const aggregatedPriceKey = aggregatedPrice.toFormat()

        orders[aggregatedPriceKey] = [
          ...(orders[aggregatedPriceKey] || []),
          {
            ...record,
            aggregatedPrice
          }
        ]
      })

      return Object.entries(orders).map(([, orderGroup]) => {
        const [firstOrder] = orderGroup

        const quantity = orderGroup.reduce(
          (sum: BigNumberInWei, order: UiPriceLevel) => {
            return sum.plus(new BigNumberInWei(order.quantity))
          },
          new BigNumberInWei(0)
        )

        const notional = orderGroup.reduce(
          (sum: BigNumberInWei, order: UiPriceLevel) => {
            const notional = new BigNumberInWei(order.quantity)
              .times(order.price)
              .toBase(market.quoteToken.decimals)

            return sum.plus(notional)
          },
          new BigNumberInBase(0)
        )

        const aggregatePrices = orderGroup.map(
          ({ price }: UiPriceLevel) => price
        )

        return {
          ...firstOrder,
          aggregatePrices,
          notional,
          quantity
        }
      })
    },

    buysWithDepth(): UiOrderbookPriceLevel[] {
      const { aggregatedBuyOrders, buysTotalNotional, market } = this

      if (!market) {
        return []
      }

      return aggregatedBuyOrders
        .sort((v1: UiOrderbookPriceLevel, v2: UiOrderbookPriceLevel) => {
          const v1Price = new BigNumberInWei(v1.price)
          const v2Price = new BigNumberInWei(v2.price)

          return v2Price.minus(v1Price).toNumber()
        })
        .map((record: UiPriceLevel) => {
          const total = new BigNumberInWei(record.quantity)
            .times(record.price)
            .toBase(market.quoteToken.decimals)

          return {
            ...record,
            total,
            depth: total.dividedBy(buysTotalNotional).times(100).toNumber()
          } as UiOrderbookPriceLevel
        })
    },

    buyOrdersSummary(): UiOrderbookSummary | undefined {
      const { buysWithDepth, buyHoverPosition, market } = this

      if (!market || buysWithDepth.length === 0 || buyHoverPosition === null) {
        return
      }

      return buysWithDepth
        .slice(0, Number(buyHoverPosition) + 1)
        .reduce(computeOrderbookSummary, {
          quantity: new BigNumberInBase(0),
          total: new BigNumberInBase(0)
        })
    },

    aggregatedSellOrders(): UiOrderbookPriceLevel[] {
      const { aggregation, sells, market } = this

      if (!market) {
        return []
      }

      const orders = {} as Record<string, any>
      sells.forEach((record: UiPriceLevel, index: number) => {
        const price = new BigNumberInBase(
          new BigNumberInWei(record.price).toBase(market.quoteToken.decimals)
        )

        const aggregatedPrice = getAggregationPrice({
          price,
          aggregation,
          isBuy: false
        })
        const aggregatedPriceKey = aggregatedPrice.toFormat()

        orders[aggregatedPriceKey] = [
          ...(orders[aggregatedPriceKey] || []),
          {
            ...record,
            aggregatedPrice
          }
        ]
      })

      return Object.entries(orders)
        .reverse()
        .map(([, orderGroup]) => {
          const [firstOrder] = orderGroup

          const quantity = orderGroup.reduce(
            (sum: BigNumberInWei, order: UiPriceLevel) => {
              return sum.plus(new BigNumberInWei(order.quantity))
            },
            new BigNumberInWei(0)
          )

          const aggregatePrices = orderGroup.map(
            ({ price }: UiPriceLevel) => price
          )

          return {
            ...firstOrder,
            aggregatePrices,
            quantity
          }
        })
    },

    sellsWithDepth(): UiOrderbookPriceLevel[] {
      const { aggregatedSellOrders, sellsTotalNotional, market } = this

      if (!market) {
        return []
      }

      return aggregatedSellOrders
        .sort((v1: UiOrderbookPriceLevel, v2: UiOrderbookPriceLevel) => {
          const v1Price = new BigNumberInWei(v1.price)
          const v2Price = new BigNumberInWei(v2.price)

          return v1Price.minus(v2Price).toNumber()
        })
        .map((record: UiPriceLevel) => {
          const total = new BigNumberInWei(record.quantity)
            .times(record.price)
            .toBase(market.quoteToken.decimals)

          return {
            ...record,
            total,
            depth: total.dividedBy(sellsTotalNotional).times(100).toNumber()
          } as UiOrderbookPriceLevel
        })
        .reverse()
    },

    sellOrdersSummary(): UiOrderbookSummary | undefined {
      const { sellsWithDepth, sellHoverPosition, market } = this

      if (
        !market ||
        sellsWithDepth.length === 0 ||
        sellHoverPosition === null
      ) {
        return
      }

      return sellsWithDepth
        .slice(Number(sellHoverPosition))
        .reduce(computeOrderbookSummary, {
          quantity: new BigNumberInBase(0),
          total: new BigNumberInBase(0)
        })
    },

    orderBookSummary(): UiOrderbookSummary | undefined {
      const {
        buyHoverPosition,
        sellHoverPosition,
        buyOrdersSummary,
        sellOrdersSummary
      } = this

      if (buyHoverPosition !== null) {
        return buyOrdersSummary
      }

      if (sellHoverPosition !== null) {
        return sellOrdersSummary
      }

      return undefined
    },

    $orderbookSummaryElement(): InstanceType<typeof HTMLElement> {
      return this.$refs.orderbookSummary as InstanceType<typeof HTMLElement>
    }
  },

  watch: {
    aggregation() {
      this.$nextTick(() => {
        this.onScrollSells()
        this.onScrollBuys()
      })
    },

    sells() {
      this.$nextTick(this.onScrollSells)
    },

    buys() {
      this.$nextTick(this.onScrollBuys)
    }
  },

  mounted() {
    this.$nextTick(() => {
      this.onScrollSells()
      this.onScrollBuys()
    })
  },

  methods: {
    onScrollSells() {
      const el = this.$refs.sellOrders as any

      if (el && !this.autoScrollSellsLocked) {
        el.scrollTop = el.scrollHeight
      }
    },

    onScrollBuys() {
      const el = this.$refs.buyOrders as any

      if (el && !this.autoScrollBuysLocked) {
        el.scrollTop = 0
      }
    },

    handleSellOrderHover(position: number | null) {
      const { $orderbookSummaryElement, popperOption } = this
      this.sellHoverPosition = position

      if (position !== null) {
        const hoverElement = this.$refs[`order-book-sell-${position}`] as {
          $el: InstanceType<typeof Element>
        }[]

        this.popper = createPopper(
          hoverElement[0].$el,
          $orderbookSummaryElement,
          popperOption
        )

        this.$nextTick(() =>
          $orderbookSummaryElement.setAttribute('data-show', '')
        )
      } else {
        if (this.popper.destroy) {
          this.popper.destroy()
        }
        $orderbookSummaryElement.removeAttribute('data-show')
      }
    },

    handleBuyOrderHover(position: number | null) {
      const { $orderbookSummaryElement, popperOption } = this
      this.buyHoverPosition = position

      if (position !== null) {
        const hoverElement = this.$refs[`order-book-buy-${position}`] as {
          $el: InstanceType<typeof Element>
        }[]

        this.popper = createPopper(
          hoverElement[0].$el,
          $orderbookSummaryElement,
          popperOption
        )

        this.$nextTick(() =>
          $orderbookSummaryElement.setAttribute('data-show', '')
        )
      } else {
        if (this.popper.destroy) {
          this.popper.destroy()
        }
        $orderbookSummaryElement.removeAttribute('data-show')
      }
    }
  }
})
</script>
