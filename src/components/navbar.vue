<template>
  <header class="qianxiao-header">
    <!-- 标题区域 -->
    <div class="title-section">
      <h1 class="title">
        <span class="title-text">星炬档案 · 千咲</span>
        <span class="title-sub">解弦之眼观测站</span>
      </h1>

      <!-- 在线人数展示 -->
      <div class="online-count" v-if="onlineCount !== null">
        <div class="count-icon">✦</div>
        <div class="count-text">
          <span class="count-label">共鸣者在线</span>
          <span class="count-value">{{ onlineCount }}人</span>
        </div>
      </div>
    </div>

    <!-- 音乐播放控制 -->
    <div class="music-control">
      <button
        class="music-btn"
        @click="toggleMusic"
        :class="{ playing: isPlaying }"
      >
        <span class="music-icon">{{ isPlaying ? "♫" : "♪" }}</span>
        <span class="music-text">{{ isPlaying ? "结线之花" : "播放BGM" }}</span>
        <div class="music-wave" v-if="isPlaying">
          <span class="wave-bar"></span>
          <span class="wave-bar"></span>
          <span class="wave-bar"></span>
          <span class="wave-bar"></span>
          <span class="wave-bar"></span>
        </div>
        <div class="scissor-cut" :class="{ active: isPlaying }"></div>
      </button>
      <audio ref="bgmPlayer" loop>
        <source :src="musicUrl" type="audio/mpeg" />
        您的浏览器不支持音频元素
      </audio>
    </div>

    <!-- 移动端汉堡按钮 -->
    <button
      class="hamburger"
      @click="toggleMobileNav"
      aria-label="Toggle navigation"
    >
      <span :class="{ open: mobileNavOpen }"></span>
      <span :class="{ open: mobileNavOpen }"></span>
      <span :class="{ open: mobileNavOpen }"></span>
    </button>

    <!-- PC端导航 -->
    <nav class="nav-links-pc">
      <RouterLink
        v-for="item in mainNavItems"
        :key="item.name"
        :to="item.path"
        class="nav-item"
        active-class="active-link"
      >
        <span class="nav-text">{{ item.name }}</span>
        <span class="nav-line"></span>
      </RouterLink>

      <!-- 下拉菜单 (AI对话之后的项目) -->
      <div
        class="dropdown"
        @mouseenter="dropdownOpen = true"
        @mouseleave="dropdownOpen = false"
      >
        <button class="dropdown-toggle">
          <span class="dropdown-text">更多功能</span>
          <span class="dropdown-icon">✂</span>
          <div class="scissor-effect" :class="{ open: dropdownOpen }"></div>
        </button>

        <div class="dropdown-menu" :class="{ open: dropdownOpen }">
          <RouterLink
            v-for="item in dropdownNavItems"
            :key="item.name"
            :to="item.path"
            class="dropdown-item"
            @click="dropdownOpen = false"
          >
            <span class="dropdown-item-text">{{ item.name }}</span>
            <span class="dropdown-item-line"></span>
          </RouterLink>

          <!-- 外部链接 -->
          <a
            href="https://slty.site/#/redirector"
            target="_blank"
            rel="noopener"
            class="dropdown-item external"
            @click="dropdownOpen = false"
          >
            <span class="dropdown-item-text">霜落映界</span>
            <span class="external-icon">↗</span>
          </a>
        </div>
      </div>
    </nav>

    <!-- 移动端导航 -->
    <nav :class="['nav-links-mobile', { 'mobile-open': mobileNavOpen }]">
      <RouterLink
        v-for="item in navItems"
        :key="item.name"
        :to="item.path"
        class="nav-item"
        active-class="active-link"
        @click="mobileNavOpen = false"
      >
        <span class="nav-text">{{ item.name }}</span>
        <span class="nav-line"></span>
      </RouterLink>

      <a
        href="https://slty.site/#/redirector"
        target="_blank"
        rel="noopener"
        class="nav-item external"
        @click="mobileNavOpen = false"
      >
        <span class="nav-text">霜落映界</span>
        <span class="external-icon">↗</span>
      </a>
    </nav>

    <!-- 动态剪裁效果 -->
    <div class="scissor-lines">
      <div class="scissor-line line-1"></div>
      <div class="scissor-line line-2"></div>
      <div class="scissor-line line-3"></div>
    </div>

    <!-- 背景动效：弦线网格 -->
    <div class="string-grid"></div>
  </header>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, computed } from "vue";
