<!-- global-bottom.vue -->
<template>
  <footer
    v-if="$slidev.nav.currentLayout !== 'cover'
      && $slidev.nav.currentLayout !== 'center'
      && $slidev.nav.currentPage !== $slidev.nav.total"
    class="absolute bottom-0 left-0 w-full px-4 py-1 text-gray-600"
  >
    <div class="grid grid-cols-3 items-center text-sm">
      <!-- 左 -->
      <div class="text-left">
        Haoran Zhao
      </div>

      <!-- 中 -->
      <div class="text-center">
        Seminar
      </div>

      <!-- 右 -->
      <div class="text-right">
        {{ $slidev.nav.currentPage }}
      </div>
    </div>
  </footer>
</template>

<script setup lang="ts">
</script>
