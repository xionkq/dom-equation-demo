<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'

// 定义连线数据结构类型
interface Connection {
  id: number
  from: { x: number; y: number }
  to: { x: number; y: number }
  fromType: string
  fromIndex: number
  toType: string
  toIndex: number
}

interface DragState {
  from: { x: number; y: number }
  fromType: string
  fromIndex: number
}

// 连线相关的响应式数据
const connections = ref<Connection[]>([]) // 已完成的连线
const currentDrag = ref<DragState | null>(null) // 当前拖拽状态
const mousePosition = ref({ x: 0, y: 0 }) // 鼠标位置
const questionContentRef = ref<HTMLDivElement | null>(null) // 问题内容区域的引用

// 题目数据（你可以根据实际需求修改）
const topOptions = ref([
  { id: 1, text: '快乐自己，影响他人' },
  { id: 2, text: '快乐自己，影响他人' },
  { id: 3, text: '快乐自己，影响他人' },
  { id: 4, text: '快乐自己，影响他人' },
  { id: 5, text: '快乐自己，影响他人' },
])

const bottomOptions = ref([
  { id: 'A', text: '快乐自己，影响他人' },
  { id: 'B', text: '快乐自己，影响他人' },
  { id: 'C', text: '快乐自己，影响他人' },
  { id: 'D', text: '快乐自己，影响他人' },
  { id: 'E', text: '快乐自己，影响他人' },
])

const tempLineStyle = computed(() => {
  if (!currentDrag.value) return { display: 'none' }

  const startPoint = {
    x: currentDrag.value.from.x,
    y: currentDrag.value.from.y,
  }
  const endPoint = { x: mousePosition.value.x, y: mousePosition.value.y }

  // 计算 form 到 to 的距离
  const distance = Math.sqrt(Math.pow(endPoint.x - startPoint.x, 2) + Math.pow(endPoint.y - startPoint.y, 2))
  // 计算 form-to 直线与垂直线的角度，单位 deg
  const angle = Math.atan2(endPoint.x - startPoint.x, endPoint.y - startPoint.y) * (180 / Math.PI)

  if (distance < 100) return { display: 'none' }

  return {
    '--border-bottom-width': distance - 100 + 'px',
    '--before-el-top': distance + 'px',
    left: startPoint.x + 'px',
    top: startPoint.y + 'px',
    transform: `rotate(${360 - angle}deg)`,
    transformOrigin: 'top center',
    paddingTop: '100px',
  }
})

// 获取小球中心坐标
function getBallPosition(element: HTMLElement) {
  const rect = element.getBoundingClientRect()
  const containerRect = questionContentRef.value?.getBoundingClientRect()

  if (!containerRect) {
    return { x: 0, y: 0 }
  }

  return {
    x: rect.left + rect.width / 2 - containerRect.left,
    y: rect.top + rect.height / 2 - containerRect.top,
  }
}

// 小球鼠标按下事件
function onBallMouseDown(event: MouseEvent, type: string, index: number) {
  event.preventDefault()

  const ballElement = event.currentTarget as HTMLElement
  const position = getBallPosition(ballElement)

  // // 检查是否已经有连线从这个球开始，如果有则删除
  // const existingConnectionIndex = connections.value.findIndex(
  //   (conn) => conn.fromType === type && conn.fromIndex === index
  // )

  // if (existingConnectionIndex !== -1) {
  //   connections.value.splice(existingConnectionIndex, 1)
  // }

  const containerRect = questionContentRef.value?.getBoundingClientRect()
  if (!containerRect) return

  mousePosition.value = {
    x: event.clientX - containerRect.left,
    y: event.clientY - containerRect.top,
  }

  currentDrag.value = {
    from: position,
    fromType: type,
    fromIndex: index,
  }

  // 添加小球激活状态
  ballElement.classList.add('dragging')
}

// 鼠标移动事件
function onMouseMove(event: MouseEvent) {
  if (!currentDrag.value) return

  const containerRect = questionContentRef.value?.getBoundingClientRect()
  if (!containerRect) return

  mousePosition.value = {
    x: event.clientX - containerRect.left,
    y: event.clientY - containerRect.top,
  }
}

