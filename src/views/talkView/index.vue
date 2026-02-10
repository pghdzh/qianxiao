<template>
  <div class="chat-page">
    <div class="carousel carousel1" aria-hidden="true">
      <img
        v-for="(src, idx) in randomFive"
        :key="idx"
        :src="src"
        class="carousel-image"
        :class="{ active: idx === currentIndex }"
      />
    </div>
    <div class="carousel carousel2" aria-hidden="true">
      <img
        v-for="(src, idx) in randomFive2"
        :key="idx"
        :src="src"
        class="carousel-image"
        :class="{ active: idx === currentIndex }"
      />
    </div>

    <div class="chat-container">
      <!-- 统计面板（放在聊天容器顶部） -->
      <div class="stats-panel">
        <div class="stat-item">
          总对话次数：<span>{{ stats.totalChats }}</span>
        </div>
        <div class="stat-item">
          首次使用：<span>{{
            new Date(stats.firstTimestamp).toISOString().slice(0, 10)
          }}</span>
        </div>
        <div class="stat-item">
          活跃天数：<span>{{ stats.activeDates.length }}</span> 天
        </div>
        <div class="stat-item">
          今日对话：<span>{{
            stats.dailyChats[new Date().toISOString().slice(0, 10)] || 0
          }}</span>
          次
        </div>
        <button class="detail-btn" @click="showModal = true">全部</button>
      </div>
      <div class="messages" ref="msgList">
        <transition-group name="msg" tag="div">
          <div
            v-for="msg in chatLog"
            :key="msg.id"
            :class="[
              'message',
              msg.role,
              { error: msg.isError, egg: msg.isEgg },
            ]"
          >
            <div class="avatar" :class="msg.role"></div>
            <div class="bubble">
              <div class="content" v-html="msg.text"></div>
            </div>
          </div>
          <div v-if="loading" class="message bot" key="loading">
            <div class="avatar bot"></div>
            <div class="bubble loading">
              正在思考中
              <span class="dots">
                <span class="dot"></span>
                <span class="dot"></span>
                <span class="dot"></span>
              </span>
            </div>
          </div>
        </transition-group>
      </div>
      <form class="input-area" @submit.prevent="sendMessage">
        <textarea
          v-model="input"
          placeholder="向千咲提问…"
          :disabled="loading"
          @keydown="handleKeydown"
          rows="1"
        ></textarea>
        <button
          type="submit"
          class="send-btn"
          :disabled="!input.trim() || loading"
        >
          发送
        </button>
        <div class="btn-group">
          <button
            type="button"
            class="clear-btn"
            @click="clearChat"
            :disabled="loading"
            title="清空对话"
          >
            ✖
          </button>
        </div>
        <button
          type="button"
          class="copy-btn"
          :disabled="!chatLog.length || loading"
          @click="() => copyChat()"
        >
          {{ copyButtonText }}
        </button>
        <!-- 发送按钮 -->

        <!-- 统计数据按钮 -->
        <button
          type="button"
          class="Alldetail-btn"
          @click="showModal = true"
          title="查看统计"
        >
          统计数据
        </button>
      </form>
    </div>

    <!-- 详细统计弹窗 -->
    <div v-if="showModal" class="modal-overlay" @click.self="showModal = false">
      <div class="modal-content">
        <h3>详细统计</h3>
        <ul class="detail-list">
          <li>总对话次数：{{ stats.totalChats }}</li>
          <li>
            首次使用：{{
              new Date(stats.firstTimestamp).toISOString().slice(0, 10)
            }}
          </li>
          <li>活跃天数：{{ stats.activeDates.length }} 天</li>
          <li>
            今日对话：{{
              stats.dailyChats[new Date().toISOString().slice(0, 10)] || 0
            }}
            次
          </li>
          <li>总使用时长：{{ formatDuration(stats.totalTime) }}</li>
          <li>当前连续活跃：{{ stats.currentStreak }} 天</li>
          <li>最长连续活跃：{{ stats.longestStreak }} 天</li>
          <li>
            最活跃日：{{ mostActiveDayComputed }} （{{
              stats.dailyChats[mostActiveDayComputed] || 0
            }}
            次）
          </li>
        </ul>
        <button class="close-btn" @click="showModal = false">关闭</button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import {
  reactive,
  ref,
  computed,
  onMounted,
  nextTick,
  watch,
  onBeforeUnmount,
} from "vue";
import { sendMessageToHui } from "@/api/deepseekApi";

