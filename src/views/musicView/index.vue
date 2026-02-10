<template>
  <section
    class="cantarella-player"
    @keydown.space.prevent="onSpace"
    tabindex="0"
    ref="rootEl"
    aria-label="千咲 音乐播放器"
  >
    <div class="stage">
      <!-- 左侧：封面与控制 -->
      <div class="left" role="region" aria-label="播放器控制区">
        <div class="cover" :style="coverStyle">
          <!-- video 作为纯装饰性背景，屏读器隐藏 -->
          <video
            v-if="videoSrc"
            class="video-background"
            :src="videoSrc"
            autoplay
            muted
            loop
            playsinline
            aria-hidden="true"
            tabindex="-1"
            :class="videoClass"
          ></video>

          <!-- 加载遮罩 -->
          <div v-if="loadingAudio" class="loading-overlay" aria-hidden="true">
            <div class="spinner" />
            <div class="loading-text">加载中…</div>
          </div>
        </div>

        <div class="controls">
          <div class="title" :title="current?.title || '未选择曲目'">
            {{ current?.title || "未选择曲目" }}
          </div>

          <div class="meta">
            <span class="time">{{ formatTime(currentTime) }}</span>
            <span class="divider">/</span>
            <span class="time">{{ formatTime(duration) }}</span>
          </div>

          <!-- 进度条（Pointer 支持）-->
          <div
            class="progress-wrap"
            ref="progressWrap"
            @click="seekByClick"
            @pointerdown.prevent="onPointerDownProgress"
            role="slider"
            :aria-valuemin="0"
            :aria-valuemax="duration"
            :aria-valuenow="currentTime"
            aria-label="进度条"
          >
            <div class="progress-bar">
              <div
                class="progress"
                :style="{ width: progressPercent + '%' }"
              ></div>
            </div>
            <div
              class="progress-handle"
              :style="{ left: progressPercent + '%' }"
              aria-hidden="true"
            ></div>
          </div>

          <!-- 控件行 -->
          <div class="btns">
            <button class="icon" @click="prev" aria-label="上一首">⟵</button>

            <button
              class="play"
              @click="togglePlay"
              :aria-pressed="playing"
              :aria-label="playing ? '暂停' : '播放'"
            >
              <span v-if="!playing">▶</span>
              <span v-else>▌▌</span>
            </button>

            <button class="icon" @click="next" aria-label="下一首">⟶</button>

            <div class="modes" role="group" aria-label="播放模式">
              <button
                :class="{ active: shuffle }"
                @click="toggleShuffle"
                title="随机播放"
              >
                🔀
              </button>
              <button
                :class="{ active: repeatMode !== 'off' }"
                @click="toggleRepeat"
                title="循环模式"
              >
                🔁
              </button>
            </div>

            <div class="volume" aria-label="音量控制">
              <input
                type="range"
                min="0"
                max="1"
                step="0.01"
                v-model.number="volume"
                aria-label="音量"
              />
            </div>
          </div>

          <div v-if="errorMessage" class="error-msg" role="status">
            {{ errorMessage }}
          </div>
        </div>
      </div>

      <!-- 右侧：播放列表 -->
      <div
        class="right"
        :class="{ collapsed: !playlistOpen && isMobile }"
        role="region"
        aria-label="播放列表"
      >
        <div class="playlist-header">
          <div class="left-head">
            <h3>播放列表</h3>
            <!-- 文本展开/收起按钮（替换文件夹图标） -->
            <button
              class="toggle-list-text"
              @click="togglePlaylist"
              :title="playlistOpen ? '收起播放列表' : '展开播放列表'"
            >
              {{ playlistOpen ? "收起" : "展开" }}
            </button>
            <div class="api-hint">
              {{ loading ? "加载中…" : list.length ? "" : "目录为空" }}
            </div>
          </div>

          <div class="search-wrap">
            <input
              v-model="searchTerm"
              @input="onSearchInput"
              placeholder="搜索曲名..."
              aria-label="搜索曲目"
            />
            <button
              v-if="searchTerm"
              class="clear"
              @click="clearSearch"
              aria-label="清除搜索"
            >
              ✕
            </button>
          </div>
        </div>

        <div class="list-area">
          <div v-if="loading" class="list-loading">
            <div class="small-spinner" />
            加载目录...
          </div>

          <ul class="playlist" role="list">
            <li
              v-for="(item, idx) in filteredList"
              :key="item.name || idx"
              :class="{ active: idx === index }"
              @click="selectTrack(idx)"
              tabindex="0"
              @keyup.enter="selectTrack(idx)"
              role="listitem"
              :aria-current="idx === index ? 'true' : 'false'"
            >
              <div class="left-col">
                <div class="dot" aria-hidden="true"></div>
                <!-- 尽量显示完整名称：允许换行，窄屏时限制为两行 -->
                <div class="title" :title="item.title">{{ item.title }}</div>
              </div>
              <div class="right-col">
                <div class="len">
                  {{ item.duration ? formatTime(item.duration) : "--:--" }}
                </div>
              </div>
            </li>
          </ul>
        </div>
      </div>
    </div>

    <!-- audio 元素 -->
    <audio
      ref="audioRef"
      @timeupdate="onTimeUpdate"
      @ended="onEnded"
      @loadedmetadata="onLoadedMetadata"
      @error="onAudioError"
      preload="metadata"
    ></audio>
  </section>
