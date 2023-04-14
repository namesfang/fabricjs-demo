<template>
  <div class="container">
    <div class="tile">
      <button @click="togglePencil" type="button" :class="{ light }">
        画笔
      </button>
      <button @click="addText" type="button">文字</button>
    </div>
    <div class="workspace">
      <canvas ref="canvasRef"></canvas>
    </div>
    <div v-if="tile.type" class="prop">
      <Pencil v-if="tile.type == 'pencil'" :options="options" />
      <Text v-if="tile.type == 'text'" :options="options" />
    </div>
    <div v-else class="prop empty">请选择元素</div>
  </div>
  <teleport to="body">
    <ul @mouseenter="enter" @mouseleave="leave" class="contextmenu" ref="contextmenuRef">
      <li @click="clone">克隆</li>
      <li @click="copy">复制</li>
      <li @click="paste">粘贴</li>
      <li @click="lock">锁定/解锁</li>
      <li @click="visible">隐藏/显示</li>
      <li @click="remove" class="remove">删除</li>
      <li @click="clear">清空</li>
      <li @click="arrow(true)">添加开始箭头</li>
      <li @click="arrow(false)">添加结束箭头</li>
    </ul>
  </teleport>
</template>

<script setup>

import { fabric } from 'fabric';

import { reactive, ref, watch } from 'vue';

import background from './assets/images/bg.png';
import Pencil from './widgets/Pencil.vue';
import Text from './widgets/Text.vue';

let { clientHeight, clientWidth } = document.body;

let canvas = null

const canvasRef = ref()

const contextmenuRef = ref()

const light = ref(false)

const tile = ref({})


const options = reactive({
  opacity: 1,
  width: 3,
  color: '#3399ff',
  fill: '#3399ff',
  fontSize: 20,
  shadowColor: '#3399ff',
  shadowWidth: 0,
  shadowOffset: 0,
})

const togglePencil = ()=> {
  tile.value.type = 'pencil'
  canvas.isDrawingMode = !canvas.isDrawingMode;
  light.value = canvas.isDrawingMode;

  canvas.freeDrawingBrush.perPixelTargetFind = false;
  canvas.freeDrawingBrush.color = options.color;
  canvas.freeDrawingBrush.width = options.width;
  canvas.freeDrawingBrush.opacity = options.opacity;
  canvas.freeDrawingBrush.decimate = 50

  canvas.freeDrawingBrush.shadow = new fabric.Shadow({
    blur: options.shadowWidth,
    offsetX: options.shadowOffset,
    offsetY: options.shadowOffset,
    affectStroke: true,
    color: options.shadowColor,
  })
}

let textElement = null
const addText = ()=> {
  tile.value.type = 'text'
  canvas.isDrawingMode = false;
  light.value = false;

  textElement = new fabric.IText('请文本输入', {
    fill: options.fill,
    top: clientHeight / 2,
    left: clientWidth / 2 - 320
  })

  canvas.add(textElement)
}

const clone = ()=> {
  
  let element = canvas.getActiveObject();
  if(!element) {
    return
  }

  element.clone((cloned)=> {
    cloned.set({
      top: cloned.top + 50,
      left: cloned.left + 50,
    })
    canvas.add(cloned)
    canvas.setActiveObject(cloned)
    canvas.bringForward(cloned)
  })
}

let cloneElement = null;
let contextmenuPoint = {
  top: 0,
  left: 0
}
const copy = ()=> {
  canvas.getActiveObject().clone((cloned)=> {
    cloneElement = cloned;
  })
}

const paste = ()=> {
  if(!cloneElement) {
    return;
  }
  cloneElement.set(contextmenuPoint)
  if(cloneElement.type === 'activeSelection') {
    cloneElement.canvas = canvas;
    cloneElement.forEachObject((t)=> canvas.add(t))
    cloneElement.setCoords()
  } else {
    canvas.add(cloneElement)
  }
  canvas.setActiveObject(cloneElement)
  canvas.bringForward(cloneElement)
  cloneElement = null;
}

