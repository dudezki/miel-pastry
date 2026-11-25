<script setup lang="ts">
/**
 * Exploded view of the signature entremet: the slice separates into its layers as the section
 * scrolls (pinned, scrubbed), and each layer is labelled. Static and pre-separated for reduced motion.
 */
import { onBeforeUnmount, onMounted, ref } from 'vue'
import { gsap } from '@/composables/useMotion'
import { prefersReducedMotion } from '@/lib/utils'

const root = ref<HTMLElement>()
const reduced = ref(false)
let mm: gsap.MatchMedia | undefined

// Bottom → top. y/h in viewBox units of the assembled slice; `gap` = offset when exploded.
const layers = [
  { y: 640, h: 40, fill: '#c98b4a', t: 'Sablé breton', s: 'Salted-butter shortbread base' },
  { y: 590, h: 50, fill: '#e8c98f', t: 'Almond dacquoise', s: 'Light, nutty meringue sponge' },
  { y: 530, h: 60, fill: '#a8c47a', t: 'Pistachio crémeux', s: 'Sicilian pistachio, silky' },
  { y: 420, h: 110, fill: '#f2a6b6', t: 'Raspberry mousse', s: 'Whipped with fresh raspberries' },
  { y: 400, h: 20, fill: '#b8325a', t: 'Mirror glaze', s: 'Raspberry, poured at 32 °C' },
]
const EXPLODE = 58
const offset = (i: number) => 40 - i * EXPLODE // base settles onto the plate, each layer above lifts further

onMounted(() => {
  reduced.value = prefersReducedMotion()
  if (reduced.value) return
  mm = gsap.matchMedia()
  mm.add('(prefers-reduced-motion: no-preference)', () => {
    const q = gsap.utils.selector(root.value!)
    gsap.set(q('[data-layer]'), { y: 0 })
    gsap.set(q('[data-layer-label], [data-topping-label]'), { opacity: 0, x: (i: number) => (i % 2 ? 20 : -20) })
    const tl = gsap.timeline({
      defaults: { ease: 'none' },
      scrollTrigger: { trigger: q('[data-stage]')[0], start: 'top top', end: '+=160%', scrub: 0.7, pin: true, anticipatePin: 1 },
    })
    layers.forEach((_, i) => tl.to(q(`[data-layer="${i}"]`), { y: offset(i), duration: 1, ease: 'power2.inOut' }, 0))
    tl.to(q('[data-toppings]'), { y: offset(layers.length - 1) - EXPLODE, duration: 1, ease: 'power2.inOut' }, 0)
      .to(q('[data-layer-label], [data-topping-label]'), { opacity: 1, x: 0, stagger: 0.08, duration: 0.3 }, 0.6)
      .to({}, { duration: 0.4 })
  })
})
onBeforeUnmount(() => mm?.revert())
</script>

<template>
  <section id="layers" ref="root" class="bg-ink-2">
    <div data-stage class="relative flex min-h-svh flex-col justify-center py-24">
      <div class="container-x grid items-center gap-10 lg:grid-cols-[0.6fr_1.4fr]">
        <div>
          <p class="eyebrow">The signature</p>
          <h2 class="display mt-5 text-5xl sm:text-7xl">Framboise–<span class="italic text-signal">Pistache.</span></h2>
          <p class="mt-6 max-w-sm text-paper-dim">Five layers, two days, one slice. Scroll to take it apart — every layer is made in-house from scratch.</p>
        </div>
        <svg viewBox="0 0 1200 900" class="h-auto w-full" role="img"
          aria-label="Exploded view of the Framboise-Pistache entremet: from the bottom, sablé breton, almond dacquoise, pistachio crémeux, raspberry mousse, mirror glaze, topped with raspberries and gold leaf.">
          <ellipse cx="600" cy="745" rx="330" ry="42" fill="#ffffff" /><ellipse cx="600" cy="745" rx="270" ry="30" fill="none" stroke="#efe6da" stroke-width="3" />
          <g v-for="(l, i) in layers" :key="l.t" :data-layer="i" :transform="reduced ? `translate(0 ${offset(i)})` : undefined">
            <rect x="380" :y="l.y" width="440" :height="l.h" rx="10" :fill="l.fill" />
            <rect x="380" :y="l.y" width="440" :height="Math.min(10, l.h / 3)" rx="5" fill="#ffffff" opacity="0.25" />
            <template v-if="i === 0"><circle v-for="k in 12" :key="k" :cx="395 + k * 34" :cy="l.y + 22" r="3" fill="#9a5f2a" /></template>
            <template v-if="i === 1"><ellipse v-for="k in 10" :key="k" :cx="400 + k * 40" :cy="l.y + 25 + (k % 2) * 8" rx="7" ry="3" fill="#c9a06a" /></template>
            <template v-if="i === 3"><circle v-for="k in 9" :key="k" :cx="410 + k * 44" :cy="l.y + 40 + (k % 3) * 18" r="6" fill="#d6456b" opacity="0.6" /></template>
            <rect v-if="i === 4" x="400" :y="l.y + 4" width="160" height="5" rx="2.5" fill="#ffffff" opacity="0.5" />
            <g :data-layer-label="i" :opacity="reduced ? 1 : 0" font-family="Inter Variable, sans-serif" :text-anchor="i % 2 ? 'start' : 'end'">
              <line :x1="i % 2 ? 830 : 370" :y1="l.y + l.h / 2" :x2="i % 2 ? 880 : 320" :y2="l.y + l.h / 2" stroke="#4a2a1a" stroke-opacity="0.5" stroke-width="1.5" />
              <text :x="i % 2 ? 890 : 310" :y="l.y + l.h / 2 - 2" fill="#2e1a10" font-size="30" font-weight="600">{{ l.t }}</text>
              <text :x="i % 2 ? 890 : 310" :y="l.y + l.h / 2 + 28" fill="#5e4436" font-size="22">{{ l.s }}</text>
            </g>
          </g>
          <g data-toppings :transform="reduced ? `translate(0 ${offset(layers.length - 1) - EXPLODE})` : undefined">
            <circle v-for="k in 5" :key="k" :cx="440 + k * 60" cy="386" r="16" fill="#d6456b" />
            <circle v-for="k in 5" :key="'d' + k" :cx="440 + k * 60" cy="380" r="4" fill="#f2a6b6" />
            <path d="M590 372 l14 -10 l10 12 l-12 6z" fill="#e9b96e" /><path d="M700 374 l10 -8 l8 10 l-10 4z" fill="#e9b96e" />
            <circle v-for="k in 6" :key="'p' + k" :cx="420 + k * 66" cy="398" r="3" fill="#8fb46a" />
            <g data-topping-label :opacity="reduced ? 1 : 0" font-family="Inter Variable, sans-serif">
              <line x1="830" y1="380" x2="880" y2="380" stroke="#4a2a1a" stroke-opacity="0.5" stroke-width="1.5" />
              <text x="890" y="378" fill="#2e1a10" font-size="30" font-weight="600">Raspberries &amp; gold leaf</text>
              <text x="890" y="408" fill="#5e4436" font-size="22">Plus crushed pistachio</text>
            </g>
          </g>
        </svg>
      </div>
    </div>
  </section>
</template>