</template>

<script setup lang="ts">
import {
  ref,
  computed,
  onMounted,
  onBeforeUnmount,
  watch,
  nextTick,
} from "vue";
import { getMusicList, getMusicUrl } from "@/api/modules/music"; // 请确认路径

type MusicItem = {
  name: string;
  title: string;
  url?: string;
  duration?: number | null;
};

const list = ref<MusicItem[]>([]);
const loading = ref(false);
const index = ref<number>(-1);
const playing = ref(false);
const audioRef = ref<HTMLAudioElement | null>(null);
const currentTime = ref<number>(0);
const duration = ref<number>(0);
const volume = ref<number>(
  Number(localStorage.getItem("cantarella_volume") ?? 0.8)
);
const shuffle = ref<boolean>(false);
const repeatMode = ref<"off" | "one" | "all">("off");

const rootEl = ref<HTMLElement | null>(null);
const progressWrap = ref<HTMLElement | null>(null);
const dragging = ref(false);
const playlistOpen = ref(true);
const errorMessage = ref<string | null>(null);
const loadingAudio = ref(false);

// 根据窗口宽度判断移动端
const isMobile = ref<boolean>(window.innerWidth <= 920);
window.addEventListener("resize", () => {
  isMobile.value = window.innerWidth <= 920;
});

// 随机选择视频源（mp1/mp2 里）
const videoSrc = ref("");
const videoClass = ref(""); // "landscape" or "portrait"

// 搜索相关
const searchTerm = ref("");
let searchTimer: any = null;
const searchDebounceMs = 240;

// 当前曲目与进度计算
const current = computed(() =>
  index.value >= 0 && list.value[index.value] ? list.value[index.value] : null
);
const progressPercent = computed(() =>
  duration.value
    ? Math.min(100, Math.max(0, (currentTime.value / duration.value) * 100))
    : 0
);

// 封面渐变（千咲风格）
const coverStyle = computed(() => {
  const t = current.value?.title || "cantarella";
  let hash = 0;
  for (let i = 0; i < t.length; i++)
    hash = (hash << 5) - hash + t.charCodeAt(i);
  const r1 = (Math.abs(hash) % 120) + 40;
  const r2 = (Math.abs(hash * 3) % 120) + 40;
  return {
    background: `radial-gradient(circle at 28% 28%, rgba(95,224,255,0.06), transparent 12%), linear-gradient(135deg, rgba(${r1},${r2},255,0.08), rgba(111,92,230,0.08))`,
  };
});

// 过滤后的列表（基于搜索）
const filteredList = computed(() => {
  const term = (searchTerm.value || "").trim().toLowerCase();
  if (!term) return list.value;
  return list.value.filter((i) => (i.title || "").toLowerCase().includes(term));
});

// 获取列表
async function fetchList() {
  loading.value = true;
  try {
    const res = await getMusicList();
    const items =
      res?.ok && Array.isArray(res.list)
        ? res.list
        : Array.isArray(res)
        ? res
        : res?.list ?? [];
    list.value = items.map((it: any) => ({
      name: it.name,
      title: it.title ?? (it.name ? it.name.replace(/\.mp3$/i, "") : "未知"),
      url: getMusicUrl(it.name),
      duration: null,
    }));
    // // 自动选中第一首不自动播放
    // if (list.value.length > 0) {
    //    index.value = 0;
    //    await loadCurrent(false);
    // }
  } catch (e) {
    console.error("获取音乐列表失败", e);
    list.value = [];
    errorMessage.value = "加载目录失败";
  } finally {
    loading.value = false;
  }
}