const STORAGE_KEY = "hui_chat_log";

// 本地存储键名
const STORAGE_STATS_KEY = "hui_chat_stats";
const showModal = ref(false);
// Stats 类型声明，确保所有字段都有默认值
interface Stats {
  firstTimestamp: number; // 首次使用时间戳
  totalChats: number; // 总对话次数
  activeDates: string[]; // 有发言的日期列表（yyyy‑mm‑dd）
  dailyChats: Record<string, number>; // 每日对话次数
  currentStreak: number; // 当前连续活跃天数
  longestStreak: number; // 历史最长连续活跃天数

  totalTime: number; // 累计使用时长（毫秒）
}

// 默认值，用于补齐本地存储中可能缺失的字段
const defaultStats: Stats = {
  firstTimestamp: Date.now(),
  totalChats: 0,
  activeDates: [],
  dailyChats: {},
  currentStreak: 0,
  longestStreak: 0,

  totalTime: 0,
};

// 从 localStorage 加载并合并默认值
function loadStats(): Stats {
  const saved = localStorage.getItem(STORAGE_STATS_KEY);
  if (saved) {
    try {
      const parsed = JSON.parse(saved);
      return { ...defaultStats, ...parsed };
    } catch {
      console.warn("加载统计数据失败，使用默认值");
    }
  }
  return { ...defaultStats };
}

// 保存到 localStorage
function saveStats() {
  localStorage.setItem(STORAGE_STATS_KEY, JSON.stringify(stats));
}

// 更新「活跃天数」及「连续活跃」逻辑
function updateActive(date: string) {
  if (!stats.activeDates.includes(date)) {
    stats.activeDates.push(date);
    updateStreak();
    saveStats(); // 持久化活跃天数变化
  }
}
function updateStreak() {
  const dates = [...stats.activeDates].sort();
  let curr = 0,
    max = stats.longestStreak,
    prevTs = 0;
  const todayStr = new Date().toISOString().slice(0, 10);
  dates.forEach((d) => {
    const ts = new Date(d).getTime();
    if (prevTs && ts - prevTs === 86400000) curr++;
    else curr = 1;
    max = Math.max(max, curr);
    prevTs = ts;
  });
  stats.currentStreak = dates[dates.length - 1] === todayStr ? curr : 0;
  stats.longestStreak = max;
  saveStats();
}

// 更新「每日对话次数」
function updateDaily(date: string) {
  stats.dailyChats[date] = (stats.dailyChats[date] || 0) + 1;
  saveStats(); // 持久化活跃天数变化
}

// 计算最活跃日
const mostActiveDayComputed = computed(() => {
  let day = "",
    max = 0;
  for (const [d, c] of Object.entries(stats.dailyChats)) {
    if (c > max) {
      max = c;
      day = d;
    }
  }
  return day || new Date().toISOString().slice(0, 10);
});

// 格式化总使用时长
function formatDuration(ms: number): string {
  const totalMin = Math.floor(ms / 60000);
  const h = Math.floor(totalMin / 60);
  const m = totalMin % 60;
  return h ? `${h} 小时 ${m} 分钟` : `${m} 分钟`;
}

// —— Vue 响应式状态 ——
const stats = reactive<Stats>(loadStats());
// 会话开始时间，用于计算本次时长
const sessionStart = Date.now();

interface ChatMsg {
  id: number;
  role: "user" | "bot";
  text: string;
  isError?: boolean;
  isEgg?: boolean;
}

const chatLog = ref<ChatMsg[]>(loadChatLog());
const input = ref("");
const loading = ref(false);
const msgList = ref<HTMLElement>();

async function sendMessage() {
  if (!input.value.trim()) return;
  if (stats.totalChats === 0 && !localStorage.getItem(STORAGE_STATS_KEY)) {
    stats.firstTimestamp = Date.now();
    saveStats();
  }
  const date = new Date().toISOString().slice(0, 10); // 每次都取最新“今天”
  stats.totalChats++;
  updateActive(date);
  updateDaily(date);
  saveStats();

  const userText = input.value;
  chatLog.value.push({
    id: Date.now(),
    role: "user",
    text: userText,
  });
  input.value = "";
  loading.value = true;

  try {
    //  throw new Error("测试错误");
    const history = chatLog.value.filter((msg) => !msg.isEgg && !msg.isError);
    const botReply = await sendMessageToHui(userText, history);

    if (botReply == "error") {
      chatLog.value.push({
        id: Date.now() + 2,
        role: "bot",
        text: "API余额耗尽了，去b站提醒我充钱吧",
        isError: true,
      });
    } else {
      chatLog.value.push({
        id: Date.now() + 1,
        role: "bot",
        text: botReply,
      });
    }

    // —— 彩蛋结束 ——
  } catch (e) {
    console.error(e);
  } finally {
    loading.value = false;
    await scrollToBottom();
  }
}

