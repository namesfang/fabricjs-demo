<template>
  <dl>
    <dt>大小</dt>
    <dd>
      <input type="range" v-model.number="options.fontSize" />
    </dd>
  </dl>
  <dl>
    <dt>颜色</dt>
    <dd>
      <input type="color" v-model="options.fill">
    </dd>
  </dl>
  <dl>
    <dt>透明度</dt>
    <dd>
      <input type="range" v-model.number="options.opacity" min="0" max="1" step="0.1" />
    </dd>
  </dl>
</template>

<script setup>
const props = defineProps({
  options: {
    type: Object,
    requred: true
  }
})
</script>