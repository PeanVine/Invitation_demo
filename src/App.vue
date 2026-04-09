<template>
  <p class="arrival-tip" :class="{ show: showArrivalTip, leave: isIntroDone }">
    <span class="arrival-tip-icon" aria-hidden="true"></span>
    <span>叮! 有一封的邮件送达!</span>
  </p>
  <div
    class="container"
    :class="{ extracted: isExtracted, reading: isFlipped, ready: isIntroDone }"
    :style="{ '--scene-scale': String(sceneScale) }"
  >
    <div
      class="letter-wrapper"
      :class="{ opened: isOpened, extracted: isExtracted }"
    >
      <transition name="action-hint-fade">
        <p
          v-if="!isOpened"
          class="action-hint action-hint-intro"
          :class="{ show: showUnopenedHint }"
        >
          拆开看看?
        </p>
      </transition>
      <transition name="check-hint-fade">
        <p
          v-if="isOpened && showCheckHint"
          class="action-hint action-hint-check"
        >
          Check!
        </p>
      </transition>
      <p class="inner-greeting">欢迎回家!</p>
      <div class="cover" @click="openEnvelope"></div>
      <div
        class="letter"
        :class="{
          clickable: isOpened && (!isExtracted || !isFlipped),
          reading: isFlipped,
        }"
        @click="handleLetterClick"
      >
        <div class="letter-page" :class="{ flipped: isFlipped }">
          <img src="/assets/封面.png" alt="封面" class="letter-face front" />
          <img src="/assets/内页.png" alt="内页" class="letter-face back" />
        </div>
      </div>
      <div class="envelope" @click="hideCheckHint">
        <img
          src="/assets/Tenzor_Logo.png"
          alt="Tenzor Logo"
          class="envelope-logo"
        />
        <b>From Tenzor</b>
      </div>
    </div>
    <transition name="flip-hint-fade">
      <p v-if="showFlipHint && !isFlipped" class="flip-hint">
        点击封面 拨至内页
      </p>
    </transition>
  </div>
</template>
<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref, watch } from "vue";

const isOpened = ref(false);
const isExtracted = ref(false);
const isFlipped = ref(false);
const isIntroDone = ref(false);
const showArrivalTip = ref(false);
const showUnopenedHint = ref(false);
const showCheckHint = ref(false);
const showFlipHint = ref(false);
const sceneScale = ref(0.88);

let introTimer: ReturnType<typeof setTimeout> | null = null;
let hintTimer: ReturnType<typeof setTimeout> | null = null;
let checkHintTimer: ReturnType<typeof setTimeout> | null = null;
let flipHintTimer: ReturnType<typeof setTimeout> | null = null;

const updateSceneScale = () => {
  const widthScale = (window.innerWidth - 24) / 500;
  const heightScale = (window.innerHeight - 24) / 600;
  sceneScale.value = Math.min(0.88, widthScale, heightScale);
};

const hideCheckHint = () => {
  if (checkHintTimer) {
    clearTimeout(checkHintTimer);
    checkHintTimer = null;
  }
  showCheckHint.value = false;
};

const hideFlipHint = () => {
  showFlipHint.value = false;
};

const openEnvelope = () => {
  if (isOpened.value) {
    hideCheckHint();
    hideFlipHint();
    return;
  }
  isOpened.value = true;
  hideCheckHint();
  hideFlipHint();
  checkHintTimer = setTimeout(() => {
    if (isOpened.value && !isExtracted.value) {
      showCheckHint.value = true;
    }
  }, 1000);
};

const handleLetterClick = () => {
  if (!isOpened.value) return;

  if (!isExtracted.value) {
    hideCheckHint();
    isExtracted.value = true;
    showFlipHint.value = false;
    if (flipHintTimer) {
      clearTimeout(flipHintTimer);
    }
    flipHintTimer = setTimeout(() => {
      if (isExtracted.value && !isFlipped.value) {
        showFlipHint.value = true;
      }
    }, 2000);
    return;
  }

  if (!isFlipped.value) {
    hideFlipHint();
    isFlipped.value = true;
  }
};