import { io } from "socket.io-client";
const musicUrl = import.meta.env.VITE_API_BASE_URL + "/music/结线之花.mp3";
// 导航项数据
const navItems = [
  { name: "星炬首页", path: "/" },
  { name: "角色解析", path: "/timeLine" },
  { name: "观测记录", path: "/message" },
  { name: "剪影图集", path: "/gallery" },
  { name: "弦音对话", path: "/talk" },
  { name: "资源解析", path: "/resources" },
  // { name: "语音共鸣", path: "/voice" },
  { name: "弦音曲库", path: "/music" },
  { name: "学院百科", path: "/wiki" },
  // { name: "鸣谢名单", path: "/thanks" },
];

// 分割导航项：AI对话之前为主导航，之后为下拉菜单
const mainNavItems = computed(() => navItems.slice(0, 5)); // 前6项
const dropdownNavItems = computed(() => navItems.slice(5)); // 后4项

const mobileNavOpen = ref(false);
const dropdownOpen = ref(false);

function toggleMobileNav() {
  mobileNavOpen.value = !mobileNavOpen.value;
  // 移动端打开时关闭PC下拉菜单
  if (mobileNavOpen.value) {
    dropdownOpen.value = false;
  }
}

// 点击页面其他区域关闭下拉菜单
const closeDropdown = (event: MouseEvent) => {
  const target = event.target as HTMLElement;
  if (!target.closest(".dropdown")) {
    dropdownOpen.value = false;
  }
};

onMounted(() => {
  document.addEventListener("click", closeDropdown);
});

onBeforeUnmount(() => {
  document.removeEventListener("click", closeDropdown);
});

// 在线人数功能
const siteId = "qianxiao";
const onlineCount = ref<number | null>(null);
const socket: any = io(import.meta.env.VITE_API_BASE_URL, {
  query: { siteId },
});

onMounted(() => {
  socket.on("onlineCount", (count: number) => {
    onlineCount.value = count;
  });
});

onBeforeUnmount(() => {
  socket.disconnect();
});

// BGM播放控制
const bgmPlayer = ref<HTMLAudioElement | null>(null);
const isPlaying = ref(false);

const toggleMusic = () => {
  if (!bgmPlayer.value) return;

  if (isPlaying.value) {
    bgmPlayer.value.pause();
  } else {
    // 尝试播放，如果失败可能是浏览器自动播放策略阻止
    bgmPlayer.value.play().catch((error) => {
      console.log("播放失败:", error);
      // 可以在这里添加用户交互提示
    });
  }

  isPlaying.value = !isPlaying.value;
};

// 监听音频加载状态
onMounted(() => {
  if (bgmPlayer.value) {
    bgmPlayer.value.addEventListener("play", () => {
      isPlaying.value = true;
    });

    bgmPlayer.value.addEventListener("pause", () => {
      isPlaying.value = false;
    });

    bgmPlayer.value.addEventListener("ended", () => {
      isPlaying.value = false;
    });
  }
});
</script>

