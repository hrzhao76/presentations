<template>
  <div class="slidev-layout relative">
    <slot />

    <div
      v-if="$frontmatter?.credit"
      class="custom-slide-credit"
      v-html="$frontmatter.credit"
    />
  </div>
</template>

<style scoped>
.custom-slide-credit {
  position: absolute;
  /* 距离左边 16px */
  left: 18px; 
  /* 关键：距离底部 45px，这样就不会被 global-bottom 的 footer 挡住了 */
  bottom: 30px; 

  font-size: 11px;
  line-height: 1.3;
  opacity: 0.6;
  max-width: 40vw;
  color: var(--slidev-text-color);
  
  /* 提高层级，确保在最上层 */
  z-index: 110; 
  pointer-events: auto;
}

/* 即使在深色模式下也能看清 */
.custom-slide-credit :deep(a) {
  color: currentColor;
  text-decoration: underline;
  opacity: 0.8;
}
</style>