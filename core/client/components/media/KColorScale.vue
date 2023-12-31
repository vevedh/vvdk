<template>
  <div>
    <q-resize-observer @resize="onResized" />
    <canvas
      class="k-color-scale"
      :id="canvasId"
      />
  </div>
</template>

<script setup>
import _ from 'lodash'
import { computed, watch, onMounted } from 'vue'
import { uid } from 'quasar'
import chroma from 'chroma-js'

// props
const props = defineProps({
  label: {
    type: String,
    default: undefined
  },
  colors: {
    type: String,
    default: 'OrRd'
  },
  domain: {
    type: Array,
    default: () => [0, 1]
  },
  classes: {
    type: Array,
    default: () => null
  },
  layout: {
    type: Object,
    default: () => {
      return {
        gutter: 4,
        label: {
          size: 12,
          font: 'Arial',
          color: 'black',
          align: 'left'
        },
        bar: {
          height: undefined,
          width: undefined
        },
        ticks: {
          size: 10,
          font: 'Arial',
          color: 'black'
        }
      }
    }
  },
  direction: {
    type: String,
    default: 'horizontal',
    validator: (value) => {
      return ['horizontal', 'vertical'].includes(value)
    }
  }
})

// data
const canvasId = uid()
let canvas = null
let canvasContext = null
let expectedSize = null
const callRefresh = _.debounce(() => { refresh() }, 200)

