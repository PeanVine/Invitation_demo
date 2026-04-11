<template>
  <p class="arrival-tip" :class="{ show: showArrivalTip, leave: isIntroDone }">
    <span class="arrival-tip-icon" aria-hidden="true"></span>
    <span>叮咚! 您有一封新的邮件送达!</span>
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
          @click.stop="handleCheckClick"
        >
          Check!
        </p>
      </transition>
      <p class="inner-greeting">欢迎回家!</p>
      <div class="cover" @click="openEnvelope">
        <img
          src="/assets/Tenzor_Logo.png"
          alt="Tenzor Logo"
          class="cover-logo"
        />
      </div>
      <div
        class="letter"
        :class="{
          clickable: isOpened && (!isExtracted || !isFlipped),
          reading: isFlipped,
        }"
        @click="handleLetterClick"
      >
        <div
          class="letter-book"
          :class="{
            flipped: isFlipped,
            mobile: isMobile,
            'focus-left': focusState === 'left',
            'focus-right': focusState === 'right',
          }"
        >
          <!-- 翻页后的右面 (底层不动的页) -->
          <div
            class="book-page right-page"
            :class="{ interactive: isFlipped && !isMobile }"
            @click="handlePageClick('right')"
          >
            <img src="/assets/内页.png" alt="内页" class="letter-face" />

            <!-- 新增的内页文字部分 -->
            <div class="inner-typography">
              <p class="greeting">Hi, Tenzor:</p>
              <p class="indent-p">
                2026年4月8日，在西安交通大学130周年校庆之际，挑战网也将迎来25周年站庆。
              </p>
              <p class="indent-p">
                冬去春又来。挑战又走过了十个年头，现在的挑战，有技术、视频、美工三个部门，有七十多名成员。
              </p>
              <p class="indent-p">
                时过境迁，学业的担子越来越重，ai浪潮高涨不休，但挑战依旧，我们也仍在迎接时代挑战。
              </p>
              <p class="indent-p">
                已经毕业了的Tenzor，夜深人静的时候，你会不会偶示想起西南角的小阁楼，想起自己全心全意迎接挑战的样子？
              </p>
              <p class="indent-p">
                衷心感谢各位前辈长久以来对挑战的支持。在这个特殊的日子，诚挚邀请各位再聚阁楼，再话当年。
              </p>
              <p class="indent-p">
                我们也精心准备了一份小礼物，待您来时，为您献上。
              </p>
              <div class="inner-footer">
                <p class="center-text">时间：2026年4月14日</p>
                <p class="right-text">挑战网25周年站庆筹备组</p>
              </div>
              <p class="support-footer">支持我们（支付宝）</p>
            </div>
          </div>
          <!-- 可以翻动的页 -->
          <div
            class="book-page flip-page"
            :class="{ interactive: isFlipped && !isMobile }"
            @click="handlePageClick('left')"
          >
            <!-- 封面 -->
            <div class="page-face front-face">
              <img src="/assets/封面.png" alt="封面" class="letter-face" />
              <div class="typography-layout">
                <div class="header-text">邀请函</div>
                <div class="center-box">
                  <div class="split-line"></div>
                  <div class="content left">
                    <span>地点</span>
                  </div>
                  <div class="content right">
                    <span>西安交通大学</span>
                    <span>挑战网工作室</span>
                  </div>
                </div>
                <div class="footer-text">INVITATION</div>
              </div>
            </div>
            <!-- 翻开后的左面 (背身) -->
            <div class="page-face back-face">
              <img src="/assets/封面.png" alt="左面" class="letter-face" />
            </div>
          </div>
        </div>
      </div>
      <div class="envelope">
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
const envelopeBaseWidth = 540;
const isMobile = ref(false);
const focusState = ref<"center" | "left" | "right">("center");

let introTimer: ReturnType<typeof setTimeout> | null = null;
let hintTimer: ReturnType<typeof setTimeout> | null = null;
let checkHintTimer: ReturnType<typeof setTimeout> | null = null;
let flipHintTimer: ReturnType<typeof setTimeout> | null = null;