// 设置并预检音源
async function safeSetSrc(url: string) {
  const a = audioRef.value!;
  errorMessage.value = null;
  loadingAudio.value = true;
  try {
    try {
      const head = await fetch(url, { method: "HEAD" });
      if (!head.ok) throw new Error(`资源响应 ${head.status}`);
      const ct = head.headers.get("content-type") || "";
      if (!ct.includes("audio")) {
        console.warn("content-type 不是 audio:", ct);
      }
    } catch (e) {
      // 忽略 HEAD 失败
    }
    a.src = url;
    a.load();
  } catch (err) {
    console.error("设置音源失败", err);
    errorMessage.value = "无法加载音频资源";
    throw err;
  } finally {
    // loadingAudio 会在 loadedmetadata 或 error 中被关闭
  }
}

async function loadCurrent(doPlay = false) {
  const a = audioRef.value;
  const curr = current.value;
  if (!a || !curr) return;
  a.pause();
  duration.value = 0;
  currentTime.value = 0;
  try {
    await safeSetSrc(curr.url || getMusicUrl(curr.name));
    if (doPlay) await play();
  } catch {
    playing.value = false;
    loadingAudio.value = false;
  }
}

async function play() {
  const a = audioRef.value;
  if (!a) return;
  try {
    await a.play();
    playing.value = true;
    errorMessage.value = null;
  } catch (e: any) {
    console.warn("播放失败", e);
    playing.value = false;
    errorMessage.value = "播放被浏览器阻止或资源不可用";
  }
}
function pause() {
  audioRef.value?.pause();
  playing.value = false;
}
function togglePlay() {
  if (!audioRef.value) return;
  if (playing.value) pause();
  else play();
}

function selectTrack(i: number) {
  if (i < 0 || i >= list.value.length) return;
  index.value = i;
  loadCurrent(true);
}

// 音频事件
function onTimeUpdate(e: Event) {
  const t = e.target as HTMLAudioElement;
  currentTime.value = t.currentTime || 0;
}
function onLoadedMetadata(e: Event) {
  const t = e.target as HTMLAudioElement;
  duration.value = isFinite(t.duration) ? t.duration : 0;
  if (current.value && !current.value.duration)
    current.value.duration = duration.value;
  loadingAudio.value = false;
}
function onEnded() {
  loadingAudio.value = false;
  if (repeatMode.value === "one") {
    if (audioRef.value) {
      audioRef.value.currentTime = 0;
      play();
    }
    return;
  }
  if (shuffle.value) {
    playRandom();
    return;
  }
  if (index.value < list.value.length - 1) selectTrack(index.value + 1);
  else {
    if (repeatMode.value === "all") selectTrack(0);
    else playing.value = false;
  }
}
function onAudioError(e: Event) {
  const a = audioRef.value;
  console.error("audio error", a?.error);
  errorMessage.value = "音频播放出错";
  playing.value = false;
  loadingAudio.value = false;
}

// 上一/下一/随机
function next() {
  if (!list.value.length) return;
  if (shuffle.value) {
    playRandom();
    return;
  }
  if (index.value < list.value.length - 1) selectTrack(index.value + 1);
  else if (repeatMode.value === "all") selectTrack(0);
}
function prev() {
  if (!audioRef.value) return;
  if (audioRef.value.currentTime > 4) {
    audioRef.value.currentTime = 0;
    return;
  }
  if (index.value > 0) selectTrack(index.value - 1);
  else if (repeatMode.value === "all") selectTrack(list.value.length - 1);
}
function playRandom() {
  if (!list.value.length) return;
  if (list.value.length === 1) {
    selectTrack(0);
    return;
  }
  let i = index.value;
  while (i === index.value) i = Math.floor(Math.random() * list.value.length);
  selectTrack(i);
}

