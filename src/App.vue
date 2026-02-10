<template>
  <div id="app">
    <transition name="fade" v-if="showIntro">
      <div class="intro-container" @click="showIntro = false">
        <!-- 视频背景 -->
        <div class="video-wrapper">
          <video
            class="video-background"
            :src="videoSrc"
            autoplay
            muted
            loop
            playsinline
          ></video>
          <div class="video-overlay"></div>
        </div>

        <!-- 动态弦线覆盖层 -->
        <div class="string-overlay">
          <div
            class="string"
            v-for="(str, i) in strings"
            :key="i"
            :style="getStringStyle(str, i)"
          ></div>
        </div>

        <!-- 剪断效果 -->
        <div class="cutting-effect" :style="cuttingStyle"></div>

        <div class="intro-content" aria-live="polite">
          <div class="intro-inner">
            <!-- 剪刀动画装饰 -->
            <div class="scissors-animation">
              <div class="scissors-icon">✂</div>
              <div class="cutting-line"></div>
            </div>

            <!-- 主标题 -->
            <div class="main-title">
              <span
                class="title-char"
                v-for="(char, i) in titleChars"
                :key="i"
                :style="getCharStyle(i)"
                >{{ char }}</span
              >
            </div>

            <!-- 打字机语句 -->
            <div class="typewriter-container">
              <div class="typewriter-wrapper">
                <p class="typewriter-text">
                  {{ displayText }}
                  <span class="cursor" v-if="isTyping"></span>
                </p>
              </div>
              <div class="subtitle">解弦之眼 · 千咲</div>
            </div>

         
          </div>

          <!-- 交互提示 -->
          <div class="interaction-hint" :class="{ visible: showHint }">
            <div class="hint-line"></div>
            <div class="hint-text">点击切断入场</div>
            <div class="hint-scissors">✂</div>
          </div>
        </div>
      </div>
    </transition>
    <div v-else>
      <navbar />
      <main class="main-content">
        <RouterView />
      </main>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, computed } from "vue";
import { RouterView } from "vue-router";
import navbar from "@/components/navbar.vue";

const showIntro = ref(true);
const videoSrc = ref("");
const displayText = ref("");
const isTyping = ref(true);
const showHint = ref(false);
const titleAnimationProgress = ref(0);

// 千咲风格语句
const lines = [
  "在废都的断章光影中，千咲以剪断弱点的眼，注视着你的步伐。",
  "欢迎，漂泊者。在这片时隙废都，她将与你一同拆解未知的结构。",
  "时间不停轮回，但你的脚步不会停——千咲与共。",
  "在弱点绽放为线的瞬间，她的剪刀划破迷雾，引路而行。",
  "当废墟的回声与湮灭交织，握紧你的信念，我们一同探索。",
  "你穿越了未知之地，这里是断裂与重构的交汇——千咲在此等候。",
  "在废都樱花飘落之处，你的旅途与理性之刃再次交错。",
  "漂泊者，前方是无尽的绞碎与解构，千咲将与你同行。",
] as const;

// 弦线数据
interface StringData {
  id: number;
  x: number;
  y: number;
  length: number;
  opacity: number;
  speed: number;
  swing: number;
  swingSpeed: number;
  isCut: boolean;
  cutProgress: number;
}

const strings = ref<StringData[]>([]);
const STRING_COUNT = 15;
const titleChars = "千咲".split("");

let typingTimer: number | null = null;
let hintTimer: number | null = null;
let titleTimer: number | null = null;
let stringTimer: number | null = null;

// 剪断效果
const cuttingProgress = ref(0);
const cuttingAngle = ref(0);

const cuttingStyle = computed(() => ({
  "--cut-progress": `${cuttingProgress.value}`,
  "--cut-angle": `${cuttingAngle.value}deg`,
  "--cut-opacity": `${cuttingProgress.value > 0 ? 0.6 : 0}`,
}));

// 初始化弦线
const initStrings = () => {
  strings.value = [];
  for (let i = 0; i < STRING_COUNT; i++) {
    strings.value.push({
      id: i,
      x: Math.random() * 100,
      y: Math.random() * 100,
      length: 20 + Math.random() * 60,
      opacity: 0.1 + Math.random() * 0.15,
      speed: 0.2 + Math.random() * 0.4,
      swing: 2 + Math.random() * 8,
      swingSpeed: 0.5 + Math.random() * 1.5,
      isCut: Math.random() > 0.7,
      cutProgress: Math.random(),
    });
  }
};