const lock = ()=> {
  let ao = canvas.getActiveObject();
  console.log(ao.lockMovementX)
  if(ao.lockMovementX) {
    ao.set('lockMovementX', false);
    ao.set('lockMovementY', false);
    ao.set('hasBorders', true);
    ao.set('hasControls', true);
  } else {
    ao.set('lockMovementX', true);
    ao.set('lockMovementY', true);
    ao.set('hasBorders', false);
    ao.set('hasControls', false);
  }
  canvas.renderAll()
}

const visible = ()=> {
  let ao = canvas.getActiveObject();
  ao.set('opacity', ao.opacity === 1 ? 0 : 1);
  canvas.renderAll()
}

const arrow = (start) => {
  let element = canvas.getActiveObject();
  if(!element) {
    return;
  }

  let arrow = new fabric.Triangle({
    width: 10,
    height: 20,
    fill: element.stroke
  })

  if(element.type === 'group') {
    let objects = element.getObjects();
    // 定位 线的索引
    let index = objects.findIndex((t)=> t.type === 'path')

    if(length === 3) {
     // return;
    }

    // 已添加开始箭头则退出 ['triangle', 'path']
    if(index === 1 && start) {
      return;
    }
    // 已添加结束箭头则退出 ['path', 'triangle']
    if(index == 0 && start === false) {
      return;
    }

    let pathElement = objects.at(index);

    // 颜色
    arrow.fill = pathElement.stroke;

    if(start) {
      arrow.left = pathElement.path.at(0).at(1) - 5
      arrow.top = pathElement.path.at(0).at(2) - 20
      element.insertAt(arrow)
    } else {
      arrow.left = pathElement.path.at(-1).at(1) - 5
      arrow.top = pathElement.path.at(-1).at(2)
      element.addWithUpdate(arrow)
    }
    canvas.renderAll()
    return;
  }

  element.clone((cloned)=> {
    let elements = [
      cloned
    ]
    
    if(start) {
      arrow.left = element.path.at(0).at(1) - 5
      arrow.top = element.path.at(0).at(2) - 20
      elements.unshift(arrow)
    } else {
      arrow.left = element.path.at(-1).at(1) - 5
      arrow.top = element.path.at(-1).at(2) - 20
      elements.push(arrow)
    }

    let el = new fabric.Group(elements, {
      top: element.top,
      left: element.left
    })
    canvas.add(el)
    canvas.setActiveObject(el)
    canvas.bringForward(el)

    canvas.remove(element)
  })
}

const remove = ()=> {
  canvas.remove(canvas.getActiveObject())
}

const clear = ()=> {
  canvas.forEachObject((t)=> {
    canvas.remove(t)
  })

  tile.value.type = '';
  canvas.isDrawingMode = false;
  light.value = false;
  contextmenuRef.value.style.display = 'none'
}

const hover = ref(false)
const enter = ()=> {
  hover.value = true;
}

const leave = ()=> {
  hover.value = false;
}

document.body.addEventListener('click', ()=> {
  contextmenuRef.value.style.display = 'none'
})

watch(canvasRef, ()=> {

    fabric.Object.prototype.transparentCorners = false;

    canvas = new fabric.Canvas(canvasRef.value, {
      width: clientWidth - 640,
      height: clientHeight,
      fireRightClick: true,
      stopContextMenu: true
    })

    canvas.setBackgroundColor({
      originX: 'left',
      originY: 'top',
      repeat: 'repeat',
      source: background
    }, canvas.renderAll.bind(canvas))

    // 右键
    canvas.on('mouse:up', ({ button, target, e })=> {

      if(button === 3) {
        light.value = false;
        canvas.isDrawingMode = false;

        const find = canvas.findTarget(e, false);

        if(find) {
          canvas.setActiveObject(find)
          canvas.bringForward(find)
        }

        contextmenuPoint.top = e.clientY
        contextmenuPoint.left = e.clientX - 340

        contextmenuRef.value.style.display = 'block'
        contextmenuRef.value.style.left = `${e.clientX}px`
        contextmenuRef.value.style.top = `${e.clientY}px`
      }
    })
})