// 进度条：点击 & 拖拽
function seekByClick(e: MouseEvent | TouchEvent) {
  if (!progressWrap.value || !duration.value || !audioRef.value) return;
  const rect = progressWrap.value.getBoundingClientRect();
  const clientX =
    (e as MouseEvent).clientX ?? (e as TouchEvent).touches?.[0]?.clientX;
  if (clientX == null) return;
  const x = Math.min(Math.max(0, clientX - rect.left), rect.width);
  const ratio = x / rect.width;
  audioRef.value.currentTime = ratio * duration.value;
  currentTime.value = audioRef.value.currentTime;
}
function onPointerDownProgress(e: PointerEvent) {
  if (!progressWrap.value || !audioRef.value || !duration.value) return;
  dragging.value = true;
  (e.target as Element).setPointerCapture?.(e.pointerId);
  window.addEventListener("pointermove", onPointerMoveProgress);
  window.addEventListener("pointerup", onPointerUpProgress);
  handlePointer(e);
}
function onPointerMoveProgress(e: PointerEvent) {
  handlePointer(e);
}
function onPointerUpProgress(e: PointerEvent) {
  dragging.value = false;
  window.removeEventListener("pointermove", onPointerMoveProgress);
  window.removeEventListener("pointerup", onPointerUpProgress);
}
function handlePointer(e: PointerEvent) {
  if (!progressWrap.value || !audioRef.value || !duration.value) return;
  const rect = progressWrap.value.getBoundingClientRect();
  const x = Math.min(Math.max(0, e.clientX - rect.left), rect.width);
  const ratio = x / rect.width;
  audioRef.value.currentTime = ratio * duration.value;
  currentTime.value = audioRef.value.currentTime;
}

// 音量持久化
watch(volume, (v) => {
  if (audioRef.value) audioRef.value.volume = v;
  localStorage.setItem("cantarella_volume", String(v));
});

// 模式切换
function toggleShuffle() {
  shuffle.value = !shuffle.value;
}
function toggleRepeat() {
  if (repeatMode.value === "off") repeatMode.value = "all";
  else if (repeatMode.value === "all") repeatMode.value = "one";
  else repeatMode.value = "off";
}

// 键盘空格
function onSpace() {
  if (
    document.activeElement &&
    ["INPUT", "TEXTAREA"].includes(document.activeElement.tagName)
  )
    return;
  togglePlay();
}

// 切换播放列表（文字）
function togglePlaylist() {
  playlistOpen.value = !playlistOpen.value;
  if (playlistOpen.value)
    nextTick(() => {
      const el = document.querySelector(".playlist li.active");
      el?.scrollIntoView({ block: "center", behavior: "smooth" });
    });
}

// 搜索防抖
function onSearchInput() {
  if (searchTimer) clearTimeout(searchTimer);
  searchTimer = setTimeout(() => {
    // 过滤由 computed filteredList 完成
    searchTimer = null;
  }, searchDebounceMs);
}
function clearSearch() {
  searchTerm.value = "";
}

// 格式化时间
function formatTime(sec?: number) {
  if (!sec || !isFinite(sec)) return "--:--";
  const s = Math.floor(sec % 60);
  const m = Math.floor(sec / 60);
  return `${String(m).padStart(2, "0")}:${String(s).padStart(2, "0")}`;
}

// 生命周期：绑定 audio ref、初始化列表和 videoSrc
onMounted(async () => {
  audioRef.value =
    (document.querySelector(".cantarella-player audio") as HTMLAudioElement) ??
    null;
  if (audioRef.value) audioRef.value.volume = volume.value;

  // 设置 videoSrc 与 class：桌面优先横屏(mp1), 移动优先竖屏(mp2)
  const isM = isMobile.value;
  const folder = isM ? "/mp1" : "/mp2";
  // 随机选择 1..4，如果没有资源请保证路径存在
  const idx = Math.floor(Math.random() * 4) + 1;
  videoSrc.value = `${folder}/1 (${idx}).mp4`;
  videoClass.value = isM ? "landscape" : "portrait";

  await fetchList();

  window.addEventListener("keydown", globalKeydown);
});

onBeforeUnmount(() => {
  window.removeEventListener("keydown", globalKeydown);
});

// 全局键盘
function globalKeydown(e: KeyboardEvent) {
  if (e.code === "Space") {
    if (
      document.activeElement &&
      ["INPUT", "TEXTAREA"].includes(document.activeElement.tagName)
    )
      return;
    e.preventDefault();
    togglePlay();
  } else if (e.code === "Escape") {
    pause();
  }
}
</script>