watch(isFlipped, (flipped) => {
  document.documentElement.classList.toggle("reading-scroll", flipped);
  document.body.classList.toggle("reading-scroll", flipped);
});

onMounted(() => {
  showArrivalTip.value = true;
  updateSceneScale();
  window.addEventListener("resize", updateSceneScale);
  introTimer = setTimeout(() => {
    isIntroDone.value = true;
  }, 1500);
  hintTimer = setTimeout(() => {
    if (!isOpened.value) {
      showUnopenedHint.value = true;
    }
  }, 3000);
});

onBeforeUnmount(() => {
  window.removeEventListener("resize", updateSceneScale);
  document.documentElement.classList.remove("reading-scroll");
  document.body.classList.remove("reading-scroll");
  if (introTimer) {
    clearTimeout(introTimer);
  }
  if (hintTimer) {
    clearTimeout(hintTimer);
  }
  if (checkHintTimer) {
    clearTimeout(checkHintTimer);
  }
  if (flipHintTimer) {
    clearTimeout(flipHintTimer);
  }
});
</script>
<style>
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}
:root {
  --clr: #94ee6a89;
  --flip-scale-end: 1.5;
  --flip-y-start: 0;
  --flip-y-mid: 10px;
  --flip-y-end: 15vh;
}
html {
  overflow: hidden;
}
body {
  width: 100%;
  min-height: 100vh;
  overflow: hidden;
  background: var(--clr);
  display: flex;
  justify-content: center;
  align-items: center;
}
html.reading-scroll,
body.reading-scroll {
  overflow-y: auto;
  overflow-x: hidden;
  scrollbar-width: none;
  -ms-overflow-style: none;
}
html.reading-scroll::-webkit-scrollbar,
body.reading-scroll::-webkit-scrollbar {
  width: 0;
  height: 0;
  display: none;
}
.arrival-tip {
  position: fixed;
  top: 2vh;
  left: 50%;
  transform: translate(-50%, -26px) scale(0.92);
  opacity: 0;
  z-index: 20;
  padding: 7px 12px 7px 9px;
  border-radius: 999px;
  background: #fff;
  color: #101010;
  font-size: 16px;
  letter-spacing: 0.2px;
  display: inline-flex;
  align-items: center;
  gap: 7px;
  box-shadow: 0 8px 18px rgba(0, 0, 0, 0.22);
  border: 1px solid rgba(0, 0, 0, 0.08);
  pointer-events: none;
}
.arrival-tip .arrival-tip-icon {
  width: 16px;
  height: 16px;
  border-radius: 50%;
  background: radial-gradient(
    circle at 35% 35%,
    #f1f1f1 0%,
    #cbcbcb 60%,
    #8e8e8e 100%
  );
  position: relative;
  flex-shrink: 0;
  box-shadow: 0 1px 0 rgba(255, 255, 255, 0.5) inset;
}
.arrival-tip .arrival-tip-icon::after {
  content: "";
  position: absolute;
  width: 6px;
  height: 2px;
  border-radius: 3px;
  background: #3d3d3d;
  left: 50%;
  top: 3px;
  transform: translateX(-50%);
  opacity: 0.85;
}
.arrival-tip.show {
  animation: 0.3s ease-out 0.3s forwards tipDropIn;
}
.arrival-tip.show .arrival-tip-icon {
  animation: 1s ease-in-out 0.5s 2 tipIconShake;
}
.arrival-tip.leave {
  animation: 0.28s ease-in forwards tipFadeOut;
}
.container {
  position: relative;
  width: 500px;
  height: 600px;
  transform: scale(var(--scene-scale));
  transform-origin: center bottom;
  overflow: visible;
  clip-path: inset(-100vh 0 1px 0);
  display: flex;
  flex-direction: column;
  justify-content: flex-end;
  align-items: center;
  transition: clip-path 0s linear;
  opacity: 0;
  pointer-events: none;
}
.container.ready {
  animation: 1.2s cubic-bezier(0.37, -0.09, 0, 1.6) bounceInUp;
  opacity: 1;
  pointer-events: auto;
}
.container.extracted {
  clip-path: inset(-100vh 0 -100vh 0);
  transition-delay: 0.2s;
}
.container.reading {
  clip-path: inset(-100vh -100vw -100vh -100vw);
  transition: clip-path 0.12s ease-out;
  transition-delay: 0s;
}
@keyframes tipDropIn {
  from {
    opacity: 0;
    transform: translate(-50%, -26px) scale(0.92);
  }
  to {
    opacity: 1;
    transform: translate(-50%, 0) scale(1);
  }
}
@keyframes tipFadeOut {
  from {
    opacity: 1;
    transform: translate(-50%, 0) scale(1);
  }
  to {
    opacity: 0;
    transform: translate(-50%, -12px) scale(0.98);
  }
}
@keyframes tipIconShake {
  0% {
    transform: rotate(0deg);
  }
  25% {
    transform: rotate(-9deg);
  }
  50% {
    transform: rotate(8deg);
  }
  75% {
    transform: rotate(-6deg);
  }
  100% {
    transform: rotate(0deg);
  }
}
@keyframes bounceInUp {
  from {
    opacity: 0;
    transform: translateY(2000px) scale(calc(var(--scene-scale) * 0.1));
  }
  65% {
    opacity: 1;
    transform: translateY(-100px) scale(var(--scene-scale));
  }
  75% {
    transform: translateY(20px) scale(var(--scene-scale));
  }
  90% {
    transform: translateY(-20px) scale(var(--scene-scale));
  }
  100% {
    transform: translateY(0) scale(var(--scene-scale));
  }
}
.container .letter-wrapper {
  position: relative;
  width: 100%;
  height: 360px;
  background-image: url(/assets/bg.png);
  background-repeat: no-repeat;
  display: flex;
  justify-content: center;
  align-items: flex-end;
}
.container .letter-wrapper .action-hint {
  position: absolute;
  top: -44px;
  left: 0;
  width: 100%;
  text-align: center;
  font-size: 30px;
  font-weight: 700;
  color: #48240a;
  letter-spacing: 1px;
  pointer-events: none;
  user-select: none;
  z-index: 8;
}
.container .letter-wrapper .action-hint-intro {
  opacity: 0;
  z-index: 2;
}
.container .letter-wrapper .action-hint-check {
  top: -70px;
  color: #fff;
  text-shadow: 0 3px 10px rgba(0, 0, 0, 0.28);
}
.check-hint-fade-enter-active,
.check-hint-fade-leave-active {
  transition:
    opacity 0.32s ease,
    transform 0.32s ease;
}
.check-hint-fade-enter-from,
.check-hint-fade-leave-to {
  opacity: 0;
  transform: translateY(6px);
}
.container .letter-wrapper .action-hint-intro.show {
  animation: 0.55s cubic-bezier(0.2, 0.86, 0.28, 1) forwards hintRiseIn;
}
.action-hint-fade-leave-active {
  transition: opacity 0.35s ease;
}
.action-hint-fade-leave-to {
  opacity: 0;
}
@keyframes hintRiseIn {
  from {
    opacity: 0;
    transform: translateY(420px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
@keyframes checkFadeIn {
  from {
    opacity: 0;
    transform: translateY(6px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
.container .letter-wrapper .inner-greeting {
  position: absolute;
  top: 50px;
  left: 0;
  width: 100%;
  text-align: center;
  z-index: 1;
  font-size: 34px;
  font-weight: 700;
  color: #6f3e11;
  letter-spacing: 2px;
  pointer-events: none;
  user-select: none;
}
.container .flip-hint {
  position: absolute;
  top: 82vh;
  left: 0;
  width: 100%;
  text-align: center;
  font-size: 22px;
  font-weight: 700;
  color: #5a2e0c;
  letter-spacing: 1px;
  z-index: 7;
  pointer-events: none;
  user-select: none;
}
.flip-hint-fade-enter-active,
.flip-hint-fade-leave-active {
  transition:
    opacity 0.4s ease,
    transform 0.4s ease;
}
.flip-hint-fade-enter-from {
  opacity: 0;
  transform: translateY(10px);
}
.flip-hint-fade-leave-to {
  opacity: 0;
  transform: translateY(120vh);
}
.container .letter-wrapper .cover {
  position: absolute;
  width: 100%;
  height: 210px;
  z-index: 5;
  background-image: url(/assets/cover-front.png);
  background-position: 0;
  background-repeat: no-repeat;
  background-size: cover;
  top: 0;
  right: 3px;
  cursor: pointer;
  transform-origin: 0 5px;
}
.container .letter-wrapper.opened .cover {
  animation: 0.3s forwards rollingOver;
}
@keyframes rollingOver {
  10% {
    background-image: url(/assets/cover-front.png);
  }
  100% {
    z-index: 2;
    background-image: url(/assets/cover-back.png);
    transform: rotateX(-180deg);
  }
}
.container .letter-wrapper .letter {
  position: absolute;
  top: -110px;
  z-index: 3;
  width: 450px;
  height: 675px;
  margin: 0;
  background: transparent;
  box-shadow: none;
  overflow: hidden;
  transform: translateY(180px);
}
.container .letter-wrapper .letter.reading {
  overflow: visible;
  z-index: 6;
}
.container .letter-wrapper .letter.clickable {
  cursor: pointer;
}
.container .letter-wrapper .letter .letter-page {
  position: relative;
  width: 100%;
  height: 100%;
  transform-style: preserve-3d;
  transform-origin: center center;
  transition: transform 0.55s ease;
}
.container .letter-wrapper .letter .letter-page.flipped {
  animation: 2s cubic-bezier(0.2, 0.7, 0.16, 1) forwards flipToInnerZoom;
}
.container .letter-wrapper .letter .letter-face {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  border: 0;
  object-fit: cover;
  object-position: top;
  backface-visibility: hidden;
  -webkit-backface-visibility: hidden;
  display: block;
}
.container .letter-wrapper .letter .letter-face.back {
  transform: rotateY(180deg);
}
@keyframes flipToInnerZoom {
  0% {
    transform: rotateY(0deg) scale(1) translateY(var(--flip-y-start));
  }
  20% {
    transform: rotateY(90deg) scale(1.2) translateY(var(--flip-y-mid));
  }
  100% {
    transform: rotateY(180deg) scale(var(--flip-scale-end)) translateY(var(--flip-y-end));
  }
}
.container .letter-wrapper.opened .letter {
  animation: 0.8s 0.08s forwards letterBounceInUp;
}

.container .letter-wrapper.extracted {
  animation: 0.75s 0s forwards wrapperDrop;
}

.container .letter-wrapper.extracted .letter {
  animation: 1s 0s forwards letterLiftUp;
}
@keyframes letterBounceInUp {
  from {
    transform: translateY(180px);
  }
  40% {
    transform: translateY(-20px);
  }
  75% {
    transform: translateY(0px);
  }
  100% {
    transform: translateY(0);
  }
}
@keyframes letterLiftUp {
  from {
    transform: translateY(0);
  }
  40% {
    transform: translateY(-150vh);
  }
  100% {
    transform: translateY(min(-130vh, -950px));
  }
}
@keyframes wrapperDrop {
  from {
    transform: translateY(0);
  }
  to {
    transform: translateY(100vh);
  }
}
.container .letter-wrapper .envelope {
  position: absolute;
  width: 500px;
  height: 360px;
  z-index: 4;
  background-image: url(/assets/envelope.png);
  background-position: center;
  background-repeat: no-repeat;
  background-size: cover;
  display: flex;
  flex-direction: column;
  justify-content: flex-end;
  align-items: center;
  padding-bottom: 40px;
  font-size: 24px;
  color: #000;
}
.container .letter-wrapper .envelope b {
  opacity: 0.3;
}
.container .letter-wrapper .envelope .envelope-logo {
  width: 88px;
  height: auto;
  margin-bottom: 8px;
  opacity: 0.82;
}
.container .letter-wrapper .envelope small {
  opacity: 0.3;
  font-weight: 200;
}

@media (max-width: 768px) {
  :root {
    --flip-scale-end: 1.2;
    --flip-y-start: 0;
    --flip-y-mid: -5px;
    --flip-y-end: -15px;
  }
}
</style>
