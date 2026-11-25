<script setup lang="ts">
import { ref } from 'vue'
import { useSectionMotion } from '@/composables/useMotion'
const root = ref<HTMLElement>()
useSectionMotion(() => root.value)
// Fictional demo menu — kept in sync with the voice assistant persona ("miel").
const menu = [
  { group: 'Viennoiserie', items: [['Butter croissant', '3.20'], ['Pain au chocolat', '3.60'], ['Almond croissant', '4.20']] },
  { group: 'Pâtisserie', items: [['Raspberry tart', '6.50'], ['Pistachio éclair', '5.80'], ['Framboise–Pistache, slice', '7.90']] },
  { group: 'Macarons', items: [['Single macaron', '2.20'], ['Box of six', '12.00'], ['Box of twelve', '22.00']] },
  { group: 'To drink', items: [['Espresso', '2.80'], ['Café crème', '3.80'], ['Chocolat chaud', '4.50']] },
]
</script>

<template>
  <section id="menu" ref="root" class="container-x py-28 sm:py-40">
    <div class="grid gap-6 lg:grid-cols-[1fr_auto] lg:items-end">
      <div>
        <p class="eyebrow" data-reveal>Today's menu</p>
        <h2 class="display mt-5 text-5xl sm:text-7xl" data-reveal="0.1">Small menu. <span class="italic text-signal">Made today.</span></h2>
      </div>
      <p class="max-w-xs text-sm text-paper-dim" data-reveal>Prices in euros. Whole entremets (serves 8) €42 with 48 hours' notice.</p>
    </div>
    <div class="mt-16 grid gap-x-16 gap-y-14 md:grid-cols-2">
      <div v-for="g in menu" :key="g.group" data-reveal>
        <h3 class="display border-b border-line pb-3 text-3xl text-signal">{{ g.group }}</h3>
        <ul class="mt-2">
          <li v-for="[name, price] in g.items" :key="name" class="flex items-baseline gap-3 py-3">
            <span>{{ name }}</span>
            <span class="flex-1 translate-y-[-4px] border-b border-dotted border-paper/25" aria-hidden="true" />
            <span class="font-mono tabular-nums text-paper-dim"><span class="sr-only">€</span>{{ price }}</span>
          </li>
        </ul>
      </div>
    </div>
  </section>
</template>