<style scoped lang="scss">
/* 千咲风格设计系统 - 红黑剪裁 · 星炬学院 */
.qianxiao-header {
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

  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: 1000;
  height: 80px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 40px;
  background: var(--deep-bg);
  backdrop-filter: blur(10px) saturate(1.8);
  box-shadow: 0 10px 30px var(--shadow-deep),
    0 0 0 1px rgba(198, 40, 40, 0.1) inset,
    0 0 20px rgba(198, 40, 40, 0.05) inset;
  border-bottom: 1px solid rgba(245, 245, 245, 0.05);
  overflow: visible;
  animation: header-appear 0.6s ease-out both;

  /* 网格背景纹理 - 学院风 */
  &::before {
    content: "";
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-image: linear-gradient(
        rgba(198, 40, 40, 0.03) 1px,
        transparent 1px
      ),
      linear-gradient(90deg, rgba(198, 40, 40, 0.03) 1px, transparent 1px);
    background-size: 30px 30px;
    pointer-events: none;
    opacity: 0.4;
  }

  /* 右侧剪刀光效 */
  &::after {
    content: "";
    position: absolute;
    right: 5%;
    top: 0;
    width: 200px;
    height: 100%;
    background: radial-gradient(
      ellipse at center,
      rgba(244, 67, 54, 0.1) 0%,
      rgba(244, 67, 54, 0.05) 30%,
      transparent 70%
    );
    pointer-events: none;
    filter: blur(15px);
    animation: scissor-glow 4s ease-in-out infinite;
  }
}

/* 标题区域 */
.title-section {
  display: flex;
  align-items: center;
  gap: 30px;
  position: relative;

  .title {
    display: flex;
    flex-direction: column;
    font-family: "Noto Serif SC", "Source Han Serif", serif;
    line-height: 1.2;
    position: relative;

    .title-text {
      font-size: 24px;
      font-weight: 800;
      color: var(--white);
      letter-spacing: 1px;
      text-shadow: 0 2px 4px var(--shadow-deep), 0 0 20px rgba(198, 40, 40, 0.3);
      position: relative;

      &::before {
        content: "✧";
        position: absolute;
        left: -25px;
        top: 50%;
        transform: translateY(-50%);
        color: var(--accent);
        font-size: 18px;
        animation: star-twinkle 3s infinite;
      }
    }

    .title-sub {
      font-size: 12px;
      font-weight: 400;
      color: rgba(245, 245, 245, 0.7);
      letter-spacing: 2px;
      margin-top: 4px;
      position: relative;
      padding-left: 20px;

      &::before {
        content: "";
        position: absolute;
        left: 0;
        top: 50%;
        width: 15px;
        height: 1px;
        background: linear-gradient(90deg, var(--accent), transparent);
      }
    }
  }
}

/* 在线人数 */
.online-count {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 8px 16px;
  background: rgba(26, 26, 26, 0.7);
  border-radius: 20px;
  border: 1px solid rgba(198, 40, 40, 0.2);
  backdrop-filter: blur(5px);
  position: relative;
  overflow: hidden;
  transition: all 0.3s ease;

  &::before {
    content: "";
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: linear-gradient(
      90deg,
      transparent,
      rgba(198, 40, 40, 0.1),
      transparent
    );
    transform: translateX(-100%);
  }

  &:hover {
    border-color: rgba(198, 40, 40, 0.4);
    box-shadow: 0 0 15px rgba(198, 40, 40, 0.2);

    &::before {
      animation: shine 1.5s ease;
    }
  }

  .count-icon {
    color: var(--accent);
    font-size: 16px;
    animation: pulse 2s infinite;
  }

  .count-text {
    display: flex;
    flex-direction: column;
    line-height: 1.2;
  }

  .count-label {
    font-size: 10px;
    color: rgba(245, 245, 245, 0.6);
    font-weight: 300;
    letter-spacing: 0.5px;
  }

  .count-value {
    font-size: 14px;
    font-weight: 700;
    color: var(--accent-light);
    text-shadow: 0 0 10px rgba(244, 67, 54, 0.5);
  }
}

