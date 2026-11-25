<script setup lang="ts">
/**
 * Scroll-driven pâtisserie tour: storefront → display window (pastries labelled) → push into the
 * croissant → macro croissant anatomy. Pure SVG + GSAP ScrollTrigger (scrubbed, pinned).
 * Narrow screens show labels as chips; reduced-motion users get a static, labelled version.
 */
import { nextTick, onBeforeUnmount, onMounted, ref } from 'vue'
import { Mic } from 'lucide-vue-next'
import { gsap, ScrollTrigger } from '@/composables/useMotion'
import { prefersReducedMotion } from '@/lib/utils'
import { useCallStore } from '@/stores/call'
import Button from '@/components/ui/Button.vue'

const call = useCallStore()
const root = ref<HTMLElement>()
const reduced = ref(false)
const narrow = ref(false)
let mm: gsap.MatchMedia | undefined

/** Camera target: the croissant on the middle shelf (viewBox units). */
const O = { x: 735, y: 585 }
const WINDOW_ZOOM = 2.1
/** Where a world point lands on screen at a given zoom around O. */
const project = (x: number, y: number, s: number) => ({ x: O.x + (x - O.x) * s, y: O.y + (y - O.y) * s })

type Label = { x: number; y: number; tx: number; ty: number; t: string; s: string }
const shopLabels: Label[] = [
  { x: 1150, y: 195, tx: 1180, ty: 120, t: 'Baked at dawn', s: 'Ovens on at 5 am' },
  { x: 600, y: 420, tx: 215, ty: 380, t: 'Window of the day', s: 'Fresh trays every 2 hours' },
  { x: 1180, y: 520, tx: 1110, ty: 290, t: 'Coffee bar inside', s: 'Espresso & chocolat chaud' },
]
const shelf = [
  { wx: 560, wy: 468, dx: -170, dy: -110, t: 'Macarons', s: 'Six flavours, filled to order' },
  { wx: 890, wy: 466, dx: 70, dy: -70, t: 'Pistachio éclair', s: 'Sicilian pistachio cream' },
  { wx: 560, wy: 588, dx: -150, dy: 60, t: 'Raspberry tart', s: 'Vanilla crème, fresh berries' },
  { wx: 735, wy: 578, dx: -110, dy: -175, t: 'Butter croissant', s: 'Our signature — look closer' },
  { wx: 905, wy: 590, dx: 20, dy: 50, t: 'Pain au chocolat', s: 'Two bars of dark chocolate' },
  { wx: 880, wy: 700, dx: 30, dy: -20, t: 'Raspberry entremet', s: 'Mirror glaze, pistachio heart' },
]
const windowLabels: Label[] = shelf.map((l) => {
  const p = project(l.wx, l.wy, WINDOW_ZOOM)
  return { x: p.x, y: p.y, tx: p.x + l.dx, ty: p.y + l.dy, t: l.t, s: l.s }
})
const crumbLabels: Label[] = [
  { x: 800, y: 300, tx: 880, ty: 120, t: 'Golden egg-wash crust', s: 'Shatters at the first bite' },
  { x: 655, y: 450, tx: 230, ty: 250, t: '27 laminated layers', s: 'Folded three times, rested overnight' },
  { x: 425, y: 640, tx: 215, ty: 700, t: 'Crisp, caramelised tips', s: 'The baker’s favourite part' },
  { x: 1255, y: 735, tx: 1180, ty: 540, t: 'Honeycomb crumb', s: 'Open, airy, buttery' },
  { x: 330, y: 765, tx: 360, ty: 820, t: 'French AOP butter', s: '84% butterfat' },
]
const pillW = (l: { t: string; s: string }) => Math.max(l.t.length * 13.5, l.s.length * 9.2) + 40

