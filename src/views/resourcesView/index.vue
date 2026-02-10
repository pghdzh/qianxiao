<template>
  <div class="yuzuki-resources">
    <div class="carousel carousel1" aria-hidden="true">
      <img
        v-for="(src, idx) in randomFive"
        :key="idx"
        :src="src"
        class="carousel-image"
        :class="{ active: idx === currentIndex }"
      />
      <div class="carousel-overlay"></div>
    </div>
    <div class="carousel carousel2" aria-hidden="true">
      <img
        v-for="(src, idx) in randomFive2"
        :key="idx"
        :src="src"
        class="carousel-image"
        :class="{ active: idx === currentIndex }"
      />
      <div class="carousel-overlay"></div>
    </div>

    <header class="hero">
      <div class="hero-inner">
        <h1>
          <span class="hero-icon">✂</span> 资源分享
          <span class="hero-icon">✂</span>
        </h1>
        <p class="subtitle">可自由上传关于千咲的相关链接</p>
      </div>
      <div class="hero-decoration">
        <div class="star left"></div>
        <div class="star right"></div>
      </div>
    </header>

    <main class="container">
      <section class="uploader" :class="{ collapsed: uploaderCollapsed }">
        <div class="uploader-head">
          <button
            class="toggle"
            @click="toggleUploader"
            :aria-expanded="!uploaderCollapsed"
          >
            <span class="toggle-icon">{{ uploaderCollapsed ? "▼" : "▲" }}</span>
            <span v-if="uploaderCollapsed">展开上传区</span>
            <span v-else>收起上传区</span>
          </button>
        </div>

        <form
          @submit.prevent="addResource"
          class="upload-form"
          :aria-hidden="uploaderCollapsed"
        >
          <div class="row">
            <input
              v-model="form.title"
              type="text"
              placeholder="标题（必填，如果有解压码之类的也写这里吧）"
              aria-label="标题"
            />
            <input
              v-model="form.type"
              type="text"
              placeholder="链接类型(网页链接、b站视频、网盘链接等等)"
              aria-label="来源"
            />
          </div>

          <div class="row">
            <input
              v-model="form.uploader"
              type="text"
              placeholder="上传人（可选）"
              aria-label="上传人"
            />
            <input
              v-model="form.link"
              type="url"
              placeholder="链接(只输入网址不能有中文)"
              aria-label="链接"
            />
          </div>

          <div class="actions">
            <button type="submit" class="btn primary">上传</button>
          </div>
        </form>
      </section>

      <section class="list">
        <div class="list-header">
          <h2>
            <span class="header-accent">资源列表</span>（{{
              resources.length
            }}）
          </h2>
          <div class="sort">
            <label>
              排序：
              <select v-model="sortBy">
                <option value="time">按时间（新→旧）</option>
                <option value="likes">按点赞（高→低）</option>
              </select>
            </label>
          </div>
        </div>

        <ul class="items">
          <li v-for="item in sortedResources" :key="item.id" class="item">
            <a
              :href="item.link"
              target="_blank"
              rel="noopener noreferrer"
              class="title"
            >
              <span class="title-text">{{ item.title }}</span>
              <span class="link-icon">↗</span>
            </a>

            <div class="meta">
              <div class="left">
                <span class="uploader">{{ item.uploader || "匿名" }}</span>
                <span class="dot">•</span>
                <time :datetime="item.time">{{ formatTime(item.time) }}</time>
              </div>

              <div class="right">
                <button
                  @click.prevent="handleLike(item)"
                  :aria-pressed="likedIds.has(String(item.id))"
                  class="like-btn"
                  :class="{ active: likedIds.has(String(item.id)) }"
                >
                  <img
                    :src="
                      likedIds.has(String(item.id))
                        ? '/icons/heart-red-filled.svg'
                        : '/icons/heart-red-outline.svg'
                    "
                    class="heart-icon"
                    alt="heart"
                  />
                  <span class="count">{{ item.likes }}</span>
                </button>

                <span class="badge">{{ item.type }}</span>
              </div>
            </div>
          </li>
        </ul>

        <p v-if="resources.length === 0" class="empty">
          目前没有资源，快来上传第一条吧！
        </p>
      </section>
    </main>

    <footer class="foot">
      <div class="footer-content">
        <span class="gold-text">提示：点击标题将直接跳转</span>
        <div class="footer-decoration"></div>
      </div>
    </footer>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onBeforeUnmount } from "vue";
