<template>
  <section class="voice-gallery">
    <div class="bg-carousel" aria-hidden="true">
      <transition-group name="bg-fade" tag="div" class="bg-layer">
        <img
          v-for="(src, idx) in activeImages"
          :key="`bg-${idx}-${isMobile ? 'm' : 'd'}`"
          :src="src"
          :class="['bg-img', { active: idx === currentIndex }]"
          alt=""
        />
      </transition-group>
    </div>

    <div class="vg__wrap">
      <header class="vg__header">
        <div class="logo">
          <div class="title-group">
            <h1 class="title">千咲 · 语音馆</h1>
            <p class="subtitle">和千咲对话有概率解锁语音彩蛋哦</p>
          </div>
        </div>
      </header>

      <!-- 列表（已解锁放前，未解锁放后） -->
      <ul class="vg__list" role="list">
        <li
          v-for="id in allVoiceIds"
          :key="id"
          class="vg__item"
          :class="{
            unlocked: isUnlocked(id),
            locked: !isUnlocked(id),
            playing: id === currentId,
          }"
        >
          <div class="item__left">
            <div class="index">{{ String(id).padStart(3, "0") }}</div>
            <div class="info">
              <div class="name">语音 {{ String(id).padStart(3, "0") }}</div>
              <div class="note" v-if="isUnlocked(id)">已解锁</div>
              <div class="note note--locked" v-else>未解锁</div>
            </div>
          </div>

          <div class="item__right">
            <button
              class="btn btn--icon"
              :disabled="!isUnlocked(id)"
              @click="playEntry(id)"
              :title="
                isUnlocked(id)
                  ? currentId === id && isPlaying
                    ? '暂停'
                    : '播放'
                  : '尚未解锁'
              "
            >
              <span v-if="!isUnlocked(id)">🔒</span>
              <span v-else-if="currentId === id && isPlaying">❚❚</span>
              <span v-else>▶</span>
            </button>

            <a
              v-if="isUnlocked(id)"
              :href="voicePath(id)"
              :download="`audio_${id}.mp3`"
              title="下载"
            >
              <el-button type="primary" :icon="Download" circle />
            </a>
            <span v-else class="btn btn--hint" aria-hidden="true">—</span>
          </div>
        </li>
      </ul>
    </div>
  </section>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onBeforeUnmount, watch } from "vue";
import { Download } from "@element-plus/icons-vue";
/* ================== 配置 ================== */
const TOTAL_VOICES = 22; // 语音总数，按实际替换
const BG_INTERVAL_MS = 4500; // 背景切换间隔（毫秒）
const MOBILE_BREAKPOINT = 720; // 小于这个宽度视为移动端
/* ========================================= */

/* ========== 导入图片资源（Vite：eager） ========== */
/* 横图（用于 PC） */
const modules: Record<string, any> = import.meta.glob(
  "@/assets/images1/*.{jpg,png,jpeg,webp}",
  { eager: true }
);
const allSrcs: string[] = Object.values(modules).map(
  (m: any) => m.default || m
);

/* 竖图（用于移动端） */
const modules2: Record<string, any> = import.meta.glob(
  "@/assets/images2/*.{jpg,png,jpeg,webp}",
  { eager: true }
);
const allSrcs2: string[] = Object.values(modules2).map(
  (m: any) => m.default || m
);

/* 简单洗牌函数 */
function shuffle<T>(arr: T[]) {
  const a = arr.slice();
  for (let i = a.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [a[i], a[j]] = [a[j], a[i]];
  }
  return a;
}

/* 随机取 5 张（若不足 5 张则全部） */
const randomFive = ref<string[]>(
  shuffle(allSrcs).slice(0, Math.min(5, allSrcs.length))
);
const randomFive2 = ref<string[]>(
  shuffle(allSrcs2).slice(0, Math.min(5, allSrcs2.length))
);