// 鼠标释放事件
function onMouseUp(event: MouseEvent) {
  if (!currentDrag.value) return

  // 移除所有拖拽状态样式
  document.querySelectorAll('.option-item.dragging').forEach((el) => {
    el.classList.remove('dragging')
  })

  // 检查是否释放在有效的目标小球上
  const targetElement = document.elementFromPoint(event.clientX, event.clientY)
  const targetBall = targetElement?.closest('.option-item[data-type][data-index]') as HTMLElement

  // 释放目标处无小球则取消连线
  if (!targetBall) {
    currentDrag.value = null
    return
  }

  const targetType = targetBall.getAttribute('data-type')
  const targetIndexStr = targetBall.getAttribute('data-index')

  // 释放目标不是有效的小球则取消连线
  if (!targetType || !targetIndexStr) {
    currentDrag.value = null
    return
  }

  const targetIndex = parseInt(targetIndexStr)

  // 检查是否是有效连线（上下排互连，且不是同一个球），若同排则取消连线
  if (targetType === currentDrag.value.fromType) {
    currentDrag.value = null
    return
  }

  const targetPosition = getBallPosition(targetBall)

  // 检查目标球是否已经有连线，如果有则删除
  const existingTargetIndex = connections.value.findIndex(
    (conn) =>
      (conn.toType === targetType && conn.toIndex === targetIndex) ||
      (conn.fromType === targetType && conn.fromIndex === targetIndex)
  )

  if (existingTargetIndex !== -1) {
    connections.value.splice(existingTargetIndex, 1)
  }

  // 检查起始球是否已经有连线，如果有则删除
  const existingFromIndex = connections.value.findIndex(
    (conn) =>
      (conn.fromType === currentDrag.value?.fromType && conn.fromIndex === currentDrag.value?.fromIndex) ||
      (conn.toType === currentDrag.value?.fromType && conn.toIndex === currentDrag.value?.fromIndex)
  )

  if (existingFromIndex !== -1) {
    connections.value.splice(existingFromIndex, 1)
  }

  // 创建新连线
  const newConnection: Connection = {
    id: Date.now(),
    from: currentDrag.value.from,
    to: targetPosition,
    fromType: currentDrag.value.fromType,
    fromIndex: currentDrag.value.fromIndex,
    toType: targetType,
    toIndex: targetIndex,
  }

  connections.value.push(newConnection)

  currentDrag.value = null
}

// // 删除指定连线
// function removeConnection(connectionId: number) {
//   const index = connections.value.findIndex((conn) => conn.id === connectionId)
//   if (index !== -1) {
//     connections.value.splice(index, 1)
//   }
// }
//
// // 清空所有连线
// function clearAllConnections() {
//   connections.value = []
// }

function getLineStyle(connection: Connection) {
  const startPoint = {
    x: connection.from.x,
    // y: connection.from.y + (connection.fromType === 'top' ? 100 : -100),
    y: connection.from.y,
  }
  const endPoint = {
    x: connection.to.x,
    // y: connection.to.y + (connection.toType === 'top' ? 100 : -100),
    y: connection.to.y,
  }

  // 计算 form 到 to 两点之间的距离
  const distance = Math.round(
    Math.sqrt(Math.pow(endPoint.x - startPoint.x, 2) + Math.pow(endPoint.y - startPoint.y, 2))
  )
  // 计算 form-to 直线与垂直线的角度，单位 deg
  const angle = Math.round(Math.atan2(endPoint.x - startPoint.x, endPoint.y - startPoint.y) * (180 / Math.PI))

  return {
    '--distance': distance + 'px',
    '--border-bottom-width': `calc(var(--distance) - 200px)`,
    '--before-el-top': `calc(var(--distance) - 100px)`,
    left: startPoint.x + 'px',
    top: startPoint.y + 'px',
    transform: `rotate(${360 - angle}deg)`,
    transformOrigin: 'top center',
    paddingTop: '100px',
  }
}

