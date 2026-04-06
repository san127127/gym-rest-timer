<script setup lang="ts">
import { onUnmounted, ref, useTemplateRef } from 'vue';

const canvasRef = useTemplateRef<HTMLCanvasElement>('canvasElement');
const videoRef = useTemplateRef<HTMLVideoElement>('videoElement');

const restTime = ref<number>(60);
const timeLeft = ref<number>(0);
const vibrateTime = ref<number>(0);
const isCounting = ref<boolean>(false);
const isVibrating = ref<boolean>(false);
const hasSnoozed = ref<boolean>(false);
const historyStartTime = ref<Date>(new Date());
const holdProgress = ref<number>(0);
const history = ref<{ startTime: string; endTime: string; totalSeconds: number }[]>([]);

let timerId: ReturnType<typeof setInterval> | null = null;
let vibrateId: ReturnType<typeof setInterval> | null = null;
let vibrateCounterId: ReturnType<typeof setInterval> | null = null;
let holdId: ReturnType<typeof setInterval> | null = null;
let snoozeTimer: ReturnType<typeof setInterval> | null = null;

const setTime = (seconds: number): void => {
  restTime.value = seconds;
};

const drawCanvas = (
  text: string,
  subText: string = '',
  color: string = '#00FF00',
  progress: number = 1,
): void => {
  const canvas = canvasRef.value!;
  const ctx = canvas.getContext('2d')!;
  const cx = canvas.width / 2;
  const cy = canvas.height / 2;

  ctx.fillStyle = '#000000';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  ctx.beginPath();
  ctx.arc(cx, cy, 85, 0, Math.PI * 2);
  ctx.strokeStyle = '#222';
  ctx.lineWidth = 10;
  ctx.stroke();

  if (progress > 0) {
    ctx.beginPath();
    ctx.arc(cx, cy, 85, -Math.PI / 2, -Math.PI / 2 + Math.PI * 2 * progress);
    ctx.strokeStyle = color;
    ctx.lineWidth = 10;
    ctx.lineCap = 'round';
    ctx.stroke();
  }

  ctx.fillStyle = color;
  ctx.font = 'bold 60px sans-serif';
  ctx.textAlign = 'center';
  ctx.fillText(text, cx, cy + (subText ? -5 : 20));

  if (subText) {
    ctx.font = 'bold 25px sans-serif';
    ctx.fillText(subText, cx, cy + 35);
  }
};

const enablePiP = async (): Promise<void> => {
  const canvas = canvasRef.value!;
  const video = videoRef.value!;
  // @ts-ignore
  video.srcObject = canvas.captureStream(10);
  await video.play();
  await video.requestPictureInPicture();
};

const startTimer = async (): Promise<void> => {
  if (isCounting.value) return;
  timeLeft.value = restTime.value;
  vibrateTime.value = 0;
  isCounting.value = true;
  isVibrating.value = false;
  hasSnoozed.value = false;
  holdProgress.value = 0;
  historyStartTime.value = new Date();

  drawCanvas(`${timeLeft.value}s`, '', '#00FF00', 1);
  await enablePiP();

  timerId = setInterval(() => {
    timeLeft.value--;
    const p = timeLeft.value / restTime.value;
    drawCanvas(`${timeLeft.value}s`, '', '#00FF00', p);
    if (timeLeft.value <= 0) {
      clearInterval(timerId!);
      triggerAlarm();
    }
  }, 1000);
};

const triggerAlarm = (): void => {
  isVibrating.value = true;
  if (!vibrateCounterId) {
    vibrateCounterId = setInterval(() => {
      vibrateTime.value++;
      if (isVibrating.value) {
        drawCanvas('GO!', `Vib: ${vibrateTime.value}s`, '#FF4444', 1);
      }
    }, 1000);
  }
  navigator.vibrate([1000]);
  vibrateId = setInterval(() => {
    navigator.vibrate([1000]);
  }, 5000);
};

const snoozeVibration = (): void => {
  if (hasSnoozed.value) return;
  if (vibrateId) clearInterval(vibrateId);
  isVibrating.value = false;
  hasSnoozed.value = true;
  navigator.vibrate(0);

  let snoozeLeft = 90;
  drawCanvas(`${snoozeLeft}s`, 'PAUSE', '#FFA500', 1);

  snoozeTimer = setInterval(() => {
    snoozeLeft--;
    drawCanvas(`${snoozeLeft}s`, 'PAUSE', '#FFA500', snoozeLeft / 90);
    if (snoozeLeft <= 0) {
      clearInterval(snoozeTimer!);
      isVibrating.value = true;
      triggerAlarm();
    }
  }, 1000);
};

const startHold = (): void => {
  if (!isVibrating.value && !isCounting.value && !hasSnoozed.value) return;
  holdId = setInterval(() => {
    holdProgress.value += 3.34;
    if (holdProgress.value >= 100) stopAll();
  }, 100);
};

const stopHold = (): void => {
  if (holdId) clearInterval(holdId);
  if (holdProgress.value < 100) holdProgress.value = 0;
};