const updateSceneScale = () => {
  // 处理移动端手势放大时 visualViewport 尺寸缩小导致二次缩小的冲突问题
  const zoomScale = window.visualViewport?.scale || 1;
  // 如果处于双指缩放状态，直接跳过重新计算避免引发频繁回流打断原生手势
  if (zoomScale > 1) return;

  const viewportWidth = window.visualViewport?.width ?? window.innerWidth;
  const viewportHeight = window.visualViewport?.height ?? window.innerHeight;
  isMobile.value = viewportWidth <= 768;
  const simulatedMobileUiInset = isMobile.value
    ? Number.parseFloat(
        getComputedStyle(document.documentElement).getPropertyValue(
          "--mobile-ui-simulated-inset",
        ),
      ) || 0
    : 0;
  const usableViewportHeight = Math.max(
    320,
    viewportHeight - simulatedMobileUiInset,
  );
  const widthScale = (viewportWidth - 24) / envelopeBaseWidth;
  const heightScale = (usableViewportHeight - 24) / 600;
  let newScale = Math.min(0.78, widthScale, heightScale);
  if (isMobile.value) {
    // 手机端将整体信封场景再等比缩小一些，弄成0.8
    newScale *= 0.8;
  }
  sceneScale.value = newScale;
  document.documentElement.style.setProperty(
    "--app-vh",
    `${usableViewportHeight}px`,
  );
  document.documentElement.style.setProperty("--app-vw", `${viewportWidth}px`);
};