/* 音乐播放控制 */
.music-control {
  z-index: 10;

  .music-btn {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 10px 20px;
    background: rgba(20, 20, 20, 0.8);
    border: 1px solid rgba(198, 40, 40, 0.3);
    border-radius: 30px;
    color: rgba(245, 245, 245, 0.9);
    font-family: "Noto Sans SC", sans-serif;
    font-size: 14px;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.3s ease;
    position: relative;
    overflow: hidden;

    &:hover {
      background: rgba(30, 30, 30, 0.9);
      border-color: var(--accent);
      color: var(--white);
      box-shadow: 0 0 20px rgba(198, 40, 40, 0.3);

      .music-icon {
        transform: scale(1.2);
      }
    }

    &.playing {
      background: rgba(198, 40, 40, 0.1);
      border-color: var(--accent);
      box-shadow: 0 0 25px rgba(244, 67, 54, 0.2);

      .music-icon {
        animation: music-note 1.5s infinite;
      }
    }

    .music-icon {
      font-size: 18px;
      transition: transform 0.3s ease;
    }

    .music-text {
      position: relative;
      z-index: 2;
    }

    .music-wave {
      display: flex;
      align-items: center;
      gap: 2px;
      height: 16px;
      margin-left: 4px;

      .wave-bar {
        width: 3px;
        height: 8px;
        background: var(--accent-light);
        border-radius: 2px;
        animation: wave-animation 1.2s ease-in-out infinite;

        &:nth-child(1) {
          animation-delay: 0s;
        }
        &:nth-child(2) {
          animation-delay: 0.1s;
        }
        &:nth-child(3) {
          animation-delay: 0.2s;
          height: 12px;
        }
        &:nth-child(4) {
          animation-delay: 0.3s;
        }
        &:nth-child(5) {
          animation-delay: 0.4s;
        }
      }
    }

    .scissor-cut {
      position: absolute;
      top: 0;
      left: -100%;
      width: 100%;
      height: 100%;
      background: linear-gradient(
        90deg,
        transparent,
        rgba(198, 40, 40, 0.3),
        transparent
      );
      transition: left 0.8s ease;
      z-index: 1;

      &.active {
        left: 100%;
      }
    }
  }

  audio {
    display: none;
  }
}

/* PC端导航 */
.nav-links-pc {
  display: flex;
  align-items: center;
  gap: 8px;
  position: relative;

  .nav-item {
    position: relative;
    padding: 12px 16px;
    text-decoration: none;
    color: rgba(245, 245, 245, 0.8);
    font-family: "Noto Sans SC", sans-serif;
    font-size: 14px;
    font-weight: 500;
    letter-spacing: 0.5px;
    transition: all 0.3s ease;
    overflow: hidden;

    .nav-text {
      position: relative;
      z-index: 2;
      display: inline-block;
      transition: transform 0.3s ease;
    }

    .nav-line {
      position: absolute;
      bottom: 0;
      left: 50%;
      transform: translateX(-50%);
      width: 0;
      height: 2px;
      background: linear-gradient(
        90deg,
        transparent,
        var(--accent),
        transparent
      );
      transition: width 0.4s ease;
      opacity: 0;
    }

    &:hover {
      color: var(--white);

      .nav-text {
        transform: translateY(-2px);
      }

      .nav-line {
        width: 80%;
        opacity: 1;
        animation: line-glow 1.5s infinite;
      }
    }

    &.active-link {
      color: var(--accent-light);
      font-weight: 600;

      .nav-line {
        width: 100%;
        opacity: 1;
        background: linear-gradient(
          90deg,
          transparent,
          var(--accent-light),
          var(--accent),
          transparent
        );
        animation: line-glow 1.5s infinite;
        box-shadow: 0 0 10px rgba(244, 67, 54, 0.5);
      }

      .nav-text::after {
        content: "✂";
        position: absolute;
        right: -18px;
        top: 0;
        font-size: 12px;
        color: var(--accent-light);
        animation: scissor-float 2s infinite;
      }
    }
  }
}

