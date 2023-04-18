<template>
  <div class="container">
    <div class="tile">
      <button @click="togglePencil" type="button" :class="{ light }">
        画笔
      </button>
      <!--
      <button @click="toggleArrowPencil" type="button" :class="{ light: lightArrow }">
        箭头画笔
      </button>
      -->
      <button @click="addText" type="button">文字</button>
    </div>
    <div class="workspace">
      <canvas ref="canvasRef"></canvas>
    </div>
    <div v-if="tile.type" class="prop">
      <Pencil v-if="tile.type == 'pencil'" :options="options" />
      <Text v-if="tile.type == 'text'" :options="options" @typing="typing" ref="textRef"/>
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

const textRef = ref()
const canvasRef = ref()

const contextmenuRef = ref()

const light = ref(false)
const lightArrow = ref(false)

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
  fontBackgroundColor: '#ffffff',
  fontBorderColor: '#ffffff',
  fontAutobreak: false,
  fontStyle: 'normal',
  fontWeight: 'normal',
  startArrow: false,
  endArrow: false,
  fillBackground: false,
  fillBackgroundColor: '#a7d1ff'
})

const initPencil = ()=> {
  canvas.isDrawingMode = !canvas.isDrawingMode;
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

const togglePencil = ()=> {
  initPencil()
  tile.value.type = 'pencil'
  light.value = canvas.isDrawingMode;
}

const toggleArrowPencil = ()=> {
  initPencil()
  tile.value.type = 'pencil'
  lightArrow.value = canvas.isDrawingMode;
}

// const coords2path = (coords) => {
//   let path = [];
//   coords.forEach((t)=> {
//     path.push(t.join(' '))
//   })
//   return path.join(' ')
// }

const activeObject = (e, callback)=> {
  canvas.isDrawingMode = false;

  let find = canvas.findTarget(e, false);

  if(find) {
    canvas.setActiveObject(find)
    canvas.bringForward(find)
    callback && callback(find)
  }
}

const _createArrowPath = (start, fromX, fromY, toX, toY, theta, headlen)=> {
  theta = typeof theta !== 'undefined' ? theta : 30
  headlen = typeof theta !== 'undefined' ? headlen : 10 
  // 计算各角度和对应的P2,P3坐标
  let angle = Math.atan2(fromY - toY, fromX - toX) * 90 / Math.PI,
  angle1 = (angle + theta) * Math.PI / 90,
  angle2 = (angle - theta) * Math.PI / 90,
  topX = headlen * Math.cos(angle1),
  topY = headlen * Math.sin(angle1),
  botX = headlen * Math.cos(angle2),    botY = headlen * Math.sin(angle2)
  let arrowX = fromX - topX,    arrowY = fromY - topY
  let path = ' M ' + fromX + ' ' + fromY
  path += ' L ' + toX + ' ' + toY

  if(start) {
    arrowX = toX - topX
    arrowY = toY - topY
  } else {
    arrowX = toX + topX
    arrowY = toY + topY
  }
  
  path += ' M ' + arrowX + ' ' + arrowY
  path += ' L ' + toX + ' ' + toY
  if(start) {
    arrowX = toX - botX
    arrowY = toY - botY
  } else {
    arrowX = toX + botX
    arrowY = toY + botY
  }
  
  path += ' L ' + arrowX + ' ' + arrowY
  return path
}

// const DROG_CIRCLE = 8;

// const createCircle = (left, top)=> {
//   let element = new fabric.Circle({
//     left,
//     top,
//     strokeWidth: 2,
//     radius: DROG_CIRCLE - 1,
//     // fill: '#fff',
//     // stroke: '#666',
//     fill: '#fff',
//     stroke: '#3399ff',
//     selectable: false,
//     hoverCursor: 'default',
//     // hasBorders: true, // 它指定是否使边框可见
//     hasControls: false, // 它指定是否禁用控件
//     opacity: 0.8,
//     centeredRotation: true,
//     originX: 'center',
//     originY: 'center',
//   });

//   canvas.add(element)
// }

// const createTriangle = (left, top)=> {
//   let element = new fabric.Triangle({
//     width: DROG_CIRCLE * 2,
//     height: DROG_CIRCLE * 2,
//     left: left,
//     top: top,
//     fill: '#39f',
//     opacity: 1,
//     centeredRotation: true,
//     originX: 'center',
//     originY: 'center',
//     // 禁止选中
//     selectable: true,
//     hasControls: false, // 当设置为 `false` 时，对象的控件不显示并且不能用于操作对象
//   });

//   canvas.add(element)
// }


let textElement = null
const addText = ()=> {
  tile.value.type = 'text'
  canvas.isDrawingMode = false;
  light.value = false;

  textElement = new fabric.Textbox('请输入文本', {
    width: 10,
    fill: options.fill,
    top: clientHeight / 2,
    left: clientWidth / 2 - 320,
    hasControls: false
  })

  textElement.onKeyUp = function(e) {
    textRef.value.updateText(this.text);
    canvas.renderAll(textElement)
  }

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

const drawingArrow = (element) => {
  // if(element.type === 'group') {
  //   let objects = element.getObjects();
  //   // 定位 线的索引
  //   let index = objects.findIndex((t)=> t.type === 'path')

  //   if(length === 3) {
  //    // return;
  //   }

  //   // 已添加开始箭头则退出 ['triangle', 'path']
  //   if(index === 1 && start) {
  //     return;
  //   }
  //   // 已添加结束箭头则退出 ['path', 'triangle']
  //   if(index == 0 && start === false) {
  //     return;
  //   }

  //   let pathElement = objects.at(index);

  //   // 颜色
  //   arrow.fill = pathElement.stroke;

  //   if(start) {
  //     arrow.left = pathElement.path.at(0).at(1) - 5
  //     arrow.top = pathElement.path.at(0).at(2) - 20
  //     element.insertAt(arrow)
  //   } else {
  //     arrow.left = pathElement.path.at(-1).at(1) - 5
  //     arrow.top = pathElement.path.at(-1).at(2)
  //     element.addWithUpdate(arrow)
  //   }
  //   canvas.renderAll()
  //   return;
  // }

  element.clone((cloned)=> {

    let fromX = cloned.path.at(-2).at(1)
    let fromY = cloned.path.at(-2).at(2)
    let toX = cloned.path.at(-1).at(1)
    let toY = cloned.path.at(-1).at(2)

    let arrow = new fabric.Path(_createArrowPath(fromX, fromY, toX, toY, 20, 20), {
      fill: 'transparent',
      stroke: options.color,
      strokeWidth: options.width,
      objectCaching: false,
      // 禁止选中
      selectable: false,
      hasBorders: false, //当设置为 `false` 时，对象的控制边界不会被渲染
      hasControls: false, // 当设置为 `false` 时，对象的控件不显示并且不能用于操作对象
      hoverCursor: 'default',
    })

    let elements = [
      cloned,
      arrow
    ]

    let el = new fabric.Group(elements, {
      top: element.top,
      left: element.left
    })

    // el = arrow;

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

const typing = (value)=> {
  let element = canvas.getActiveObject();
  if(element) {
    element.text = value
    canvas.renderAll(element)
  }
}

document.body.addEventListener('click', ()=> {
  if(contextmenuRef.value) {
    contextmenuRef.value.style.display = 'none'
  }
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
    canvas.on('mouse:up', ({ button, e })=> {
      if(button === 1) {
        if(lightArrow.value) {
          lightArrow.value = false;
          activeObject(e, (find)=> {
            drawingArrow(find)
          })
        }
        if(light.value) {
          light.value = false;
          activeObject(e)
        }
      }
      // 右击快捷方式
      else if(button === 3) {
        
        activeObject(e)

        light.value = false;
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
  ()=> options.fontBackgroundColor,
  (value)=> {
    let element = canvas.getActiveObject();
    if(element) {
      element.set('backgroundColor', value)
      canvas.renderAll(element)
    }
  }
)

watch(
  ()=> options.fontBorderColor,
  (value)=> {
    let element = canvas.getActiveObject();
    if(element) {
      element.borderColor = value
      canvas.renderAll(element)
    }
  }
)
watch(
  ()=> options.fontAutobreak,
  (value)=> {
    let element = canvas.getActiveObject();
    if(element) {
      if(value) {
        element.set('splitByGrapheme', true)
      } else {
        element.set('splitByGrapheme', false)
      }
      canvas.renderAll(element)
    }
  }
)
watch(
  ()=> options.fontStyle,
  (value)=> {
    let element = canvas.getActiveObject();
    if(element) {
      element.set('fontStyle', value)
      canvas.renderAll(element)
    }
  }
)
watch(
  ()=> options.fontWeight,
  (value)=> {
    let element = canvas.getActiveObject();
    if(element) {
      element.set('fontWeight', value)
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
      if(element.type === 'group') {
        element.getObjects().forEach((t)=> {
          t.set('stroke', value)
          canvas.renderAll(t)
        })
      } else {
        element.set('stroke', value)
        canvas.renderAll(element)
      }
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
      if(element.type === 'group') {
        element.getObjects().forEach((t)=> {
          t.set('strokeWidth', value)
        })
        canvas.renderAll()
      } else {
        element.set('strokeWidth', value)
        canvas.renderAll(element)
      }
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

const createArrow = (start, cloned)=> {
  let pos = start ? [0, 1] : [-2, -1]

  let a = pos.shift();
  let b = pos.shift();

  let fromX = cloned.path.at(a).at(1);
  let fromY = cloned.path.at(a).at(2);
  let toX = cloned.path.at(b).at(1);
  let toY = cloned.path.at(b).at(2);

  return new fabric.Path(_createArrowPath(start, fromX, fromY, toX, toY, 20, 20), {
    name: 'arrow',
    fill: 'transparent',
    stroke: options.color,
    strokeWidth: options.width,
    objectCaching: false,
    // 禁止选中
    selectable: false,
    hasBorders: false, //当设置为 `false` 时，对象的控制边界不会被渲染
    hasControls: false, // 当设置为 `false` 时，对象的控件不显示并且不能用于操作对象
    hoverCursor: 'default',
  })
}

const drawArrow = (add, start=true)=> {
  let element = canvas.getActiveObject();
  if(!element) {
    return;
  }

  if(element.type == 'group') {
    let elements = element.getObjects();
    let length = elements.length;
    let pencil = elements.find((t)=> t.name =='pencil')
    if(add) {

      let arrow = createArrow(start, pencil)

      if(start) {
        if(elements.at(0).type == 'arrow') {
          return;
        }
      } else {
        if(elements.at(-1).type == 'arrow') {
          return;
        }
      }

      start ? element.insertAt(arrow, 0) : element.addWithUpdate(arrow)

      canvas.renderAll(element)
    } else {
      let index = start ? 0 : 2;
      let el = element.item(index);

      if(el.name == 'arrow') {
        element.remove(el)
        canvas.renderAll(element)
      }

      if(length === 2) {
        canvas.add(pencil)
        canvas.setActiveObject(pencil)
        canvas.bringForward(pencil)
        canvas.remove(element)
      }
    }
    return;
  }

  element.clone((cloned)=> {

    let arrow = createArrow(start, cloned)

    cloned.name = 'pencil'
    let elements = [
      cloned,
    ]

    start ? elements.unshift(arrow) : elements.push(arrow)

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

// 开始箭头
watch(
  ()=> options.startArrow,
  (value)=> {
    drawArrow(value, true)
  }
)
// 结束箭头
watch(
  ()=> options.endArrow,
  (value)=> {
    drawArrow(value, false)
  }
)

watch(
  ()=> options.fillBackground,
  (value)=> {
    let element = canvas.getActiveObject();
    if(element) {
      if(value) {
        if(element.type =='group') {
          let elements = element.getObjects()
          for(let el of elements) {
            if(el.name === 'pencil') {
              el.set('fill', options.fillBackgroundColor)
            }
          }
        } else {
          element.set('fill', options.fillBackgroundColor)
        }
      } else {
        if(element.type =='group') {
          let elements = element.getObjects()
          for(let el of elements) {
            if(el.name === 'pencil') {
              el.set('fill', null)
            }
          }
        } else {
          element.set('fill', null)
        }
      }
      canvas.renderAll(element)
    }
  }
)

watch(
  ()=> options.fillBackgroundColor,
  (value)=> {
    let element = canvas.getActiveObject();
    if(element && options.fillBackground) {
      if(element.type === 'group') {
        let el = element.getObjects().find(t=> t.name === 'pencil')
        el.set('fill', value)
        canvas.renderAll(el)
      } else {
        element.set('fill', value)
        canvas.renderAll(element)
      }
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
      margin: 20px 0 0 0;
      padding: 0;
      display: flex;
      dt {
        width: 60px;
        padding: 0;
        margin: 0;
        font-size: 14px;
        color: #686868;
        margin-right: 20px;
      }
      dd {
        flex: 1;
        margin: 0;
        padding: 0;
        display: flex;
        align-items: center;

        span {
          width: 80px;
          color: #a9a9a9;
        }

        input {
          border: 0;
        }

        textarea {
          width: 100%;
          height: 200px;
          border: 1px solid #f2f2f2;
          resize: none;
          outline: 1px solid #8a8a8a;
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