function handleKeydown(e: KeyboardEvent) {
  if (e.key === "Enter") sendMessage();
}

function clearChat() {
  if (confirm("确定要清空全部对话吗？")) {
    chatLog.value = [
      {
        id: Date.now(),
        role: "bot",
        text: "现在的雨声，和穗波市的不同，有了清晰的节奏。……抱歉，只是突然想到了。",
      },
    ];
    localStorage.removeItem(STORAGE_KEY);
  }
}

function loadChatLog(): ChatMsg[] {
  const saved = localStorage.getItem(STORAGE_KEY);
  if (saved) {
    try {
      return JSON.parse(saved);
    } catch (e) {
      console.error("chatLog 解析失败：", e);
    }
  }
  return [
    {
      id: Date.now(),
      role: "bot",
      text: "现在的雨声，和穗波市的不同，有了清晰的节奏。……抱歉，只是突然想到了。",
    },
  ];
}

async function scrollToBottom() {
  await nextTick();
  if (msgList.value) {
    msgList.value.scrollTop = msgList.value.scrollHeight;
  }
}

//复制文本相关
// 复制按钮文字与定时器（加入到 script setup）
const copyButtonText = ref("复制");
let _copyTimer: number | null = null;

// 把 HTML 文本转为纯文本（保留 <br> 换行）
function stripHtml(html = ""): string {
  if (typeof document === "undefined") {
    // SSR 安全返回
    return html.replace(/<br\s*\/?>/gi, "\n").replace(/<[^>]+>/g, "");
  }
  const div = document.createElement("div");
  const withBreaks = String(html).replace(/<br\s*\/?>/gi, "\n");
  div.innerHTML = withBreaks;
  return div.textContent || div.innerText || "";
}

// 根据 chatLog.build 出可读的纯文本（含时间戳）
function buildChatPlainText(): string {
  // chatLog 是你已有的 ref<ChatMsg[]>
  return chatLog.value
    .map((msg) => {
      // 尝试把 msg.id 当作时间戳（你的代码里确实用 Date.now() 作为 id）
      const time = new Date(msg.id).toLocaleString();
      const who = msg.role === "user" ? "你" : "千咲";
      const text = stripHtml(msg.text);
      return `[${time}] ${who}: ${text}`;
    })
    .join("\n\n");
}

// 回退复制：execCommand
function fallbackCopyText(text: string) {
  const textarea = document.createElement("textarea");
  textarea.value = text;
  textarea.style.position = "fixed"; // 防止滚动到页面底部
  textarea.style.left = "-9999px";
  textarea.style.top = "0";
  document.body.appendChild(textarea);
  textarea.select();
  textarea.setSelectionRange(0, textarea.value.length);
  const ok = document.execCommand("copy");
  document.body.removeChild(textarea);
  if (!ok) throw new Error("execCommand copy failed");
}

// 主复制函数（在模板里绑定 @click="copyChat"）
async function copyChat() {
  const text = buildChatPlainText();
  if (!text) {
    copyButtonText.value = "无内容可复制";
    clearTimeout(_copyTimer as number);
    _copyTimer = window.setTimeout(() => (copyButtonText.value = "复制"), 1600);
    return;
  }

  try {
    if (navigator.clipboard && navigator.clipboard.writeText) {
      await navigator.clipboard.writeText(text);
    } else {
      fallbackCopyText(text);
    }
    copyButtonText.value = "已复制";
  } catch (err) {
    try {
      // 再试一次回退方案
      fallbackCopyText(text);
      copyButtonText.value = "已复制";
    } catch (e) {
      console.error("复制失败：", e);
      copyButtonText.value = "复制失败";
    }
  } finally {
    clearTimeout(_copyTimer as number);
    _copyTimer = window.setTimeout(() => (copyButtonText.value = "复制"), 1600);
  }
}