// 弦线样式
const getStringStyle = (str: StringData, index: number) => {
  const time = Date.now() / 1000;
  const swingOffset = Math.sin(time * str.swingSpeed + index) * str.swing;
  const cutHeight = str.length * str.cutProgress;

  return {
    left: `${str.x}%`,
    top: `${str.y}%`,
    height: `${str.length}px`,
    opacity: str.opacity.toString(),
    transform: `translateX(${swingOffset}px)`,
    "--cut-height": `${cutHeight}px`,
    animationDelay: `${index * 0.1}s`,
    animationDuration: `${3 + Math.random() * 2}s`,
  };
};

// 标题字符样式
const getCharStyle = (index: number) => {
  const delay = index * 0.2;
  const isVisible = titleAnimationProgress.value > delay;

  return {
    opacity: isVisible ? "1" : "0",
    transform: isVisible
      ? "translateY(0) scale(1)"
      : "translateY(20px) scale(0.8)",
    transitionDelay: `${delay}s`,
    textShadow: isVisible ? "0 0 10px rgba(244, 67, 54, 0.5)" : "none",
  };
};

// 动画弦线
const animateStrings = () => {
  stringTimer = window.setInterval(() => {
    strings.value.forEach((str, i) => {
      // 移动弦线
      str.y += str.speed;
      if (str.y > 100) {
        str.y = -str.length / 2;
        str.x = Math.random() * 100;
        str.isCut = Math.random() > 0.7;
        str.cutProgress = 0;
      }

      // 剪断动画
      if (str.isCut && str.cutProgress < 1) {
        str.cutProgress += 0.01;
      }

      // 随机剪断
      if (!str.isCut && Math.random() < 0.002) {
        str.isCut = true;
        str.cutProgress = 0;
      }
    });
  }, 50);
};

// 剪断效果动画
const animateCuttingEffect = () => {
  let progress = 0;
  const interval = window.setInterval(() => {
    if (progress >= 1) {
      clearInterval(interval);
      // 重置
      setTimeout(() => {
        cuttingProgress.value = 0;
      }, 1000);
      return;
    }

    progress += 0.03;
    cuttingProgress.value = progress;
    cuttingAngle.value = 15 + Math.sin(Date.now() / 200) * 10;
  }, 30);

  // 随机触发剪断效果
  return interval;
};

// 随机剪断效果
const startRandomCutting = () => {
  const randomCut = () => {
    if (Math.random() > 0.7 && cuttingProgress.value === 0) {
      animateCuttingEffect();
    }
    setTimeout(randomCut, 2000 + Math.random() * 3000);
  };
  randomCut();
};

/* 打字机效果 */
const typingSpeed = 80;

function pickRandomLine(): string {
  const idx = Math.floor(Math.random() * lines.length);
  return lines[idx];
}

function startTypewriter(line: string) {
  const reduce =
    window.matchMedia &&
    window.matchMedia("(prefers-reduced-motion: reduce)").matches;

  if (reduce) {
    displayText.value = line;
    isTyping.value = false;
    showHintAfterDelay();
    return;
  }

  let i = 0;
  isTyping.value = true;
  displayText.value = "";

  typingTimer = window.setInterval(() => {
    i++;
    displayText.value = line.slice(0, i);

    if (i >= line.length) {
      if (typingTimer) {
        clearInterval(typingTimer);
        typingTimer = null;
      }
      isTyping.value = false;
      showHintAfterDelay();
    }
  }, typingSpeed);
}

// 标题动画
const startTitleAnimation = () => {
  titleAnimationProgress.value = 0;
  titleTimer = window.setInterval(() => {
    titleAnimationProgress.value += 0.05;
    if (titleAnimationProgress.value >= 1) {
      if (titleTimer) clearInterval(titleTimer);
    }
  }, 100);
};

// 显示提示
function showHintAfterDelay() {
  hintTimer = window.setTimeout(() => {
    showHint.value = true;
  }, 800);
}

onMounted(() => {
  // 检测是否为移动端
  const isMobile = window.innerWidth <= 768;
  const folder = isMobile ? "/mp2" : "/mp1";
  const index = Math.floor(Math.random() * 4) + 1; // 随机 1 或 2
  videoSrc.value = `${folder}/1 (${index}).mp4`;

  // 初始化
  initStrings();
  animateStrings();
  startTitleAnimation();
  startRandomCutting();

  // 延迟开始打字
  setTimeout(() => {
    const line = pickRandomLine();
    startTypewriter(line);
  }, 600);

  // 8秒后自动进入主页
  setTimeout(() => {
    if (showIntro.value) {
      showIntro.value = false;
    }
  }, 5000);
});