/* 下拉菜单 */
.dropdown {
  position: relative;
  margin-left: 10px;

  .dropdown-toggle {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 12px 20px;
    background: rgba(26, 26, 26, 0.8);
    border: 1px solid rgba(198, 40, 40, 0.3);
    border-radius: 8px;
    color: rgba(245, 245, 245, 0.9);
    font-family: "Noto Sans SC", sans-serif;
    font-size: 14px;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.3s ease;
    position: relative;
    overflow: hidden;
    z-index: 10;

    &:hover {
      background: rgba(38, 38, 38, 0.9);
      border-color: var(--accent);
      color: var(--white);
      box-shadow: 0 0 15px rgba(198, 40, 40, 0.3);

      .dropdown-icon {
        transform: rotate(15deg);
      }
    }

    .dropdown-text {
      position: relative;
      z-index: 2;
    }

    .dropdown-icon {
      font-size: 16px;
      transition: transform 0.3s ease;
    }

    .scissor-effect {
      position: absolute;
      top: 0;
      left: -100%;
      width: 100%;
      height: 100%;
      background: linear-gradient(
        90deg,
        transparent,
        rgba(198, 40, 40, 0.2),
        transparent
      );
      transition: left 0.5s ease;
      z-index: 1;

      &.open {
        left: 100%;
      }
    }
  }

  .dropdown-menu {
    position: absolute;
    top: calc(100% + 10px);
    right: 0;
    width: 200px;
    background: rgba(20, 20, 20, 0.95);
    backdrop-filter: blur(15px);
    border-radius: 8px;
    border: 1px solid rgba(198, 40, 40, 0.2);
    box-shadow: 0 10px 30px rgba(10, 10, 10, 0.8),
      0 0 20px rgba(198, 40, 40, 0.1);
    overflow: hidden;
    opacity: 0;
    visibility: hidden;
    transform: translateY(-10px);
    transition: all 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.1);
    z-index: 100;

    &::before {
      content: "";
      position: absolute;
      top: -2px;
      left: 0;
      width: 100%;
      height: 2px;
      background: linear-gradient(
        90deg,
        transparent,
        var(--accent),
        transparent
      );
      animation: scanning 2s linear infinite;
    }

    &.open {
      opacity: 1;
      visibility: visible;
      transform: translateY(0);
    }

    .dropdown-item {
      display: flex;
      align-items: center;
      justify-content: space-between;
      padding: 14px 20px;
      text-decoration: none;
      color: rgba(245, 245, 245, 0.8);
      border-bottom: 1px solid rgba(245, 245, 245, 0.05);
      transition: all 0.3s ease;
      position: relative;
      overflow: hidden;

      &::before {
        content: "";
        position: absolute;
        top: 0;
        left: -100%;
        width: 100%;
        height: 100%;
        background: linear-gradient(
          90deg,
          transparent,
          rgba(198, 40, 40, 0.1),
          transparent
        );
        transition: left 0.5s ease;
      }

      &:hover {
        color: var(--white);
        background: rgba(198, 40, 40, 0.1);

        &::before {
          left: 100%;
        }

        .dropdown-item-text {
          transform: translateX(5px);
        }

        .dropdown-item-line {
          width: 100%;
        }

        .external-icon {
          transform: translate(3px, -3px);
        }
      }

      .dropdown-item-text {
        position: relative;
        z-index: 2;
        transition: transform 0.3s ease;
      }

      .dropdown-item-line {
        position: absolute;
        bottom: 0;
        left: 0;
        width: 0;
        height: 1px;
        background: var(--accent);
        transition: width 0.4s ease;
      }

      .external-icon {
        font-size: 12px;
        color: var(--accent-light);
        transition: transform 0.3s ease;
      }

      &.external {
        background: rgba(198, 40, 40, 0.05);

        &:hover {
          background: rgba(198, 40, 40, 0.15);
        }
      }
    }
  }
}

