<template>
  <div class="slidev-layout default">
    <slot />
    <div class="absolute bottom-4 right-4 text-sm opacity-60">
      {{ $slidev.nav.currentPage }}
    </div>
  </div>
</template>