const stopAll = (): void => {
  if (isCounting.value) {
    const now = new Date();
    history.value.unshift({
      startTime: historyStartTime.value.toLocaleTimeString(),
      endTime: now.toLocaleTimeString(),
      totalSeconds: Math.floor((now.getTime() - historyStartTime.value.getTime()) / 1000),
    });
  }

  if (timerId) clearInterval(timerId);
  if (vibrateId) clearInterval(vibrateId);
  if (vibrateCounterId) clearInterval(vibrateCounterId);
  if (holdId) clearInterval(holdId);
  if (snoozeTimer) clearInterval(snoozeTimer);

  timerId = vibrateId = vibrateCounterId = holdId = snoozeTimer = null;
  isCounting.value = isVibrating.value = hasSnoozed.value = false;
  holdProgress.value = 0;
  navigator.vibrate(0);
  if (document.pictureInPictureElement) document.exitPictureInPicture();
};

onUnmounted(stopAll);
</script>

<template>
  <div class="gym-container" :class="{ 'is-vibrating': isVibrating }">
    <div class="timer-card">
      <canvas ref="canvasElement" width="300" height="300" style="display: none"></canvas>
      <video ref="videoElement" muted playsinline style="display: none"></video>

      <div v-if="!isCounting && !isVibrating && !hasSnoozed" class="controls">
        <div class="input-group">
          <input type="number" v-model.number="restTime" class="time-input" />
          <label>SEC</label>
        </div>
        <div class="presets">
          <button
            v-for="t in [60, 90, 120, 150, 180]"
            :key="t"
            @click="setTime(t)"
            class="preset-btn"
          >
            {{ t }}
          </button>
        </div>
        <button @click="startTimer" class="main-btn">START REST</button>

        <div v-if="history.length > 0" class="history-list">
          <div v-for="(item, i) in history.slice(0, 3)" :key="i" class="history-item">
            <span>{{ item.startTime }} - {{ item.endTime }}</span>
            <span>{{ item.totalSeconds }}s</span>
          </div>
        </div>
      </div>

      <div v-else class="active-status">
        <div class="countdown-display" :class="{ warning: timeLeft <= 10 && timeLeft > 0 }">
          {{ timeLeft > 0 ? timeLeft : isVibrating ? 'GO!' : 'PAUSE' }}
        </div>
        <div class="vibe-timer">
          Total: <span>{{ restTime + vibrateTime }}s</span>
        </div>
        <button v-if="isVibrating && !hasSnoozed" @click="snoozeVibration" class="snooze-btn">
          PAUSE VIBE (90s)
        </button>
        <div class="hold-container">
          <button
            @mousedown="startHold"
            @mouseup="stopHold"
            @mouseleave="stopHold"
            @touchstart.prevent="startHold"
            @touchend.prevent="stopHold"
            class="stop-btn"
          >
            {{ holdProgress > 0 ? 'KEEP HOLDING...' : 'HOLD 3S TO STOP' }}
            <div class="progress-bar" :style="{ width: holdProgress + '%' }"></div>
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.gym-container {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #0a0a0a;
  color: #fff;
  font-family: system-ui, sans-serif;
  user-select: none;
}
.is-vibrating {
  background: #500;
}
.timer-card {
  width: 85%;
  max-width: 380px;
  background: #1a1a1a;
  padding: 2.5rem 2rem;
  border-radius: 40px;
  text-align: center;
}
.time-input {
  background: transparent;
  border: none;
  border-bottom: 3px solid #0f0;
  color: #0f0;
  font-size: 5rem;
  width: 160px;
  text-align: center;
  outline: none;
  font-weight: bold;
}
.presets {
  display: flex;
  justify-content: center;
  gap: 15px;
  margin: 2rem 0;
}
.preset-btn {
  background: #333;
  border: none;
  color: #fff;
  padding: 12px 20px;
  border-radius: 12px;
}
.main-btn {
  width: 100%;
  padding: 1.2rem;
  background: #0f0;
  color: #000;
  border: none;
  border-radius: 20px;
  font-size: 1.3rem;
  font-weight: 800;
}
.countdown-display {
  font-size: 7rem;
  font-weight: 900;
  color: #0f0;
}
.vibe-timer {
  margin-bottom: 1rem;
  color: #f44;
  font-weight: bold;
}
.vibe-timer span {
  font-size: 1.5rem;
}
.history-list {
  margin-top: 2rem;
  border-top: 1px solid #333;
  padding-top: 1rem;
  font-size: 0.9rem;
  color: #888;
}
.history-item {
  display: flex;
  justify-content: space-between;
  margin-bottom: 5px;
}
.snooze-btn {
  width: 100%;
  padding: 1rem;
  background: #ffa500;
  color: #000;
  border: none;
  border-radius: 20px;
  font-weight: bold;
  margin-bottom: 1rem;
}
.hold-container {
  position: relative;
  width: 100%;
  height: 60px;
  background: #333;
  border-radius: 20px;
  overflow: hidden;
}
.stop-btn {
  position: relative;
  width: 100%;
  height: 100%;
  background: transparent;
  color: #fff;
  border: none;
  font-weight: bold;
  z-index: 2;
}
.progress-bar {
  position: absolute;
  top: 0;
  left: 0;
  height: 100%;
  background: #f44;
  transition: width 0.1s linear;
  z-index: 1;
}
.warning {
  color: #ff0;
}
</style>