/* 移动端导航 */
.hamburger {
  display: none;
  flex-direction: column;
  justify-content: space-between;
  width: 30px;
  height: 24px;
  background: transparent;
  border: none;
  cursor: pointer;
  padding: 0;
  z-index: 1001;

  span {
    display: block;
    width: 100%;
    height: 3px;
    background: var(--white);
    border-radius: 3px;
    transition: all 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.1);
    box-shadow: 0 0 5px rgba(198, 40, 40, 0.5);

    &:nth-child(1) {
      transform-origin: top left;
    }

    &:nth-child(2) {
      transform-origin: center;
    }

    &:nth-child(3) {
      transform-origin: bottom left;
    }

    &.open:nth-child(1) {
      transform: rotate(45deg) translate(5px, -3px);
      background: var(--accent-light);
    }

    &.open:nth-child(2) {
      opacity: 0;
      transform: scaleX(0);
    }

    &.open:nth-child(3) {
      transform: rotate(-45deg) translate(5px, 3px);
      background: var(--accent-light);
    }
  }
}

.nav-links-mobile {
  display: none;
  position: absolute;
  top: 80px;
  left: 0;
  right: 0;
  flex-direction: column;
  background: rgba(15, 15, 20, 0.98);
  backdrop-filter: blur(20px);
  border-top: 1px solid rgba(198, 40, 40, 0.2);
  border-bottom: 1px solid rgba(198, 40, 40, 0.1);
  box-shadow: 0 10px 30px rgba(10, 10, 10, 0.9);
  max-height: 0;
  overflow: hidden;
  transition: max-height 0.5s cubic-bezier(0.175, 0.885, 0.32, 1.1);
  z-index: 999;

  &.mobile-open {
    max-height: 600px;
  }

  .nav-item {
    padding: 18px 25px;
    text-decoration: none;
    color: rgba(245, 245, 245, 0.8);
    font-family: "Noto Sans SC", sans-serif;
    font-size: 16px;
    border-bottom: 1px solid rgba(245, 245, 245, 0.05);
    position: relative;
    overflow: hidden;
    transition: all 0.3s ease;

    &:hover {
      color: var(--white);
      background: rgba(198, 40, 40, 0.1);

      .nav-text {
        transform: translateX(10px);
      }

      .nav-line {
        width: 100%;
      }
    }

    &.active-link {
      color: var(--accent-light);
      font-weight: 600;
      background: rgba(198, 40, 40, 0.05);

      .nav-line {
        width: 100%;
        opacity: 1;
      }

      .nav-text::after {
        content: "✂";
        position: absolute;
        right: 25px;
        animation: scissor-float 2s infinite;
      }
    }

    .nav-text {
      position: relative;
      transition: transform 0.3s ease;
    }

    .nav-line {
      position: absolute;
      bottom: 0;
      left: 0;
      width: 0;
      height: 2px;
      background: linear-gradient(90deg, var(--accent), transparent);
      transition: width 0.4s ease;
    }

    &.external {
      background: rgba(198, 40, 40, 0.05);
      display: flex;
      justify-content: space-between;
      align-items: center;

      &:hover {
        background: rgba(198, 40, 40, 0.15);
      }

      .external-icon {
        color: var(--accent-light);
        font-size: 14px;
      }
    }
  }
}

/* 动态剪裁线效果 */
.scissor-lines {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 1;

  .scissor-line {
    position: absolute;
    height: 1px;
    background: linear-gradient(90deg, transparent, var(--accent), transparent);
    opacity: 0;

    &.line-1 {
      top: 20%;
      width: 30%;
      left: 10%;
      animation: line-scan 8s infinite;
    }

    &.line-2 {
      top: 50%;
      width: 40%;
      left: 30%;
      animation: line-scan 10s infinite 1s;
    }

    &.line-3 {
      top: 80%;
      width: 25%;
      left: 60%;
      animation: line-scan 12s infinite 2s;
    }
  }
}

/* 弦线网格背景动效 */
.string-grid {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: -1;
  background-image: radial-gradient(
      circle at 30% 30%,
      rgba(198, 40, 40, 0.03) 0%,
      transparent 50%
    ),
    radial-gradient(
      circle at 70% 70%,
      rgba(244, 67, 54, 0.02) 0%,
      transparent 50%
    );

  &::before {
    content: "";
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-image: linear-gradient(
        90deg,
        rgba(198, 40, 40, 0.02) 1px,
        transparent 1px
      ),
      linear-gradient(0deg, rgba(198, 40, 40, 0.02) 1px, transparent 1px);
    background-size: 40px 40px;
    animation: grid-move 20s linear infinite;
  }
}