<style scoped lang="scss">
/* 千咲主题配色变量 */
$deep-bg: linear-gradient(145deg, #0a0a14 0%, #1a0a14 35%, #0a0a1a 100%);
$accent: #c62828; /* 主色：深红（制服/剪刀） */
$accent-light: #f44336; /* 亮红（高光/特效） */
$accent-dark: #8e0000; /* 暗红（阴影） */
$black: #0a0a0a; /* 纯黑（头发/底色） */
$gray-dark: #1a1a1a; /* 深灰（背景） */
$gray-light: #2a2a2a; /* 浅灰（边框） */
$white: #f5f5f5; /* 珍珠白（衬衫/文字） */
$gold: #ffd700; /* 金色（装饰/星炬） */

$shadow-core: rgba(198, 40, 40, 0.15);
$shadow-deep: rgba(10, 10, 20, 0.8);

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

/* ====== 千咲风格 · 音乐播放器 ====== */
.cantarella-player {
  padding: 20px;
  min-height: 100vh;
  background: $deep-bg;
  color: $white;
  padding-top: 80px;
  font-family: "Noto Sans SC", "Segoe UI", "Helvetica Neue", Arial, sans-serif;
  outline: none;
  position: relative;
  overflow-x: hidden;

  /* 血色弦线背景纹理 */
  &::before {
    content: "";
    position: fixed;
    left: 0;
    top: 100px;
    width: 100%;
    height: 200px;
    background-image: repeating-linear-gradient(
      to bottom,
      $accent-transparent-10 0px,
      $accent-transparent-10 1px,
      transparent 1px,
      transparent 20px
    );
    opacity: 0.15;
    pointer-events: none;
    mix-blend-mode: overlay;
    z-index: 0;
  }

  /* 动态血色光晕 */
  &::after {
    content: "";
    position: fixed;
    width: 500px;
    height: 500px;
    top: -100px;
    left: -100px;
    background: radial-gradient(
      circle,
      $accent-transparent-20 0%,
      transparent 70%
    );
    border-radius: 50%;
    pointer-events: none;
    z-index: 0;
    animation: pulse 12s ease-in-out infinite alternate;
  }
}

/* 主舞台区域 */
.stage {
  display: flex;
  gap: 24px;
  max-width: 1200px;
  margin: 0 auto;
  align-items: flex-start;
  position: relative;
  z-index: 10;
}

/* ====== 左侧：封面与控制区域 ====== */
.left {
  width: 450px;
  background: linear-gradient(
    145deg,
    rgba(26, 10, 20, 0.9) 0%,
    rgba(10, 10, 10, 0.95) 100%
  );
  border: 1px solid $accent-transparent-30;
  border-radius: 16px;
  padding: 20px;
  box-shadow: 0 20px 60px $shadow-deep, inset 0 1px 0 $white-transparent-10;
  backdrop-filter: blur(10px);
  position: relative;
  min-height: 400px;

  /* 剪刀剪影装饰 */
  &::before {
    content: "✂";
    position: absolute;
    top: 15px;
    right: 15px;
    font-size: 20px;
    color: $accent-transparent-20;
    transform: rotate(45deg);
    z-index: 1;
  }
}

/* 封面区域 */
.cover {
  width: 100%;
  height: 600px;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
  position: relative;
  box-shadow: inset 0 1px 0 $white-transparent-10, 0 12px 32px $shadow-deep;
  border: 1px solid $accent-transparent-20;

  /* 封面动态渐变背景 */
  &::before {
    content: "";
    position: absolute;
    inset: 0;
    background: linear-gradient(
      135deg,
      $accent-transparent-30 0%,
      $accent-transparent-10 50%,
      transparent 100%
    );
    opacity: 0.3;
    animation: gradient-shift 8s ease-in-out infinite alternate;
  }
}

/* 视频背景 */
.video-background {
  width: 100%;
  height: 100%;
  object-fit: cover;
  opacity: 0.8;
  filter: brightness(0.6) contrast(1.1) saturate(0.9) hue-rotate(0deg);
  z-index: 1;
  position: relative;

  &.landscape {
    aspect-ratio: 16 / 9;
  }

  &.portrait {
    aspect-ratio: 9 / 16;
    width: auto;
    height: 110%;
  }
}

/* 加载遮罩 */
.loading-overlay {
  position: absolute;
  inset: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background: $black-transparent-70;
  z-index: 2;
  border-radius: 12px;

  .spinner {
    width: 40px;
    height: 40px;
    border-radius: 50%;
    border: 3px solid $white-transparent-10;
    border-top-color: $accent-light;
    animation: spin 1s linear infinite;
  }

  .loading-text {
    margin-top: 12px;
    color: $white-transparent-90;
    font-weight: 500;
    font-size: 0.9rem;
  }
}

/* 控件区域 */
.controls {
  margin-top: 20px;

  .title {
    font-size: 1.2rem;
    font-weight: 700;
    color: $white;
    letter-spacing: 0.3px;
    padding-bottom: 8px;
    border-bottom: 1px solid $accent-transparent-20;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;

    &::before {
      content: "♪";
      margin-right: 8px;
      color: $accent-light;
    }
  }

  .meta {
    margin-top: 10px;
    font-size: 0.9rem;
    color: $white-transparent-70;
    display: flex;
    gap: 6px;
    align-items: center;

    .time {
      font-family: "Courier New", monospace;
      font-weight: 600;
    }

    .divider {
      opacity: 0.5;
    }
  }
}

/* 进度条 */
.progress-wrap {
  margin-top: 16px;
  cursor: pointer;
  touch-action: none;
  position: relative;

  &:hover {
    .progress-handle {
      transform: translateX(-50%) scale(1.1);
    }
  }
}

.progress-bar {
  height: 8px;
  background: $black-transparent-30;
  border-radius: 4px;
  overflow: hidden;
  position: relative;
  border: 1px solid $accent-transparent-10;
  box-shadow: inset 0 2px 4px $shadow-deep;
}

.progress {
  height: 100%;
  width: 0%;
  background: linear-gradient(90deg, $accent, $accent-dark);
  box-shadow: inset 0 1px 0 $white-transparent-20,
    0 0 10px $accent-transparent-50;
  transition: width 80ms linear;
  position: relative;

  &::after {
    content: "";
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: linear-gradient(
      90deg,
      transparent,
      $white-transparent-10,
      transparent
    );
    animation: shimmer 2s linear infinite;
  }
}

.progress-handle {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: $accent-light;
  transform: translateX(-50%);
  position: absolute;
  top: -5px;
  box-shadow: 0 0 0 4px $accent-transparent-30,
    0 6px 24px $accent-transparent-50;
  transition: transform 0.2s ease, left 80ms linear;
  border: 2px solid $white;
  z-index: 1;
}

/* 按钮行 */
.btns {
  margin-top: 20px;
  display: flex;
  align-items: center;
  gap: 12px;
  flex-wrap: wrap;
}

.icon,
.play {
  border: none;
  background: $gray-dark;
  color: $white;
  font-size: 18px;
  cursor: pointer;
  padding: 10px;
  border-radius: 10px;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  border: 1px solid $accent-transparent-20;
  box-shadow: 0 6px 20px $shadow-deep;

  &:hover:not(:disabled) {
    background: $accent-transparent-30;
    transform: translateY(-2px);
    box-shadow: 0 10px 28px $accent-transparent-30,
      0 0 0 1px $accent-transparent-50;
  }

  &:active:not(:disabled) {
    transform: translateY(0);
  }

  &:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }
}