const handlePageClick = (side: "left" | "right") => {
  if (isMobile.value) return;
  if (isFlipped.value) {
    focusState.value = side;
  }
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

const startExtractLetter = () => {
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
};

const handleCheckClick = () => {
  if (!isOpened.value || isExtracted.value || !showCheckHint.value) return;
  startExtractLetter();
};

const handleLetterClick = () => {
  if (!isOpened.value) return;

  if (!isExtracted.value) {
    // 未抽出前只允许点击 Check 触发抽信，其他位置点击无效。
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
  window.visualViewport?.addEventListener("resize", updateSceneScale);
  window.visualViewport?.addEventListener("scroll", updateSceneScale);

  // 初次加载优化：预加载重型图片后再开始渲染动画，防止首次绘制卡顿
  const preloadUrls = [
    "/assets/bg.png",
    "/assets/cover-front.png",
    "/assets/envelope.png",
    "/assets/Tenzor_Logo.png",
    "/assets/page_bg.jpg",
  ];
  let loaded = 0;
  const startTime = Date.now();

  const startIntro = () => {
    // 保证提示气泡至少有 1.5 秒的展示时间，即使图片瞬间从缓存加载完也不会闪退
    const elapsed = Date.now() - startTime;
    const delay = Math.max(1500 - elapsed, 100);

    introTimer = setTimeout(() => {
      isIntroDone.value = true;
    }, delay);
    hintTimer = setTimeout(() => {
      if (!isOpened.value) {
        showUnopenedHint.value = true;
      }
    }, delay + 1500);
  };

  preloadUrls.forEach((url) => {
    const img = new Image();
    img.onload = img.onerror = () => {
      loaded++;
      if (loaded === preloadUrls.length) {
        startIntro();
      }
    };
    img.src = url;
  });
});

onBeforeUnmount(() => {
  window.removeEventListener("resize", updateSceneScale);
  window.visualViewport?.removeEventListener("resize", updateSceneScale);
  window.visualViewport?.removeEventListener("scroll", updateSceneScale);
  document.documentElement.style.removeProperty("--app-vh");
  document.documentElement.style.removeProperty("--app-vw");
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
  /* 禁止电脑端选中文字和图片拖拽 */
  user-select: none;
  -webkit-user-select: none;
  -webkit-user-drag: none;
  /* 禁止手机端点击时出现蓝色/灰色高亮背景框 */
  -webkit-tap-highlight-color: transparent;
  /* 移除特定元素的焦点外边框 */
  outline: none;
}
:root {
  --clr: #fcfcf589;
  --intro-landing-offset: 24px;
  --mobile-ui-simulated-inset: 0px;
  /*信封整体宽度*/
  --envelope-width: 700px;
  --flip-scale-end: 1.5;
  --flip-y-start: 0;
  --flip-y-mid: 10px;
  --flip-y-end: 15vh;
  /* ===👇 调试信封颜色的位置 👇=== */
  /* 通过调节 filter 中的 sepia/hue-rotate/brightness 给外壳上色 */
  --envelope-red-filter: saturate(0) sepia(0.2) hue-rotate(10deg)
    brightness(2.3) contrast(0.95);
  /* 信封内衬背景，也采用滤镜统一控制 */
  --envelope-inner-filter: saturate(0.15) sepia(0.2) hue-rotate(10deg)
    brightness(3) contrast(0.8);
  /* 翻开折角的内侧颜色，比外侧略深一点产生阴影感 */
  --envelope-ivory-filter: saturate(0.15) sepia(0.2) hue-rotate(10deg)
    brightness(2.1) contrast(0.75);
  /* ============================ */
}

html {
  height: var(--app-vh, 100dvh);
  overflow: hidden;
}
body {
  width: 100%;
  min-height: var(--app-vh, 100dvh);
  overflow: hidden;
  background-color: var(--clr);
  display: flex;
  justify-content: center;
  align-items: center;
  padding-top: env(safe-area-inset-top);
  padding-bottom: env(safe-area-inset-bottom);
}
body::before {
  content: "";
  position: fixed;
  top: 50%;
  left: 50%;
  width: 100vh;
  height: 100vw;
  background-image: url(/assets/page_bg.jpg);
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
  transform: translate(-50%, -50%) rotate(90deg);
  z-index: -10;
}
html.reading-scroll,
body.reading-scroll {
  height: auto;
  min-height: 100dvh;
  overflow: auto;
  /* 允许缩放和多向平移，修复部分手机端浏览器因强制屏蔽横向滚动导致双指失效 */
  touch-action: pan-x pan-y pinch-zoom;
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
  transform: translate(-50%, -26px);
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
  animation: none;
}
.arrival-tip.leave {
  animation: 0.28s ease-in forwards tipFadeOut;
}
.container {
  position: relative;
  width: var(--envelope-width);
  height: 600px;
  transform: translate3d(0, var(--intro-landing-offset), 0)
    scale(var(--scene-scale));
  transform-origin: center bottom;
  overflow: visible;
  /* 基础的下部裁剪，控制信封底部隐藏 */
  clip-path: inset(-100vh 0 0 0);
  display: flex;
  flex-direction: column;
  justify-content: flex-end;
  align-items: center;
  transition: clip-path 0s linear;
  /* 将基础透明度设为极小值而非0，迫使浏览器在开场加载阶段就把这个庞大的组件完整绘制并上传到GPU缓存中，避免动画开始时瞬间暴增开销 */
  opacity: 0.001;
  pointer-events: none;
  /* 提前挂载硬件加速层，防止添加 .ready 瞬间因重新分层导致计算卡顿 */
  will-change: transform, opacity;
  backface-visibility: hidden;
  -webkit-backface-visibility: hidden;
  perspective: 1000px;
}
.container.ready {
  animation: 1.2s cubic-bezier(0.37, -0.09, 0, 1.6) bounceInUp;
  opacity: 1;
  pointer-events: auto;
}
.container.extracted {
  /* 抽出后彻底干掉裁剪蒙版，避免在任何缩放级别下误切底部 */
  clip-path: none;
  /* 👇 抽出信时，下部裁剪区域取消的过渡动画时长调节 */
  transition: clip-path 0.1s ease-in-out;
}
.container.reading {
  clip-path: none;
  transition: clip-path 0.12s ease-out;
  transition-delay: 0s;
}
@keyframes tipDropIn {
  from {
    opacity: 0;
    transform: translate(-50%, -26px);
  }
  to {
    opacity: 1;
    transform: translate(-50%, 0);
  }
}
@keyframes tipFadeOut {
  from {
    opacity: 1;
    transform: translate(-50%, 0);
  }
  to {
    opacity: 0;
    transform: translate(-50%, -26px);
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
    transform: translate3d(0, calc(var(--intro-landing-offset) + 2000px), 0)
      scale(calc(var(--scene-scale) * 0.1));
  }
  65% {
    opacity: 1;
    transform: translate3d(0, calc(var(--intro-landing-offset) - 100px), 0)
      scale(var(--scene-scale));
  }
  75% {
    transform: translate3d(0, calc(var(--intro-landing-offset) + 20px), 0)
      scale(var(--scene-scale));
  }
  90% {
    transform: translate3d(0, calc(var(--intro-landing-offset) - 20px), 0)
      scale(var(--scene-scale));
  }
  100% {
    transform: translate3d(0, var(--intro-landing-offset), 0)
      scale(var(--scene-scale));
  }
}
.container .letter-wrapper {
  position: relative;
  width: 100%;
  height: 360px;
  display: flex;
  justify-content: center;
  align-items: flex-end;
}
.container .letter-wrapper::before {
  content: "";
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-image: url(/assets/bg.png);
  background-repeat: no-repeat;
  /* 应用相同的米黄色滤镜 */
  filter: var(--envelope-inner-filter);
  z-index: 0;
}
.container .letter-wrapper .action-hint {
  position: absolute;
  top: -44px;
  left: 2%;
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

/*check最终停留位置*/
.container .letter-wrapper .action-hint-check {
  top: -30px;
  color: #fff;
  text-shadow: 0 3px 10px rgba(0, 0, 0, 0.28);
  pointer-events: auto;
  cursor: pointer;
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
  top: 30%;
  right: -30px;
  width: auto;
  font-size: 22px;
  font-weight: 700;
  color: #5a2e0c;
  letter-spacing: 8px;
  writing-mode: vertical-rl;
  transform: translateY(-50%);
  z-index: 10;
  pointer-events: none;
  user-select: none;
}
.flip-hint-fade-enter-active {
  /* 渐入的同时向右移动并减速(ease-out 曲线自带减速效果) */
  transition:
    opacity 0.8s ease-out,
    transform 0.8s cubic-bezier(0.1, 0.9, 0.2, 1);
}
.flip-hint-fade-leave-active {
  /* 消失时要求0.2秒内消失 */
  transition: opacity 0.2s ease;
}
.flip-hint-fade-enter-from {
  opacity: 0;
  /* 从左边偏 -30px 的位置开始，这样到原位 0 就形成了“向右”动作 */
  transform: translate(-30px, -50%);
}
.flip-hint-fade-leave-to {
  opacity: 0;
  /* 原地消失即可，也可以微调 translate 保持平滑 */
  transform: translateY(-50%);
}

/* 闭合三角封口 */
.container .letter-wrapper .cover {
  position: absolute;
  width: 100%;
  height: 300px;
  z-index: 5;
  background-image: url(/assets/cover-front.png);
  background-position: 0;
  background-repeat: no-repeat;
  background-size: cover;
  top: 0;
  right: 3px;
  cursor: pointer;
  transform-origin: 0 5px;
  filter: var(--envelope-red-filter);
  /* 为了支持内部logo的透视隐藏效果 */
  transform-style: preserve-3d;
}

.container .letter-wrapper .cover .cover-logo {
  position: absolute;
  bottom: 10px; /* 控制 Logo在中间偏下三角的位置 */
  left: 52%;
  width: 95px;
  height: auto;
  transform: translateX(-50%);
  opacity: 0.85;
  /* 翻开时随着背面隐藏 */
  backface-visibility: hidden;
  -webkit-backface-visibility: hidden;
  transition: opacity 0.15s ease;
}

/* 刚开始翻开时立刻把 logo 隐藏掉 */
.container .letter-wrapper.opened .cover .cover-logo {
  opacity: 0;
}

/* 展开三角封口 */
.container .letter-wrapper.opened .cover {
  animation: 0.3s forwards rollingOver;
}

/* 三角封口翻转动画 */
@keyframes rollingOver {
  0% {
    background-image: url(/assets/cover-front.png);
    filter: var(--envelope-red-filter);
  }
  50% {
    background-image: url(/assets/cover-front.png);
    filter: var(--envelope-red-filter);
  }
  50% {
    z-index: 2;
    background-image: url(/assets/cover-back.png);
    filter: var(--envelope-ivory-filter);
  }
  100% {
    z-index: 2;
    background-image: url(/assets/cover-back.png);
    transform: rotateX(-180deg);
    filter: var(--envelope-ivory-filter);
  }
}
.container .letter-wrapper .letter {
  position: absolute;
  top: auto;
  bottom: 0;
  z-index: 3;
  /* 信的高度及宽度 */
  width: calc(var(--envelope-width) * 0.95);
  height: calc(var(--envelope-width) * 0.63);
  min-height: 320px;
  max-height: 420px;
  margin: 0;
  background: transparent;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.25);
  overflow: visible;
  transform: translateY(180px);
}
.container .letter-wrapper .letter.reading {
  overflow: visible;
  z-index: 6;
}
.container .letter-wrapper .letter.clickable {
  cursor: pointer;
}
.container .letter-wrapper .letter .letter-book {
  position: relative;
  width: 100%;
  height: 100%;
  transform-style: preserve-3d;
  /* 添加透视，让3D翻转更真实 */
  perspective: 1500px;
  /* 整体平移时间 */
  transition: transform 1.2s cubic-bezier(0.2, 0.7, 0.16, 1);
  container-type: size;
}
/* 因为.letter局部转了-90度，原本的Y轴变成了屏幕的水平方向(向右)。整体向右移动即translateY(50%)，调小可以偏左 */
.container .letter-wrapper .letter .letter-book.flipped {
  transform: translateY(30%);
}
.container .letter-wrapper .letter .letter-book.flipped.focus-left {
  transform: translateY(70%);
}
.container .letter-wrapper .letter .letter-book.flipped.focus-right {
  transform: translateY(30%);
}
.container .letter-wrapper .letter .letter-book.flipped.mobile {
  transform: translateY(0%);
}

.container .letter-wrapper .letter .book-page {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  transform-style: preserve-3d;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.25);
  border-radius: 4px;
}
.container .letter-wrapper .letter .book-page.interactive {
  cursor: pointer;
}

.container .letter-wrapper .letter .right-page {
  z-index: 1;
}

.container .letter-wrapper .letter .flip-page {
  /* .letter逆时针旋90度后，局部的top变成了屏幕的left。所以沿着局部top旋转即是在屏幕上像书一样横向翻页 */
  transform-origin: top center;
  transition: transform 1.2s cubic-bezier(0.2, 0.7, 0.16, 1);
  z-index: 2;
}

/* 书页翻开。正对着视野沿X轴旋转180度 */
.container .letter-wrapper .letter .letter-book.flipped .flip-page {
  transform: rotateX(180deg);
}

/* 手机端直接翻转到背部(360度) 并在Z轴后退以免与背部重叠 */
.container .letter-wrapper .letter .letter-book.flipped.mobile .flip-page {
  transform: rotateX(360deg) translateZ(-5px);
}

.container .letter-wrapper .letter .page-face {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  /* 隐藏背板 */
  backface-visibility: hidden;
  -webkit-backface-visibility: hidden;
  border-radius: 4px;
  overflow: hidden;
  background-color: var(--clr);
}

.container .letter-wrapper .letter .front-face {
  z-index: 2;
}

/* 封面文字排版 */
.typography-layout {
  position: absolute;
  top: 50%;
  left: 48%;
  transform: translate(-50%, -50%) rotate(90deg);
  display: flex;
  flex-direction: column;
  color: #fff;
  z-index: 10;
  pointer-events: none;
  /*字体粗细*/
  font-weight: 700;
  /*字号大小*/
  font-size: 16px;
  font-family: "SimHei", "黑体", sans-serif;
}
.header-text {
  align-self: flex-start;
  letter-spacing: 0rem;
  margin-bottom: 0.2rem;
  margin-left: 2.5rem;
}
.center-box {
  position: relative;
  width: 240px;
  height: 76px;
  border: 1px solid #fff;
  background: transparent;
}
.split-line {
  position: absolute;
  top: 70%;
  left: 33%;
  width: 260px;
  height: 1px;
  background: #fff;
  transform: translate(-50%, -50%) rotate(-64deg);
}
.content.left {
  position: absolute;
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  left: 10%;
}
.content.left span {
  letter-spacing: 0rem;
}
.content.right {
  position: absolute;
  height: 100%;
  right: 10%;
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  justify-content: center;
  gap: 0.6rem;
}
.content.right span:last-child {
  margin-right: 1rem;
}
.footer-text {
  align-self: flex-end;
  font-family: "DengXian", "等线", sans-serif;
  letter-spacing: 0rem;
  margin-top: 0.4rem;
  margin-right: 4.5rem;
  font-weight: bold;
}

/* 内页文字排版（与封面同层覆盖在图片上） */
.inner-typography {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 100cqh;
  height: 95cqw;
  transform: translate(-50%, -50%) rotate(90deg);
  z-index: 10;
  pointer-events: none;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
  padding: 34px 42px 26px;
  color: #222;
  line-height: 1.6;
}

.inner-typography p {
  margin: 0;
  line-height: 1.65;
}

.inner-typography .greeting {
  font-size: 1.2rem;
  font-weight: 600;
  margin-bottom: 20px;
}

.inner-typography .indent-p {
  font-weight: 400;
  font-size: 0.9rem;
  text-indent: 2em;
}

.inner-typography .inner-footer {
  margin-top: 10px;
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.inner-typography .center-text {
  align-self: center;
  font-size: 0.82rem;
}

.inner-typography .right-text {
  align-self: flex-end;
  font-size: 0.82rem;
}

.support-footer {
  align-self: flex-start;
  padding-top: 8px;
  padding-left: 10px;
  font-size: 0.9rem;
}

/* 左侧页在翻开前背对着我们，所以沿X周预先转180度 */
.container .letter-wrapper .letter .back-face {
  transform: rotateX(180deg);
  z-index: 1;
  /* ======👇调试左侧页边框 👇======== */
  /* 可以修改 `4px` 调整粗细，或者修改 `red` 调整颜色 */
  border-style: solid;
  border-color: rgb(129, 0, 0);
  border-width: 8px 20px 8px 20px;
  box-sizing: border-box;
  /* ============================== */
}

/* 给翻开后左侧页内的图片加上黑白滤镜 */
.container .letter-wrapper .letter .back-face .letter-face {
  /* ======👇调试图片黑白滤镜👇======== */
  /* grayscale() 是纯黑白，opacity() 则可以通过降低透明度透出底层底色，从而让黑色减淡 */
  filter: grayscale(100%) opacity(100%);
  /* ================================ */
}

/* 由于外层已经是正确的宽长比了，图片需要像之前一样竖转横 */
.container .letter-wrapper .letter .letter-face {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 100cqh;
  height: 100cqw;
  border: 0;
  object-fit: cover;
  display: block;
  transform: translate(-50%, -50%) rotate(90deg);
  transform-origin: center center;
}
.container .letter-wrapper.opened .letter {
  animation: 0.8s 0.08s forwards letterBounceInUp;
}

.container .letter-wrapper.extracted {
  animation: 0.75s 0s forwards wrapperDrop;
}

/*信抬升动画*/
.container .letter-wrapper.extracted .letter {
  animation: 2s 0s forwards letterLiftUp;
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
/*封面旋并放大动画*/
@keyframes letterLiftUp {
  from {
    transform: translateY(0) rotate(0deg) scale(1);
  }
  20% {
    transform: translateY(-150vh) rotate(-0deg) scale(1);
  }
  50% {
    transform: translateY(-105vh) rotate(-90deg) scale(1.4);
  }
  100% {
    transform: translateY(min(-100vh, -890px)) rotate(-90deg) scale(1.4);
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
  width: var(--envelope-width);
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
  filter: var(--envelope-red-filter);
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
    --intro-landing-offset: -16px;
    --flip-scale-end: 1.2;
    --flip-y-start: 0;
    --flip-y-mid: -5px;
    --flip-y-end: -15px;
  }

  body {
    min-height: var(--app-vh, 100dvh);
  }

  /* 手机端抽信阶段向上移动更大，彻底放宽裁剪限制避免被切边 */
  .container.extracted,
  .container.reading {
    clip-path: none;
  }

  .container.extracted {
    transition-delay: 0s;
  }

  /* 手机端覆盖提示气泡动画，使其等比缩小至 0.8 */
  @keyframes tipDropIn {
    from {
      opacity: 0;
      transform: translate(-50%, -26px);
    }
    to {
      opacity: 1;
      transform: translate(-50%, 0);
    }
  }
  @keyframes tipFadeOut {
    from {
      opacity: 1;
      transform: translate(-50%, 0);
    }
    to {
      opacity: 0;
      transform: translate(-50%, -26px);
    }
  }

  /* 手机端翻页提示改为横排在底部 */
  .container .flip-hint {
    top: calc(100% - 40px);
    bottom: auto;
    left: 50%;
    right: auto;
    width: max-content;
    max-width: calc(100vw - 24px);
    transform: translateX(-50%);
    writing-mode: horizontal-tb;
    letter-spacing: 4px;
    text-align: center;
    white-space: nowrap;
  }
  /* 手机端向下渐入 */
  .flip-hint-fade-enter-from {
    opacity: 0;
    transform: translate(-50%, -30px); /* 偏上，恢复时往下形成向下渐入效果 */
  }
  /* 手机端快速渐出 */
  .flip-hint-fade-leave-to {
    opacity: 0;
    transform: translate(-50%, 0);
  }

  /* 手机端覆盖抽取动画，让信件抽出来后挪得更高一点 */
  @keyframes letterLiftUp {
    from {
      transform: translateY(0) rotate(0deg) scale(1);
    }
    20% {
      transform: translateY(-168vh) rotate(0deg) scale(1.2);
    }
    50% {
      transform: translateY(-132vh) rotate(-90deg) scale(1.55);
    }
    100% {
      transform: translateY(min(-128vh, -1100px)) rotate(-90deg) scale(1.55);
    }
  }
}
</style>