watch(
  ()=> options.fill,
  (value)=> {
    let element = canvas.getActiveObject();
    if(element) {
      element.set('fill', value)
      canvas.renderAll(element)
    }
  }
)
watch(
  ()=> options.fontSize,
  (value)=> {
    let element = canvas.getActiveObject();
    if(element) {
      element.fontSize = value
      canvas.renderAll(element)
    }
  }
)

watch(
  ()=> options.opacity,
  (value)=> {
    let element = canvas.getActiveObject();
    if(element) {
      if(element.type == 'path') {
        element.set('opacity', value)
      } else {
        element.opacity = value
      }
      canvas.renderAll(element)
    }
  }
)

watch(
  ()=> options.color,
  (value)=> {
    let element = canvas.getActiveObject();
    if(element) {
      element.set('stroke', value)
      canvas.renderAll(element)
    } else {
      canvas.freeDrawingBrush.color = value
    }
  }
)

watch(
  ()=> options.width,
  (value)=> {
    let element = canvas.getActiveObject();
    if(element) {
      element.set('strokeWidth', value)
      canvas.renderAll(element)
    } else {
      canvas.freeDrawingBrush.width = value
    }
  }
)

watch(
  ()=> options.shadowWidth,
  (value)=> {
    let element = canvas.getActiveObject();
    if(element) {
      element.shadow.blur = value
      canvas.renderAll(element)
    } else {
      canvas.freeDrawingBrush.shadow.blur = value
    }
  }
)

watch(
  ()=> options.shadowColor,
  (value)=> {
    let element = canvas.getActiveObject();
    if(element) {
      element.shadow.color = value
      canvas.renderAll(element)
    } else {
      canvas.freeDrawingBrush.shadow.color = value
    }
  }
)

watch(
  ()=> options.shadowOffset,
  (value)=> {
    let element = canvas.getActiveObject();
    if(element) {
      element.shadow.offsetX = value
      element.shadow.offsetY = value
      canvas.renderAll(element)
    } else {
      canvas.freeDrawingBrush.shadow.offsetX = value
      canvas.freeDrawingBrush.shadow.offsetY = value
    }
  }
)
</script>

<style lang="scss">
html {
  height: 100%;
}
body {
  height: 100%;
  margin: 0;
  #app {
    height: 100%;
  }
}

.container {
  height: 100%;
  overflow: hidden;
  display: flex;

  .tile {
    width: 320px;
    padding: 20px 32px;
    button {
      width: 100%;
      border: 0;
      color: #fff;
      padding: 10px 0;
      background-color: rgb(40, 144, 255);
      margin-top: 10px;
      position: relative;
    }
    .light {
      &::after {
        content: '';
        width: 10px;
        height: 10px;
        border-radius: 50%;
        border: 1px solid #fff;
        background-color: rgb(30, 207, 39);
        position: absolute;
        top: 50%;
        left: 50%;
        margin: -5px 0 0 25px;
      }
    }
  }

  .workspace {
    flex: 1;
    background-color: #fafafa;
  }

  .prop {
    width: 320px;
    background-color: #f2f2f2;
    box-sizing: border-box;
    padding: 20px 32px;
    dl {
      height: 30px;
      margin: 20px 0 0 0;
      padding: 0;
      float: left;
      dt {
        width: 60px;
        padding: 0;
        margin: 0;
        font-size: 14px;
        color: #686868;
        float: left;
        margin-right: 20px;
      }
      dd {
        margin: 0;
        padding: 0;
        float: left;

        span {
          width: 80px;
          float: left;
          color: #a9a9a9;
        }

        input {
          float: left;
        }
      }
    }
  }

  .empty {
    text-align: center;
  }
}

.contextmenu {
  width: 130px;
  height: auto;
  overflow: hidden;
  background-color: #fff;
  border: 1px solid #f2f2f2;
  border-radius: 4px;
  box-shadow: 0 0 10px #aaa;
  position: fixed;
  left: 10px;
  top: 0;
  padding: 8px 0;
  margin: 0;
  display: none;
  li {
    line-height: 32px;
    font-size: 14px;
    padding: 0 18px;
    cursor: pointer;
    &:hover {
      background-color: #f2f2f2;
    }
  }
  .remove {
    color: #ff0000;
  }
}
</style>