.play {
  width: 60px;
  height: 60px;
  font-size: 22px;
  background: linear-gradient(145deg, $accent, $accent-dark);
  border-radius: 50%;
  box-shadow: 0 10px 30px $accent-transparent-50,
    inset 0 1px 0 $white-transparent-20;

  &:hover:not(:disabled) {
    background: linear-gradient(145deg, $accent-light, $accent);
    transform: translateY(-3px) scale(1.05);
    box-shadow: 0 14px 36px $accent-transparent-70,
      inset 0 1px 0 $white-transparent-30;
  }
}

.modes {
  display: flex;
  gap: 8px;

  button {
    width: 40px;
    height: 40px;
    border: none;
    background: $gray-dark;
    color: $white-transparent-70;
    border-radius: 8px;
    cursor: pointer;
    transition: all 0.3s ease;
    display: flex;
    align-items: center;
    justify-content: center;
    border: 1px solid $accent-transparent-10;

    &:hover {
      background: $accent-transparent-20;
      color: $white;
    }

    &.active {
      background: $accent-transparent-30;
      color: $accent-light;
      border-color: $accent-transparent-50;
      box-shadow: 0 0 15px $accent-transparent-30;
    }
  }
}

.volume {
  flex: 1;
  min-width: 120px;
  max-width: 200px;

  input[type="range"] {
    width: 100%;
    height: 6px;
    -webkit-appearance: none;
    appearance: none;
    background: $black-transparent-30;
    border-radius: 3px;
    outline: none;
    border: 1px solid #fff;

    &::-webkit-slider-thumb {
      -webkit-appearance: none;
      appearance: none;
      width: 16px;
      height: 16px;
      border-radius: 50%;
      background: $accent-light;
      cursor: pointer;
      border: 2px solid $white;
      box-shadow: 0 4px 12px $accent-transparent-50;
    }

    &::-moz-range-thumb {
      width: 16px;
      height: 16px;
      border-radius: 50%;
      background: $accent-light;
      cursor: pointer;
      border: 2px solid $white;
      box-shadow: 0 4px 12px $accent-transparent-50;
    }

    &:hover {
      &::-webkit-slider-thumb {
        transform: scale(1.1);
      }

      &::-moz-range-thumb {
        transform: scale(1.1);
      }
    }
  }
}

