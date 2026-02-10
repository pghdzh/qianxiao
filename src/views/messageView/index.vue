<template>
  <div class="chisaki-message-board" aria-live="polite">

    <!-- 顶部标题区（熔金牌匾） -->
    <header class="board-header" role="banner">
      <div class="header-plaque">
        <div class="plaque-glow"></div>
        <div class="title-wrap">
          <div class="title-icon">
            <!-- 剪刀与弦线组合图标 -->
            <svg width="22" height="22" viewBox="0 0 24 24" fill="none">
              <path
                d="M7 6L17 18M7 18L17 6"
                stroke="currentColor"
                stroke-width="2.2"
                stroke-linecap="round"
              />
              <circle
                cx="12"
                cy="12"
                r="3.2"
                stroke="currentColor"
                stroke-width="1.8"
              />
              <path
                d="M5 12H9M15 12H19"
                stroke="currentColor"
                stroke-width="1.8"
                stroke-dasharray="3 2"
              />
            </svg>
          </div>
          <h1>弦上留言</h1>
          <span class="title-count">// 共 {{ count }} 条记录</span>
          <p class="subtitle">「于静默中，解析心弦。」</p>
        </div>
        <div class="header-divider">
          <div class="divider-line"></div>
          <div class="divider-scissor">
            <svg width="16" height="16" viewBox="0 0 24 24" fill="none">
              <path
                d="M7 6L17 18M7 18L17 6"
                stroke="var(--gold)"
                stroke-width="2"
                stroke-linecap="round"
              />
            </svg>
          </div>
        </div>
      </div>
    </header>

    <!-- 留言展示区 -->
    <section class="message-list">
      <transition-group name="msg" tag="div" class="message-list-inner">
        <!-- 骨架屏 -->
        <div v-if="loading" class="skeleton-wrap" key="skeleton">
          <div class="skeleton" v-for="i in 3" :key="i">
            <div class="sk-avatar">
              <div class="sk-avatar-inner"></div>
            </div>
            <div class="sk-lines">
              <div class="sk-line short"></div>
              <div class="sk-line"></div>
              <div class="sk-line shorter"></div>
            </div>
          </div>
        </div>

        <!-- 留言卡片 -->
        <div
          v-for="(msg, idx) in messages"
          :key="msg.id || msg._tempId || idx"
          class="message-card"
          :class="{ 'card-hovered': hoveredCardIndex === idx }"
          :data-index="idx"
          tabindex="0"
          role="article"
          :aria-label="`留言来自 ${msg.name || '匿名'}：${msg.content}`"
          @mouseenter="hoveredCardIndex = idx"
          @mouseleave="hoveredCardIndex = null"
          @focus="hoveredCardIndex = idx"
          @blur="hoveredCardIndex = null"
        >
          <!-- 剪刀裁切状缺口 -->
          <div class="card-notch top-left"></div>
          <div class="card-notch top-right"></div>
          <div class="card-notch bottom-left"></div>
          <div class="card-notch bottom-right"></div>

          <!-- 红线扫描效果 -->
          <div
            class="scan-line"
            :class="{ active: hoveredCardIndex === idx }"
          ></div>

          <div class="message-meta">
            <div class="left-meta">
              <!-- 熔金镶边头像 -->
              <div class="name-avatar" :title="msg.name || '匿名'">
                <div class="avatar-border"></div>
                <div class="avatar-inner">
                  {{ getInitial(msg.name) }}
                </div>
                <!-- 红绳标记 -->
                <div class="red-string" aria-hidden="true"></div>
              </div>
              <div class="meta-texts">
                <div class="message-name">{{ msg.name || "匿名" }}</div>
                <div class="message-time">{{ formatTime(msg.created_at) }}</div>
             
              </div>
            </div>
            <!-- 剪刀状态指示器 -->
            <div
              class="scissor-indicator"
              :class="{ hovered: hoveredCardIndex === idx }"
              aria-hidden="true"
            >
              <svg width="18" height="18" viewBox="0 0 24 24" fill="none">
                <path
                  d="M7 6L17 18M7 18L17 6"
                  stroke="currentColor"
                  stroke-width="2.2"
                  stroke-linecap="round"
                />
                <circle
                  cx="12"
                  cy="12"
                  r="2.8"
                  stroke="currentColor"
                  stroke-width="1.6"
                />
              </svg>
            </div>
          </div>

          <div class="message-content-wrapper">
            <p class="message-content">{{ msg.content }}</p>
            <!-- 弦线装饰 -->
            <div class="content-chord" aria-hidden="true"></div>
          </div>

      
        </div>
      </transition-group>
    </section>

    <!-- 底部发送区（熔金锻造台） -->
    <section class="message-form" aria-label="留言输入区">
      <div class="form-container">
        <div class="input-row">
          <div class="input-wrapper">
            <label class="sr-only" for="mb-name">你的昵称</label>
            <input
              id="mb-name"
              v-model="name"
              type="text"
              placeholder="署名（可选）"
              @focus="inputFocused = true"
              @blur="inputFocused = false"
              @keydown.enter.prevent
              :class="{ focused: inputFocused }"
            />
            <div
              class="input-underline"
              :class="{ active: inputFocused }"
            ></div>
          </div>
       
        </div>

        <div class="input-row">
          <div class="input-wrapper">
            <label class="sr-only" for="mb-content">留言内容</label>
            <textarea
              id="mb-content"
              v-model="content"
              ref="textareaRef"
              placeholder="以理性剪裁言语，以情感连接心弦..."
              @input="autoGrow"
              @focus="textareaFocused = true"
              @blur="textareaFocused = false"
              @keydown.ctrl.enter.prevent="submitMessage"
              :class="{ focused: textareaFocused }"
            ></textarea>
            <div
              class="input-underline"
              :class="{ active: textareaFocused }"
            ></div>
            <div
              class="text-counter"
              :class="{ warning: content.length > 280 }"
            >
              {{ content.length }}/280
            </div>
          </div>
        </div>

        <div class="form-actions">
          <div class="actions-left">
            <div class="shortcut-hint">
              <kbd>Ctrl</kbd> + <kbd>Enter</kbd>
              <span class="hint-label">快速剪送</span>
            </div>
           
          </div>
          <button
            class="submit-button"
            @click="submitMessage"
            :disabled="isSending || !content.trim() || content.length > 280"
            :class="{
              sending: isSending,
              enabled: content.trim() && content.length <= 280,
            }"
            aria-live="polite"
          >
            <span class="button-bg"></span>
            <span class="button-content">
              <span class="button-icon" aria-hidden="true">
                <svg width="20" height="20" viewBox="0 0 24 24" fill="none">
                  <path
                    d="M7 6L17 18M7 18L17 6"
                    stroke="currentColor"
                    stroke-width="2.2"
                    stroke-linecap="round"
                  />
                  <circle
                    cx="12"
                    cy="12"
                    r="3.5"
                    stroke="currentColor"
                    stroke-width="1.8"
                  />
                </svg>
              </span>
              <span class="button-text">
                {{ isSending ? "剪送中..." : "剪开发送" }}
              </span>
            </span>
            <span class="button-pulse" :class="{ active: isSending }"></span>
          </button>
        </div>
      </div>
    </section>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, nextTick, onBeforeUnmount } from "vue";
