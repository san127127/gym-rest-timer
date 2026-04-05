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
const holdProgress = ref<number>(0);

let timerId: ReturnType<typeof setInterval> | null = null;
let vibrateId: ReturnType<typeof setInterval> | null = null;
let vibrateCounterId: ReturnType<typeof setInterval> | null = null;
let holdId: ReturnType<typeof setInterval> | null = null;
let snoozeTimer: ReturnType<typeof setInterval> | null = null;

const setTime = (seconds: number): void => {
  restTime.value = seconds;
};

const drawCanvas = (text: string, subText: string = '', color: string = '#00FF00'): void => {
  const canvas = canvasRef.value!;
  const ctx = canvas.getContext('2d')!;
  ctx.fillStyle = '#000000';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  ctx.fillStyle = color;
  ctx.font = 'bold 70px sans-serif';
  ctx.textAlign = 'center';
  ctx.fillText(text, canvas.width / 2, canvas.height / 2 + (subText ? -10 : 25));

  if (subText) {
    ctx.font = 'bold 30px sans-serif';
    ctx.fillText(subText, canvas.width / 2, canvas.height / 2 + 40);
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

  drawCanvas(`${timeLeft.value}s`);
  await enablePiP();

  timerId = setInterval(() => {
    timeLeft.value--;
    drawCanvas(`${timeLeft.value}s`);
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
        drawCanvas('GO!', `Vib: ${vibrateTime.value}s`, '#FF4444');
      }
    }, 1000);
  }

  vibrateId = setInterval(() => {
    navigator.vibrate([1000, 500, 1000]);
  }, 5000);
};

const snoozeVibration = (): void => {
  if (hasSnoozed.value) return;
  if (vibrateId) clearInterval(vibrateId);
  isVibrating.value = false;
  hasSnoozed.value = true;
  navigator.vibrate(0);

  let snoozeLeft = 90;
  drawCanvas(`Pause ${snoozeLeft}s`, `Vib: ${vibrateTime.value}s`, '#FFA500');

  snoozeTimer = setInterval(() => {
    snoozeLeft--;
    drawCanvas(`Pause ${snoozeLeft}s`, `Vib: ${vibrateTime.value}s`, '#FFA500');
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
    if (holdProgress.value >= 100) {
      stopAll();
    }
  }, 100);
};

const stopHold = (): void => {
  if (holdId) clearInterval(holdId);
  if (holdProgress.value < 100) {
    holdProgress.value = 0;
  }
};

const stopAll = (): void => {
  if (timerId) clearInterval(timerId);
  if (vibrateId) clearInterval(vibrateId);
  if (vibrateCounterId) clearInterval(vibrateCounterId);
  if (holdId) clearInterval(holdId);
  if (snoozeTimer) clearInterval(snoozeTimer);

  timerId = null;
  vibrateId = null;
  vibrateCounterId = null;
  holdId = null;
  snoozeTimer = null;

  isCounting.value = false;
  isVibrating.value = false;
  hasSnoozed.value = false;
  holdProgress.value = 0;
  navigator.vibrate(0);

  if (document.pictureInPictureElement) {
    document.exitPictureInPicture();
  }
};

onUnmounted(stopAll);
</script>

<template>
  <div class="gym-container" :class="{ 'is-vibrating': isVibrating }">
    <div class="timer-card">
      <canvas ref="canvasElement" width="320" height="200" style="display: none"></canvas>
      <video ref="videoElement" muted playsinline style="display: none"></video>

      <div v-if="!isCounting && !isVibrating && !hasSnoozed" class="controls">
        <div class="input-group">
          <input type="number" v-model.number="restTime" class="time-input" />
          <label>SEC</label>
        </div>
        <div class="presets">
          <button v-for="t in [60, 120, 180]" :key="t" @click="setTime(t)" class="preset-btn">
            {{ t }}
          </button>
        </div>
        <button @click="startTimer" class="main-btn">START REST</button>
      </div>

      <div v-else class="active-status">
        <div class="countdown-display" :class="{ warning: timeLeft <= 10 && timeLeft > 0 }">
          {{ timeLeft > 0 ? timeLeft : isVibrating ? 'GO!' : 'PAUSE' }}
        </div>

        <div class="vibe-timer">
          Vibrating total: <span>{{ vibrateTime }}s</span>
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
            {{ holdProgress > 0 ? 'HOLDING...' : 'HOLD 3S TO STOP' }}
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
  padding: 3rem 2rem;
  border-radius: 40px;
  text-align: center;
  position: relative;
  overflow: hidden;
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
  margin: 2.5rem 0;
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
  margin-bottom: 1.5rem;
  color: #f44;
  font-weight: bold;
  font-size: 1.2rem;
}
.vibe-timer span {
  font-size: 1.5rem;
}
.warning {
  color: #ff0;
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
  cursor: pointer;
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
</style>
