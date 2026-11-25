<script setup lang="ts">
import { useSmoothScroll } from '@/composables/useMotion'
import SiteNav from '@/components/layout/SiteNav.vue'
import SiteFooter from '@/components/layout/SiteFooter.vue'
import PastryTour from '@/components/tour/PastryTour.vue'
import LayersSection from '@/components/sections/LayersSection.vue'
import MenuSection from '@/components/sections/MenuSection.vue'
import HowItWorksSection from '@/components/sections/HowItWorksSection.vue'
import ContactSection from '@/components/sections/ContactSection.vue'
import CallDock from '@/components/voice/CallDock.vue'

useSmoothScroll()
</script>

<template>
  <SiteNav />
  <main id="main" tabindex="-1" class="outline-none">
    <PastryTour /><LayersSection /><MenuSection /><HowItWorksSection /><ContactSection />
  </main>
  <SiteFooter />
  <CallDock />
</template>