import { getMessageList, createMessage } from "@/api/modules/message";

// 数据状态
const messages = ref<any[]>([]);
const count = ref(0);
const name = ref(localStorage.getItem("message_name") || "");
const content = ref("");
const loading = ref(true);
const isSending = ref(false);
const inputFocused = ref(false);
const textareaFocused = ref(false);
const textareaRef = ref<HTMLTextAreaElement | null>(null);
const hoveredCardIndex = ref<number | null>(null);

// 获取留言列表
const fetchMessages = async () => {
  loading.value = true;
  try {
    const res = await getMessageList({ page: 1, pageSize: 999 });
    messages.value = res.data || [];
    count.value = res.pagination.total;
    await nextTick();
  } catch (err) {
    console.error(err);
  } finally {
    loading.value = false;
  }
};

// 提交留言
const submitMessage = async () => {
  if (!content.value.trim() || isSending.value || content.value.length > 280)
    return;
  isSending.value = true;
  const payload = { name: name.value || "匿名", content: content.value };
  try {
    localStorage.setItem("message_name", name.value);
    content.value = "";
    await nextTick();
    await createMessage(payload);
    await fetchMessages();
  } catch (err) {
    console.error(err);
  } finally {
    isSending.value = false;
  }
};

// 时间格式化
const formatTime = (time: string, short: boolean = false) => {
  if (!time) return short ? "--:--" : "";
  const d = new Date(time);
  if (short) {
    const hh = String(d.getHours()).padStart(2, "0");
    const mm = String(d.getMinutes()).padStart(2, "0");
    return `${hh}:${mm}`;
  }
  const y = d.getFullYear();
  const m = String(d.getMonth() + 1).padStart(2, "0");
  const day = String(d.getDate()).padStart(2, "0");
  const hh = String(d.getHours()).padStart(2, "0");
  const mm = String(d.getMinutes()).padStart(2, "0");
  return `${y}.${m}.${day} ${hh}:${mm}`;
};