/* 轮播索引（共享，但 index 会根据 activeImages 长度做取模） */
const currentIndex = ref(0);
let imgTimer: number | null = null;

/* ========== 设备判断（响应式） ========== */
const isMobile = ref(window.innerWidth < MOBILE_BREAKPOINT);
function handleResize() {
  const nowMobile = window.innerWidth < MOBILE_BREAKPOINT;
  if (nowMobile !== isMobile.value) {
    isMobile.value = nowMobile;
    // 切换图片组时重置索引以避免越界
    currentIndex.value = 0;
  }
}

/* activeImages 根据 isMobile 返回对应数组 */
const activeImages = computed(() =>
  isMobile.value ? randomFive2.value : randomFive.value
);
/* ========== 语音列表与播放逻辑 ========== */

/* 已解锁集合（由 localStorage.triggeredVoices 提供，数组） */
const unlockedSet = ref<Set<number>>(new Set<number>());

function loadUnlocked() {
  try {
    const raw = localStorage.getItem("triggeredVoices") || "[]";
    const arr = JSON.parse(raw);
    const s = new Set<number>();
    if (Array.isArray(arr)) {
      arr.forEach((v: any) => {
        const n = Number(v);
        if (!Number.isNaN(n)) s.add(n);
      });
    }
    unlockedSet.value = s;
  } catch (e) {
    console.warn("读取 triggeredVoices 失败", e);
    unlockedSet.value = new Set<number>();
  }
}

/* 生成所有 id，并保持已解锁在前、未解锁在后 */
const allIds = Array.from({ length: TOTAL_VOICES }, (_, i) => i);
const allVoiceIds = computed(() => {
  const unlocked = Array.from(unlockedSet.value)
    .filter((n) => allIds.includes(n))
    .sort((a, b) => a - b);
  const locked = allIds.filter((id) => !unlockedSet.value.has(id));
  return [...unlocked, ...locked];
});

/* audio 单例 */
let currentAudio: HTMLAudioElement | null = null;
const currentId = ref<number | null>(null);
const isPlaying = ref(false);
const currentTime = ref(0);
const currentDuration = ref(0);

function createAudio(src?: string) {
  destroyAudio();
  currentAudio = new Audio();
  currentAudio.preload = "auto";
  if (src) currentAudio.src = src;
  currentAudio.addEventListener("timeupdate", onTimeUpdate);
  currentAudio.addEventListener("loadedmetadata", onLoadedMeta);
  currentAudio.addEventListener("ended", onEnded);
  currentAudio.addEventListener("error", onAudioError);
}
function destroyAudio() {
  if (!currentAudio) return;
  try {
    currentAudio.pause();
    currentAudio.removeEventListener("timeupdate", onTimeUpdate);
    currentAudio.removeEventListener("loadedmetadata", onLoadedMeta);
    currentAudio.removeEventListener("ended", onEnded);
    currentAudio.removeEventListener("error", onAudioError);
    currentAudio.src = "";
  } catch (e) {}
  currentAudio = null;
}
function onTimeUpdate() {
  if (currentAudio) currentTime.value = currentAudio.currentTime || 0;
}
function onLoadedMeta() {
  if (currentAudio) currentDuration.value = currentAudio.duration || 0;
}
function onEnded() {
  isPlaying.value = false; /* 不自动下一条 */
}
function onAudioError(e?: any) {
  console.error("audio error", e);
  isPlaying.value = false;
}

function voicePath(id: number) {
  return `/voice/audio (${id}).mp3`;
}
function isUnlocked(id: number) {
  return unlockedSet.value.has(id);
}