/* 错误消息 */
.error-msg {
  margin-top: 12px;
  padding: 10px;
  background: linear-gradient(
    145deg,
    rgba(140, 30, 30, 0.3) 0%,
    rgba(100, 20, 20, 0.4) 100%
  );
  border: 1px solid $accent-transparent-50;
  border-radius: 8px;
  color: $accent-light;
  font-weight: 600;
  font-size: 0.9rem;
  text-align: center;
  animation: error-pulse 2s ease-in-out infinite;
}

/* ====== 右侧：播放列表 ====== */
.right {
  flex: 1;
  background: linear-gradient(
    145deg,
    rgba(26, 10, 20, 0.9) 0%,
    rgba(10, 10, 10, 0.95) 100%
  );
  border: 1px solid $accent-transparent-30;
  border-radius: 16px;
  padding: 20px;
  box-shadow: 0 20px 60px $shadow-deep, inset 0 1px 0 $white-transparent-10;
  backdrop-filter: blur(10px);
  display: flex;
  flex-direction: column;
  max-height: 70vh;
  overflow: hidden;

  /* 折叠状态 */
  &.collapsed {
    max-height: 72px;

    .list-area,
    .search-wrap {
      display: none;
    }
  }
}

/* 播放列表头部 */
.playlist-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
  position: relative;
  z-index: 1;

  &::after {
    content: "";
    position: absolute;
    bottom: -8px;
    left: 0;
    right: 0;
    height: 1px;
    background: linear-gradient(
      90deg,
      transparent,
      $accent-transparent-30,
      transparent
    );
  }
}

.left-head {
  display: flex;
  gap: 16px;
  align-items: center;

  h3 {
    margin: 0;
    font-size: 1.3rem;
    font-weight: 700;
    color: $white;
    position: relative;

    &::before {
      content: "🎵";
      margin-right: 8px;
    }
  }
}

/* 切换列表按钮 */
.toggle-list-text {
  padding: 6px 12px;
  border-radius: 8px;
  background: $accent-transparent-10;
  border: 1px solid $accent-transparent-30;
  color: $white;
  font-size: 0.85rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s ease;
  display: none;
  &:hover {
    background: $accent-transparent-30;
    transform: translateY(-2px);
    box-shadow: 0 6px 18px $accent-transparent-20;
  }
}

.api-hint {
  color: $white-transparent-50;
  font-size: 0.85rem;
  font-style: italic;
}

/* 搜索框 */
.search-wrap {
  display: flex;
  align-items: center;
  gap: 8px;
  position: relative;

  input {
    padding: 10px 14px;
    padding-right: 40px;
    border-radius: 10px;
    border: 1px solid $accent-transparent-20;
    background: $black-transparent-50;
    color: $white;
    width: 240px;
    font-size: 0.95rem;
    transition: all 0.3s ease;

    &::placeholder {
      color: $white-transparent-50;
    }

    &:focus {
      outline: none;
      border-color: $accent-transparent-50;
      box-shadow: 0 0 0 3px $accent-transparent-20;
      width: 280px;
    }
  }

  .clear {
    position: absolute;
    right: 10px;
    background: transparent;
    border: none;
    color: $white-transparent-50;
    cursor: pointer;
    font-size: 1.2rem;
    transition: all 0.3s ease;

    &:hover {
      color: $accent-light;
      transform: scale(1.2);
    }
  }
}

/* 列表区域 */
.list-area {
  flex: 1;
  overflow: hidden;
  position: relative;
}

.list-loading {
  display: flex;
  align-items: center;
  gap: 12px;
  color: $white-transparent-70;
  padding: 20px 0;
  justify-content: center;

  .small-spinner {
    width: 20px;
    height: 20px;
    border-radius: 50%;
    border: 2px solid $white-transparent-10;
    border-top-color: $accent-light;
    animation: spin 1s linear infinite;
  }
}