onBeforeUnmount(() => {
  if (typingTimer) clearInterval(typingTimer);
  if (hintTimer) clearTimeout(hintTimer);
  if (titleTimer) clearInterval(titleTimer);
  if (stringTimer) clearInterval(stringTimer);
});
</script>

<style scoped lang="scss">
#app {
  position: relative;
  min-height: 100vh;
}

/* 淡入淡出动画 */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.8s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

/* 千咲风格开屏页 */
.intro-container {
  position: fixed;
  inset: 0;
  background: #0a0a14;
  z-index: 9999;
  display: flex;
  justify-content: center;
  align-items: center;
  overflow: hidden;
  cursor: pointer;

  /* 视频包装器 */
  .video-wrapper {
    position: absolute;
    inset: 0;
    overflow: hidden;

    .video-background {
      position: absolute;
      inset: 0;
      width: 100%;
      height: 100%;
      object-fit: cover;
      z-index: 1;
      filter: saturate(0.7) contrast(1.2) brightness(0.6) sepia(0.3)
        hue-rotate(-20deg);
    }

    .video-overlay {
      position: absolute;
      inset: 0;
      background: linear-gradient(
        45deg,
        rgba(198, 40, 40, 0.15) 0%,
        rgba(10, 10, 20, 0.6) 50%,
        rgba(142, 0, 0, 0.15) 100%
      );
      z-index: 2;
    }
  }

  /* 弦线覆盖层 */
  .string-overlay {
    position: absolute;
    inset: 0;
    z-index: 3;
    pointer-events: none;

    .string {
      position: absolute;
      width: 1px;
      background: linear-gradient(to bottom, transparent, #c62828, transparent);
      transform-origin: top center;
      animation: stringSway ease-in-out infinite alternate;

      /* 剪断效果 */
      &::before {
        content: "";
        position: absolute;
        top: var(--cut-height, 0);
        left: 50%;
        transform: translateX(-50%);
        width: 3px;
        height: 3px;
        background: #ffd700;
        border-radius: 50%;
        box-shadow: 0 0 8px #ffd700;
        opacity: calc(var(--cut-height, 0) / 100);
      }

      /* 剪断后的部分 */
      &::after {
        content: "";
        position: absolute;
        top: var(--cut-height, 0);
        left: 50%;
        transform: translateX(-50%);
        width: 1px;
        height: calc(100% - var(--cut-height, 0));
        background: linear-gradient(
          to bottom,
          rgba(198, 40, 40, 0.3),
          transparent
        );
      }
    }

    @keyframes stringSway {
      0% {
        transform: translateX(0) rotate(0.5deg);
      }
      100% {
        transform: translateX(5px) rotate(-0.5deg);
      }
    }
  }

  /* 剪断效果 */
  .cutting-effect {
    position: absolute;
    inset: 0;
    z-index: 4;
    pointer-events: none;
    opacity: var(--cut-opacity, 0);
    transition: opacity 0.3s ease;

    &::before {
      content: "";
      position: absolute;
      top: 0;
      left: 50%;
      width: 3px;
      height: 100%;
      background: linear-gradient(
        to bottom,
        transparent,
        rgba(255, 215, 0, 0.8),
        rgba(244, 67, 54, 0.9),
        rgba(255, 215, 0, 0.8),
        transparent
      );
      transform: translateX(-50%) rotate(var(--cut-angle, 0deg));
      filter: blur(1px);
      clip-path: polygon(
        0% 0%,
        100% 0%,
        100% calc(var(--cut-progress, 0) * 100%),
        0% calc(var(--cut-progress, 0) * 100%)
      );
    }

    &::after {
      content: "";
      position: absolute;
      top: calc(var(--cut-progress, 0) * 100% - 10px);
      left: 0;
      width: 100%;
      height: 20px;
      background: linear-gradient(
        90deg,
        transparent,
        rgba(244, 67, 54, 0.3),
        rgba(255, 215, 0, 0.4),
        rgba(244, 67, 54, 0.3),
        transparent
      );
      transform: rotate(var(--cut-angle, 0deg));
    }
  }
}

.intro-content {
  position: relative;
  z-index: 5;
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  padding: 20px;

  .intro-inner {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 40px;
    max-width: 800px;
    width: 100%;

    /* 剪刀动画 */
    .scissors-animation {
      position: relative;
      margin-bottom: 10px;

      .scissors-icon {
        font-size: 2.5rem;
        color: #f44336;
        animation: scissorsFloat 3s ease-in-out infinite;
        text-shadow: 0 0 15px rgba(244, 67, 54, 0.5);
      }

      .cutting-line {
        position: absolute;
        top: 50%;
        left: 50%;
        width: 100px;
        height: 1px;
        background: linear-gradient(90deg, transparent, #c62828, transparent);
        transform: translate(-50%, -50%);
        opacity: 0.5;
      }

      @keyframes scissorsFloat {
        0%,
        100% {
          transform: translateY(0) rotate(0deg);
        }
        50% {
          transform: translateY(-10px) rotate(10deg);
        }
      }
    }

    /* 主标题 */
    .main-title {
      font-size: 5.5rem;
      font-weight: 300;
      letter-spacing: 8px;
      text-align: center;
      margin: 10px 0;

      .title-char {
        display: inline-block;
        background: linear-gradient(135deg, #f5f5f5 30%, #f44336 70%);
        -webkit-background-clip: text;
        -webkit-text-fill-color: transparent;
        background-clip: text;
        transition: all 0.6s cubic-bezier(0.34, 1.56, 0.64, 1);

        &:first-child {
          margin-right: 20px;
        }
      }

      @media (max-width: 768px) {
        font-size: 3.5rem;
        letter-spacing: 6px;
      }

      @media (max-width: 480px) {
        font-size: 2.5rem;
        letter-spacing: 4px;
      }
    }

    /* 打字机容器 */
    .typewriter-container {
      text-align: center;
      margin: 20px 0;

      .typewriter-wrapper {
        background: rgba(26, 26, 26, 0.4);
        border-radius: 10px;
        padding: 20px 30px;
        border: 1px solid rgba(198, 40, 40, 0.2);
        backdrop-filter: blur(5px);
        margin-bottom: 15px;

        .typewriter-text {
          font-size: 1.5rem;
          font-weight: 400;
          line-height: 1.4;
          margin: 0;
          min-height: 1.5em;
          color: #f5f5f5;
          text-shadow: 0 2px 8px rgba(0, 0, 0, 0.5);

          .cursor {
            display: inline-block;
            width: 2px;
            height: 1.2em;
            margin-left: 5px;
            background: #f44336;
            animation: blink 1s infinite;
            vertical-align: middle;
          }

          @media (max-width: 768px) {
            font-size: 1.2rem;
          }

          @media (max-width: 480px) {
            font-size: 1rem;
          }
        }
      }

      .subtitle {
        font-size: 0.9rem;
        color: rgba(244, 67, 54, 0.8);
        letter-spacing: 3px;
        text-transform: uppercase;
        margin-top: 5px;
      }
    }

  
  }

  /* 交互提示 */
  .interaction-hint {
    position: absolute;
    bottom: 40px;
    left: 0;
    width: 100%;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 12px;
    opacity: 0;
    transition: opacity 0.5s ease;

    &.visible {
      opacity: 1;
    }

    .hint-line {
      width: 100px;
      height: 1px;
      background: linear-gradient(90deg, transparent, #c62828, transparent);
    }

    .hint-text {
      font-size: 0.9rem;
      color: rgba(245, 245, 245, 0.7);
      letter-spacing: 2px;
      text-transform: uppercase;
    }

    .hint-scissors {
      font-size: 1.2rem;
      color: #f44336;
      animation: hintBounce 1.5s infinite;
    }

    @keyframes hintBounce {
      0%,
      100% {
        transform: translateY(0);
      }
      50% {
        transform: translateY(-5px);
      }
    }
  }
}

/* 动画定义 */
@keyframes blink {
  0%,
  50% {
    opacity: 1;
  }
  51%,
  100% {
    opacity: 0;
  }
}

@keyframes fadeIn {
  to {
    opacity: 1;
  }
}

/* 响应式调整 */
@media (max-width: 768px) {
  .intro-inner {
    gap: 30px !important;

    .main-title {
      font-size: 3.5rem !important;
    }

    .typewriter-container .typewriter-wrapper {
      padding: 15px 20px !important;
    }

  
  }

  .interaction-hint {
    bottom: 30px !important;
  }
}

@media (max-width: 480px) {
  .intro-inner {
    gap: 25px !important;

    .main-title {
      font-size: 2.5rem !important;
      letter-spacing: 4px !important;
    }

    .scissors-animation .scissors-icon {
      font-size: 2rem !important;
    }

    .typewriter-container .typewriter-wrapper {
      padding: 12px 16px !important;
    }

  
  }

  .interaction-hint {
    bottom: 25px !important;

    .hint-text {
      font-size: 0.8rem !important;
    }
  }
}
</style>