async function playEntry(id: number) {
  if (!isUnlocked(id)) return;
  // 同一条 -> 切换暂停/恢复
  if (currentId.value === id && isPlaying.value) {
    currentAudio?.pause();
    isPlaying.value = false;
    return;
  }
  if (currentId.value === id && !isPlaying.value && currentAudio) {
    try {
      await currentAudio.play();
      isPlaying.value = true;
    } catch (e) {
      console.warn(e);
    }
    return;
  }

  // 新条目
  currentId.value = id;
  createAudio(voicePath(id));
  try {
    await currentAudio!.play();
    isPlaying.value = true;
  } catch (e) {
    console.warn("播放被阻止或失败", e);
    isPlaying.value = false;
  }
}

/* ========== 背景轮播控制 ========== */
function startBgTimer() {
  stopBgTimer();
  imgTimer = window.setInterval(() => {
    const len = Math.max(1, activeImages.value.length);
    // 保证在当前 activeImages 长度范围内循环
    currentIndex.value = (currentIndex.value + 1) % len;
  }, BG_INTERVAL_MS);
}
function stopBgTimer() {
  if (imgTimer !== null) {
    clearInterval(imgTimer);
    imgTimer = null;
  }
}

/* 监听 storage 事件（跨 tab 更新） */
function onStorage(e: StorageEvent) {
  if (e.key === "triggeredVoices") loadUnlocked();
}

/* 生命周期 */
onMounted(() => {
  loadUnlocked();
  window.addEventListener("storage", onStorage);
  window.addEventListener("resize", handleResize);

  // 如果数组为空（没有图片），也避免报错：确保至少有一个占位空数组
  if (!randomFive.value.length && allSrcs.length)
    randomFive.value = shuffle(allSrcs).slice(0, Math.min(5, allSrcs.length));
  if (!randomFive2.value.length && allSrcs2.length)
    randomFive2.value = shuffle(allSrcs2).slice(
      0,
      Math.min(5, allSrcs2.length)
    );

  // 启动轮播
  startBgTimer();
});

onBeforeUnmount(() => {
  window.removeEventListener("storage", onStorage);
  window.removeEventListener("resize", handleResize);
  stopBgTimer();
  destroyAudio();
});

/* 监听 activeImages 长度变化（切换设备时重置 index 并保持循环） */
watch(activeImages, (nv) => {
  currentIndex.value = 0;
});
</script>
<style lang="scss" scoped>
/* 千咲主题配色变量 */

/* 透明度变量 */
$accent-transparent-10: rgba(198, 40, 40, 0.1);
$accent-transparent-20: rgba(198, 40, 40, 0.2);
$accent-transparent-30: rgba(198, 40, 40, 0.3);
$accent-transparent-50: rgba(198, 40, 40, 0.5);
$accent-transparent-70: rgba(198, 40, 40, 0.7);
$white-transparent-10: rgba(245, 245, 245, 0.1);
$white-transparent-20: rgba(245, 245, 245, 0.2);
$white-transparent-30: rgba(245, 245, 245, 0.3);
$white-transparent-50: rgba(245, 245, 245, 0.5);
$white-transparent-70: rgba(245, 245, 245, 0.7);
$white-transparent-90: rgba(245, 245, 245, 0.9);
$black-transparent-30: rgba(10, 10, 10, 0.3);
$black-transparent-50: rgba(10, 10, 10, 0.5);
$black-transparent-70: rgba(10, 10, 10, 0.7);
$gold-transparent-20: rgba(255, 215, 0, 0.2);
$gold-transparent-30: rgba(255, 215, 0, 0.3);