// 如果你的工程使用 ts 路径别名 @ 指向 src，可以用 '@/api/resource'，否则根据实际路径调整
import {
  getResourceList,
  createResource,
  likeResource,
} from "@/api/modules/resource";
import { ElMessage } from "element-plus";

interface Resource {
  id: number | string;
  title: string;
  uploader?: string;
  time: string; // ISO 或 created_at
  likes: number;
  link: string;
  type: string;
  role_key?: string;
}

const STORAGE_KEY = "fll_resources_v1";
const DEFAULT_ROLE = "qianxiao";

const form = ref<{
  title: string;
  uploader: string;
  link: string;
  type: string;
}>({
  title: "",
  uploader: "",
  link: "",
  type: "",
});

const resources = ref<Resource[]>([]);
const likedIds = ref(new Set<string>());
const sortBy = ref<"time" | "likes">("time");
const uploaderCollapsed = ref(false);

function mapServerToLocal(row: any): Resource {
  return {
    id: row.id,
    title: row.title,
    uploader: row.uploader || "匿名",
    time: row.created_at || row.time || new Date().toISOString(),
    likes: row.likes ?? 0,
    link: row.link,
    type: row.storage_type || row.type || "other",
    role_key: row.role_key,
  };
}

async function loadResources() {
  try {
    // 尝试从后端拉取（分页可扩展，这里一次拉足够 demo）
    const res: any = await getResourceList({
      role_key: DEFAULT_ROLE,
      page: 1,
      pageSize: 100,
    });
    if (res && res.success && Array.isArray(res.data)) {
      resources.value = res.data.map(mapServerToLocal);
      // 可恢复本地点赞状态（仅 UI 记忆）
      const raw = localStorage.getItem(STORAGE_KEY);
      if (raw) {
        try {
          const parsed = JSON.parse(raw);
          if (parsed.liked && Array.isArray(parsed.liked)) {
            parsed.liked.forEach((id: string) => likedIds.value.add(id));
          }
        } catch (e) {
          /* ignore */
        }
      }
      return;
    }
  } catch (err) {
    console.warn("拉取资源失败，使用本地缓存", err);
  }
  // 回退：本地缓存（仅恢复点赞状态）
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (raw) {
      const parsed = JSON.parse(raw);
      if (parsed.liked && Array.isArray(parsed.liked)) {
        parsed.liked.forEach((id: string) => likedIds.value.add(id));
      }
    }
  } catch (e) {
    console.warn("本地加载失败", e);
  }
}

function saveLocalCache() {
  try {
    const liked = Array.from(likedIds.value);
    localStorage.setItem(STORAGE_KEY, JSON.stringify({ liked }));
  } catch (e) {
    console.warn("保存本地缓存失败", e);
  }
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
  loadResources();
  Imgtimer = window.setInterval(() => {
    currentIndex.value =
      (currentIndex.value + 1) % Math.max(1, randomFive.value.length);
  }, 5200);
  uploaderCollapsed.value = window.innerWidth <= 640;
});
function toggleUploader() {
  uploaderCollapsed.value = !uploaderCollapsed.value;
}
onBeforeUnmount(() => {
  if (Imgtimer) clearInterval(Imgtimer);
});

async function addResource() {
  const t = form.value.title.trim();
  const l = form.value.link.trim();
  if (!form.value.title.trim() || !form.value.link.trim()) {
    return ElMessage.warning("请填写完整信息");
  }
  if (!/^https?:\/\//i.test(l)) {
    return ElMessage.error("请输入正确的链接(https开头)");
  }
  // 尝试调用后端接口
  try {
    const payload = {
      title: t,
      uploader: form.value.uploader.trim() || "匿名",
      link: l,
      storage_type: form.value.type,
      role_key: DEFAULT_ROLE,
    };
    const res: any = await createResource(payload);
    if (res && res.success && res.data) {
      const added = mapServerToLocal(res.data);
      resources.value.unshift(added);
      // 自动展开到顶部展示（可选）
      saveLocalCache();
      resetForm();
      ElMessage.success("上传成功");
      return;
    }
    ElMessage.error("上传失败");
  } catch (err) {
    console.warn("创建资源失败", err);
  }
}

