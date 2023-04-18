<template>
  <dl>
    <dt>线宽</dt>
    <dd>
      <input type="range" v-model.number="options.width" />
    </dd>
  </dl>
  <dl>
    <dt>颜色</dt>
    <dd>
      <input type="color" v-model="options.color">
    </dd>
  </dl>
  <dl>
    <dt>透明度</dt>
    <dd>
      <input type="range" v-model.number="options.opacity" min="0" max="1" step="0.1" />
    </dd>
  </dl>
  <dl>
    <dt>阴影</dt>
    <dd>
      <input type="color" v-model="options.shadowColor">
    </dd>
  </dl>
  <dl>
    <dt>阴影大小</dt>
    <dd>
      <input type="range" v-model.number="options.shadowWidth" />
    </dd>
  </dl>
  <dl>
    <dt>阴影距离</dt>
    <dd>
      <input type="range" v-model.number="options.shadowOffset" />
    </dd>
  </dl>
  <dl>
    <dt>开始箭头</dt>
    <dd>
      <input type="checkbox" v-model="options.startArrow" :true-value="true" :false-value="false" />
    </dd>
  </dl>
  <dl>
    <dt>结束箭头</dt>
    <dd>
      <input type="checkbox" v-model="options.endArrow" :true-value="true" :false-value="false" />
    </dd>
  </dl>
  <dl>
    <dt>添加背景</dt>
    <dd>
      <input type="checkbox" v-model="options.fillBackground" :true-value="true" :false-value="false" />
    </dd>
  </dl>
  <dl v-if="options.fillBackground">
    <dt>背景</dt>
    <dd>
      <input type="color" v-model="options.fillBackgroundColor">
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