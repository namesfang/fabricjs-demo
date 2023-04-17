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
  <dl>
    <dt>背景</dt>
    <dd>
      <input type="color" v-model="options.fontBackgroundColor">
    </dd>
  </dl>
  <dl>
    <dt>边框</dt>
    <dd>
      <input type="color" v-model="options.fontBorderColor">
    </dd>
  </dl>
  <dl>
    <dt>加粗</dt>
    <dd>
      <input type="checkbox" v-model="options.fontWeight" true-value="bold" false-value="normal"/>
    </dd>
  </dl>
  <dl>
    <dt>斜体</dt>
    <dd>
      <input type="checkbox" v-model="options.fontStyle" true-value="italic" false-value="normal"/>
    </dd>
  </dl>
  <dl>
    <dt>自动换行</dt>
    <dd>
      <input type="checkbox" v-model="options.fontAutobreak" :true-value="true" :false-value="false"/>
    </dd>
  </dl>
  <dl>
    <dt>文字</dt>
    <dd>
      <textarea @input="typing" v-model="text"></textarea>
    </dd>
  </dl>
</template>

<script setup>
import { ref, watch } from 'vue';

const props = defineProps({
  options: {
    type: Object,
    requred: true
  }
})

const emit = defineEmits([
  'typing'
])

const text = ref('请输入文本')

const typing = (e)=> {
  emit('typing', e.target.value)
}


const updateText = (value)=> {
  text.value = value  
}
defineExpose({
  updateText
})
</script>