// 获取姓名首字母
const getInitial = (n?: string) => {
  if (!n) return "匿";
  const firstChar = n.trim().charAt(0);
  // 判断是否为中文字符
  return /[\u4e00-\u9fa5]/.test(firstChar)
    ? firstChar
    : firstChar.toUpperCase();
};

// 文本框自动增高
const autoGrow = () => {
  const ta = textareaRef.value;
  if (!ta) return;
  requestAnimationFrame(() => {
    ta.style.height = "auto";
    const newHeight = Math.min(ta.scrollHeight, 200);
    ta.style.height = `${newHeight}px`;
  });
};



onMounted(() => {
  fetchMessages();
  nextTick(() => {
    autoGrow();
  });
});

onBeforeUnmount(() => {
  // 清理工作
});
</script>

<style scoped lang="scss">
/* 千咲血色暗夜色板（基于用户输入） */
.chisaki-message-board {
  /* CSS变量定义 */
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

  /* 衍生变量 */
  --card-bg: rgba(20, 10, 15, 0.68);
  --card-border: rgba(198, 40, 40, 0.12);
  --card-glow: rgba(244, 67, 54, 0.08);
  --text-primary: var(--white);
  --text-secondary: rgba(245, 245, 245, 0.72);
  --text-tertiary: rgba(245, 245, 245, 0.5);

  position: relative;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  background: var(--deep-bg);
  color: var(--text-primary);
  font-family: "Noto Sans SC", "Segoe UI", -apple-system, system-ui, sans-serif;
  overflow-x: hidden;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  padding-top: 80px;

  /* 顶部标题区 */
  .board-header {
    position: relative;
    z-index: 10;
    padding: 30px 20px 20px;
    max-width: 1000px;
    margin: 0 auto;
    width: 100%;

    .header-plaque {
      position: relative;
      background: linear-gradient(
        145deg,
        rgba(30, 10, 15, 0.85) 0%,
        rgba(20, 5, 10, 0.92) 100%
      );
      border-radius: 18px;
      padding: 22px 28px;
      border: 1px solid rgba(198, 40, 40, 0.2);
      box-shadow: 0 20px 60px var(--shadow-deep),
        inset 0 1px 0 rgba(255, 215, 0, 0.1);
      backdrop-filter: blur(12px) saturate(1.8);
      overflow: hidden;

      .plaque-glow {
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        height: 1px;
        background: linear-gradient(
          90deg,
          transparent 0%,
          var(--gold) 50%,
          transparent 100%
        );
        filter: blur(1px);
        opacity: 0.6;
      }

      .title-wrap {
        display: flex;
        align-items: center;
        flex-wrap: wrap;
        gap: 14px;
        position: relative;
        z-index: 2;

        .title-icon {
          color: var(--accent-light);
          display: flex;
          align-items: center;
          animation: icon-pulse 4s ease-in-out infinite;
        }

        h1 {
          margin: 0;
          font-size: 28px;
          font-weight: 800;
          background: linear-gradient(
            135deg,
            var(--white) 0%,
            var(--accent-light) 30%,
            var(--gold) 100%
          );
          -webkit-background-clip: text;
          -webkit-text-fill-color: transparent;
          letter-spacing: 0.5px;
          text-shadow: 0 4px 20px rgba(198, 40, 40, 0.2);
        }

        .title-count {
          color: var(--text-secondary);
          font-size: 15px;
          font-weight: 500;
          font-family: "Consolas", "Monaco", monospace;
          margin-left: 4px;
          opacity: 0.9;
        }

        .subtitle {
          margin: 4px 0 0 0;
          width: 100%;
          color: var(--text-tertiary);
          font-size: 14px;
          font-weight: 500;
          font-style: italic;
          letter-spacing: 0.3px;
        }
      }

      .header-divider {
        position: relative;
        margin-top: 18px;
        height: 1px;

        .divider-line {
          height: 1px;
          background: linear-gradient(
            90deg,
            transparent 0%,
            rgba(198, 40, 40, 0.4) 20%,
            rgba(198, 40, 40, 0.4) 80%,
            transparent 100%
          );
        }

        .divider-scissor {
          position: absolute;
          top: 50%;
          left: 50%;
          transform: translate(-50%, -50%);
          background: rgba(10, 10, 20, 0.9);
          padding: 0 12px;
          display: flex;
          align-items: center;
          justify-content: center;
        }
      }
    }
  }

  /* 留言列表区 */
  .message-list {
    position: relative;
    z-index: 5;
    flex: 1;
    padding: 0 20px 200px;
    max-width: 1000px;
    width: 100%;
    margin: 0 auto;

    .message-list-inner {
      display: flex;
      flex-direction: column;
      gap: 20px;
      padding-bottom: 160px;
    }

    /* 骨架屏 */
    .skeleton-wrap {
      display: flex;
      flex-direction: column;
      gap: 18px;

      .skeleton {
        display: flex;
        gap: 16px;
        align-items: center;
        padding: 20px;
        background: rgba(25, 12, 18, 0.6);
        border-radius: 16px;
        border: 1px solid rgba(198, 40, 40, 0.08);

        .sk-avatar {
          position: relative;
          width: 56px;
          height: 56px;
          border-radius: 14px;
          overflow: hidden;
          flex-shrink: 0;

          .sk-avatar-inner {
            width: 100%;
            height: 100%;
            background: linear-gradient(
              90deg,
              rgba(198, 40, 40, 0.15) 25%,
              rgba(244, 67, 54, 0.1) 50%,
              rgba(198, 40, 40, 0.15) 75%
            );
            background-size: 200% 100%;
            animation: skeleton-shimmer 1.8s infinite;
          }
        }

        .sk-lines {
          flex: 1;

          .sk-line {
            height: 12px;
            border-radius: 6px;
            background: rgba(255, 255, 255, 0.05);
            margin-bottom: 10px;

            &.short {
              width: 60%;
            }

            &.shorter {
              width: 40%;
              margin-top: 14px;
            }
          }
        }
      }
    }
  }

  /* 留言卡片 */
  .message-card {
    position: relative;
    background: var(--card-bg);
    border-radius: 16px;
    padding: 24px;
    border: 1px solid var(--card-border);
    box-shadow: 0 12px 40px var(--shadow-deep), inset 0 1px 0 var(--card-glow);
    transition: all 0.4s cubic-bezier(0.2, 0.9, 0.1, 1);
    overflow: hidden;

    /* 剪刀裁切状缺口 */
    .card-notch {
      position: absolute;
      width: 16px;
      height: 16px;
      border: 2px solid var(--accent);
      opacity: 0.4;

      &.top-left {
        top: -1px;
        left: -1px;
        border-right: none;
        border-bottom: none;
        border-radius: 8px 0 0 0;
      }
      &.top-right {
        top: -1px;
        right: -1px;
        border-left: none;
        border-bottom: none;
        border-radius: 0 8px 0 0;
      }
      &.bottom-left {
        bottom: -1px;
        left: -1px;
        border-right: none;
        border-top: none;
        border-radius: 0 0 0 8px;
      }
      &.bottom-right {
        bottom: -1px;
        right: -1px;
        border-left: none;
        border-top: none;
        border-radius: 0 0 8px 0;
      }
    }

    /* 红线扫描效果 */
    .scan-line {
      position: absolute;
      top: 0;
      left: -100%;
      width: 100%;
      height: 2px;
      background: linear-gradient(
        90deg,
        transparent 0%,
        var(--accent-light) 50%,
        transparent 100%
      );
      filter: blur(1px);
      opacity: 0;
      transition: opacity 0.3s;
      z-index: 1;

      &.active {
        opacity: 0.8;
        animation: scan 1.2s cubic-bezier(0.4, 0, 0.2, 1) forwards;
      }
    }

    &:hover,
    &.card-hovered {
      transform: translateY(-8px);
      border-color: rgba(198, 40, 40, 0.3);
      box-shadow: 0 24px 70px rgba(10, 10, 20, 0.9),
        0 0 0 1px rgba(244, 67, 54, 0.15), inset 0 1px 0 rgba(255, 215, 0, 0.05);

      .scissor-indicator {
        color: var(--accent-light);
        transform: rotate(15deg) scale(1.2);

        &.hovered {
          animation: scissor-cut 0.6s ease-out;
        }
      }
    }

    .message-meta {
      display: flex;
      justify-content: space-between;
      align-items: flex-start;
      margin-bottom: 18px;
      position: relative;
      z-index: 2;

      .left-meta {
        display: flex;
        gap: 16px;
        align-items: center;
        flex: 1;

        .name-avatar {
          position: relative;
          width: 60px;
          height: 60px;
          flex-shrink: 0;

          .avatar-border {
            position: absolute;
            inset: 0;
            border-radius: 16px;
            background: linear-gradient(
              135deg,
              var(--accent) 0%,
              var(--gold) 100%
            );
            padding: 2px;
            -webkit-mask: linear-gradient(#fff 0 0) content-box,
              linear-gradient(#fff 0 0);
            -webkit-mask-composite: xor;
            mask-composite: exclude;
            opacity: 0.7;
          }

          .avatar-inner {
            position: absolute;
            inset: 2px;
            border-radius: 14px;
            background: linear-gradient(
              135deg,
              rgba(30, 10, 15, 0.9) 0%,
              rgba(20, 5, 10, 0.95) 100%
            );
            display: flex;
            align-items: center;
            justify-content: center;
            font-weight: 900;
            color: var(--accent-light);
            font-size: 24px;
            box-shadow: inset 0 -4px 12px rgba(0, 0, 0, 0.4);
          }

          .red-string {
            position: absolute;
            top: -4px;
            right: -4px;
            width: 12px;
            height: 12px;
            border-radius: 50%;
            background: radial-gradient(
              circle at 30% 30%,
              var(--accent-light) 0%,
              transparent 70%
            );
            filter: blur(1.5px);
            z-index: 3;
          }
        }

        .meta-texts {
          flex: 1;

          .message-name {
            font-size: 18px;
            font-weight: 700;
            color: var(--white);
            letter-spacing: 0.3px;
            margin-bottom: 4px;
          }

          .message-time {
            font-size: 14px;
            color: var(--text-secondary);
            font-family: "Consolas", "Monaco", monospace;
            margin-bottom: 2px;
          }

       
        }
      }

      .scissor-indicator {
        color: rgba(198, 40, 40, 0.4);
        transition: all 0.3s cubic-bezier(0.2, 0.9, 0.1, 1);
        flex-shrink: 0;
        margin-left: 12px;
      }
    }

    .message-content-wrapper {
      position: relative;
      margin-bottom: 16px;
      z-index: 2;

      .message-content {
        font-size: 16px;
        line-height: 1.7;
        color: rgba(245, 245, 245, 0.95);
        margin: 0;
        padding-right: 20px;
        white-space: pre-wrap;
        word-break: break-word;
      }

      .content-chord {
        position: absolute;
        right: 0;
        top: 0;
        bottom: 0;
        width: 1px;
        background: linear-gradient(
          to bottom,
          transparent 0%,
          var(--accent) 20%,
          var(--accent) 80%,
          transparent 100%
        );
        opacity: 0.3;
      }
    }

  
  }

  /* 底部发送区 */
  .message-form {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    z-index: 20;
    padding: 20px;
    background: linear-gradient(
      to top,
      rgba(10, 10, 20, 0.98) 0%,
      rgba(10, 10, 20, 0.92) 30%,
      transparent 100%
    );
    backdrop-filter: blur(10px);

    .form-container {
      max-width: 1000px;
      margin: 0 auto;
      background: linear-gradient(
        145deg,
        rgba(25, 12, 18, 0.85) 0%,
        rgba(20, 8, 14, 0.92) 100%
      );
      border-radius: 18px;
      padding: 22px;
      border: 1px solid rgba(198, 40, 40, 0.2);
      box-shadow: 0 20px 60px var(--shadow-deep),
        inset 0 1px 0 rgba(255, 215, 0, 0.08);

      .input-row {
        margin-bottom: 18px;
        position: relative;

        &:last-of-type {
          margin-bottom: 22px;
        }

        .input-wrapper {
          position: relative;

          input,
          textarea {
            width: 100%;
            padding: 16px 18px;
            border-radius: 14px;
            border: 1px solid rgba(198, 40, 40, 0.15);
            background: rgba(15, 6, 10, 0.7);
            color: var(--text-primary);
            font-size: 16px;
            font-family: inherit;
            transition: all 0.3s cubic-bezier(0.2, 0.9, 0.1, 1);
            resize: none;

            &:focus {
              outline: none;
              border-color: var(--accent-light);
              background: rgba(20, 8, 12, 0.85);
              box-shadow: 0 0 0 3px rgba(198, 40, 40, 0.1),
                inset 0 2px 8px rgba(0, 0, 0, 0.2);

              &::placeholder {
                color: rgba(245, 245, 245, 0.4);
              }
            }

            &::placeholder {
              color: rgba(245, 245, 245, 0.5);
              transition: color 0.3s;
            }

            &.focused {
              border-color: var(--accent-light);
            }
          }

          textarea {
            min-height: 80px;
            line-height: 1.6;
          }

          .input-underline {
            position: absolute;
            bottom: 0;
            left: 18px;
            right: 18px;
            height: 2px;
            background: linear-gradient(
              90deg,
              transparent 0%,
              var(--accent-light) 50%,
              transparent 100%
            );
            transform: scaleX(0);
            transition: transform 0.4s cubic-bezier(0.2, 0.9, 0.1, 1);
            filter: blur(1px);

            &.active {
              transform: scaleX(1);
            }
          }

          .text-counter {
            position: absolute;
            bottom: -22px;
            right: 4px;
            font-size: 13px;
            color: var(--text-tertiary);
            font-family: "Consolas", "Monaco", monospace;
            transition: color 0.3s;

            &.warning {
              color: var(--accent-light);
              font-weight: 600;
            }
          }
        }

  
      }

      .form-actions {
        display: flex;
        align-items: center;
        justify-content: space-between;

        .actions-left {
          display: flex;
          align-items: center;
          gap: 20px;

          .shortcut-hint {
            display: flex;
            align-items: center;
            gap: 8px;
            font-size: 14px;
            color: var(--text-secondary);

            kbd {
              padding: 4px 8px;
              background: rgba(198, 40, 40, 0.15);
              border: 1px solid rgba(198, 40, 40, 0.3);
              border-radius: 8px;
              font-size: 12px;
              font-weight: 600;
              color: var(--accent-light);
              font-family: "Consolas", "Monaco", monospace;
              min-width: 32px;
              text-align: center;
            }

            .hint-label {
              margin-left: 4px;
              opacity: 0.9;
            }
          }

        }

        .submit-button {
          position: relative;
          padding: 0;
          border: none;
          background: transparent;
          cursor: pointer;
          min-width: 160px;
          height: 52px;
          overflow: hidden;
          border-radius: 14px;

          .button-bg {
            position: absolute;
            inset: 0;
            background: linear-gradient(
              135deg,
              rgba(198, 40, 40, 0.9) 0%,
              rgba(142, 0, 0, 0.8) 100%
            );
            border-radius: 14px;
            transition: all 0.3s cubic-bezier(0.2, 0.9, 0.1, 1);
          }

          .button-content {
            position: relative;
            z-index: 2;
            display: flex;
            align-items: center;
            justify-content: center;
            gap: 10px;
            height: 100%;
            padding: 0 24px;
            color: var(--white);
            font-weight: 700;
            font-size: 16px;
            letter-spacing: 0.5px;

            .button-icon {
              transition: transform 0.3s;
            }
          }

          .button-pulse {
            position: absolute;
            inset: 0;
            border-radius: 14px;
            box-shadow: 0 0 0 0 rgba(244, 67, 54, 0.7);
            opacity: 0;
            z-index: 1;

            &.active {
              animation: button-pulse 1.2s infinite;
            }
          }

          &:hover:not(:disabled) {
            .button-bg {
              background: linear-gradient(
                135deg,
                rgba(244, 67, 54, 0.95) 0%,
                rgba(198, 40, 40, 0.9) 100%
              );
              transform: scale(1.02);
            }

            .button-icon {
              transform: rotate(15deg);
            }
          }

          &:active:not(:disabled) {
            transform: translateY(2px);

            .button-bg {
              transform: scale(0.98);
            }
          }

          &.sending {
            .button-icon {
              animation: scissor-cut 0.8s infinite;
            }
          }

          &:disabled {
            cursor: not-allowed;
            opacity: 0.5;

            .button-bg {
              background: rgba(42, 42, 42, 0.7);
            }

            .button-content {
              color: rgba(245, 245, 245, 0.6);
            }
          }

          &.enabled {
            .button-bg {
              box-shadow: 0 8px 32px rgba(198, 40, 40, 0.3),
                inset 0 1px 0 rgba(255, 255, 255, 0.1);
            }
          }
        }
      }
    }
  }

  /* 动画定义 */
  @keyframes blood-float-1 {
    0% {
      transform: translateY(100vh) rotate(0deg);
      opacity: 0;
    }
    10% {
      opacity: 0.7;
    }
    90% {
      opacity: 0.7;
    }
    100% {
      transform: translateY(-100px) rotate(360deg);
      opacity: 0;
    }
  }

  @keyframes blood-float-2 {
    0% {
      transform: translate(0, 100vh) rotate(0deg);
      opacity: 0;
    }
    10% {
      opacity: 0.5;
    }
    90% {
      opacity: 0.5;
    }
    100% {
      transform: translate(100px, -100px) rotate(180deg);
      opacity: 0;
    }
  }

  @keyframes blood-float-3 {
    0%,
    100% {
      transform: translateY(0) rotate(0deg);
      opacity: 0.3;
    }
    50% {
      transform: translateY(-40px) rotate(180deg);
      opacity: 0.6;
    }
  }

  @keyframes scan {
    0% {
      left: -100%;
      opacity: 0;
    }
    10% {
      opacity: 0.8;
    }
    90% {
      opacity: 0.8;
    }
    100% {
      left: 100%;
      opacity: 0;
    }
  }

  @keyframes scissor-cut {
    0%,
    100% {
      transform: rotate(0deg) scale(1.2);
    }
    50% {
      transform: rotate(30deg) scale(1.3);
    }
  }

  @keyframes icon-pulse {
    0%,
    100% {
      opacity: 1;
      transform: scale(1);
    }
    50% {
      opacity: 0.7;
      transform: scale(1.05);
    }
  }

  @keyframes skeleton-shimmer {
    0% {
      background-position: -200% 0;
    }
    100% {
      background-position: 200% 0;
    }
  }

  @keyframes button-pulse {
    0% {
      box-shadow: 0 0 0 0 rgba(244, 67, 54, 0.7);
      opacity: 1;
    }
    70% {
      box-shadow: 0 0 0 12px rgba(244, 67, 54, 0);
      opacity: 0;
    }
    100% {
      box-shadow: 0 0 0 0 rgba(244, 67, 54, 0);
      opacity: 0;
    }
  }

  @keyframes warning-pulse {
    0%,
    100% {
      opacity: 1;
    }
    50% {
      opacity: 0.6;
    }
  }

  /* 响应式设计 */
  @media (max-width: 768px) {
    .board-header {
      padding: 20px 16px 16px;

      .header-plaque {
        padding: 18px 20px;

        .title-wrap {
          gap: 10px;

          h1 {
            font-size: 24px;
          }

          .title-count {
            font-size: 14px;
          }

          .subtitle {
            font-size: 13px;
          }
        }
      }
    }

    .message-list {
      padding: 0 16px 180px;

      .message-card {
        padding: 20px 18px;

        .message-meta {
          .left-meta {
            .name-avatar {
              width: 52px;
              height: 52px;

              .avatar-inner {
                font-size: 22px;
              }
            }

            .meta-texts {
              .message-name {
                font-size: 17px;
              }
            }
          }
        }
      }
    }

    .message-form {
      padding: 16px;

      .form-container {
        padding: 18px;

        .input-row {
          .input-wrapper {
            input,
            textarea {
              padding: 14px 16px;
              font-size: 15px;
            }
          }
        }

        .form-actions {
          flex-direction: column;
          gap: 16px;
          align-items: stretch;

          .actions-left {
            justify-content: space-between;
          }

          .submit-button {
            min-width: 100%;
          }
        }
      }
    }
  }
}

/* 无障碍隐藏 */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
</style>