/* ====== 千咲风格 · 语音馆 ====== */
.voice-gallery {
  position: relative;
  min-height: 100vh;
  font-family: "Noto Sans SC", "Segoe UI", "Helvetica Neue", Arial, sans-serif;
  color: var(--white);
  overflow-x: hidden;
  padding: 20px;
  padding-top: 80px;
  background: var(--deep-bg);
  --deep-bg: linear-gradient(145deg, #0a0a14 0%, #1a0a14 35%, #0a0a1a 100%);
  --accent: #c62828; /* 主色：深红（制服/剪刀） */
  --accent-light: #f44336; /* 亮红（高光/特效） */
  --accent-dark: #8e0000; /* 暗红（阴影） */
  --black: #0a0a0a; /* 纯黑（头发/底色） */
  --gray-dark: #1a1a1a; /* 深灰（背景） */
  --gray-light: #2a2a2a; /* 浅灰（边框） */
  --white: #f5f5f5; /* 珍珠白（衬衫/文字） */
  --gold: #ffd700; /* 金色（装饰/星炬） */
  --shadow-core: rgba(198, 40, 40, 0.15);
  --shadow-deep: rgba(10, 10, 20, 0.8);

  /* 血色丝线背景纹理，象征千咲的剪刀与弦线 */
  &::before {
    content: "";
    position: absolute;
    inset: 0;
    z-index: 0;
    pointer-events: none;
    background-image: linear-gradient(
        45deg,
        transparent 49%,
        $accent-transparent-10 50%,
        transparent 51%
      ),
      linear-gradient(
        -45deg,
        transparent 49%,
        $accent-transparent-10 50%,
        transparent 51%
      ),
      radial-gradient(
        circle at 20% 30%,
        $accent-transparent-20 0%,
        transparent 25%
      ),
      radial-gradient(
        circle at 80% 70%,
        $accent-transparent-20 0%,
        transparent 25%
      );
    background-size: 60px 60px, 60px 60px, 300px 300px, 300px 300px;
    opacity: 0.3;
  }

  /* 动态血色光晕 */
  &::after {
    content: "";
    position: fixed;
    width: 500px;
    height: 500px;
    top: -100px;
    right: -100px;
    background: radial-gradient(
      circle,
      $accent-transparent-20 0%,
      transparent 70%
    );
    border-radius: 50%;
    pointer-events: none;
    z-index: 0;
    animation: pulse 10s ease-in-out infinite alternate;
  }

  /* 背景轮播层 */
  .bg-carousel {
    position: fixed;
    inset: 0;
    z-index: 1;
    pointer-events: none;

    .bg-layer {
      position: absolute;
      inset: 0;
      overflow: hidden;


      .bg-img {
        position: absolute;
        left: 0;
        top: 0;
        width: 100%;
        height: 100%;
        object-fit: cover;
        display: block;
        opacity: 0;
        transform: scale(1.02);
        transition: opacity 1.2s ease, transform 1.2s ease;
        pointer-events: none;
        filter: brightness(0.4) contrast(1.1) saturate(0.8) hue-rotate(0deg);
      }

      .bg-img.active {
        opacity: 1;
        transform: scale(1);
        filter: brightness(0.5) contrast(1.2) saturate(0.9);
      }
    }
  }

  /* 前景容器（血色档案匣） */
  .vg__wrap {
    position: relative;
    z-index: 10;
    max-width: 900px;
    margin: 0 auto;
    border-radius: 16px;
    padding: 24px;
    box-shadow: 0 20px 60px var(--shadow-deep),
      inset 0 1px 0 $white-transparent-10;
    background: linear-gradient(
      145deg,
      rgba(26, 26, 26, 0.45) 0%,
      rgba(10, 10, 10, 0.2) 100%
    );
    border: 1px solid $accent-transparent-30;
    backdrop-filter: blur(2px) saturate(1.5);
    overflow: hidden;

    /* 右上角剪刀剪影装饰 */
    &::before {
      content: "✂";
      position: absolute;
      top: 20px;
      right: 20px;
      font-size: 24px;
      color: $accent-transparent-20;
      transform: rotate(45deg);
      z-index: 0;
    }
  }

  /* 头部 */
  .vg__header {
    display: flex;
    gap: 16px;
    align-items: center;
    margin-bottom: 32px;
    position: relative;
    z-index: 1;

    .logo {
      display: flex;
      gap: 16px;
      align-items: center;
      width: 100%;
      position: relative;

      /* 血色呼吸动画 */
      @keyframes heart-beat {
        0% {
          transform: scale(1);
          opacity: 0.8;
          filter: drop-shadow(0 0 10px $accent-transparent-30);
        }
        50% {
          transform: scale(1.05);
          opacity: 1;
          filter: drop-shadow(0 0 20px $accent-transparent-50);
        }
        100% {
          transform: scale(1);
          opacity: 0.8;
          filter: drop-shadow(0 0 10px $accent-transparent-30);
        }
      }

      @keyframes blood-drop {
        0% {
          opacity: 0;
          transform: translateY(0) scale(0.8);
        }
        50% {
          opacity: 1;
          transform: translateY(20px) scale(1);
        }
        100% {
          opacity: 0;
          transform: translateY(40px) scale(1.2);
        }
      }

      .title-group {
        flex: 1;
        display: flex;
        flex-direction: column;
        position: relative;
        z-index: 1;

        .title {
          margin: 0;
          font-size: 2rem;
          font-weight: 800;
          background: linear-gradient(
            90deg,
            var(--accent-light) 0%,
            var(--accent) 50%,
            var(--accent-dark) 100%
          );
          -webkit-background-clip: text;
          background-clip: text;
          color: transparent;
          -webkit-text-fill-color: transparent;
          text-shadow: 0 4px 20px $accent-transparent-30;
          letter-spacing: 1px;
          padding-bottom: 8px;
          border-bottom: 2px solid $accent-transparent-30;
          position: relative;

          &::after {
            content: "Chisaki";
            position: absolute;
            bottom: -6px;
            left: 0;
            font-size: 0.7rem;
            font-weight: 300;
            color: $white-transparent-50;
            letter-spacing: 3px;
          }
        }

        .subtitle {
          margin: 12px 0 0;
          color: $white-transparent-70;
          font-size: 1rem;
          line-height: 1.5;
          max-width: 600px;
          position: relative;
          padding-left: 20px;

          &::before {
            content: "⋆";
            position: absolute;
            left: 0;
            top: 50%;
            transform: translateY(-50%);
            color: var(--gold);
            font-size: 1.2rem;
          }
        }
      }
    }
  }

  /* 列表区域 */
  .vg__list {
    display: grid;
    gap: 14px;
    margin: 0;
    padding: 0;
    list-style: none;
    max-height: calc(100vh - 220px);
    overflow-y: auto;
    padding-right: 8px;
    position: relative;
    z-index: 1;

    &::-webkit-scrollbar {
      width: 8px;
    }

    &::-webkit-scrollbar-thumb {
      background: linear-gradient(
        180deg,
        $accent-transparent-50 0%,
        $accent-transparent-30 100%
      );
      border-radius: 4px;
      border: 2px solid transparent;
      background-clip: padding-box;
    }

    &::-webkit-scrollbar-track {
      background: $black-transparent-30;
      border-radius: 4px;
    }
  }

  /* 语音卡片项 */
  .vg__item {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 16px;
    padding: 16px 20px;
    border-radius: 12px;
    background: linear-gradient(
      145deg,
      rgba(42, 42, 42, 0.3) 0%,
      rgba(26, 26, 26, 0.4) 100%
    );
    border: 1px solid $accent-transparent-20;
    backdrop-filter: blur(1px);
    transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
    position: relative;
    overflow: hidden;
    min-height: 80px;

    /* 播放状态 */
    &.playing {
      transform: translateY(-3px) scale(1.01);
      box-shadow: 0 15px 40px $accent-transparent-30,
        inset 0 0 0 1px $accent-transparent-50;
      border-color: $accent-transparent-50;
      background: linear-gradient(
        145deg,
        rgba(66, 26, 26, 0.8) 0%,
        rgba(42, 16, 16, 0.9) 100%
      );

      /* 播放脉冲效果 */
      &::before {
        content: "";
        position: absolute;
        inset: 0;
        background: radial-gradient(
          circle at 80% 50%,
          $accent-transparent-20,
          transparent 40%
        );
        animation: playing-pulse 1.5s ease-in-out infinite;
        z-index: 0;
        pointer-events: none;
      }

      .index {
        background: linear-gradient(
          145deg,
          var(--accent) 0%,
          var(--accent-dark) 100%
        ) !important;
        color: var(--white) !important;
        box-shadow: 0 6px 20px $accent-transparent-50 !important;
      }
    }

    /* 已解锁状态 */
    &.unlocked {
      &:hover {
        transform: translateY(-2px);
        box-shadow: 0 12px 32px $accent-transparent-20,
          inset 0 0 0 1px $accent-transparent-30;
        border-color: $accent-transparent-30;
        cursor: pointer;
      }

      .index {
        background: linear-gradient(
          145deg,
          rgba(42, 42, 42, 0.9) 0%,
          rgba(26, 26, 26, 1) 100%
        );
        color: var(--accent-light);
      }
    }

    /* 锁定状态 */
    &.locked {
      opacity: 0.5;
      filter: grayscale(0.3) brightness(0.6);

      .index {
        background: linear-gradient(
          145deg,
          rgba(26, 26, 26, 0.9) 0%,
          rgba(10, 10, 10, 1) 100%
        );
        color: $white-transparent-30;
      }

      .note--locked {
        color: $white-transparent-30 !important;
        font-style: italic;
      }
    }

    /* 左侧内容 */
    .item__left {
      display: flex;
      gap: 16px;
      align-items: center;
      flex: 1;
      min-width: 0;
      z-index: 1;

      .index {
        flex-shrink: 0;
        width: 60px;
        height: 60px;
        border-radius: 10px;
        display: flex;
        align-items: center;
        justify-content: center;
        font-weight: 700;
        font-size: 1.2rem;
        box-shadow: 0 6px 16px var(--shadow-deep),
          inset 0 1px 0 $white-transparent-10;
        border: 1px solid $accent-transparent-20;
        transition: all 0.3s ease;
      }

      .info {
        flex: 1;
        min-width: 0;

        .name {
          color: var(--white);
          font-weight: 600;
          font-size: 1.1rem;
          margin-bottom: 4px;
          letter-spacing: 0.5px;
          white-space: nowrap;
          overflow: hidden;
          text-overflow: ellipsis;
        }

        .note {
          color: var(--accent-light);
          font-size: 0.85rem;
          font-weight: 500;
          opacity: 0.9;
        }

        .note--locked {
          color: $white-transparent-50 !important;
        }
      }
    }

    /* 右侧操作按钮 */
    .item__right {
      display: flex;
      gap: 12px;
      align-items: center;
      z-index: 1;
      flex-shrink: 0;

      .btn {
        &--icon {
          width: 56px;
          height: 56px;
          border-radius: 12px;
          border: none;
          display: flex;
          align-items: center;
          justify-content: center;
          background: linear-gradient(
            145deg,
            var(--accent) 0%,
            var(--accent-dark) 100%
          );
          color: var(--white);
          font-size: 1.4rem;
          font-weight: 700;
          cursor: pointer;
          box-shadow: 0 8px 24px $accent-transparent-30,
            inset 0 1px 0 $white-transparent-20;
          transition: all 0.3s ease;
          border: 1px solid $accent-transparent-50;
          position: relative;
          overflow: hidden;

          &:hover:not(:disabled) {
            transform: translateY(-2px) scale(1.05);
            box-shadow: 0 12px 32px $accent-transparent-50,
              inset 0 1px 0 $white-transparent-30;
            background: linear-gradient(
              145deg,
              var(--accent-light) 0%,
              var(--accent) 100%
            );
          }

          &:active:not(:disabled) {
            transform: translateY(0) scale(0.98);
          }

          &:disabled {
            background: $black-transparent-50;
            color: $white-transparent-30;
            cursor: not-allowed;
            box-shadow: none;
            border-color: $white-transparent-10;
          }

          span {
            position: relative;
            z-index: 1;
          }
        }

        &--hint {
          color: $white-transparent-30;
          font-size: 1.2rem;
          opacity: 0.5;
        }
      }

      a {
        .el-button {
          width: 48px;
          height: 48px;
          background: linear-gradient(
            145deg,
            rgba(42, 42, 42, 0.8) 0%,
            rgba(26, 26, 26, 0.9) 100%
          );
          border: 1px solid $accent-transparent-30;
          color: var(--accent-light);
          transition: all 0.3s ease;
          box-shadow: 0 6px 20px var(--shadow-deep);

          &:hover {
            background: linear-gradient(
              145deg,
              $accent-transparent-20 0%,
              $accent-transparent-10 100%
            );
            border-color: $accent-transparent-50;
            transform: translateY(-2px);
            box-shadow: 0 10px 28px $accent-transparent-30;
          }

          :deep(svg) {
            width: 20px;
            height: 20px;
            color: var(--accent-light);
          }
        }
      }
    }
  }

  /* 背景过渡动画 */
  .bg-fade-enter-active,
  .bg-fade-leave-active {
    transition: opacity 1.2s ease, transform 1.2s ease;
  }

  .bg-fade-enter-from,
  .bg-fade-leave-to {
    opacity: 0;
    transform: scale(1.05);
  }

  .bg-fade-enter-to,
  .bg-fade-leave-from {
    opacity: 1;
    transform: scale(1);
  }

  /* ====== 动画定义 ====== */
  @keyframes pulse {
    0%,
    100% {
      opacity: 0.3;
      transform: scale(1);
    }
    50% {
      opacity: 0.6;
      transform: scale(1.1);
    }
  }

  @keyframes playing-pulse {
    0%,
    100% {
      opacity: 0.1;
      transform: scale(1);
    }
    50% {
      opacity: 0.3;
      transform: scale(1.1);
    }
  }

  @keyframes heart-beat {
    0%,
    100% {
      transform: scale(1);
      opacity: 0.8;
    }
    50% {
      transform: scale(1.05);
      opacity: 1;
    }
  }

  @keyframes blood-drop {
    0% {
      opacity: 0;
      transform: translateY(-20px) scale(0.8);
    }
    50% {
      opacity: 1;
      transform: translateY(0) scale(1);
    }
    100% {
      opacity: 0;
      transform: translateY(20px) scale(1.2);
    }
  }

  /* ====== 响应式设计 ====== */
  @media (max-width: 768px) {
    padding: 12px;
    padding-top: 70px;

    .vg__wrap {
      padding: 20px;
      border-radius: 12px;
    }

    .vg__header {
      margin-bottom: 24px;

      .logo {
        .title-group {
          .title {
            font-size: 1.5rem;
          }

          .subtitle {
            font-size: 0.9rem;
            padding-left: 16px;
          }
        }
      }
    }

    .vg__list {
      max-height: calc(100vh - 180px);
      gap: 12px;
    }

    .vg__item {
      padding: 12px 16px;
      gap: 12px;

      .item__left {
        gap: 12px;

        .index {
          width: 50px;
          height: 50px;
          font-size: 1rem;
        }

        .info {
          .name {
            font-size: 1rem;
          }
        }
      }

      .item__right {
        gap: 8px;

        .btn--icon {
          width: 48px;
          height: 48px;
          font-size: 1.2rem;
        }

        a .el-button {
          width: 44px;
          height: 44px;
        }
      }
    }
  }

  @media (max-width: 480px) {
    .vg__item {
      flex-direction: column;
      align-items: stretch;
      gap: 12px;

      .item__left {
        justify-content: space-between;
      }

      .item__right {
        justify-content: flex-end;
        width: 100%;
        padding-top: 8px;
        border-top: 1px solid $white-transparent-10;
      }
    }
  }
}
</style>