// computed
const labelSize = computed(() => {
  return _.get(props.layout, 'label.size', 12)
})
const labelFont = computed(() => {
  return labelSize.value + 'px ' + _.get(props.layout, 'label.font', 'Arial')
})
const labelColor = computed(() => {
  return _.get(props.layout, 'label.color', 'black')
})
const labelAlign = computed(() => {
  return _.get(props.layout, 'label.align', 'left')
})
const barHeight = computed(() => {
  return _.get(props.layout, 'bar.height', props.direction === 'horizontal' ? 16 : undefined)
})
const barWidth = computed(() => {
  return _.get(props.layout, 'bar.width', props.direction === 'vertical' ? 16 : undefined)
})
const ticksSize = computed(() => {
  return _.get(props.layout, 'ticks.size', 8)
})
const ticksFont = computed(() => {
  return ticksSize.value + 'px ' + _.get(props.layout, 'ticks.font', 'Arial')
})
const ticksColor = computed(() => {
  return _.get(props.layout, 'ticks.color', 'black')
})
const gutter = computed(() => {
  return _.get(props.layout, 'gutter', 4)
})
// functions
function drawLabel () {
  if (_.isNil(props.label)) return
  canvasContext.font = labelFont.value
  canvasContext.fillStyle = labelColor.value
  canvasContext.textAlign = labelAlign.value
  let xLabel = canvas.width
  switch (labelAlign.value) {
    case 'left':
      xLabel = 0
      break
    case 'center':
      xLabel = canvas.width / 2
      break
    default:
  }
  canvasContext.fillText(props.label, xLabel, labelSize.value)
}
function drawDiscreteHorizontalScale () {
  drawLabel()
  // draw colorbar
  const yBar = _.isNil(props.label) ? 0 : labelSize.value + gutter.value
  const length = props.classes.length - 1
  const boxWidth = canvas.width / length
  const classToColor = chroma.scale(props.colors).classes(props.classes)
  for (let i = 0; i < length; i++) {
    canvasContext.fillStyle = classToColor(props.classes[i])
    canvasContext.fillRect(i * boxWidth, yBar, boxWidth, barHeight.value)
  }
  // draw ticks
  const yTicks = yBar + barHeight.value + gutter.value + ticksSize.value
  canvasContext.font = ticksFont.value
  canvasContext.fillStyle = ticksColor.value
  for (let i = 0; i < props.classes.length; ++i) {
    let tick
    if (i === 0) {
      if (props.classes[i] !== Number.MIN_VALUE) {
        canvasContext.textAlign = 'left'
        tick = props.classes[i]
      }
    } else if (i === props.classes.length - 1) {
      if (props.classes[i] !== Number.MAX_VALUE) {
        canvasContext.textAlign = 'right'
        tick = props.classes[i]
      }
    } else {
      canvasContext.textAlign = 'center'
      tick = props.classes[i]
    }
    if (tick) canvasContext.fillText(tick, i * boxWidth, yTicks)
  }
}
function drawDiscreteVerticalScale () {
  drawLabel()
  // draw colorbar
  const yBar = _.isNil(props.label) ? 0 : labelSize.value + gutter.value
  const length = props.classes.length - 1
  const boxHeight = (canvas.height - yBar) / length
  const classToColor = chroma.scale(props.colors).classes(props.classes)
  for (let i = 0; i < length; i++) {
    canvasContext.fillStyle = classToColor(props.classes[i])
    canvasContext.fillRect(0, yBar + (length - i - 1) * boxHeight, barWidth.value, boxHeight)
  }
  // draw ticks
  canvasContext.font = ticksFont.value
  canvasContext.fillStyle = ticksColor.value
  canvasContext.textAlign = 'left'
  const x = barWidth.value + gutter.value
  for (let i = 0; i < props.classes.length; ++i) {
    let tick
    if (i === 0) {
      if (props.classes[i] !== Number.MIN_VALUE) tick = props.classes[i]
    } else if (i === props.classes.length - 1) {
      if (props.classes[i] !== Number.MAX_VALUE) tick = props.classes[i]
    } else tick = props.classes[i]
    if (tick) canvasContext.fillText(tick, x, yBar + (length - i) * boxHeight + ticksSize.value / 2)
  }
}
function drawContinuousHorizontalScale () {
  drawLabel()
  // draw colorbar
  const yBar = _.isNil(props.label) ? 0 : labelSize.value + gutter.value
  const length = canvas.width
  const colors = chroma.scale(props.colors).colors(length)
  for (let i = 0; i < length; i++) {
    canvasContext.fillStyle = colors[i]
    canvasContext.fillRect(i, yBar, 1, barHeight.value)
  }
  // draw ticks
  const yTicks = yBar + barHeight.value + gutter.value + ticksSize.value
  canvasContext.font = ticksFont.value
  canvasContext.fillStyle = ticksColor.value
  canvasContext.textAlign = 'left'
  canvasContext.fillText(props.domain[0], 0, yTicks)
  canvasContext.textAlign = 'right'
  canvasContext.fillText(props.domain[1], canvas.width, yTicks)
}
function drawContinuousVerticalScale () {
  drawLabel()
  // draw colorbar
  const yBar = _.isNil(props.label) ? 0 : labelSize.value + gutter.value
  const length = canvas.height - yBar
  const colors = chroma.scale(props.colors).colors(length)
  for (let i = 0; i < length; i++) {
    canvasContext.fillStyle = colors[i]
    canvasContext.fillRect(0, yBar + length - i, barWidth.value, 1)
  }
  // draw ticks
  canvasContext.font = ticksFont.value
  canvasContext.fillStyle = ticksColor.value
  const x = barWidth.value + gutter.value
  canvasContext.textAlign = 'left'
  canvasContext.fillText(props.domain[0], x, canvas.height)
  canvasContext.fillText(props.domain[1], x, yBar + ticksSize.value)
}
function refresh () {
  if (!canvas || !expectedSize) return
  if (canvas.width !== expectedSize.width) canvas.width = expectedSize.width
  if (canvas.height !== expectedSize.height) canvas.height = expectedSize.height
  canvasContext.clearRect(0, 0, canvas.width, canvas.height)
  canvasContext.fillText(props.label, canvas.width / 2, labelFont.value)
  if (props.classes) {
    if (props.direction === 'horizontal') drawDiscreteHorizontalScale()
    else drawDiscreteVerticalScale()
  } else {
    if (props.direction === 'horizontal') drawContinuousHorizontalScale()
    else drawContinuousVerticalScale()
  }
}
function onResized (size) {
  expectedSize = size
  if (canvas) callRefresh()
}

// watch
watch(props, () => { if (canvas) refresh() })

// hooks
onMounted(() => {
  canvas = document.getElementById(canvasId)
  canvasContext = canvas.getContext('2d')
  refresh()
})
</script>

<style lang="scss" scoped>
  .k-color-scale {
    width: 100%;
  }
</style>