function resetForm() {
  form.value.title = "";
  form.value.uploader = "";
  form.value.link = "";
  form.value.type = "";
}

async function handleLike(item: Resource) {
  // UI 乐观更新
  const id = item.id;
  const wasLiked = likedIds.value.has(String(id));
  if (wasLiked) {
    likedIds.value.delete(String(id));
    item.likes = Math.max(0, item.likes - 1);
  } else {
    likedIds.value.add(String(id));
    item.likes++;
  }
  saveLocalCache();

  // 同步后端（不依赖返回值进行 UI 回滚，简单处理：若失败则回退）
  try {
    const action = wasLiked ? "unlike" : "like";
    const res: any = await likeResource(id, action);
    if (
      res &&
      res.success &&
      res.data &&
      typeof res.data.likes !== "undefined"
    ) {
      item.likes = res.data.likes;
    }
  } catch (err) {
    console.warn("点赞接口调用失败，回滚本地状态", err);
    // 回滚
    if (wasLiked) {
      // 本来是已赞，取消失败 -> 重新添加
      likedIds.value.add(String(id));
      item.likes++;
    } else {
      likedIds.value.delete(String(id));
      item.likes = Math.max(0, item.likes - 1);
    }
    saveLocalCache();
  }
}

const sortedResources = computed(() => {
  const arr = [...resources.value];
  if (sortBy.value === "time") {
    arr.sort((a, b) => +new Date(b.time) - +new Date(a.time));
  } else {
    arr.sort((a, b) => b.likes - a.likes);
  }
  return arr;
});

function formatTime(iso: string) {
  try {
    const d = new Date(iso);
    return new Intl.DateTimeFormat("zh-CN", {
      month: "2-digit",
      day: "2-digit",
      hour: "2-digit",
      minute: "2-digit",
    }).format(d);
  } catch (e) {
    return iso;
  }
}
</script>

<style scoped lang="scss">
// 定义CSS变量

