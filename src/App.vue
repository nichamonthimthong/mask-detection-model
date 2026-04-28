<script setup>
import { ref, onUnmounted, nextTick } from 'vue' // ✅ เพิ่ม nextTick เข้ามา

// --- Logic ส่วนเดิม ---
const URL = "/my_model/"; 
let model, webcam, maxPredictions, requestID;

const webcamContainer = ref(null);
const isRunning = ref(false);
const isLoading = ref(false);
const predictions = ref([]); 

async function init() {
  if (isRunning.value) return; 
  isLoading.value = true;

  try {
    const modelURL = URL + "model.json";
    const metadataURL = URL + "metadata.json";

    // 1. โหลดโมเดล
    model = await window.tmImage.load(modelURL, metadataURL);
    maxPredictions = model.getTotalClasses();

    // 2. เตรียมกล้อง
    const flip = true; 
    webcam = new window.tmImage.Webcam(200, 200, flip);
    await webcam.setup(); 
    await webcam.play();
    
    // 3. เปลี่ยนสถานะหน้าจอ
    isRunning.value = true;
    isLoading.value = false;
    requestID = window.requestAnimationFrame(loop);

    // ✅ แก้ไขตรงนี้: รอให้หน้าจอสร้างกล่องเสร็จก่อน ค่อยเอากล้องไปใส่
    await nextTick(); 
    
    if (webcamContainer.value) {
        webcamContainer.value.innerHTML = "";
        webcamContainer.value.appendChild(webcam.canvas);
    }
    
  } catch (error) {
    console.error(error);
    alert("เกิดข้อผิดพลาด: " + error.message); // แจ้งเตือนชัดขึ้น
    isLoading.value = false;
    isRunning.value = false;
  }
}

async function loop() {
  if (!isRunning.value) return;
  webcam.update(); 
  await predict();
  requestID = window.requestAnimationFrame(loop);
}

async function predict() {
  if (!model || !webcam.canvas) return; // กัน Error
  
  const prediction = await model.predict(webcam.canvas);
  predictions.value = prediction
    .map(p => {
        let colorClass = 'bg-red'; 
        if (p.probability > 0.8) colorClass = 'bg-green';
        else if (p.probability > 0.4) colorClass = 'bg-yellow';

        return {
            className: p.className,
            probability: (p.probability * 100).toFixed(0),
            rawProb: p.probability,
            colorClass: colorClass
        }
    })
    .sort((a, b) => b.rawProb - a.rawProb);
}

const stopCamera = () => {
    isRunning.value = false;
    isLoading.value = false;
    if (requestID) window.cancelAnimationFrame(requestID);
    if (webcam) webcam.stop();
}

onUnmounted(() => {
    stopCamera();
});
</script>

<template>
  <div class="app-wrapper">
    <div class="card">
      
      <div class="card-header">
        <div class="header-icon">🤖</div>
        <div class="header-text">
            <h2>AI Recognition</h2>
            <p>ระบบจำแนกวัตถุอัจฉริยะ</p>
        </div>
        <div v-if="isRunning" class="live-badge">
            <span class="blink">●</span> LIVE
        </div>
      </div>

      <div class="card-body">
          
          <div v-if="!isRunning && !isLoading" class="start-screen">
              <div class="placeholder-icon">📷</div>
              <p>กดปุ่มด้านล่างเพื่อเริ่มใช้งานกล้อง</p>
              <button @click="init" class="btn btn-primary btn-lg">
                เริ่มการทำงาน
              </button>
          </div>

          <div v-else-if="isLoading" class="loading-screen">
              <div class="spinner"></div>
              <p>กำลังโหลดโมเดลและเปิดกล้อง...</p>
          </div>

          <div v-else class="active-screen">
              
              <div class="cam-wrapper">
                  <div ref="webcamContainer" class="webcam-view"></div>
              </div>

              <div class="stats-wrapper">
                  <h3 class="stats-title">ผลการวิเคราะห์</h3>
                  <div class="stats-list">
                      <div v-for="(item, index) in predictions" :key="index" class="stat-row">
                          <div class="stat-header">
                              <span class="stat-name">{{ item.className }}</span>
                              <span class="stat-val">{{ item.probability }}%</span>
                          </div>
                          <div class="progress-track">
                              <div class="progress-fill" 
                                   :class="item.colorClass"
                                   :style="{ width: item.probability + '%' }">
                              </div>
                          </div>
                      </div>
                  </div>
                  
                  <button @click="stopCamera" class="btn btn-outline">
                    หยุดการทำงาน
                  </button>
              </div>
          </div>

      </div>
    </div>
  </div>
