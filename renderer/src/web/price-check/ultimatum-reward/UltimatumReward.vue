<template>
  <div
    v-if="rewardValue"
    class="p-2 border-dashed border border-gray-600 rounded mt-2"
  >
    <div class="flex items-center text-gray-400 leading-none justify-center">
      <span class="text-gray-500 mx-1">{{ t("Estimated Value:") }}</span>
      <span>{{ rewardItem?.quantity || 1 }}</span>
      <span class="font-sans mx-1"> × </span>
      <item-quick-price
        :price="rewardValue.price"
        :item-img="rewardValue.icon"
        approx
        class="inline-flex"
      />
    </div>

    <div v-if="rewardValue.tier !== undefined" class="flex items-center leading-none justify-center mt-2">
      <span class="text-gray-500 mx-1">{{ t("Tier") }}</span>
      <span :class="tierClass">{{ rewardValue.tier }}</span>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, PropType, computed } from 'vue'
import { useI18n } from 'vue-i18n'
import { usePoeninja } from '@/web/background/Prices'
import { ParsedItem, UltimatumRewardType } from '@/parser'
import { ITEM_BY_TRANSLATED } from '@/assets/data'
import ItemQuickPrice from '@/web/ui/ItemQuickPrice.vue'

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

    const rewardType = computed(() => props.item.inscribedUltimatum?.reward_type ?? null)

    const rewardItem = computed(() => {
      if (!props.item.inscribedUltimatum) return null

      if (rewardType.value === UltimatumRewardType.MirroredCopy) {
        // Mirror case: reward is equivalent to 1 Mirror of Kalandra
        return {
          name: 'Mirror of Kalandra',
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
          quantity: props.item.inscribedUltimatum.sacrifice_quantity
        }
      } else {
        // Use reward item name
        return {
          name: props.item.inscribedUltimatum.reward_unique,
          quantity: 1
        }
      }
    })

    const dbItem = computed(() => {
      if (!rewardItem.value) return null

      const namespace =
        rewardType.value === UltimatumRewardType.DivinationCard
          ? 'DIVINATION_CARD'
          : rewardType.value === UltimatumRewardType.Unique
            ? 'UNIQUE'
            : 'ITEM'
      const found = ITEM_BY_TRANSLATED(namespace, rewardItem.value.name)
      return found?.[0]
    })

    const rewardValue = computed(() => {
      if (!dbItem.value || !rewardItem.value) return null

      // Special case: Mirror of Kalandra reward - just show mirror price
      if (rewardType.value === UltimatumRewardType.MirroredCopy) {
        const price = findPriceByQuery({
          ns: 'ITEM',
          name: 'Mirror of Kalandra',
          variant: undefined
        })

        return {
          price: price ? autoCurrency(price.chaos * 0.95) : undefined, // 5% fee, undefined shows "?"
          icon: dbItem.value.icon,
          tier: props.item.inscribedUltimatum?.tier
        }
      }

      // Special case: Chaos Orb is the base currency, always worth 1 chaos
      if (rewardType.value === UltimatumRewardType.Currency && dbItem.value.refName === 'Chaos Orb') {
        const rewardChaos = rewardItem.value.quantity * 0.9 // 10% fee

        return {
          price: autoCurrency(rewardChaos),
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

      // Calculate: (value of sacrifice) * 0.9
      let calculatedPrice
      if (price) {
        let rewardChaos: number
        if (
          rewardType.value === UltimatumRewardType.Currency ||
          rewardType.value === UltimatumRewardType.DivinationCard
        ) {
          // For currency/divination rewards: value of sacrificed items minus 10% fee
          rewardChaos = price.chaos * rewardItem.value.quantity * 0.9
        } else {
          // For unique items: just the value of the unique minus nothing (no sacrifice cost for unique rewards)
          rewardChaos = price.chaos * rewardItem.value.quantity * 0.9
        }
        calculatedPrice = autoCurrency(rewardChaos)
      }

      return {
        price: calculatedPrice, // undefined shows "?" in ItemQuickPrice
        icon: dbItem.value.icon,
        tier: props.item.inscribedUltimatum?.tier
      }
    })

    const tierClass = computed(() => {
      if (!rewardValue.value?.tier) return ''
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