.yuzuki-resources {
  --deep-bg: linear-gradient(145deg, #0a0a14 0%, #1a0a14 35%, #0a0a1a 100%);
  --accent: #c62828;
  --accent-light: #f44336;
  --accent-dark: #8e0000;
  --black: #0a0a0a;
  --gray-dark: #1a1a1a;
  --gray-light: #2a2a2a;
  --white: #f5f5f5;
  --gold: #ffd700;
  --shadow-core: rgba(198, 40, 40, 0.15);
  --shadow-deep: rgba(10, 10, 20, 0.8);

  color: var(--white);
  display: flex;
  flex-direction: column;
  padding-top: 60px;
  font-family: "Noto Sans SC", "PingFang SC", "Helvetica Neue", Arial,
    sans-serif;
  -webkit-font-smoothing: antialiased;
  background: var(--deep-bg);
  min-height: 100vh;
  position: relative;

  .carousel {
    position: fixed;
    inset: 0;
    z-index: 0;
    pointer-events: none;
    overflow: hidden;

    .carousel-overlay {
      position: absolute;
      inset: 0;
      background: linear-gradient(
        145deg,
        rgba(10, 10, 20, 0.8) 0%,
        rgba(26, 10, 20, 0.7) 35%,
        rgba(10, 10, 26, 0.85) 100%
      );
      pointer-events: none;
      z-index: 1;
    }

    .carousel-image {
      position: absolute;
      width: 100%;
      height: 100%;
      object-fit: cover;
 
      opacity: 0;
      transition: opacity 1.2s cubic-bezier(0.4, 0, 0.2, 1);

      &.active {
        opacity: 1;
      }
    }
  }

  .carousel2 {
    display: none;
  }

  .hero {
    position: relative;
    padding: 32px 12px;
    background: linear-gradient(
      180deg,
      rgba(10, 10, 20, 0.9) 0%,
      rgba(26, 10, 20, 0.7) 100%
    );
    backdrop-filter: blur(12px);
    border-bottom: 1px solid rgba(42, 42, 42, 0.3);
    box-shadow: 0 4px 30px var(--shadow-deep);
    z-index: 1;
    overflow: hidden;

    &::before {
      content: "";
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      height: 1px;
      background: linear-gradient(
        90deg,
        transparent,
        var(--accent-light),
        var(--gold),
        var(--accent-light),
        transparent
      );
      opacity: 0.3;
    }

    .hero-inner {
      max-width: 1000px;
      margin: 0 auto;
      display: flex;
      flex-direction: column;
      gap: 12px;
      text-align: center;
      position: relative;
      z-index: 2;

      h1 {
        margin: 0;
        font-size: 2.5rem;
        font-weight: 900;
        letter-spacing: 2px;
        color: var(--white);
        text-shadow: 0 2px 10px var(--shadow-core),
          0 0 20px rgba(198, 40, 40, 0.3);
        position: relative;
        display: inline-flex;
        align-items: center;
        justify-content: center;
        gap: 16px;

        .hero-icon {
          color: var(--accent-light);
          font-size: 1.8rem;
          animation: pulse 2s infinite;
        }
      }

      .subtitle {
        margin-top: 8px;
        color: rgba(245, 245, 245, 0.8);
        font-size: 1.1rem;
        letter-spacing: 1px;
      }
    }

    .hero-decoration {
      position: absolute;
      top: 50%;
      left: 0;
      right: 0;
      transform: translateY(-50%);
      pointer-events: none;

      .star {
        position: absolute;
        width: 4px;
        height: 4px;
        background: var(--gold);
        border-radius: 50%;
        filter: blur(1px);
        opacity: 0.5;

        &.left {
          left: 20%;
          animation: twinkle 3s infinite;
        }

        &.right {
          right: 20%;
          animation: twinkle 3s infinite 1.5s;
        }
      }
    }
  }

  .container {
    max-width: 1000px;
    margin: 32px auto;
    padding: 0 20px;
    width: 100%;
    box-sizing: border-box;
    z-index: 2;
    position: relative;

    .uploader {
     
      overflow: hidden;
      box-shadow: 0 20px 40px var(--shadow-deep),
        0 0 0 1px rgba(42, 42, 42, 0.3);
      background: rgba(26, 26, 26, 0.7);
      backdrop-filter: blur(10px);
      margin-bottom: 32px;
      border: 1px solid rgba(66, 66, 66, 0.2);
      transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);

      .uploader-head {
        display: flex;
        justify-content: flex-end;
        padding: 16px 20px;
        background: rgba(10, 10, 10, 0.8);

        .toggle {
          background: linear-gradient(
            135deg,
            var(--accent) 0%,
            var(--accent-dark) 100%
          );
          border: 1px solid rgba(245, 245, 245, 0.1);
          color: var(--white);
          padding: 10px 20px;
          border-radius: 10px;
          cursor: pointer;
          font-weight: 600;
          transition: all 0.2s ease;
          display: flex;
          align-items: center;
          gap: 8px;
          box-shadow: 0 4px 12px var(--shadow-core);

          &:hover {
            transform: translateY(-2px);
            box-shadow: 0 6px 20px rgba(198, 40, 40, 0.25);
          }

          &:active {
            transform: translateY(0);
          }

          .toggle-icon {
            font-size: 12px;
          }
        }
      }

      .upload-form {
        padding: 24px;
        max-height: 500px;
        overflow: hidden;
        transition: all 0.4s cubic-bezier(0.4, 0, 0.2, 1);

        .row {
          display: flex;
          gap: 16px;
          margin-bottom: 20px;

          input,
          select {
            flex: 1 1 0;
            padding: 14px 16px;
            border-radius: 10px;
            border: 1px solid rgba(66, 66, 66, 0.5);
            font-size: 15px;
            background: rgba(10, 10, 10, 0.8);
            color: var(--white);
            font-weight: 500;
            outline: none;
            transition: all 0.2s ease;

            &::placeholder {
              color: rgba(245, 245, 245, 0.5);
            }

            &:focus {
              border-color: var(--accent-light);
              box-shadow: 0 0 0 2px rgba(244, 67, 54, 0.2);
              background: rgba(10, 10, 10, 0.9);
            }
          }
        }

        .actions {
          display: flex;
          gap: 12px;

          .btn {
            padding: 14px 28px;
            border-radius: 10px;
            border: none;
            font-weight: 600;
            cursor: pointer;
            transition: all 0.2s ease;
            font-size: 15px;

            &.primary {
              background: linear-gradient(
                135deg,
                var(--accent) 0%,
                var(--accent-light) 100%
              );
              color: var(--white);
              box-shadow: 0 4px 20px var(--shadow-core);

              &:hover {
                transform: translateY(-2px);
                box-shadow: 0 8px 25px rgba(198, 40, 40, 0.4);
              }

              &:active {
                transform: translateY(0);
              }
            }
          }
        }
      }

      &.collapsed {
        .upload-form {
          max-height: 0;
          padding-top: 0;
          padding-bottom: 0;
          border-top: 1px solid transparent;
        }
      }
    }

    .list {
      .list-header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin-bottom: 24px;
        padding-bottom: 16px;
        border-bottom: 1px solid rgba(66, 66, 66, 0.5);

        h2 {
          font-size: 1.8rem;
          margin: 0;
          color: var(--white);
          font-weight: 700;
          display: flex;
          align-items: center;
          gap: 8px;

          .header-accent {
            color: var(--accent-light);
          }
        }

        .sort {
          select {
            padding: 10px 16px;
            border-radius: 8px;
            border: 1px solid rgba(66, 66, 66, 0.5);
            background: rgba(10, 10, 10, 0.8);
            color: var(--white);
            font-size: 14px;
            cursor: pointer;
            transition: all 0.2s ease;

            &:focus {
              border-color: var(--accent-light);
              outline: none;
            }
          }
        }
      }

      .items {
        list-style: none;
        padding: 0;
        margin: 0;
        max-height: 60vh;
        overflow-y: auto;
        scrollbar-width: thin;
        scrollbar-color: var(--accent) rgba(42, 42, 42, 0.3);

        &::-webkit-scrollbar {
          width: 6px;
        }

        &::-webkit-scrollbar-track {
          background: rgba(42, 42, 42, 0.3);
          border-radius: 3px;
        }

        &::-webkit-scrollbar-thumb {
          background: var(--accent);
          border-radius: 3px;
        }
      }

      .item {
        border-radius: 12px;
        padding: 20px;
        margin-bottom: 16px;
        background: rgba(26, 26, 26, 0.7);
        backdrop-filter: blur(8px);
        border: 1px solid rgba(66, 66, 66, 0.3);
        transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
        position: relative;
        overflow: hidden;

        &::before {
          content: "";
          position: absolute;
          top: 0;
          left: 0;
          right: 0;
          height: 1px;
          background: linear-gradient(
            90deg,
            transparent,
            var(--accent-light),
            transparent
          );
          opacity: 0;
          transition: opacity 0.3s ease;
        }

        &:hover {
          transform: translateY(-4px);
          border-color: rgba(244, 67, 54, 0.3);
          box-shadow: 0 10px 30px var(--shadow-deep),
            0 0 20px rgba(198, 40, 40, 0.1);

          &::before {
            opacity: 1;
          }
        }

        .title {
          display: flex;
          justify-content: space-between;
          align-items: center;
          color: var(--white);
          font-weight: 600;
          text-decoration: none;
          margin-bottom: 16px;
          font-size: 1.1rem;
          transition: color 0.2s ease;

          &:hover {
            color: var(--accent-light);

            .link-icon {
              transform: translate(3px, -3px);
            }
          }

          .title-text {
            flex: 1;
            overflow: hidden;
            text-overflow: ellipsis;
            white-space: nowrap;
          }

          .link-icon {
            color: var(--accent-light);
            font-size: 0.9rem;
            transition: transform 0.2s ease;
          }
        }

        .meta {
          display: flex;
          justify-content: space-between;
          align-items: center;
          font-size: 0.9rem;

          .left {
            display: flex;
            align-items: center;
            gap: 12px;

            .uploader {
              color: var(--gold);
              font-weight: 500;
              margin: auto 0;
            }

            .dot {
              color: rgba(245, 245, 245, 0.5);
            }

            time {
              color: rgba(245, 245, 245, 0.7);
            }
          }

          .right {
            display: flex;
            align-items: center;
            gap: 12px;

            .like-btn {
              background: rgba(10, 10, 10, 0.6);
              border: 1px solid rgba(66, 66, 66, 0.5);
              cursor: pointer;
              padding: 8px 16px;
              border-radius: 8px;
              font-weight: 500;
              display: inline-flex;
              align-items: center;
              gap: 8px;
              transition: all 0.2s ease;
              color: rgba(245, 245, 245, 0.8);

              &:hover {
                background: rgba(26, 26, 26, 0.8);
                border-color: var(--accent);
              }

              &.active {
                background: rgba(198, 40, 40, 0.15);
                border-color: var(--accent);
                color: var(--accent-light);
              }
            }

            .heart-icon {
              width: 18px;
              height: 18px;
              display: block;
              transition: transform 0.2s ease;
            }

            .like-btn.active .heart-icon {
              transform: scale(1.1);
            }

            .badge {
              padding: 6px 12px;
              border-radius: 20px;
              font-size: 0.8rem;
              font-weight: 600;
              background: rgba(10, 10, 10, 0.8);
              color: var(--accent-light);
              border: 1px solid rgba(244, 67, 54, 0.3);
              white-space: nowrap;
            }
          }
        }
      }

      .empty {
        text-align: center;
        color: rgba(245, 245, 245, 0.6);
        padding: 48px 0;
        font-size: 1.1rem;
        background: rgba(26, 26, 26, 0.5);
        border-radius: 12px;
        border: 1px dashed rgba(66, 66, 66, 0.5);
      }
    }
  }

  .foot {
    margin-top: auto;
    padding: 24px;
    text-align: center;
    position: relative;
    z-index: 2;

    .footer-content {
      color: rgba(245, 245, 245, 0.7);
      font-size: 0.9rem;
      position: relative;
      display: inline-block;

      .gold-text {
        color: var(--gold);
        font-weight: 500;
      }

      .footer-decoration {
        position: absolute;
        bottom: -8px;
        left: 50%;
        transform: translateX(-50%);
        width: 100px;
        height: 1px;
        background: linear-gradient(
          90deg,
          transparent,
          var(--gold),
          transparent
        );
        opacity: 0.5;
      }
    }
  }

  @media (max-width: 640px) {
    padding-top: 40px;

    .carousel1 {
      display: none;
    }

    .carousel2 {
      display: block;
    }

    .hero {
      padding: 24px 16px;
      padding-top: 80px;
      .hero-inner {
        h1 {
          font-size: 1.8rem;
          gap: 12px;

          .hero-icon {
            font-size: 1.4rem;
          }
        }

        .subtitle {
          font-size: 1rem;
        }
      }
    }

    .container {
      margin: 24px auto;
      padding: 0 16px;

      .uploader {
        .upload-form {
          .row {
            flex-direction: column;
            gap: 12px;
          }
        }
      }

      .list {
        .list-header {
          flex-direction: column;
          align-items: stretch;
          gap: 16px;

          h2 {
            font-size: 1.5rem;
          }
        }

        .item {
          padding: 16px;

          .meta {
            flex-direction: column;
            align-items: stretch;
            gap: 12px;

            .left,
            .right {
              width: 100%;
            }

            .left {
              justify-content: flex-start;
            }

            .right {
              justify-content: space-between;
            }
          }
        }
      }
    }
  }
}

// 动画
@keyframes pulse {
  0%,
  100% {
    opacity: 1;
    transform: scale(1);
  }
  50% {
    opacity: 0.7;
    transform: scale(1.1);
  }
}

@keyframes twinkle {
  0%,
  100% {
    opacity: 0.2;
    transform: scale(1);
  }
  50% {
    opacity: 0.8;
    transform: scale(1.2);
  }
}
</style>