const macarons = ['#f2a3b5', '#b9d48f', '#f6dc7a', '#c7b3e0', '#7a4a32', '#d6456b']
// Croissant segments (back to front): [cx, cy, rx, ry, rotation]
const segs: [number, number, number, number, number][] = [
  [420, 630, 50, 66, -62], [1180, 630, 50, 66, 62],
  [505, 548, 84, 100, -46], [1095, 548, 84, 100, 46],
  [640, 470, 108, 128, -24], [960, 470, 108, 128, 24],
  [800, 430, 124, 152, 0],
]
const holes = Array.from({ length: 14 }, (_, i) => {
  const r = (n: number) => { const v = Math.sin(i * 77.7 + n * 3.1) * 1000; return v - Math.floor(v) }
  return { x: 1195 + r(1) * 120, y: 712 + r(2) * 50, rx: 5 + r(3) * 9, ry: 3 + r(4) * 6 }
})

const setNarrow = () => { narrow.value = window.innerWidth / window.innerHeight < 1 }

onMounted(async () => {
  reduced.value = prefersReducedMotion()
  setNarrow()
  window.addEventListener('resize', setNarrow)
  if (reduced.value) return
  await nextTick()

  mm = gsap.matchMedia()
  mm.add('(prefers-reduced-motion: no-preference)', () => {
    const q = gsap.utils.selector(root.value!)
    const origin = `${O.x} ${O.y}`
    gsap.set(q('[data-l]'), { opacity: 0, y: 12 })
    gsap.set(q('[data-macro]'), { opacity: 0, scale: 1.14, svgOrigin: '800 450' })
    gsap.set(q('[data-flash], [data-chapter]'), { opacity: 0 })

    const tl = gsap.timeline({
      defaults: { ease: 'none' },
      scrollTrigger: { trigger: root.value, start: 'top top', end: '+=600%', scrub: 0.8, pin: true, anticipatePin: 1, refreshPriority: 1 },
    })
    tl.to(q('[data-intro]'), { opacity: 0, y: -40, duration: 0.5 }, 0)
      .to(q('[data-scroll-hint]'), { opacity: 0, duration: 0.3 }, 0)
      // 1 · The shop
      .to(q('[data-chapter="1"]'), { opacity: 1, duration: 0.3 }, 0.3)
      .to(q('[data-l="shop"]'), { opacity: 1, y: 0, stagger: 0.15, duration: 0.35, ease: 'power2.out' }, 0.4)
      .to(q('[data-world]'), { scale: 1.08, svgOrigin: origin, duration: 1.5 }, 0.2)
      .to(q('[data-l="shop"], [data-chapter="1"]'), { opacity: 0, duration: 0.3 }, 1.6)
      // 2 · The window
      .to(q('[data-world]'), { scale: WINDOW_ZOOM, svgOrigin: origin, duration: 1.1, ease: 'power2.inOut' }, 1.9)
      .to(q('[data-chapter="2"]'), { opacity: 1, duration: 0.3 }, 2.9)
      .to(q('[data-l="window"]'), { opacity: 1, y: 0, stagger: 0.1, duration: 0.35, ease: 'power2.out' }, 3.0)
      .to(q('[data-l="window"], [data-chapter="2"]'), { opacity: 0, duration: 0.25 }, 4.3)
      // 3 · Into the croissant
      .to(q('[data-world]'), { scale: 18, svgOrigin: origin, duration: 1.5, ease: 'power2.in' }, 4.5)
      .to(q('[data-flash]'), { opacity: 1, duration: 0.3 }, 5.7)
      .to(q('[data-macro]'), { opacity: 1, duration: 0.3 }, 6.0)
      .to(q('[data-world-svg]'), { opacity: 0, duration: 0.2 }, 6.0)
      .to(q('[data-flash]'), { opacity: 0, duration: 0.5 }, 6.1)
      .to(q('[data-macro]'), { scale: 1, svgOrigin: '800 450', duration: 1, ease: 'power2.out' }, 6.0)
      .to(q('[data-chapter="3"]'), { opacity: 1, duration: 0.3 }, 6.4)
      .to(q('[data-l="crumb"]'), { opacity: 1, y: 0, stagger: 0.14, duration: 0.35, ease: 'power2.out' }, 6.6)
      .to({}, { duration: 0.8 })

    // This trigger is created after later sections' triggers (we awaited nextTick), so sort by
    // refreshPriority and recompute: sections below must account for this pin's spacing.
    ScrollTrigger.sort()
    ScrollTrigger.refresh()

    // Ambient life: lamps glow, steam rises from the coffee cup.
    gsap.to(q('[data-lamp]'), { opacity: 0.5, duration: 1.8, repeat: -1, yoyo: true, ease: 'sine.inOut', stagger: 0.5 })
    gsap.to(q('[data-steam]'), { y: -18, opacity: 0, duration: 2.4, repeat: -1, ease: 'sine.out', stagger: 0.8 })
  })
})
onBeforeUnmount(() => { mm?.revert(); window.removeEventListener('resize', setNarrow) })
</script>

