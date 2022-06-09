<script setup lang="ts">
import type { QRCode } from 'jsqr'
import jsQR from 'jsqr'
interface Props {
  drawOnfound?: boolean
  lineColor?: string
  lineWidth?: number
}
const props = withDefaults(defineProps<Props>(), {
  drawOnfound: true,
  lineColor: '#03C03C',
  lineWidth: 2,
})
const emits = defineEmits<{
  (e: 'scanned', code: QRCode): void
}>()
/* Ref */
const videoRef = ref<HTMLVideoElement | null>(null)
const canvasRef = ref<HTMLCanvasElement | null>(null)
/* Variable */
const canvas = ref<CanvasRenderingContext2D>()
const videoVisible = ref(false)
interface VideoWH {
  width: number
  height: number
}
const videoWH = computed<VideoWH>(() => {
  return {
    width: document.documentElement.clientWidth || document.body.clientWidth,
    height: document.documentElement.clientHeight || document.body.clientHeight,
  }
})
interface Point {
  x: number
  y: number
}
// 画线
function drawLine(begin: Point, end: Point) {
  if (canvas.value) {
    canvas.value.beginPath()
    canvas.value.moveTo(begin.x, begin.y)
    canvas.value.lineTo(end.x, end.y)
    canvas.value.lineWidth = props.lineWidth
    canvas.value.strokeStyle = props.lineColor
    canvas.value.stroke()
  }
}
// 画框
function drawBox(location: QRCode['location']) {
  if (props.drawOnfound) {
    drawLine(location.topLeftCorner, location.topRightCorner)
    drawLine(location.topRightCorner, location.bottomRightCorner)
    drawLine(location.bottomRightCorner, location.bottomLeftCorner)
    drawLine(location.bottomLeftCorner, location.topLeftCorner)
  }
}
function found(code: QRCode) {
  emits('scanned', code)
}
function run() {
  requestAnimationFrame(() => {
    if (videoRef.value && canvasRef.value) {
      canvasRef.value.height = videoWH.value.height
      canvasRef.value.width = videoWH.value.width
      canvas.value?.drawImage(videoRef.value, 0, 0, canvasRef.value.width, canvasRef.value.height)
      const imageData = canvas.value?.getImageData(0, 0, canvasRef.value.width, canvasRef.value.height)
      let code: QRCode | null = null
      try {
        code = jsQR(imageData!.data, imageData!.width, imageData!.height)
      }
      catch (e) {
        console.error(e)
      }
      if (code) {
        drawBox(code.location)
        found(code)
      }
    }
    run()
  })
}
function fullStop() {
  if (videoRef.value && videoRef.value.srcObject)
    (videoRef.value.srcObject as MediaStream).getTracks().forEach(t => t.stop())
}
async function init() {
  await nextTick()
  if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
    canvas.value = canvasRef.value!.getContext('2d')!
    navigator.mediaDevices.getUserMedia({ video: true }).then((stream: MediaStream) => {
      if (videoRef.value) {
        videoRef.value.srcObject = stream
        videoRef.value.playsInline = true
        const play = videoRef.value.play()
        play.then(run).catch(() => {
          videoVisible.value = true
        })
      }
    })
  }
}
onMounted(() => {
  init()
})
onBeforeUnmount(() => {
  fullStop()
})
</script>

<template>
  <div ref="scanCode" class="scan-code">
    <!-- <div v-if="notSupport" class="banner">
      <i class="close_icon" @click="() => notSupport = false" />
      <p class="text">
        若当前浏览器无法扫码，请切换其他浏览器尝试
      </p>
    </div> -->
    <div class="scan-code-cover">
      <p class="line" />
      <span class="square top left" />
      <span class="square top right" />
      <span class="square bottom right" />
      <span class="square bottom left" />
      <p class="tips">
        将二维码放入框内，即可自动扫描
      </p>
    </div>
    <video
      v-show="videoVisible"
      ref="videoRef"
      class="video"
      :width="videoWH.width"
      :height="videoWH.height"
    />
    <canvas v-show="!videoVisible" ref="canvasRef" />
  </div>
</template>

<style scoped>
.scan-code {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}
.scan-code .scan-code-cover {
  height: 220px;
  width: 220px;
  position: absolute;
  top:50%;
  left:50%;
  -webkit-transform: translate(-50%,-50%);
  -moz-transform: translate(-50%,-50%);
  -ms-transform: translate(-50%,-50%);
  -o-transform: translate(-50%,-50%);
  transform: translate(-50%,-50%);
  border: .5px solid #999999;
  z-index: 1111;
}
.scan-code .scan-code-cover .line {
  width: 200px;
  height: 1px;
  margin-left: 10px;
  background: #5F68E8;
  background: linear-gradient(to right, transparent, #5F68E8, #0165FF, #5F68E8, transparent);
  position: absolute;
  -webkit-animation: scan 1.75s infinite linear;
  -moz-animation: scan 1.75s infinite linear;
  -ms-animation: scan 1.75s infinite linear;
  -o-animation: scan 1.75s infinite linear;
  animation: scan 1.75s infinite linear;
  -webkit-animation-fill-mode: both;
  -moz-animation-fill-mode: both;
  -ms-animation-fill-mode: both;
  -o-animation-fill-mode: both;
  animation-fill-mode: both;
  border-radius: 1px;
}
.scan-code .scan-code-cover .square {
  display: inline-block;
  height: 20px;
  width: 20px;
  position: absolute;
}
.scan-code .scan-code-cover .square.top {
  top: 0;
  border-top: 1px solid #5F68E8;
}
.scan-code .scan-code-cover .square.left {
  left: 0;
  border-left: 1px solid #5F68E8;
}
.scan-code .scan-code-cover .square.bottom {
  bottom: 0;
  border-bottom: 1px solid #5F68E8;
}
.scan-code .scan-code-cover .square.right {
 right: 0;
  border-right: 1px solid #5F68E8;
}
.scan-code .scan-code-cover .tips {
  position: absolute;
  bottom: -48px;
  width: 100%;
  font-size: 14px;
  color: #FFFFFF;
  opacity: 0.8;
}
@-webkit-keyframes scan {
  0% {top: 0}
  25% {top: 50px}
  50% {top: 100px}
  75% {top: 150px}
  100% {top: 200px}
}
@-moz-keyframes scan {
  0% {top: 0}
  25% {top: 50px}
  50% {top: 100px}
  75% {top: 150px}
  100% {top: 200px}
}
@-o-keyframes scan {
  0% {top: 0}
  25% {top: 50px}
  50% {top: 100px}
  75% {top: 150px}
  100% {top: 200px}
}
@keyframes scan {
  0% {top: 0}
  25% {top: 50px}
  50% {top: 100px}
  75% {top: 150px}
  100% {top: 200px}
}
</style>
