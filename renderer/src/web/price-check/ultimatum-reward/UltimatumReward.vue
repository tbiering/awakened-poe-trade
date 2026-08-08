<template>
  <div
    v-if="rewardValue"
    class="p-2 border-dashed border border-gray-600 rounded mt-2"
  >
    <div class="flex items-center text-gray-400 leading-none justify-center">
      <span class="text-gray-500 mx-1">{{ t('trade_result.estimated_value') }}</span>
      <span>{{ rewardItem?.quantity || 1 }}</span>
      <span class="font-sans mx-1"> × </span>
      <item-quick-price
        :price="rewardValue.price"
        :item-img="rewardValue.icon"
        approx
        class="inline-flex"
      />
    </div>

    <div v-if="rewardValue.tier != null" class="flex items-center leading-none justify-center mt-2">
      <span class="text-gray-500 mx-1">{{ t('trade_result.tier') }}</span>
      <span :class="tierClass">{{ rewardValue.tier }}</span>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, PropType, computed } from 'vue'
import { useI18n } from 'vue-i18n'
import { usePoeninja } from '@/web/background/Prices'
import { ParsedItem, UltimatumRewardType } from '@/parser'
import { ITEM_BY_REF, ITEM_BY_TRANSLATED } from '@/assets/data'
import { AppConfig } from '@/web/Config'
import type { PriceCheckWidget } from '@/web/overlay/interfaces'
import ItemQuickPrice from '@/web/ui/ItemQuickPrice.vue'

const MIRROR_OF_KALANDRA = 'Mirror of Kalandra'

export default defineComponent({
  components: { ItemQuickPrice },
  props: {
    item: {
      type: Object as PropType<ParsedItem>,
      required: true
    }
  },
  setup (props) {
    const { findPriceByQuery, autoCurrency } = usePoeninja()
    const { t } = useI18n()

    const TIER_THRESHOLD = 2

    // fraction of the reward's market value left after trade/sacrifice overhead
    const fee = computed(() =>
      AppConfig<PriceCheckWidget>('price-check')!.ultimatumRewardFee ?? 0.9)

    const rewardType = computed(() => props.item.inscribedUltimatum?.rewardType ?? null)

    const rewardItem = computed(() => {
      if (!props.item.inscribedUltimatum) return null

      if (rewardType.value === UltimatumRewardType.MirroredCopy) {
        // Mirror case: reward is equivalent to 1 Mirror of Kalandra
        return {
          name: MIRROR_OF_KALANDRA,
          quantity: 1
        }
      }

      if (
        rewardType.value === UltimatumRewardType.Currency ||
        rewardType.value === UltimatumRewardType.DivinationCard
      ) {
        // Use sacrifice item name and multiply quantity by reward multiplier
        return {
          name: props.item.inscribedUltimatum.sacrifice,
          quantity: props.item.inscribedUltimatum.sacrificeQuantity
        }
      } else {
        // Use reward item name
        return {
          name: props.item.inscribedUltimatum.rewardUnique,
          quantity: 1
        }
      }
    })

    const dbItem = computed(() => {
      if (!rewardItem.value) return null

      // the mirror name is hardcoded in English, every other name comes from
      // the (possibly localized) clipboard text
      if (rewardType.value === UltimatumRewardType.MirroredCopy) {
        return ITEM_BY_REF('ITEM', MIRROR_OF_KALANDRA)?.[0]
      }

      const namespace =
        rewardType.value === UltimatumRewardType.DivinationCard
          ? 'DIVINATION_CARD'
          : rewardType.value === UltimatumRewardType.Unique
            ? 'UNIQUE'
            : 'ITEM'

      // prefer the English refName resolved at parse time (works for
      // localized clients), fall back to the translated-name lookup
      const refName =
        rewardType.value === UltimatumRewardType.Unique
          ? props.item.inscribedUltimatum?.rewardUniqueRefName
          : props.item.inscribedUltimatum?.sacrificeRefName
      if (refName) {
        return ITEM_BY_REF(namespace, refName)?.[0]
      }
      return ITEM_BY_TRANSLATED(namespace, rewardItem.value.name)?.[0]
    })

    const rewardValue = computed(() => {
      if (!dbItem.value || !rewardItem.value) return null

      // Special case: Mirror of Kalandra reward - just show mirror price
      if (rewardType.value === UltimatumRewardType.MirroredCopy) {
        const price = findPriceByQuery({
          ns: 'ITEM',
          name: dbItem.value.refName,
          variant: undefined
        })

        return {
          price: price ? autoCurrency(price.chaos * fee.value) : undefined, // undefined shows "?"
          icon: dbItem.value.icon,
          tier: props.item.inscribedUltimatum?.tier
        }
      }

      // Special case: Chaos Orb is the base currency, always worth 1 chaos
      if (rewardType.value === UltimatumRewardType.Currency && dbItem.value.refName === 'Chaos Orb') {
        return {
          price: autoCurrency(rewardItem.value.quantity * fee.value),
          icon: dbItem.value.icon,
          tier: props.item.inscribedUltimatum?.tier
        }
      }

      // Default: Find price for the reward item (Currency, Divination Card, or Unique)
      const price = findPriceByQuery({
        ns: dbItem.value.namespace,
        name: dbItem.value.refName,
        variant: dbItem.value.unique?.base
      })

      return {
        // undefined shows "?" in ItemQuickPrice
        price: price
          ? autoCurrency(price.chaos * rewardItem.value.quantity * fee.value)
          : undefined,
        icon: dbItem.value.icon,
        tier: props.item.inscribedUltimatum?.tier
      }
    })

    const tierClass = computed(() => {
      if (rewardValue.value?.tier == null) return ''
      return rewardValue.value.tier < TIER_THRESHOLD ? 'text-yellow-500' : 'text-gray-400'
    })

    return {
      t,
      rewardValue,
      rewardItem,
      tierClass
    }
  }
})
</script>