watch(
  chatLog,
  async () => {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(chatLog.value));
    await scrollToBottom();
  },
  { deep: true }
);

function handleBeforeUnload() {
  stats.totalTime += Date.now() - sessionStart;
  saveStats();
}

// ========== 背景图片导入与轮播 ==========
const modules = import.meta.glob("@/assets/images1/*.{jpg,png,jpeg,webp}", {
  eager: true,
});
const allSrcs: string[] = Object.values(modules).map((mod: any) => mod.default);

const modules2 = import.meta.glob("@/assets/images2/*.{jpg,png,jpeg,webp}", {
  eager: true,
});
const allSrcs2: string[] = Object.values(modules2).map(
  (mod: any) => mod.default
);

function shuffle<T>(arr: T[]): T[] {
  const a = arr.slice();
  for (let i = a.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [a[i], a[j]] = [a[j], a[i]];
  }
  return a;
}
const randomFive = ref<string[]>(shuffle(allSrcs).slice(0, 5));
const randomFive2 = ref<string[]>(shuffle(allSrcs2).slice(0, 5));

const currentIndex = ref(0);
let Imgtimer: number | undefined;

onMounted(() => {
  scrollToBottom();
  window.addEventListener("beforeunload", handleBeforeUnload);
  Imgtimer = window.setInterval(() => {
    currentIndex.value =
      (currentIndex.value + 1) % Math.max(1, randomFive.value.length);
  }, 5200);
});