/* 响应式设计 */
@media (max-width: 1200px) {
  .qianxiao-header {
    padding: 0 20px;
  }

  .nav-links-pc .nav-item {
    padding: 12px 12px;
    font-size: 13px;
  }

  .dropdown .dropdown-toggle {
    padding: 12px 15px;
    font-size: 13px;
  }

  .music-control {
    position: relative;
    left: 0;
    transform: none;
    margin-left: 20px;
  }
}

@media (max-width: 992px) {
  .qianxiao-header {
    height: 70px;
  }

  .title .title-text {
    font-size: 20px;
  }

  .nav-links-pc {
    gap: 5px;
  }

  .nav-links-pc .nav-item {
    padding: 10px 8px;
    font-size: 12px;
  }

  .music-control .music-btn {
    padding: 8px 15px;
    font-size: 13px;
  }
}

@media (max-width: 768px) {
  .qianxiao-header {
    padding: 0 15px;
    height: 70px;
    justify-content: flex-end;
  }

  .title-section {
    position: absolute;
    left: 15px;
    top: 50%;
    transform: translateY(-50%);

    .title .title-text {
      font-size: 18px;

      &::before {
        left: -20px;
        font-size: 14px;
      }
    }
  }

  .music-control {
    position: absolute;
    right: 60px;
    left: auto;
    transform: none;
    margin-left: 0;

    .music-btn {
      padding: 6px 12px;
      font-size: 12px;

      .music-text {
        display: none;
      }

      .music-wave {
        margin-left: 0;
      }
    }
  }

  .nav-links-pc {
    display: none;
  }

  .hamburger {
    display: flex;
  }

  .nav-links-mobile {
    display: flex;
  }
}

@media (max-width: 480px) {
  .title .title-text {
    font-size: 16px;
  }

  .title .title-sub {
    font-size: 10px;
  }

  .music-control {
    right: 50px;
  }
}

/* 动画关键帧 */
@keyframes header-appear {
  0% {
    opacity: 0;
    transform: translateY(-20px);
  }
  100% {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes scissor-glow {
  0%,
  100% {
    opacity: 0.5;
  }
  50% {
    opacity: 0.8;
  }
}

@keyframes star-twinkle {
  0%,
  100% {
    opacity: 1;
  }
  50% {
    opacity: 0.3;
  }
}

@keyframes shine {
  0% {
    transform: translateX(-100%);
  }
  100% {
    transform: translateX(100%);
  }
}

@keyframes pulse {
  0%,
  100% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.1);
  }
}

@keyframes line-glow {
  0%,
  100% {
    opacity: 0.8;
  }
  50% {
    opacity: 1;
    box-shadow: 0 0 10px rgba(244, 67, 54, 0.7);
  }
}

@keyframes scissor-float {
  0%,
  100% {
    transform: translateY(0) rotate(0deg);
  }
  50% {
    transform: translateY(-3px) rotate(10deg);
  }
}

@keyframes scanning {
  0% {
    transform: translateX(-100%);
  }
  100% {
    transform: translateX(100%);
  }
}

@keyframes line-scan {
  0% {
    opacity: 0;
    transform: translateX(-100%);
  }
  10% {
    opacity: 0.7;
    transform: translateX(0);
  }
  20% {
    opacity: 0;
    transform: translateX(100%);
  }
  100% {
    opacity: 0;
    transform: translateX(100%);
  }
}

@keyframes music-note {
  0%,
  100% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.3);
  }
}

@keyframes wave-animation {
  0%,
  100% {
    height: 8px;
  }
  50% {
    height: 16px;
  }
}

@keyframes grid-move {
  0% {
    transform: translate(0, 0);
  }
  100% {
    transform: translate(20px, 20px);
  }
}
</style>