<template>
  <section id="tour" ref="root" class="relative bg-[#f6e6d3]" aria-label="Scroll tour of MIEL. Pâtisserie">
    <div class="sr-only">
      <h2>Tour of MIEL. Pâtisserie</h2>
      <ul><li v-for="l in [...shopLabels, ...windowLabels, ...crumbLabels]" :key="l.t">{{ l.t }}: {{ l.s }}</li></ul>
    </div>

    <div :class="reduced ? 'space-y-px' : 'relative h-svh overflow-hidden'">
      <!-- ============ STOREFRONT + WINDOW (one world, zoomed by the camera) ============ -->
      <svg
        data-world-svg :class="reduced ? 'block aspect-video w-full' : 'absolute inset-0 size-full'"
        viewBox="0 0 1600 900" :preserveAspectRatio="narrow ? 'xMidYMid meet' : 'xMidYMid slice'" aria-hidden="true"
      >
        <defs>
          <linearGradient id="pSky" x1="0" y1="0" x2="0" y2="1"><stop offset="0" stop-color="#f9e4cc" /><stop offset="1" stop-color="#fbf0e2" /></linearGradient>
          <linearGradient id="pGlass" x1="0" y1="0" x2="1" y2="1"><stop offset="0" stop-color="#fffaf2" /><stop offset="1" stop-color="#f7e8d4" /></linearGradient>
          <radialGradient id="pGold" cx="0.4" cy="0.35" r="0.75"><stop offset="0" stop-color="#f7d28c" /><stop offset="0.55" stop-color="#d98d3e" /><stop offset="1" stop-color="#9a5420" /></radialGradient>
          <radialGradient id="pLamp"><stop offset="0" stop-color="#ffe3a6" stop-opacity="0.9" /><stop offset="1" stop-color="#ffe3a6" stop-opacity="0" /></radialGradient>
          <pattern id="pTiles" width="80" height="40" patternUnits="userSpaceOnUse"><rect width="80" height="40" fill="#e6d2b6" /><path d="M0 40H80M40 0V40" stroke="#d5bd9c" stroke-width="2" /></pattern>
          <clipPath id="pWin"><rect x="425" y="365" width="620" height="390" /></clipPath>
        </defs>

        <g data-world>
          <rect width="1600" height="900" fill="url(#pSky)" />
          <!-- facade -->
          <rect x="260" y="70" width="1080" height="700" fill="#ecc2ae" />
          <rect x="245" y="62" width="1110" height="26" fill="#f8ead9" />
          <rect x="260" y="88" width="1080" height="10" fill="#d9a690" />
          <!-- sign -->
          <rect x="430" y="140" width="740" height="104" rx="6" fill="#b8325a" />
          <text x="800" y="208" text-anchor="middle" fill="#fbf4ea" font-family="Instrument Serif, serif" font-size="70" letter-spacing="6">MIEL.</text>
          <text x="800" y="235" text-anchor="middle" fill="#f6c9d3" font-family="Inter Variable, sans-serif" font-size="15" letter-spacing="6">PÂTISSERIE · BOULANGERIE</text>
          <!-- lamps -->
          <circle data-lamp cx="400" cy="300" r="60" fill="url(#pLamp)" /><circle data-lamp cx="1200" cy="300" r="60" fill="url(#pLamp)" />
          <path d="M392 262h16v22h-16z M1192 262h16v22h-16z" fill="#4a2a1a" /><circle cx="400" cy="294" r="10" fill="#ffe3a6" /><circle cx="1200" cy="294" r="10" fill="#ffe3a6" />
          <!-- awning: stripes + scallops -->
          <rect x="380" y="262" width="840" height="14" fill="#8f2444" />
          <g>
            <rect v-for="i in 17" :key="'s' + i" :x="380 + (i - 1) * 49.4" y="276" width="49.4" height="56" :fill="i % 2 ? '#b8325a' : '#fbf4ea'" />
            <circle v-for="i in 17" :key="'c' + i" :cx="380 + (i - 1) * 49.4 + 24.7" cy="332" r="24.7" :fill="i % 2 ? '#b8325a' : '#fbf4ea'" />
          </g>
          <!-- display window -->
          <rect x="410" y="352" width="650" height="418" fill="#4a2a1a" />
          <rect x="425" y="365" width="620" height="390" fill="url(#pGlass)" />
          <g clip-path="url(#pWin)">
            <!-- shelves -->
            <rect x="425" y="488" width="620" height="9" fill="#c9a27a" /><rect x="425" y="608" width="620" height="9" fill="#c9a27a" /><rect x="425" y="728" width="620" height="9" fill="#c9a27a" />
            <!-- shelf 1: macarons + éclairs -->
            <g v-for="(c, i) in macarons" :key="'m' + i" :transform="`translate(${478 + i * 30} 462)`">
              <rect x="-12" y="0" width="24" height="9" rx="4.5" :fill="c" /><rect x="-11" y="8" width="22" height="4" fill="#fff3e0" /><rect x="-12" y="11" width="24" height="9" rx="4.5" :fill="c" />
            </g>
            <g>
              <rect x="830" y="456" width="120" height="16" rx="8" fill="#e3b06d" /><rect x="834" y="452" width="112" height="9" rx="4.5" fill="#8fb46a" />
              <rect x="838" y="472" width="118" height="16" rx="8" fill="#e3b06d" /><rect x="842" y="468" width="110" height="9" rx="4.5" fill="#5a3322" />
            </g>
            <!-- shelf 2: tart, croissant, pain au chocolat -->
            <ellipse cx="560" cy="600" rx="56" ry="12" fill="#c9772f" /><rect x="505" y="584" width="110" height="16" rx="6" fill="#d98d3e" />
            <ellipse cx="560" cy="584" rx="52" ry="9" fill="#fff1d6" />
            <circle v-for="i in 7" :key="'r' + i" :cx="518 + i * 11" :cy="580 - (i % 2) * 3" r="6" fill="#d6456b" />
            <g transform="translate(735 585) scale(0.14) translate(-800 -480)">
              <g v-for="(s, i) in segs" :key="'mini' + i" :transform="`rotate(${s[4]} ${s[0]} ${s[1]})`"><ellipse :cx="s[0]" :cy="s[1]" :rx="s[2]" :ry="s[3]" fill="url(#pGold)" /></g>
            </g>
            <rect x="860" y="574" width="90" height="34" rx="10" fill="#e0a256" /><path d="M880 580v22M928 580v22" stroke="#4a2a1a" stroke-width="5" stroke-linecap="round" />
            <!-- shelf 3: opera slice, entremet -->
            <g><rect x="515" y="676" width="92" height="52" fill="#3b2418" /><rect x="515" y="688" width="92" height="6" fill="#c9a27a" /><rect x="515" y="704" width="92" height="6" fill="#c9a27a" /><rect x="515" y="676" width="92" height="6" fill="#1f120b" /></g>
            <path d="M820 728 A60 52 0 0 1 940 728 Z" fill="#b8325a" /><path d="M836 708 A44 30 0 0 1 870 690" stroke="#fbd3de" stroke-width="6" fill="none" stroke-linecap="round" />
            <circle cx="880" cy="672" r="9" fill="#d6456b" /><path d="M880 664l4-8" stroke="#8fb46a" stroke-width="3" />
            <!-- glass reflection -->
            <polygon points="460,365 560,365 440,755 425,755 425,430" fill="#ffffff" opacity="0.35" />
            <polygon points="960,365 1000,365 900,755 860,755" fill="#ffffff" opacity="0.25" />
          </g>
          <!-- door with arched glass + open sign -->
          <rect x="1100" y="372" width="160" height="398" fill="#4a2a1a" />
          <path d="M1116 520 V430 A64 64 0 0 1 1244 430 V520 Z" fill="url(#pGlass)" />
          <rect x="1116" y="534" width="128" height="220" fill="#6d3b26" /><rect x="1228" y="600" width="6" height="40" rx="3" fill="#e9b96e" />
          <path d="M1150 470 L1180 448 L1210 470" stroke="#4a2a1a" stroke-width="2" fill="none" />
          <rect x="1145" y="470" width="70" height="30" rx="4" fill="#fbf4ea" /><text x="1180" y="491" text-anchor="middle" font-family="Inter Variable, sans-serif" font-size="14" font-weight="700" fill="#b8325a">OUVERT</text>
          <!-- planters + coffee cup on a bistro table -->
          <rect x="290" y="700" width="100" height="70" fill="#8fb46a" /><circle v-for="i in 5" :key="'f' + i" :cx="290 + i * 18" :cy="690 - (i % 2) * 10" r="14" :fill="i % 2 ? '#d6456b' : '#f6dc7a'" />
          <rect x="1280" y="700" width="50" height="70" fill="#8fb46a" /><circle cx="1295" cy="690" r="14" fill="#f2a3b5" /><circle cx="1318" cy="682" r="14" fill="#fbf4ea" />
          <!-- pavement -->
          <rect x="0" y="770" width="1600" height="130" fill="url(#pTiles)" />
          <rect x="1390" y="740" width="110" height="10" rx="5" fill="#4a2a1a" /><rect x="1440" y="750" width="8" height="90" fill="#4a2a1a" />
          <rect x="1428" y="716" width="28" height="24" rx="4" fill="#fbf4ea" />
          <path data-steam d="M1436 708 q6 -10 0 -20" stroke="#fbf4ea" stroke-width="3" fill="none" stroke-linecap="round" />
          <path data-steam d="M1448 708 q6 -10 0 -20" stroke="#fbf4ea" stroke-width="3" fill="none" stroke-linecap="round" />
        </g>

        <template v-if="!narrow">
          <g v-for="l in shopLabels" :key="l.t" data-l="shop" :opacity="reduced ? 1 : 0" font-family="Inter Variable, sans-serif">
            <line :x1="l.x" :y1="l.y" :x2="l.tx + 10" :y2="l.ty + 30" stroke="#4a2a1a" stroke-opacity="0.5" stroke-width="1.5" />
            <circle :cx="l.x" :cy="l.y" r="7" fill="#b8325a" stroke="#fbf4ea" stroke-width="3" />
            <rect :x="l.tx" :y="l.ty" :width="pillW(l)" height="60" rx="10" fill="#fbf4ea" fill-opacity="0.95" stroke="#4a2a1a" stroke-opacity="0.15" />
            <text :x="l.tx + 18" :y="l.ty + 26" fill="#2e1a10" font-size="20" font-weight="600">{{ l.t }}</text>
            <text :x="l.tx + 18" :y="l.ty + 48" fill="#5e4436" font-size="15">{{ l.s }}</text>
          </g>
          <g v-if="!reduced">
            <g v-for="l in windowLabels" :key="l.t" data-l="window" opacity="0" font-family="Inter Variable, sans-serif">
              <line :x1="l.x" :y1="l.y" :x2="l.tx + 10" :y2="l.ty + 30" stroke="#4a2a1a" stroke-opacity="0.5" stroke-width="1.5" />
              <circle :cx="l.x" :cy="l.y" r="8" fill="#b8325a" stroke="#fbf4ea" stroke-width="3" />
              <rect :x="l.tx" :y="l.ty" :width="pillW(l)" height="60" rx="10" fill="#fbf4ea" fill-opacity="0.95" stroke="#4a2a1a" stroke-opacity="0.15" />
              <text :x="l.tx + 18" :y="l.ty + 26" fill="#2e1a10" font-size="20" font-weight="600">{{ l.t }}</text>
              <text :x="l.tx + 18" :y="l.ty + 48" fill="#5e4436" font-size="15">{{ l.s }}</text>
            </g>
          </g>
        </template>
      </svg>

      <!-- ============ MACRO CROISSANT ============ -->
      <svg
        :class="reduced ? 'block aspect-video w-full' : 'absolute inset-0 size-full'"
        viewBox="0 0 1600 900" :preserveAspectRatio="narrow ? 'xMidYMid meet' : 'xMidYMid slice'" aria-hidden="true"
        :style="reduced ? '' : 'pointer-events:none'"
      >
        <defs>
          <radialGradient id="mGold" cx="0.4" cy="0.3" r="0.8"><stop offset="0" stop-color="#f9d995" /><stop offset="0.5" stop-color="#dc9443" /><stop offset="1" stop-color="#9a5420" /></radialGradient>
          <linearGradient id="mMarble" x1="0" y1="0" x2="1" y2="1"><stop offset="0" stop-color="#f6f1ea" /><stop offset="1" stop-color="#e9e1d6" /></linearGradient>
          <radialGradient id="mShadow"><stop offset="0" stop-color="#4a2a1a" stop-opacity="0.35" /><stop offset="1" stop-color="#4a2a1a" stop-opacity="0" /></radialGradient>
        </defs>
        <g data-macro :opacity="reduced ? 1 : 0">
          <rect width="1600" height="900" fill="url(#mMarble)" />
          <g fill="none" stroke="#cfc4b6" stroke-width="2" opacity="0.7">
            <path d="M0 200 C 300 260 500 120 800 180 S 1300 300 1600 220" /><path d="M0 760 C 400 700 700 840 1100 760 S 1500 700 1600 740" /><path d="M200 0 C 260 200 180 400 260 900" stroke-width="1" />
          </g>
          <!-- plate -->
          <ellipse cx="800" cy="720" rx="560" ry="110" fill="url(#mShadow)" />
          <ellipse cx="800" cy="690" rx="540" ry="100" fill="#ffffff" /><ellipse cx="800" cy="690" rx="440" ry="72" fill="none" stroke="#efe6da" stroke-width="3" />
          <!-- croissant -->
          <g v-for="(s, i) in segs" :key="'seg' + i" :transform="`rotate(${s[4]} ${s[0]} ${s[1]})`">
            <ellipse :cx="s[0]" :cy="s[1]" :rx="s[2]" :ry="s[3]" fill="url(#mGold)" stroke="#8a4a1a" stroke-opacity="0.35" stroke-width="2" />
            <path
              v-for="k in [-0.45, 0, 0.45]" :key="k"
              :d="`M ${s[0] - s[2] * 0.86} ${s[1] + s[3] * k} Q ${s[0]} ${s[1] + s[3] * k + 16} ${s[0] + s[2] * 0.86} ${s[1] + s[3] * k}`"
              fill="none" stroke="#8a4a1a" stroke-opacity="0.35" stroke-width="3" stroke-linecap="round"
            />
            <ellipse :cx="s[0] - s[2] * 0.3" :cy="s[1] - s[3] * 0.45" :rx="s[2] * 0.35" :ry="s[3] * 0.18" fill="#fff6dc" opacity="0.45" />
          </g>
          <!-- torn piece showing honeycomb crumb -->
          <path d="M1180 700 Q1250 640 1330 690 Q1340 760 1260 780 Q1190 785 1180 700 Z" fill="#c9772f" />
          <path d="M1190 705 Q1252 660 1318 698 Q1322 752 1258 766 Q1200 770 1190 705 Z" fill="#f6dcae" />
          <ellipse v-for="(h, i) in holes" :key="'h' + i" :cx="h.x" :cy="h.y" :rx="h.rx" :ry="h.ry" fill="#e3b778" />
          <!-- butter pat + crumbs -->
          <rect x="280" y="740" width="110" height="46" rx="8" fill="#fbe7a1" /><rect x="280" y="740" width="110" height="12" rx="6" fill="#fff4c6" />
          <circle v-for="i in 9" :key="'cr' + i" :cx="560 + i * 47" :cy="770 + (i % 3) * 8" :r="3 + (i % 3)" fill="#d98d3e" />
        </g>
        <template v-if="!narrow">
          <g v-for="l in crumbLabels" :key="l.t" data-l="crumb" :opacity="reduced ? 1 : 0" font-family="Inter Variable, sans-serif">
            <line :x1="l.x" :y1="l.y" :x2="l.tx + 10" :y2="l.ty + 30" stroke="#4a2a1a" stroke-opacity="0.5" stroke-width="1.5" />
            <circle :cx="l.x" :cy="l.y" r="8" fill="#b8325a" stroke="#fbf4ea" stroke-width="3" />
            <rect :x="l.tx" :y="l.ty" :width="pillW(l)" height="60" rx="10" fill="#fbf4ea" fill-opacity="0.95" stroke="#4a2a1a" stroke-opacity="0.15" />
            <text :x="l.tx + 18" :y="l.ty + 26" fill="#2e1a10" font-size="20" font-weight="600">{{ l.t }}</text>
            <text :x="l.tx + 18" :y="l.ty + 48" fill="#5e4436" font-size="15">{{ l.s }}</text>
          </g>
        </template>
      </svg>

      <!-- Narrow screens: readable chips -->
      <template v-if="narrow && !reduced">
        <ul v-for="(set, key) in { shop: shopLabels, window: windowLabels, crumb: crumbLabels }" :key="key" class="absolute inset-x-4 bottom-20 flex flex-wrap gap-2" aria-hidden="true">
          <li v-for="l in set" :key="l.t" :data-l="key" class="rounded-full border border-line bg-ink/95 px-3 py-1.5 text-xs shadow-sm">
            <span class="font-semibold text-paper">{{ l.t }}</span> <span class="text-paper-dim">· {{ l.s }}</span>
          </li>
        </ul>
      </template>

      <div v-if="!reduced" data-flash class="pointer-events-none absolute inset-0 bg-[radial-gradient(circle_at_50%_55%,#fff8ea,#f2c98a_75%)]" />

      <!-- Intro -->
      <div v-if="!reduced" data-intro class="pointer-events-none absolute inset-0 bg-gradient-to-r from-[#fbf4ea]/95 via-[#fbf4ea]/60 to-transparent pt-32 sm:pt-40">
        <div class="container-x">
          <p class="eyebrow">MIEL. Pâtisserie · Baked at dawn</p>
          <h1 class="display mt-5 max-w-[12ch] text-[clamp(3rem,8vw,7.5rem)]">Butter, sugar, <span class="italic text-signal">patience.</span></h1>
          <p class="mt-6 max-w-md text-lg text-paper-dim">Croissants with twenty-seven layers, tarts finished by hand. Scroll to step up to the window — or ask our counter anything.</p>
          <Button size="lg" class="pointer-events-auto mt-8" :class="call.active ? '' : 'cta-pulse'" :disabled="call.active" @click="call.start()">
            <Mic :size="20" aria-hidden="true" />Ask the counter
          </Button>
        </div>
      </div>
      <p v-if="!reduced" data-scroll-hint class="absolute bottom-8 left-1/2 -translate-x-1/2 text-xs uppercase tracking-[0.3em] text-paper-dim">Scroll to the window</p>

      <div v-if="!reduced" class="pointer-events-none absolute bottom-8 left-0 right-0">
        <div class="container-x">
          <p data-chapter="1" class="absolute display text-3xl">01 <span class="text-paper-dim">— The shop</span></p>
          <p data-chapter="2" class="absolute display text-3xl">02 <span class="text-paper-dim">— The window</span></p>
          <p data-chapter="3" class="absolute display text-3xl">03 <span class="text-paper-dim">— The croissant</span></p>
        </div>
      </div>
    </div>

    <div v-if="reduced" class="container-x py-16">
      <h1 class="display text-6xl">Butter, sugar, <span class="italic text-signal">patience.</span></h1>
      <Button size="lg" class="mt-8" :disabled="call.active" @click="call.start()"><Mic :size="20" aria-hidden="true" />Ask the counter</Button>
    </div>
  </section>
</template>