onMounted(() => {
  // 防止拖拽时选中文本
  document.addEventListener('selectstart', (e) => {
    if (currentDrag.value) {
      e.preventDefault()
    }
  })
})
</script>

<template>
  <div class="choice-question" @mousemove="onMouseMove" @mouseup="onMouseUp">
    <div class="question-content" ref="questionContentRef">
      <div
        v-for="connection in connections"
        :key="connection.id"
        :style="getLineStyle(connection)"
        class="connection-line"
      ></div>

      <div v-if="currentDrag" :style="tempLineStyle" class="temp-line"></div>

      <div class="top">
        <div
          v-for="(option, index) in topOptions"
          :key="option.id"
          class="option-item"
          :data-type="'top'"
          :data-index="index"
          @mousedown="onBallMouseDown($event, 'top', index)"
        >
          <span>{{ option.text }}</span>
        </div>
      </div>
      <div class="bottom">
        <div
          v-for="(option, index) in bottomOptions"
          :key="option.id"
          class="option-item"
          :data-type="'bottom'"
          :data-index="index"
          @mousedown="onBallMouseDown($event, 'bottom', index)"
        >
          <span>{{ option.text }}</span>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.choice-question {
  width: 100%;
  height: 100%;
  overflow: hidden;
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
  user-select: none;

  .question-content {
    width: 1444px;
    height: 602px;
    margin-top: 42px;
    display: flex;
    flex-direction: column;
    align-items: center;
    position: relative;

    .temp-line,
    .connection-line {
      position: absolute;
      width: 0;
      height: 0;
      border-left: 4px solid transparent;
      border-right: 4px solid transparent;
      border-bottom: 150px solid #F9F8CA;
      border-bottom-width: var(--border-bottom-width);

      &::before {
        content: '';
        position: absolute;
        width: 10px;
        height: 10px;
        border-radius: 50%;
        background: #F9F8CA;
        top: calc(var(--before-el-top) - 5px);
        left: 50%;
        transform: translateX(-50%);
      }

      &::after {
        content: '';
        position: absolute;
        width: 16px;
        height: 16px;
        border-radius: 50%;
        background: #F9F8CA;
        top: calc(var(--before-el-top) - 8px);
        left: 50%;
        transform: translateX(-50%);
        filter: blur(3px);
      }
    }

    .connection-line {
      animation: dash 500ms;

      &::before {
        animation: dash-before 500ms;
      }

      &::after {
        animation: dash-after 500ms;
      }
    }

    .top,
    .bottom {
      display: flex;
      align-items: center;
      justify-content: space-between;
      width: 1325px;
      z-index: 2;

      .option-item {
        width: 185px;
        height: 185px;
        border-radius: 92.5px;
        background: linear-gradient(157deg, #6096d3 11.59%, #2c3ac3 78.99%);
        box-shadow: 0 4px 44px 0 #bfdee4 inset;
        display: flex;
        align-items: center;
        justify-content: center;
        font-size: 16px;
        font-weight: 500;
        color: #fff;
        cursor: pointer;
        transition: all 0.3s ease;
        position: relative;

        &:hover {
          transform: scale(1.05);
          box-shadow:
            0 4px 44px 0 #bfdee4 inset,
            0 0 20px rgba(255, 255, 255, 0.3);
        }

        &.dragging {
          transform: scale(1.1);
          box-shadow:
            0 4px 44px 0 #bfdee4 inset,
            0 0 30px rgba(255, 255, 255, 0.5);
          z-index: 10;
        }

        span {
          user-select: none;
          pointer-events: none;
        }
      }
    }

    .bottom {
      margin-top: 200px;
    }
  }
}


@keyframes dash {
  0% {
    border-bottom-width: 0;
  }

  100% {
    border-bottom-width: var(--border-bottom-width);
  }
}


@keyframes dash-before {
  0% {
    top: 100px;
  }

  100% {
    top: calc(var(--before-el-top) - 5px);
  }
}


@keyframes dash-after {
  0% {
    top: 100px;
  }

  100% {
    top: calc(var(--before-el-top) - 8px);
  }
}
</style>