/* 播放列表 */
.playlist {
  list-style: none;
  padding: 0;
  margin: 0;
  overflow-y: auto;
  max-height: calc(70vh - 90px);
  display: block;
  scrollbar-width: thin;
  scrollbar-color: $accent-transparent-30 transparent;

  &::-webkit-scrollbar {
    width: 8px;
  }

  &::-webkit-scrollbar-track {
    background: transparent;
    border-radius: 4px;
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
}

/* 列表项 */
.playlist li {
  display: grid;
  grid-template-columns: 1fr auto;
  gap: 16px;
  align-items: center;
  padding: 14px 16px;
  border-radius: 12px;
  cursor: pointer;
  transition: all 0.3s ease;
  margin-bottom: 8px;
  position: relative;
  overflow: hidden;

  /* 列表项背景光晕效果 */
  &::before {
    content: "";
    position: absolute;
    inset: 0;
    background: linear-gradient(
      145deg,
      transparent 0%,
      $accent-transparent-10 100%
    );
    opacity: 0;
    transition: opacity 0.3s ease;
    z-index: 0;
  }

  &:hover {
    transform: translateX(4px);

    &::before {
      opacity: 1;
    }

    .dot {
      background: $accent-light;
      box-shadow: 0 0 15px $accent-transparent-50;
    }
  }

  /* 当前播放项 */
  &.active {
    background: linear-gradient(
      145deg,
      rgba(198, 40, 40, 0.2) 0%,
      rgba(142, 0, 0, 0.15) 100%
    );
    border: 1px solid $accent-transparent-50;
    box-shadow: 0 10px 30px $accent-transparent-30,
      inset 0 1px 0 $white-transparent-10;

    .dot {
      background: $accent-light;
      animation: dot-pulse 1.5s ease-in-out infinite;
    }

    .title {
      color: $white;
      font-weight: 700;
    }
  }
}

/* 左列内容 */
.left-col {
  display: flex;
  gap: 14px;
  align-items: center;
  min-width: 0;
  position: relative;
  z-index: 1;
}

.dot {
  width: 12px;
  height: 12px;
  border-radius: 50%;
  background: $white-transparent-30;
  flex: 0 0 auto;
  transition: all 0.3s ease;
  box-shadow: 0 4px 12px $shadow-core;
}

.title {
  font-weight: 600;
  color: $white-transparent-90;
  overflow-wrap: anywhere;
  max-width: 100%;
  line-height: 1.4;

  /* 两行限制 */
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* 右列时长 */
.right-col {
  position: relative;
  z-index: 1;
}

.len {
  color: $white-transparent-50;
  font-weight: 600;
  min-width: 48px;
  text-align: right;
  font-family: "Courier New", monospace;
  font-size: 0.9rem;
}

/* ====== 音频元素 ====== */
audio {
  display: none;
}

/* ====== 响应式设计 ====== */
@media (max-width: 920px) {
  .volume {
    display: none;
  }
  .stage {
    flex-direction: column;
    gap: 16px;
  }

  .left {
    width: 100%;
    min-height: auto;
  }

  .cover {
    height: 240px;
  }

  .right {
    width: 100%;
    max-height: 400px;

    &.collapsed {
      max-height: 72px;
    }
  }

  .toggle-list-text {
    display: block;
  }

  .search-wrap input {
    width: 180px;

    &:focus {
      width: 220px;
    }
  }

  .playlist {
    max-height: 300px;
  }
}

@media (max-width: 520px) {
  .cantarella-player {
    padding: 12px;
    padding-top: 70px;
  }

  .cover {
    height: 200px;
  }

  .btns {
    flex-wrap: nowrap;
    overflow-x: auto;
    padding-bottom: 8px;

    &::-webkit-scrollbar {
      height: 4px;
    }

    &::-webkit-scrollbar-thumb {
      background: $accent-transparent-30;
      border-radius: 2px;
    }
  }

  .search-wrap input {
    width: 140px;

    &:focus {
      width: 180px;
    }
  }

  .playlist li {
    padding: 12px;
    grid-template-columns: 1fr;
    gap: 8px;

    .right-col {
      text-align: left;
    }
  }
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

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

@keyframes gradient-shift {
  0% {
    background-position: 0% 0%;
  }
  100% {
    background-position: 100% 100%;
  }
}

@keyframes shimmer {
  0% {
    background-position: -200px 0;
  }
  100% {
    background-position: calc(200px + 100%) 0;
  }
}

@keyframes error-pulse {
  0%,
  100% {
    opacity: 0.8;
  }
  50% {
    opacity: 1;
  }
}

@keyframes dot-pulse {
  0%,
  100% {
    transform: scale(1);
    box-shadow: 0 0 10px $accent-transparent-30;
  }
  50% {
    transform: scale(1.2);
    box-shadow: 0 0 20px $accent-transparent-50;
  }
}

@keyframes playing {
  0%,
  100% {
    opacity: 0.6;
  }
  50% {
    opacity: 1;
  }
}
</style>