onBeforeUnmount(() => {
  window.removeEventListener("beforeunload", handleBeforeUnload);
  if (Imgtimer) clearInterval(Imgtimer);
  if (_copyTimer) clearTimeout(_copyTimer);
});
</script>
<style scoped lang="scss">
/* 千咲主题核心配色 - 使用CSS变量 */
.chat-page {
  /* 配色变量 */
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

  /* 派生颜色（不使用darken/lighten函数） */
  --accent-transparent-10: rgba(198, 40, 40, 0.1);
  --accent-transparent-20: rgba(198, 40, 40, 0.2);
  --accent-transparent-30: rgba(198, 40, 40, 0.3);
  --accent-transparent-50: rgba(198, 40, 40, 0.5);
  --white-transparent-10: rgba(245, 245, 245, 0.1);
  --white-transparent-20: rgba(245, 245, 245, 0.2);
  --white-transparent-30: rgba(245, 245, 245, 0.3);
  --white-transparent-50: rgba(245, 245, 245, 0.5);
  --white-transparent-70: rgba(245, 245, 245, 0.7);
  --white-transparent-90: rgba(245, 245, 245, 0.9);
  --black-transparent-30: rgba(10, 10, 10, 0.3);
  --black-transparent-50: rgba(10, 10, 10, 0.5);
  --black-transparent-70: rgba(10, 10, 10, 0.7);
  --gold-transparent-20: rgba(255, 215, 0, 0.2);
  --gold-transparent-30: rgba(255, 215, 0, 0.3);

  /* ====== 千咲风格 · 整体页面 ====== */
  padding-top: 64px;
  min-height: 100vh;
  font-family: "Noto Sans SC", "Segoe UI", "Helvetica Neue", Arial, sans-serif;
  color: var(--white);
  display: flex;
  flex-direction: column;
  background: var(--deep-bg);
  position: relative;
  overflow-x: hidden;

  /* 背景"血色月光"纹理 */
  &::before {
    content: "";
    position: fixed;
    inset: 0;
    background-image: radial-gradient(
        circle at 20% 30%,
        var(--accent-transparent-10) 0%,
        transparent 25%
      ),
      radial-gradient(
        circle at 80% 70%,
        var(--accent-transparent-10) 0%,
        transparent 25%
      ),
      linear-gradient(
        45deg,
        transparent 49%,
        var(--white-transparent-10) 50%,
        transparent 51%
      ),
      linear-gradient(
        -45deg,
        transparent 49%,
        var(--white-transparent-10) 50%,
        transparent 51%
      );
    background-size: 200% 200%, 200% 200%, 60px 60px, 60px 60px;
    pointer-events: none;
    z-index: 0;
    opacity: 0.4;
  }

  /* 动态红色光晕，象征赤瞳与战斗能量 */
  &::after {
    content: "";
    position: fixed;
    width: 400px;
    height: 400px;
    top: 10%;
    right: 5%;
    background: radial-gradient(
      circle,
      var(--accent-transparent-20) 0%,
      transparent 70%
    );
    border-radius: 50%;
    pointer-events: none;
    z-index: 0;
    animation: pulse 8s ease-in-out infinite alternate;
  }

  .carousel {
    position: absolute;
    inset: 0;
    z-index: 1;
    pointer-events: none;

    &::before {
      content: "";
      position: absolute;
      inset: 0;
      background: linear-gradient(
        to bottom,
        var(--black-transparent-70) 0%,
        var(--black-transparent-50) 50%,
        var(--black-transparent-70) 100%
      );
      pointer-events: none;
      z-index: 2;
      mix-blend-mode: multiply;
    }

    .carousel-image {
      position: absolute;
      width: 100%;
      height: 100%;
      object-fit: cover;
      opacity: 0;
      filter: grayscale(0.3) contrast(1.2) brightness(0.7);
      transition: opacity 1.2s cubic-bezier(0.4, 0, 0.2, 1);

      &.active {
        opacity: 1;
      }
    }
  }

  .carousel2 {
    display: none;
  }

  .chat-container {
    flex: 1;
    width: 920px;
    max-width: calc(100% - 32px);
    margin: 0 auto;
    padding: 24px;
    display: flex;
    flex-direction: column;
    gap: 20px;
    z-index: 10;
    position: relative;
  }

  /* ====== 统计面板 - 血色数据板 ====== */
  .stats-panel {
    display: flex;
    align-items: center;
    background: linear-gradient(135deg, var(--gray-dark) 0%, var(--black) 100%);
    padding: 12px 20px;
    border-radius: 12px;
    font-size: 13px;
    color: var(--white);
    justify-content: space-around;
    box-shadow: 0 8px 32px var(--shadow-deep),
      inset 0 1px 0 var(--white-transparent-10);
    border: 1px solid var(--accent-transparent-30);
    backdrop-filter: blur(10px);
    position: relative;
    overflow: hidden;

    /* 左上角血色三角装饰 */
    &::before {
      content: "";
      position: absolute;
      top: 0;
      left: 0;
      width: 0;
      height: 0;
      border-style: solid;
      border-width: 20px 20px 0 0;
      border-color: var(--accent) transparent transparent transparent;
      opacity: 0.5;
    }

    .stat-item {
      text-align: center;
      padding: 4px 8px;

      span {
        color: var(--accent-light);
        font-weight: 700;
        font-size: 15px;

        margin-top: 2px;
        text-shadow: 0 0 8px var(--shadow-core);
      }
    }

    .detail-btn {
      background: var(--accent-transparent-10);
      border: 1px solid var(--accent-transparent-30);
      border-radius: 8px;
      color: var(--white);
      padding: 6px 16px;
      cursor: pointer;
      font-size: 13px;
      font-weight: 500;
      transition: all 0.25s ease;
      letter-spacing: 0.5px;

      &:hover {
        background: var(--accent-transparent-20);
        box-shadow: 0 4px 12px var(--shadow-core),
          0 0 0 1px var(--accent-transparent-30);
        transform: translateY(-1px);
      }
    }
  }

  /* ====== 消息区域 ====== */
  .messages {
    flex: 1;
    overflow-y: auto;
    padding: 16px 8px 120px;
    overscroll-behavior: contain;
    scroll-behavior: smooth;
    max-height: 72vh;
    z-index: 10;
  }

  /* 消息气泡 - 左侧千咲 / 右侧用户 */
  .message {
    display: flex;
    align-items: flex-start;
    margin-bottom: 20px;
    position: relative;

    &.user {
      flex-direction: row-reverse;
    }

    &.error .bubble {
      background: linear-gradient(
        135deg,
        rgba(120, 40, 40, 0.9) 0%,
        rgba(80, 30, 30, 0.85) 100%
      );
      border-color: rgba(200, 80, 80, 0.4);
    }

    &.egg .bubble {
      border-color: var(--accent-transparent-50);
      box-shadow: 0 8px 25px var(--shadow-core),
        inset 0 0 20px var(--accent-transparent-10);
      animation: gentle-glow 2s ease-in-out infinite alternate;
    }

    .avatar {
      width: 50px;
      height: 50px;
      border-radius: 50%;
      margin: 0 14px;
      background-size: cover;
      background-position: center;
      flex-shrink: 0;
      z-index: 5;
      border: 2px solid transparent;
      background-clip: padding-box;
      box-shadow: 0 6px 20px var(--shadow-deep);

      &.bot {
        /* 请将路径替换为千咲的头像 */
        background-image: url("@/assets/avatar/changli.png");
        border-color: var(--accent-transparent-30);
        box-shadow: 0 8px 24px var(--shadow-core),
          inset 0 1px 0 var(--white-transparent-10);
        transform: scaleX(-1);
      }

      &.user {
        background: linear-gradient(
          135deg,
          var(--gray-light) 0%,
          var(--gray-dark) 100%
        );
        border-color: var(--white-transparent-20);
      }
    }

    .bubble {
      max-width: 75%;
      background: linear-gradient(
        145deg,
        var(--black-transparent-70) 0%,
        var(--gray-dark) 100%
      );
      border: 1px solid var(--accent-transparent-20);
      padding: 16px 18px;
      border-radius: 18px;
      line-height: 1.7;
      word-break: break-word;
      box-shadow: 0 10px 30px var(--shadow-deep),
        inset 0 1px 0 var(--white-transparent-10);
      transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
      color: var(--white);
      position: relative;
      backdrop-filter: blur(5px);

      /* 气泡左上角金色剪影装饰 */
      &::before {
        content: "✄";
        position: absolute;
        top: -8px;
        left: -8px;
        font-size: 16px;
        color: var(--accent);
      }

      &:hover {
        box-shadow: 0 14px 40px var(--shadow-deep),
          0 0 20px var(--accent-transparent-20),
          inset 0 1px 0 var(--white-transparent-10);
        transform: translateY(-3px);
        border-color: var(--accent-transparent-30);
      }

      &.loading {
        color: var(--white-transparent-90);
        .dots .dot {
          background-color: var(--accent);
        }
      }

      /* 千咲气泡（左侧） */
      .message.bot & {
        border-radius: 4px 18px 18px 18px;
        margin-left: 8px;
        border-left-width: 3px;
        border-left-color: var(--accent-transparent-50);
        background: linear-gradient(
          145deg,
          rgba(26, 10, 20, 0.9) 0%,
          rgba(40, 20, 30, 0.95) 100%
        );
      }

      /* 用户气泡（右侧） */
      .message.user & {
        border-radius: 18px 4px 18px 18px;
        margin-right: 8px;
        background: linear-gradient(
          145deg,
          var(--gray-dark) 0%,
          var(--black) 100%
        );
        border-color: var(--white-transparent-30);
      }

      .content {
        font-size: 15px;
      }

      .dots {
        display: inline-flex;
        align-items: center;
        margin-left: 8px;

        .dot {
          width: 6px;
          height: 6px;
          border-radius: 50%;
          margin: 0 2px;
          opacity: 0.4;
          background-color: var(--accent);
          animation: dot-pulse 1.4s ease-in-out infinite;

          &:nth-child(1) {
            animation-delay: 0s;
          }
          &:nth-child(2) {
            animation-delay: 0.2s;
          }
          &:nth-child(3) {
            animation-delay: 0.4s;
          }
        }
      }
    }
  }

  /* ====== 输入区域 - 血色操作面板 ====== */
  .input-area {
    position: sticky;
    bottom: 16px;
    display: flex;
    align-items: center;
    background: linear-gradient(135deg, var(--gray-dark) 0%, var(--black) 100%);
    backdrop-filter: blur(15px);
    padding: 16px;
    gap: 12px;
    z-index: 100;
    border-radius: 16px;
    box-shadow: 0 20px 50px var(--shadow-deep),
      inset 0 1px 0 var(--white-transparent-10);
    border: 1px solid var(--accent-transparent-30);

    textarea {
      flex: 1;
      padding: 14px 18px;
      background: var(--black-transparent-50);
      border: 1px solid var(--accent-transparent-30);
      color: var(--white);
      font-size: 15px;
      line-height: 1.5;
      outline: none;
      resize: none;
      overflow: hidden;
      min-height: 52px;
      max-height: 160px;
      border-radius: 12px;
      box-shadow: inset 0 2px 6px var(--shadow-deep);
      transition: all 0.3s ease;
      font-family: inherit;

      &::placeholder {
        color: var(--white-transparent-50);
        font-style: italic;
      }

      &:focus {
        border-color: var(--accent-transparent-50);
        box-shadow: inset 0 2px 8px var(--shadow-deep),
          0 0 0 3px var(--accent-transparent-20);
        background: var(--black-transparent-70);
      }
    }

    .btn-group {
      display: flex;
      gap: 8px;

      button {
        display: inline-flex;
        align-items: center;
        justify-content: center;
        width: 44px;
        height: 44px;
        padding: 0;
        border: none;
        border-radius: 10px;
        background: linear-gradient(
          135deg,
          var(--gray-light) 0%,
          var(--gray-dark) 100%
        );
        color: var(--white);
        cursor: pointer;
        transition: all 0.25s ease;
        box-shadow: 0 6px 16px var(--shadow-deep),
          inset 0 1px 0 var(--white-transparent-10);
        border: 1px solid var(--accent-transparent-20);
        font-size: 18px;

        &:hover:not(:disabled) {
          transform: translateY(-2px);
          background: var(--accent-transparent-20);
          box-shadow: 0 10px 22px var(--shadow-core),
            0 0 0 1px var(--accent-transparent-30);
          color: var(--accent-light);
        }

        &:active:not(:disabled) {
          transform: translateY(0);
        }

        &:disabled {
          opacity: 0.4;
          cursor: not-allowed;
        }
      }
    }

    /* 主要操作按钮 */
    .send-btn,
    .copy-btn,
    .Alldetail-btn {
      padding: 0 20px;
      height: 44px;
      border: none;
      border-radius: 10px;
      font-weight: 600;
      font-size: 14px;
      cursor: pointer;
      transition: all 0.25s ease;
      letter-spacing: 0.3px;
    }

    .send-btn {
      background: linear-gradient(
        135deg,
        var(--accent) 0%,
        var(--accent-dark) 100%
      );
      color: var(--white);
      box-shadow: 0 8px 24px var(--shadow-core),
        inset 0 1px 0 var(--white-transparent-20);

      &:hover:not(:disabled) {
        transform: translateY(-3px);
        box-shadow: 0 14px 32px var(--shadow-core),
          inset 0 1px 0 var(--white-transparent-30);
        background: linear-gradient(
          135deg,
          var(--accent-light) 0%,
          var(--accent) 100%
        );
      }

      &:disabled {
        opacity: 0.5;
        cursor: not-allowed;
        box-shadow: none;
      }
    }

    .copy-btn {
      background: var(--gray-dark);
      color: var(--white);
      border: 1px solid var(--accent-transparent-30);
      box-shadow: 0 6px 18px var(--shadow-deep);

      &:hover:not(:disabled) {
        background: var(--accent-transparent-20);
        color: var(--accent-light);
        border-color: var(--accent-transparent-50);
        transform: translateY(-2px);
        box-shadow: 0 10px 24px var(--shadow-core);
      }
    }

    .Alldetail-btn {
      display: none;
      background: transparent;
      color: var(--white);
      border: 1px solid var(--accent-transparent-30);

      &:hover {
        background: var(--accent-transparent-10);
        border-color: var(--accent-transparent-50);
      }
    }
  }

  /* ====== 模态框 - 血色档案室 ====== */
  .modal-overlay {
    position: fixed;
    inset: 0;
    background: linear-gradient(
      135deg,
      var(--black-transparent-70) 0%,
      var(--black-transparent-50) 100%
    );
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 2000;
    padding: 20px;
    backdrop-filter: blur(5px);

    .modal-content {
      width: 420px;
      max-width: 100%;
      background: linear-gradient(
        145deg,
        var(--gray-dark) 0%,
        var(--black) 100%
      );
      backdrop-filter: blur(20px);
      border-radius: 16px;
      padding: 28px;
      color: var(--white);
      box-shadow: 0 30px 70px var(--shadow-deep),
        inset 0 1px 0 var(--white-transparent-10);
      border: 1px solid var(--accent-transparent-30);
      animation: modal-appear 0.35s cubic-bezier(0.4, 0, 0.2, 1);
      position: relative;
      overflow: hidden;

      /* 右上角血色剪影装饰 */
      &::after {
        content: "✂";
        position: absolute;
        top: 15px;
        right: 20px;
        font-size: 24px;
        color: var(--accent-transparent-30);
        transform: rotate(45deg);
      }

      h3 {
        margin: 0 0 20px 0;
        font-size: 20px;
        font-weight: 700;
        text-align: center;
        color: var(--accent-light);
        padding-bottom: 12px;
        border-bottom: 1px dashed var(--accent-transparent-30);
        position: relative;

        &::before {
          content: "「";
          position: absolute;
          left: -10px;
          opacity: 0.5;
        }
        &::after {
          content: "」";
          position: absolute;
          right: -10px;
          opacity: 0.5;
        }
      }

      .detail-list {
        list-style: none;
        padding: 0;
        margin: 0 0 24px 0;
        line-height: 1.7;
        font-size: 14.5px;

        li {
          margin-bottom: 10px;
          padding: 8px 12px;
          background: var(--black-transparent-30);
          border-radius: 8px;
          border-left: 3px solid var(--accent-transparent-50);
          display: flex;
          justify-content: space-between;

          &:nth-child(even) {
            background: var(--black-transparent-50);
          }
        }
      }

      .close-btn {
        display: block;
        margin: 10px auto 0;
        padding: 10px 30px;
        background: linear-gradient(
          135deg,
          var(--accent) 0%,
          var(--accent-dark) 100%
        );
        color: var(--white);
        border: none;
        border-radius: 10px;
        cursor: pointer;
        font-weight: 600;
        font-size: 14px;
        box-shadow: 0 8px 24px var(--shadow-core);
        transition: all 0.25s ease;
        letter-spacing: 0.5px;

        &:hover {
          transform: translateY(-2px);
          box-shadow: 0 14px 32px var(--shadow-core),
            0 0 0 1px var(--accent-transparent-30);
        }

        &:focus-visible {
          outline: none;
          box-shadow: 0 0 0 3px var(--accent-transparent-30);
        }
      }
    }
  }

  /* ====== 响应式设计 ====== */
  @media (max-width: 768px) {
    .chat-container {
      width: 100%;
      padding: 16px;
      gap: 16px;

      .carousel1 {
        display: none;
      }
      .carousel2 {
        display: block;
        .carousel-image.active {
          opacity: 0.15;
        }
      }

      .stats-panel {
        display: none;
      }
    }

    .message {
      margin-bottom: 16px;

      .avatar {
        width: 42px;
        height: 42px;
        margin: 0 10px;
      }

      .bubble {
        max-width: 82%;
        padding: 14px 16px;
        font-size: 14.5px;

        &::before {
          display: none;
        }
      }
    }

    .input-area {
      flex-direction: column;
      padding: 14px;
      gap: 10px;

      textarea {
        width: 100%;
        min-height: 48px;
        font-size: 14.5px;
      }

      .btn-group {
        width: 100%;
        justify-content: space-between;

        button {
          flex: 1;
          max-width: 48px;
        }
      }

      .send-btn,
      .copy-btn {
        width: 100%;
        margin-top: 4px;
      }

      .Alldetail-btn {
        display: block;
        width: 100%;
        margin-top: 6px;
      }
    }

    .modal-overlay {
      padding: 16px;

      .modal-content {
        padding: 22px;
        border-radius: 14px;

        h3 {
          font-size: 18px;
          margin-bottom: 16px;
        }

        .detail-list {
          font-size: 14px;

          li {
            padding: 7px 10px;
            flex-direction: column;
          }
        }
      }
    }
  }

  @media (max-width: 480px) {
    .chat-container {
      padding: 12px;
    }

    .message .bubble {
      max-width: 85%;
      padding: 12px 14px;
      border-radius: 14px;
    }
  }
}

/* ====== 动画定义 ====== */
@keyframes pulse {
  0%,
  100% {
    opacity: 0.5;
    transform: scale(1);
  }
  50% {
    opacity: 0.8;
    transform: scale(1.05);
  }
}

@keyframes dot-pulse {
  0%,
  100% {
    opacity: 0.4;
    transform: scale(0.9);
  }
  50% {
    opacity: 1;
    transform: scale(1.1);
  }
}

@keyframes gentle-glow {
  from {
    box-shadow: 0 8px 25px var(--shadow-core),
      inset 0 0 20px var(--accent-transparent-10);
  }
  to {
    box-shadow: 0 8px 30px var(--shadow-core),
      inset 0 0 25px var(--accent-transparent-20);
  }
}

@keyframes modal-appear {
  from {
    opacity: 0;
    transform: translateY(20px) scale(0.98);
  }
  to {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
}
</style>