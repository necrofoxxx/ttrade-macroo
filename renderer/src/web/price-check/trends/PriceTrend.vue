<template>
  <div v-if="trend" class="flex items-center pb-4" style="min-height: 3rem;">
    <div v-if="!isValuableBasetype && !slowdown.isReady.value" class="flex flex-1 justify-center">
      <div><i class="fas fa-dna fa-spin text-gray-600"></i></div>
      <div class="pl-2">{{ t('Getting price') }} <span class="text-gray-600">{{ t('from poe.ninja ...') }}</span></div>
    </div>
    <template v-else>
      <item-quick-price class="flex-1 text-base justify-center"
        :price="trend.price"
        :fraction="filters.stackSize != null"
        :item-img="item.info.icon"
      >
        <template #item v-if="isValuableBasetype">
          <span class="text-gray-400">{{ t('Base item') }}</span>
        </template>
      </item-quick-price>
      <div v-if="trend.change" class="px-2 text-center">
        <div class="leading-tight">
          <i v-if="trend.change.forecast === 'down'" class="fas fa-angle-double-down pr-1 text-red-600"></i>
          <i v-if="trend.change.forecast === 'up'" class="fas fa-angle-double-up pr-1 text-green-500"></i>
          <span v-if="trend.change.forecast === 'const'" class="pr-1 text-gray-600 font-sans leading-none">±</span>
          <span>{{ trend.change.text }}</span>
        </div>
        <div class="text-xs text-gray-500 leading-none">{{ t('Last 7 days') }}</div>
      </div>
      <div v-if="trend.change" class="w-12 h-8">
        <vue-apexcharts
          type="area"
          :options="{
            chart: { sparkline: { enabled: true }, animations: { enabled: false } },
            stroke: { curve: 'smooth', width: 1, colors: ['#a0aec0' /* gray.500 */] },
            fill: { colors: ['#4a5568' /* gray.700 */], type: 'solid' },
            tooltip: { enabled: false },
            plotOptions: { area: { fillTo: 'end' } },
            yaxis: {
              show: false,
              min: trend.change.graph.drawMin,
              max: trend.change.graph.drawMax
            }
          }"
          :series="[{
            data: trend.change.graph.points
          }]"
        />
      </div>
    </template>
  </div>
</template>

<script lang="ts">
import { defineComponent, PropType, computed, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import { findPriceByQuery, autoCurrency } from '../../background/Prices'
import { isValuableBasetype, getDetailsId } from './getDetailsId'
import ItemQuickPrice from '@/web/ui/ItemQuickPrice.vue'
import VueApexcharts from 'vue3-apexcharts'
import { ParsedItem } from '@/parser'
import { artificialSlowdown } from '../trade/artificial-slowdown'
import { ItemFilters } from '../filters/interfaces'

const slowdown = artificialSlowdown(800)

export default defineComponent({
  components: {
    ItemQuickPrice,
    VueApexcharts
  },
  props: {
    item: {
      type: Object as PropType<ParsedItem>,
      required: true
    },
    filters: {
      type: Object as PropType<ItemFilters>,
      required: true
    }
  },
  setup (props) {
    watch(() => props.item, (item) => {
      slowdown.reset(item)
    }, { immediate: true })

    const trend = computed(() => {
      const detailsId = getDetailsId(props.item)
      const trend = detailsId && findPriceByQuery(detailsId)
      if (!trend) return

      const price = (props.item.info.refName === 'Exalted Orb')
        ? { min: trend.chaos, max: trend.chaos, currency: 'chaos' as const }
        : autoCurrency(trend.chaos, 'chaos')

      return {
        price: price,
        change: deltaFromGraph(trend.graph)
      }
    })

    const { t } = useI18n()

    return {
      t,
      trend,
      isValuableBasetype: computed(() => {
        return isValuableBasetype(props.item)
      }),
      slowdown
    }
  }
})

function deltaFromGraph (graphPoints: Array<number | null>) {
  const points = graphPoints.filter(p => p != null) as number[]
  if (points.length < 2) return null

  let forecast = 'const'
  if (points.length === 7) {
    if (
      points.filter(p => p > 0).length >= 4 ||
      points.slice(4).every(p => p > 0)
    ) {
      forecast = 'up'
    } else if (
      points.filter(p => p < 0).length >= 4 ||
      points.slice(4).every(p => p < 0)
    ) {
      forecast = 'down'
    }
  }

  const mean = points.reduce((a, b) => a + b) / points.length
  const changeVal = Math.sqrt(points.map(x => Math.pow(x - mean, 2)).reduce((a, b) => a + b) / (points.length - 1))

  return {
    graph: {
      points,
      drawMin: Math.min(...points) - (changeVal || 1),
      drawMax: Math.max(...points) + (changeVal || 1)
    },
    forecast,
    text: `${Math.round(changeVal * 2)}\u2009%`
  }
}
</script>

<i18n>
{
  "ru": {
    "Base item": "База предмета",
    "Last 7 days": "За неделю",
    "Getting price": "Получение цены",
    "from poe.ninja ...": "с poe.ninja ..."
  }
}
</i18n>