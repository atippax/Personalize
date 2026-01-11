<template>
  <div class="ocr-container">
    <video ref="videoRef" autoplay playsinline class="camera-view"></video>

    <div class="overlay">
      <div class="status-badge" :class="{ 'is-active': isProcessing }">
        {{ isProcessing ? '⚡ Processing...' : '📡 Live Scanning' }}
      </div>
      <div class="scan-frame"></div>
    </div>

    <div class="debug-panel">
      <div class="debug-header">Debug View (Last Capture)</div>
      <img v-if="debugImageUrl" :src="debugImageUrl" class="debug-preview" />
      <div v-else class="debug-placeholder">No frame captured yet</div>
      
      <div class="debug-info">
        <p>Resolution: {{ debugInfo.res }}</p>
        <p>Size: {{ debugInfo.size }} KB</p>
        <p>Last Sync: {{ debugInfo.time }}</p>
        <p>{{ text }}</p>
      </div>
    </div>

    <canvas ref="canvasRef" v-show="false"></canvas>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, onMounted, onUnmounted } from 'vue';
import { useOCR } from './composables/useOcr';
const ocr = useOCR()
ocr.initOCR('eng')

const videoRef = ref(null);
const canvasRef = ref(null);
const isProcessing = ref(false);
const text = ref('')
// State สำหรับ Debug
const debugImageUrl = ref(null);
const debugInfo = reactive({
  res: '0x0',
  size: 0,
  time: '-'
});

let stream = null;
let intervalId = null;
const initCamera = async () => {
  try {
    const constraints = {
      video: {
        facingMode: 'environment',
        width: { min: 640, ideal: 1920, max: 1920 }, 
        height: { min: 480, ideal: 1080, max: 1080 },
      },
      audio: false
    };

    stream = await navigator.mediaDevices.getUserMedia(constraints);
    
    if (videoRef.value) {
      videoRef.value.srcObject = stream;
      const track = stream.getVideoTracks()[0];
      const capabilities = track.getCapabilities();
      const advancedConstraints: any = {};
      
      if (capabilities.focusMode && capabilities.focusMode.includes('continuous')) {
        advancedConstraints.focusMode = 'continuous';
      }

      // ถ้าต้องการพยายามสั่งเปิดไฟ Flash หรือตั้งค่าอื่นๆ เพิ่มเติม (ถ้ามี)
      if (Object.keys(advancedConstraints).length > 0) {
        await track.applyConstraints({
          advanced: [advancedConstraints]
        });
      }
    }
    
    startCaptureLoop();
  } catch (err) {
    console.error("Camera Error:", err);
    alert("ไม่สามารถเปิดกล้องได้: " + err);
  }
};
const captureFrame = async () => {
  clearInterval(intervalId);

  if (!videoRef.value || !canvasRef.value || isProcessing.value) return;

  const video = videoRef.value;
  const canvas = canvasRef.value;
  const context = canvas.getContext('2d');

  canvas.width = video.videoWidth;
  canvas.height = video.videoHeight;
  context.drawImage(video, 0, 0, canvas.width, canvas.height);

  // ดึงภาพออกมาโชว์ใน Debug View (Base64 เพื่อแสดงผลทันที)
  const previewData = canvas.toDataURL('image/jpeg', 0.5);
  debugImageUrl.value = previewData;

  // แปลงเป็น Blob เพื่อส่งจริงและวัดขนาดไฟล์
  canvas.toBlob(async (blob) => {
    if (blob) {
      // อัปเดต Debug Info
      debugInfo.res = `${canvas.width}x${canvas.height}`;
      debugInfo.size = (blob.size / 1024).toFixed(2);
      debugInfo.time = new Date().toLocaleTimeString();
try{
  await sendToServer(blob);
}
finally{ 
  startCaptureLoop()
}
    }
  }, 'image/jpeg', 0.7);
};

const sendToServer = async (imageBlob) => {
  isProcessing.value = true;
  try {
    // จำลองการส่ง
    text.value    = await ocr.recognize(imageBlob)
    // await new Promise(resolve => setTimeout(resolve, 400)); 
    console.log("Frame sent successfully");
  }catch(ex){
    alert('send '+ex)
  } finally {
    isProcessing.value = false;
  }
};

const startCaptureLoop = () => {
  intervalId = setInterval(captureFrame, 1000);
};
const stopCaptureLoop = ()=>{
  clearInterval(intervalId);

}
onMounted(() => initCamera());
onUnmounted(() => {
  if (stream) stream.getTracks().forEach(t => t.stop());
  stopCaptureLoop()
});
</script>

<style scoped>
.ocr-container {
  position: relative;
  width: 100vw;
  height: 100vh;
  background: #000;
  overflow: hidden;
}

.camera-view {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.overlay {
  position: absolute;
  top: 0; left: 0; width: 100%; height: 100%;
  display: flex; flex-direction: column; align-items: center;
  pointer-events: none;
}

.status-badge {
  margin-top: 20px;
  padding: 5px 12px;
  background: rgba(0,0,0,0.7);
  color: #00ff00;
  border-radius: 4px;
  font-size: 12px;
}

.scan-frame {
  margin: auto;
  width: 80%;
  height: 25%;
  border: 1px solid rgba(255,255,255,0.3);
  box-shadow: 0 0 0 1000px rgba(0,0,0,0.5);
}

/* --- DEBUG PANEL STYLE --- */
.debug-panel {
  position: absolute;
  bottom: 20px;
  right: 20px;
  width: 150px;
  background: rgba(0, 0, 0, 0.85);
  border: 1px solid #444;
  border-radius: 8px;
  padding: 8px;
  color: #fff;
  font-family: monospace;
  font-size: 10px;
  z-index: 100;
}

.debug-header {
  border-bottom: 1px solid #444;
  margin-bottom: 5px;
  padding-bottom: 3px;
  color: #aaa;
}

.debug-preview {
  width: 100%;
  height: auto;
  border-radius: 4px;
  margin-bottom: 5px;
  border: 1px solid #555;
}

.debug-placeholder {
  height: 80px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #222;
  color: #555;
  text-align: center;
}

.debug-info p {
  margin: 2px 0;
}
</style>