</template>

<style scoped>
/* --- โครงสร้างหลัก --- */
.app-wrapper {
    min-height: 100vh;
    background: linear-gradient(120deg, #84fab0 0%, #8fd3f4 100%);
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 20px;
    font-family: 'Segoe UI', sans-serif;
}

.card {
    background: white;
    width: 100%;
    max-width: 850px;
    border-radius: 24px;
    box-shadow: 0 15px 35px rgba(0,0,0,0.1);
    overflow: hidden;
    display: flex;
    flex-direction: column;
}

/* --- Header --- */
.card-header {
    background: #f8f9fa;
    padding: 20px 30px;
    border-bottom: 1px solid #eee;
    display: flex;
    align-items: center;
    gap: 15px;
}
.header-icon { font-size: 2rem; }
.header-text h2 { margin: 0; font-size: 1.2rem; color: #333; }
.header-text p { margin: 0; font-size: 0.9rem; color: #777; }

.live-badge {
    margin-left: auto;
    background: #ffebee;
    color: #d32f2f;
    padding: 5px 12px;
    border-radius: 20px;
    font-weight: bold;
    font-size: 0.8rem;
    display: flex;
    align-items: center;
    gap: 5px;
}
.blink { animation: blink 1.5s infinite; }
@keyframes blink { 50% { opacity: 0; } }

/* --- Body --- */
.card-body {
    padding: 30px;
    min-height: 300px;
    display: flex;
    justify-content: center;
    align-items: center;
}

/* State A & B */
.start-screen, .loading-screen { text-align: center; }
.placeholder-icon { font-size: 4rem; color: #ddd; margin-bottom: 15px; }
.start-screen p, .loading-screen p { color: #666; margin-bottom: 25px; }

.spinner {
    width: 40px; height: 40px; border: 4px solid #f3f3f3;
    border-top: 4px solid #3498db; border-radius: 50%;
    animation: spin 1s linear infinite; margin: 0 auto 15px;
}
@keyframes spin { 100% { transform: rotate(360deg); } }

/* State C: Active Grid Layout */
.active-screen {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 30px;
    width: 100%;
    align-items: start;
}

/* กล้อง */
.cam-wrapper {
    background: black;
    border-radius: 16px;
    overflow: hidden;
    box-shadow: 0 4px 15px rgba(0,0,0,0.2);
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 200px; /* กันกล่องยุบ */
}
.webcam-view :deep(canvas) {
    display: block;
    width: 100%;
    height: auto; /* ให้ยืดเต็มกล่อง */
}

/* ผลลัพธ์ */
.stats-wrapper { padding: 10px; }
.stats-title { margin-top: 0; color: #444; font-size: 1.1rem; margin-bottom: 20px; }
.stat-row { margin-bottom: 18px; }
.stat-header { display: flex; justify-content: space-between; margin-bottom: 6px; font-weight: 600; color: #555; font-size: 0.95rem; }
.progress-track { height: 10px; background: #eee; border-radius: 10px; overflow: hidden; }
.progress-fill { height: 100%; border-radius: 10px; transition: width 0.3s ease; }

.bg-green { background: linear-gradient(90deg, #2ecc71, #27ae60); }
.bg-yellow { background: linear-gradient(90deg, #f1c40f, #f39c12); }
.bg-red { background: linear-gradient(90deg, #e74c3c, #c0392b); }

/* Buttons */
.btn { border: none; padding: 10px 20px; border-radius: 8px; cursor: pointer; font-size: 1rem; font-weight: 600; transition: all 0.2s; }
.btn-lg { padding: 12px 35px; font-size: 1.1rem; }
.btn-primary { background: #3498db; color: white; box-shadow: 0 4px 10px rgba(52, 152, 219, 0.3); }
.btn-primary:hover { background: #2980b9; transform: translateY(-2px); }
.btn-outline { background: transparent; border: 2px solid #eee; color: #777; width: 100%; margin-top: 20px; }
.btn-outline:hover { border-color: #e74c3c; color: #e74c3c; background: #fff5f5; }

@media (max-width: 768px) {
    .active-screen { grid-template-columns: 1fr; }
    .card-body { padding: 20px; }
}
</style>