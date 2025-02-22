<template>
  <div>
    <div class="overflow-y-auto overflow-x-auto md:overflow-x-visible w-full">
      <TableHeader v-if="isUserWalletConnected">
        <span class="col-span-3">{{ $t('Asset') }}</span>
        <span class="col-span-3">
          <div class="flex items-center">
            <span class="flex-1 text-right">{{
              $t('Injective Chain Balance')
            }}</span>
          </div>
        </span>
        <span class="col-span-3">
          <div class="flex items-center relative justify-end">
            {{ $t('ERC20 Balance') }}
          </div>
        </span>
        <span class="col-span-3">
          <div class="flex items-center relative justify-end">
            {{ $t('Total') }}
          </div>
        </span>
      </TableHeader>

      <TableBody
        v-if="isUserWalletConnected"
        :show-empty="balances.length === 0"
      >
        <v-balance
          v-for="(balance, index) in sortedBalances"
          :key="`balance-${index}`"
          class="col-span-1"
          :balance="balance"
        />
        <template slot="empty">
          <span class="col-span-1 md:col-span-3 text-center md:text-left">{{
            $t('There are no results found - Balances')
          }}</span>
        </template>
      </TableBody>
      <v-user-wallet-connect-warning v-else cta />
    </div>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'
import { BigNumberInWei } from '@injectivelabs/utils'
import VBalance from './balance.vue'
import TableBody from '~/components/elements/table-body.vue'
import TableHeader from '~/components/elements/table-header.vue'
import { TokenWithBalance } from '~/types/token'
import {
  BankBalanceWithTokenMetaData,
  BankBalanceWithTokenMetaDataAndBalance,
  BankBalanceWithTokenMetaDataAndBalanceWithUsdBalance,
  IbcBankBalanceWithTokenMetaData
} from '~/types/bank'
import { INJECTIVE_DENOM } from '~/app/utils/constants'

export default Vue.extend({
  components: {
    TableBody,
    TableHeader,
    VBalance
  },

  computed: {
    isUserWalletConnected(): boolean {
      return this.$accessor.wallet.isUserWalletConnected
    },

    bankBalances(): BankBalanceWithTokenMetaData[] {
      return this.$accessor.bank.balancesWithTokenMetaData
    },

    ibcBankBalances(): IbcBankBalanceWithTokenMetaData[] {
      return this.$accessor.bank.ibcBalancesWithTokenMetaData
    },

    erc20TokensWithBalanceAndAllowance(): TokenWithBalance[] {
      return this.$accessor.token.erc20TokensWithBalanceFromBank
    },

    ibcTokensWithBalanceFromBank(): TokenWithBalance[] {
      return this.$accessor.token.ibcTokensWithBalanceFromBank
    },

    ercBalances(): BankBalanceWithTokenMetaDataAndBalance[] {
      const { bankBalances, erc20TokensWithBalanceAndAllowance } = this

      return bankBalances.map((bankBalance) => {
        const tokenWithBalance = erc20TokensWithBalanceAndAllowance.find(
          (token) =>
            token.address.toLowerCase() ===
            bankBalance.token.address.toLowerCase()
        )

        return {
          ...bankBalance,
          token: tokenWithBalance || {
            ...bankBalance.token,
            balance: '0',
            allowance: '0'
          }
        }
      })
    },

    ibcBalances(): BankBalanceWithTokenMetaDataAndBalance[] {
      const { ibcBankBalances, ibcTokensWithBalanceFromBank } = this

      return ibcBankBalances.map((bankBalance) => {
        const tokenWithBalance = ibcTokensWithBalanceFromBank.find(
          (token) =>
            token.address.toLowerCase() ===
            bankBalance.token.address.toLowerCase()
        )

        return {
          ...bankBalance,
          token: tokenWithBalance || {
            ...bankBalance.token,
            balance: '0',
            allowance: '0'
          }
        }
      })
    },

    balances(): BankBalanceWithTokenMetaDataAndBalance[] {
      const { ercBalances, ibcBalances } = this

      // calculate and append total USD balances
      return [...ercBalances, ...ibcBalances].map((balance) => {
        const balanceInUsd = new BigNumberInWei(balance.balance)
          .toBase(balance.token.decimals)
          .times(balance.token?.priceInUsd || 0)

        return {
          ...balance,
          balanceInUsd
        }
      })
    },

    sortedBalances(): BankBalanceWithTokenMetaDataAndBalanceWithUsdBalance[] {
      const { balances } = this

      return [...balances]
        .map((balance) => {
          const balanceInUsd = new BigNumberInWei(balance.balance)
            .toBase(balance.token.decimals)
            .times(balance.token?.priceInUsd || 0)

          return {
            ...balance,
            balanceInUsd
          }
        })
        .sort(
          (
            v1: BankBalanceWithTokenMetaDataAndBalanceWithUsdBalance,
            v2: BankBalanceWithTokenMetaDataAndBalanceWithUsdBalance
          ) => {
            // sort INJ to the top
            if (v1.denom === INJECTIVE_DENOM) {
              return -1
            }

            if (v2.denom === INJECTIVE_DENOM) {
              return 1
            }

            return v2.balanceInUsd.minus(v1.balanceInUsd).toNumber()
          }
        )
    }
  }
})
</script>
