<template>
  <div id="app">
    <h1>Vue QR Code Reader - Full Demo</h1>
    
    <div class="demo-container">
      <p>
        Modern mobile phones often have a variety of different cameras installed (e.g. front, rear,
        wide-angle, infrared, desk-view). The one picked by default is sometimes not the best choice.
        For more fine-grained control, you can select a camera by device constraints or by the device
        ID:

        <select v-model="selectedConstraints">
          <option
            v-for="option in constraintOptions"
            :key="option.label"
            :value="option.constraints"
          >
            {{ option.label }}
          </option>
        </select>
      </p>

      <p>
        Detected codes are visually highlighted in real-time. Use the following dropdown to change the
        flavor:

        <select v-model="trackFunctionSelected">
          <option
            v-for="option in trackFunctionOptions"
            :key="option.text"
            :value="option"
          >
            {{ option.text }}
          </option>
        </select>
      </p>

      <p>
        By default only QR-codes are detected but a variety of other barcode formats are also
        supported. You can select one or multiple but the more you select the more expensive scanning
        becomes: <br />

        <span
          v-for="option in Object.keys(barcodeFormats)"
          :key="option"
          class="barcode-format-checkbox"
        >
          <input
            type="checkbox"
            v-model="barcodeFormats[option]"
            :id="option"
          />
          <label :for="option">{{ option }}</label>
        </span>
      </p>

      <p class="error">{{ error }}</p>

      <p class="decode-result">
        Last result: <b>{{ result }}</b>
      </p>

      <div class="camera-container">
        <qrcode-stream
          :constraints="selectedConstraints"
          :track="trackFunctionSelected.value"
          :formats="selectedBarcodeFormats"
          @error="onError"
          @detect="onDetect"
          @camera-on="onCameraReady"
        />
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import { QrcodeStream } from 'vue-qrcode-reader'

/*** detection handling ***/

const result = ref('')

function onDetect(detectedCodes: any[]) {
  console.log(detectedCodes)
  result.value = JSON.stringify(detectedCodes.map((code) => code.rawValue))
}

/*** select camera ***/

const selectedConstraints = ref({ facingMode: 'environment' })
const defaultConstraintOptions = [
  { label: 'rear camera', constraints: { facingMode: 'environment' } },
  { label: 'front camera', constraints: { facingMode: 'user' } }
]
const constraintOptions = ref(defaultConstraintOptions)

async function onCameraReady() {
  // NOTE: on iOS we can't invoke `enumerateDevices` before the user has given
  // camera access permission. `QrcodeStream` internally takes care of
  // requesting the permissions. The `camera-on` event should guarantee that this
  // has happened.
  try {
    const devices = await navigator.mediaDevices.enumerateDevices()
    const videoDevices = devices.filter(({ kind }) => kind === 'videoinput')

    constraintOptions.value = [
      ...defaultConstraintOptions,
      ...videoDevices.map(({ deviceId, label }) => ({
        label: `${label} (ID: ${deviceId})`,
        constraints: { deviceId }
      }))
    ]

    error.value = ''
  } catch (err) {
    console.error('Error enumerating devices:', err)
  }
}

/*** track functions ***/

function paintOutline(detectedCodes: any[], ctx: CanvasRenderingContext2D) {
  for (const detectedCode of detectedCodes) {
    const [firstPoint, ...otherPoints] = detectedCode.cornerPoints

    ctx.strokeStyle = 'red'

    ctx.beginPath()
    ctx.moveTo(firstPoint.x, firstPoint.y)
    for (const { x, y } of otherPoints) {
      ctx.lineTo(x, y)
    }
    ctx.lineTo(firstPoint.x, firstPoint.y)
    ctx.closePath()
    ctx.stroke()
  }
}

function paintBoundingBox(detectedCodes: any[], ctx: CanvasRenderingContext2D) {
  for (const detectedCode of detectedCodes) {
    const {
      boundingBox: { x, y, width, height }
    } = detectedCode

    ctx.lineWidth = 2
    ctx.strokeStyle = '#007bff'
    ctx.strokeRect(x, y, width, height)
  }
}

function paintCenterText(detectedCodes: any[], ctx: CanvasRenderingContext2D) {
  for (const detectedCode of detectedCodes) {
    const { boundingBox, rawValue } = detectedCode

    const centerX = boundingBox.x + boundingBox.width / 2
    const centerY = boundingBox.y + boundingBox.height / 2

    const fontSize = Math.max(12, (50 * boundingBox.width) / ctx.canvas.width)

    ctx.font = `bold ${fontSize}px sans-serif`
    ctx.textAlign = 'center'

    ctx.lineWidth = 3
    ctx.strokeStyle = '#35495e'
    ctx.strokeText(detectedCode.rawValue, centerX, centerY)

    ctx.fillStyle = '#5cb984'
    ctx.fillText(rawValue, centerX, centerY)
  }
}

const trackFunctionOptions = [
  { text: 'nothing (default)', value: undefined },
  { text: 'outline', value: paintOutline },
  { text: 'centered text', value: paintCenterText },
  { text: 'bounding box', value: paintBoundingBox }
]
const trackFunctionSelected = ref(trackFunctionOptions[1])

/*** barcode formats ***/

const barcodeFormats = ref({
  aztec: false,
  code_128: false,
  code_39: false,
  code_93: false,
  codabar: false,
  databar: false,
  databar_expanded: false,
  data_matrix: false,
  dx_film_edge: false,
  ean_13: false,
  ean_8: false,
  itf: false,
  maxi_code: false,
  micro_qr_code: false,
  pdf417: false,
  qr_code: true,
  rm_qr_code: false,
  upc_a: false,
  upc_e: false,
  linear_codes: false,
  matrix_codes: false
})

const selectedBarcodeFormats = computed(() => {
  return Object.keys(barcodeFormats.value).filter((format) => barcodeFormats.value[format as keyof typeof barcodeFormats.value])
})

/*** error handling ***/

const error = ref('')

function onError(err: any) {
  error.value = `[${err.name}]: `

  if (err.name === 'NotAllowedError') {
    error.value += 'you need to grant camera access permission'
  } else if (err.name === 'NotFoundError') {
    error.value += 'no camera on this device'
  } else if (err.name === 'NotSupportedError') {
    error.value += 'secure context required (HTTPS, localhost)'
  } else if (err.name === 'NotReadableError') {
    error.value += 'is the camera already in use?'
  } else if (err.name === 'OverconstrainedError') {
    error.value += 'installed cameras are not suitable'
  } else if (err.name === 'StreamApiNotSupportedError') {
    error.value += 'Stream API is not supported in this browser'
  } else if (err.name === 'InsecureContextError') {
    error.value +=
      'Camera access is only permitted in secure context. Use HTTPS or localhost rather than HTTP.'
  } else {
    error.value += err.message
  }
}
</script>

<style scoped>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}

.demo-container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

.error {
  font-weight: bold;
  color: red;
}

.decode-result {
  margin: 20px 0;
  font-size: 16px;
}

.barcode-format-checkbox {
  margin-right: 10px;
  white-space: nowrap;
  display: inline-block;
}

.camera-container {
  margin-top: 20px;
  max-width: 100%;
}

select {
  margin: 0 10px;
  padding: 5px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

input[type="checkbox"] {
  margin-right: 5px;
}

label {
  margin-right: 15px;
  cursor: pointer;
}

h1 {
  color: #42b883;
  margin-bottom: 30px;
}

p {
  text-align: left;
  line-height: 1.6;
  margin-bottom: 15px;
}
</style>
