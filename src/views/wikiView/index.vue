<template>
  <div class="wiki-page">
    <!-- 背景轮播放在最底层 -->
    <div class="carousel">
      <img
        v-for="(src, idx) in randomFive"
        :key="idx"
        :src="src"
        class="carousel-image"
        :class="{ active: idx === currentIndex }"
      />
    </div>
    <header class="wiki-header">
      <div class="title">
        <h1>千咲文本分享</h1>
        <p class="subtitle">弦间低语，心音注脚</p>
      </div>
      <div class="actions">
        <input
          v-model="search"
          class="search"
          placeholder="搜索标题或者标签..."
        />
        <button class="btn btn-new" @click="openCreate">新建词条</button>
      </div>
    </header>

    <main class="wiki-body">
      <div v-if="filteredEntries.length === 0" class="empty">
        没有找到匹配的词条 ✨
      </div>

      <ul class="entry-list">
        <li v-for="entry in filteredEntries" :key="entry.id" class="entry-card">
          <div class="entry-head">
            <div class="entry-meta" @click="openDetail(entry)">
              <div class="entry-title-wrap">
                <h2 class="entry-title">{{ entry.title }}</h2>
                <span class="entry-badge">#{{ entry.slug || "未设置" }}</span>
              </div>
              <div class="entry-info">
                <span class="meta-item">作者：{{ entry.author }}</span>
                <span class="meta-item"
                  >时间：{{ formatTime(entry.updatedAt) }}</span
                >
              </div>
            </div>

            <div class="entry-actions">
              <button
                class="like"
                :class="{ active: isLiked(entry.id) }"
                :aria-pressed="isLiked(entry.id)"
                @click.stop="toggleLike(entry.id)"
              >
                <img
                  :src="
                    isLiked(entry.id)
                      ? '/icons/heart-red-filled.svg'
                      : '/icons/heart-red-outline.svg'
                  "
                  alt="like"
                />
                <span class="like-count">{{ entry.likes }}</span>
              </button>
              <div class="edit-delete" v-if="canEdit(entry.id)">
                <button class="small" @click="openEdit(entry)">编辑</button>
                <button class="small danger" @click="remove(entry.id)">
                  删除
                </button>
              </div>
            </div>
          </div>
        </li>
      </ul>
    </main>

    <!-- Edit/Create Modal -->
    <transition name="fade-zoom">
      <div class="modal-overlay" v-if="showModal">
        <div class="modal">
          <header class="modal-header">
            <h3>{{ editing ? "编辑词条" : "新建词条" }}</h3>
            <button class="close" @click="closeModal">✕</button>
          </header>
          <section class="modal-body">
            <label>
              标题
              <input v-model="form.title" placeholder="输入标题" />
            </label>

            <label>
              词条（短标签）
              <input
                v-model="form.slug"
                placeholder="比如：彩蛋、考据、点位等等"
              />
            </label>

            <label>
              作者
              <input v-model="form.author" placeholder="作者昵称" />
            </label>

            <label>
              内容
              <textarea
                v-model="form.content"
                rows="8"
                placeholder="在这里输入词条内容"
              ></textarea>
            </label>
          </section>
          <footer class="modal-footer">
            <button class="btn ghost" @click="closeModal">取消</button>
            <button class="btn" @click="submit">
              {{ editing ? "保存" : "创建" }}
            </button>
          </footer>
        </div>
      </div>
    </transition>

    <!-- Detail Modal -->
    <transition name="fade-zoom">
      <div class="modal-overlay" v-if="detailEntry">
        <div class="modal detail-modal">
          <header class="modal-header">
            <h3>{{ detailEntry.title }}</h3>
            <button class="close" @click="detailEntry = null">✕</button>
          </header>
          <section class="modal-body">
            <div class="detail-content">{{ detailEntry.content }}</div>
          </section>
        </div>
      </div>
    </transition>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, computed, onMounted, onUnmounted } from "vue";
import { ElMessage } from "element-plus";
import {
  getWikiList,
  createWiki,
  updateWiki,
  deleteWiki,
  likeWiki,
} from "@/api/modules/wiki";

// 本地存储自己创建的词条 ID
const LS_MY_WIKI_IDS = "yuzuriha:wiki:my_ids";
const myWikiIds: string[] = JSON.parse(
  localStorage.getItem(LS_MY_WIKI_IDS) || "[]"
);
const markAsMine = (id: string | number) => {
  if (!myWikiIds.includes(String(id))) {
    myWikiIds.push(String(id));
    localStorage.setItem(LS_MY_WIKI_IDS, JSON.stringify(myWikiIds));
  }
};
const canEdit = (id: string | number) => myWikiIds.includes(String(id));

// 数据状态
const entries = ref<any[]>([]);

// 本地存储键
const LS_LIKED_IDS = "yuzuriha:wiki:liked_ids";
// 从 localStorage 读取已点赞 id 列表（字符串形式）
const likedIds = ref<string[]>(
  JSON.parse(localStorage.getItem(LS_LIKED_IDS) || "[]")
);

const showModal = ref(false);
const editing = ref(false);
const editingId = ref<string | number | null>(null);
const detailEntry = ref<any>(null);
const form = reactive({ title: "", slug: "", author: "", content: "" });
const search = ref("");

// 时间格式化
function formatTime(ts: string | number | null | undefined) {
  if (!ts) return "未知时间";
  const date = new Date(ts);
  if (isNaN(date.getTime())) return "未知时间";
  return `${date.getFullYear()}-${String(date.getMonth() + 1).padStart(
    2,
    "0"
  )}-${String(date.getDate()).padStart(2, "0")}`;
}

// 加载词条列表
async function loadEntries() {
  try {
    const res: any = await getWikiList();
    entries.value = res.data.map((e: any) => ({
      ...e,
      createdAt: e.createdAt || e.created_at,
      updatedAt: e.updatedAt || e.updated_at,
    }));
  } catch (err) {
    console.error(err);
    ElMessage.error("加载词条失败");
  }
}

// 打开/关闭弹窗
function openCreate() {
  editing.value = false;
  editingId.value = null;
  form.title = "";
  form.slug = "";

  form.content = "";
  showModal.value = true;
}
function openEdit(entry: any) {
  if (!canEdit(entry.id)) {
    ElMessage.warning("只有创建者可以编辑");
    return;
  }
  editing.value = true;
  editingId.value = entry.id;
  form.title = entry.title;
  form.slug = entry.slug;
  form.author = entry.author;
  form.content = entry.content;
  showModal.value = true;
}
function closeModal() {
  showModal.value = false;
}

const canSubmit = computed(() => form.title.trim() && form.content.trim());

// 提交
async function submit() {
  if (!canSubmit.value) {
    ElMessage.warning("请填写标题和内容");
    return;
  }
  const payload = {
    title: form.title.trim(),
    author: form.author.trim() || "匿名",
    content: form.content.trim(),
    slug: null,
  };
  if (form.slug.trim()) payload.slug = form.slug.trim();
  try {
    if (editing.value && editingId.value) {
      await updateWiki(editingId.value, payload);
      ElMessage.success("编辑成功");
    } else {
      const res: any = await createWiki(payload);
      markAsMine(res.data.id);
      editingId.value = res.data.id;
      ElMessage.success("创建成功");
    }
    showModal.value = false;
    loadEntries();
  } catch (err) {
    console.error(err);
    ElMessage.error("提交失败");
  }
}

// 删除
async function remove(id: string | number) {
  if (!canEdit(id)) {
    ElMessage.warning("只有创建者可以删除");
    return;
  }
  if (!confirm("确认删除该词条？此操作不可撤销")) return;
  try {
    await deleteWiki(id);
    const index = myWikiIds.indexOf(String(id));
    if (index !== -1) myWikiIds.splice(index, 1);
    localStorage.setItem(LS_MY_WIKI_IDS, JSON.stringify(myWikiIds));
    ElMessage.success("删除成功");
    loadEntries();
  } catch (err) {
    console.error(err);
    ElMessage.error("删除失败");
  }
}

// 点赞
function persistLikedIds() {
  try {
    localStorage.setItem(LS_LIKED_IDS, JSON.stringify(likedIds.value));
  } catch (e) {
    console.warn("保存 likedIds 失败", e);
  }
}

// 判断是否已点赞（供模板绑定 class/aria-pressed）
function isLiked(id: string | number) {
  return likedIds.value.includes(String(id));
}

// 点赞 / 取消点赞（乐观更新，本地仅存 id，点赞数使用 entry.likes）
async function toggleLike(id: string | number) {
  const entry = entries.value.find((e) => e.id === id);
  if (!entry) return;

  const idStr = String(id);
  const wasLiked = likedIds.value.includes(idStr);

  // 乐观更新 UI（立即反映）
  if (wasLiked) {
    // 取消点赞：保证不低于 0
    entry.likes = Math.max(0, (entry.likes || 0) - 1);
    likedIds.value = likedIds.value.filter((x) => x !== idStr);
  } else {
    // 点赞
    entry.likes = (entry.likes || 0) + 1;
    likedIds.value.push(idStr);
  }
  persistLikedIds();

  try {
    // 调用后端（action: 'like' | 'unlike' | 'toggle'）
    // 我们明确传 'like' 或 'unlike'
    const action = wasLiked ? "unlike" : "like";
    await likeWiki(id, action);

    // 可选：如果后端在响应中返回了最新的 likes 数（res.data.likes），
    // 你可以在这里用后端值覆盖本地（示例注释）
    // const res = await likeWiki(id, action)
    // if (res?.data?.likes !== undefined) entry.likes = res.data.likes
  } catch (err) {
    // 出错则回滚乐观更新
    console.error("toggleLike error", err);
    if (wasLiked) {
      // 取消点赞失败 -> 重新标记为已点赞
      entry.likes = (entry.likes || 0) + 1;
      if (!likedIds.value.includes(idStr)) likedIds.value.push(idStr);
    } else {
      // 点赞失败 -> 取消之前增加的 count
      entry.likes = Math.max(0, (entry.likes || 0) - 1);
      likedIds.value = likedIds.value.filter((x) => x !== idStr);
    }
    persistLikedIds();
    ElMessage.error("点赞失败，请稍后重试");
  }
}

// 详情弹窗
async function openDetail(entry: any) {
  detailEntry.value = entry;
}

// 搜索过滤
const filteredEntries = computed(() => {
  const q = String(search.value || "")
    .trim()
    .toLowerCase();
  let list = entries.value;

  // 搜索过滤
  if (q) {
    list = list.filter(
      (e) =>
        (e.title || "").toLowerCase().includes(q) ||
        (e.slug || "").toLowerCase().includes(q)
    );
  }

  // 按点赞数排序（默认降序：点赞多的在前）
  return [...list].sort((a, b) => (b.likes || 0) - (a.likes || 0));
});

// 1. 分别导入两套图
const pcModules = import.meta.glob("@/assets/images1/*.{jpg,png,jpeg,webp}", {
  eager: true,
});
const mobileModules = import.meta.glob(
  "@/assets/images2/*.{jpg,png,jpeg,webp}",
  { eager: true }
);

const pcSrcs: string[] = Object.values(pcModules).map((m: any) => m.default);
const mobileSrcs: string[] = Object.values(mobileModules).map(
  (m: any) => m.default
);

// 洗牌函数
function shuffle<T>(arr: T[]): T[] {
  const a = arr.slice();
  for (let i = a.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [a[i], a[j]] = [a[j], a[i]];
  }
  return a;
}

const randomFive = ref<string[]>([]);
const currentIndex = ref(0);
let timer: number;

function pickImages() {
  const isMobile = window.innerWidth < 768;
  const all = isMobile ? mobileSrcs : pcSrcs;
  randomFive.value = shuffle(all).slice(0, 5);
}

onMounted(() => {
  loadEntries();
  pickImages(); // 首次判断
  // 监听窗口大小变化
  window.addEventListener("resize", pickImages);

  // 轮播
  timer = window.setInterval(() => {
    if (randomFive.value.length > 0) {
      currentIndex.value = (currentIndex.value + 1) % randomFive.value.length;
    }
  }, 5000);
});

onUnmounted(() => {
  clearInterval(timer);
  window.removeEventListener("resize", pickImages);
});
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

/* ====== 千咲风格 · Wiki页面 ====== */
.wiki-page {
  min-height: 100vh;
  color: $white;
  padding: 20px;
  box-sizing: border-box;
  padding-top: 80px;
  background: $deep-bg;
  position: relative;
  overflow-x: hidden;

  /* 血色弦线背景纹理 */
  &::before {
    content: "";
    position: fixed;
    inset: 0;
    background-image: 
      linear-gradient(45deg, transparent 49%, $accent-transparent-10 50%, transparent 51%),
      linear-gradient(-45deg, transparent 49%, $accent-transparent-10 50%, transparent 51%);
    background-size: 80px 80px;
    opacity: 0.1;
    pointer-events: none;
    z-index: 0;
  }

  /* 动态血色光晕 */
  &::after {
    content: "";
    position: fixed;
    width: 600px;
    height: 600px;
    bottom: -200px;
    right: -200px;
    background: radial-gradient(circle, $accent-transparent-20 0%, transparent 70%);
    border-radius: 50%;
    pointer-events: none;
    z-index: 0;
    animation: pulse 15s ease-in-out infinite alternate;
  }

  /* 轮播背景 */
  .carousel {
    position: fixed;
    inset: 0;
    z-index: 0;
    
    &::before {
      content: "";
      position: absolute;
      inset: 0;
      background: linear-gradient(180deg, 
        rgba(10, 10, 20, 0.9) 0%, 
        rgba(26, 10, 20, 0.8) 50%, 
        rgba(10, 10, 26, 0.9) 100%);
      pointer-events: none;
      z-index: 1;
      mix-blend-mode: multiply;
    }
    
    .carousel-image {
      position: absolute;
      width: 100%;
      height: 100%;
      object-fit: cover;
      opacity: 0;
      transition: opacity 1.2s ease;
      filter: brightness(0.4) contrast(1.2) saturate(0.8) blur(1.5px);
      
      &.active {
        opacity: 0.15;
      }
    }
  }

  /* 页面头部 */
  .wiki-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 20px;
    padding: 20px 24px;
    background: linear-gradient(145deg, 
      rgba(26, 10, 20, 0.9) 0%, 
      rgba(10, 10, 10, 0.95) 100%);
    border-radius: 16px;
    box-shadow: 
      0 16px 48px $shadow-deep,
      inset 0 1px 0 $white-transparent-10;
    border: 1px solid $accent-transparent-30;
    backdrop-filter: blur(10px);
    position: relative;
    z-index: 10;
    margin-bottom: 24px;
    overflow: hidden;
    
    /* 右上角剪刀装饰 */
    &::after {
      content: "✄";
      position: absolute;
      top: 20px;
      right: 24px;
      font-size: 24px;
      color: $accent-transparent-20;
      transform: rotate(45deg);
      opacity: 0.5;
    }

    .title {
      h1 {
        margin: 0;
        font-size: 1.8rem;
        color: $white;
        font-weight: 800;
        letter-spacing: 1px;
        position: relative;
        padding-bottom: 8px;
        
        &::after {
          content: "";
          position: absolute;
          bottom: 0;
          left: 0;
          width: 80px;
          height: 3px;
          background: linear-gradient(90deg, $accent, $accent-dark);
          border-radius: 3px;
        }
      }
      
      .subtitle {
        font-size: 1rem;
        color: $white-transparent-70;
        margin-top: 8px;
        font-style: italic;
        padding-left: 20px;
        position: relative;
        
        &::before {
          content: "❖";
          position: absolute;
          left: 0;
          top: 50%;
          transform: translateY(-50%);
          color: $gold;
          font-size: 1.2rem;
        }
      }
    }

    .actions {
      display: flex;
      gap: 16px;
      align-items: center;
      flex-wrap: wrap;
    }

    .search {
      padding: 12px 16px;
      border-radius: 12px;
      border: 1px solid $accent-transparent-20;
      background: $black-transparent-50;
      color: $white;
      font-size: 0.95rem;
      width: 280px;
      transition: all 0.3s ease;
      box-shadow: 0 6px 20px $shadow-deep;
      
      &::placeholder {
        color: $white-transparent-50;
      }
      
      &:focus {
        outline: none;
        border-color: $accent-transparent-50;
        box-shadow: 
          0 6px 24px $shadow-deep,
          0 0 0 3px $accent-transparent-20;
        width: 320px;
      }
    }

    .btn-new {
      background: linear-gradient(145deg, $accent, $accent-dark);
      color: $white;
      border: none;
      border-radius: 12px;
      padding: 12px 24px;
      font-size: 1rem;
      font-weight: 600;
      cursor: pointer;
      box-shadow: 
        0 8px 28px $accent-transparent-30,
        inset 0 1px 0 $white-transparent-20;
      transition: all 0.3s ease;
      position: relative;
      overflow: hidden;
      
      &:hover {
        background: linear-gradient(145deg, $accent-light, $accent);
        transform: translateY(-3px);
        box-shadow: 
          0 14px 36px $accent-transparent-50,
          inset 0 1px 0 $white-transparent-30;
      }
      
      &:active {
        transform: translateY(-1px);
      }
      
      /* 按钮光晕效果 */
      &::before {
        content: "";
        position: absolute;
        inset: 0;
        background: radial-gradient(circle at center, 
          $white-transparent-20 0%, 
          transparent 70%);
        opacity: 0;
        transition: opacity 0.3s ease;
      }
      
      &:hover::before {
        opacity: 1;
      }
    }
  }

  /* 主要内容区域 */
  .wiki-body {
    position: relative;
    z-index: 10;
    
    .empty {
      text-align: center;
      padding: 60px 20px;
      color: $white-transparent-70;
      font-size: 1.2rem;
      background: $black-transparent-30;
      border-radius: 12px;
      border: 1px solid $accent-transparent-10;
      margin: 20px 0;
      backdrop-filter: blur(5px);
      
      &::before {
        content: "📝";
        display: block;
        font-size: 3rem;
        margin-bottom: 16px;
        opacity: 0.7;
      }
    }

    .entry-list {
      list-style: none;
      padding: 0;
      margin: 0;
      display: grid;
      gap: 20px;
      
      .entry-card {
        background: linear-gradient(145deg, 
          rgba(42, 26, 36, 0.9) 0%, 
          rgba(26, 10, 20, 0.95) 100%);
        border-radius: 16px;
        padding: 24px;
        box-shadow: 
          0 12px 36px $shadow-deep,
          inset 0 1px 0 $white-transparent-10;
        transition: all 0.3s ease;
        border: 1px solid $accent-transparent-20;
        backdrop-filter: blur(8px);
        position: relative;
        overflow: hidden;
        
        /* 卡片角落装饰 */
        &::before {
          content: "";
          position: absolute;
          top: 0;
          right: 0;
          width: 60px;
          height: 60px;
          background: linear-gradient(45deg, 
            transparent 50%, 
            $accent-transparent-10 50%);
          border-bottom-left-radius: 16px;
          pointer-events: none;
        }
        
        &:hover {
          transform: translateY(-6px);
          box-shadow: 
            0 20px 48px $shadow-deep,
            0 0 0 1px $accent-transparent-30;
          border-color: $accent-transparent-50;
          
          .entry-title {
            color: $accent-light;
          }
          
          .entry-badge {
            background: $accent-transparent-30;
            border-color: $accent-transparent-50;
          }
        }

        .entry-head {
          display: flex;
          justify-content: space-between;
          align-items: flex-start;
          gap: 16px;
          flex-wrap: wrap;

          .entry-meta {
            flex: 1;
            cursor: pointer;
            min-width: 0;
            
            .entry-title-wrap {
              display: flex;
              align-items: center;
              gap: 12px;
              margin-bottom: 12px;
              flex-wrap: wrap;
              
              .entry-title {
                font-size: 1.4rem;
                margin: 0;
                color: $white;
                font-weight: 700;
                transition: color 0.3s ease;
                line-height: 1.3;
                
                /* 限制为两行 */
                display: -webkit-box;
                -webkit-line-clamp: 2;
                -webkit-box-orient: vertical;
                overflow: hidden;
                text-overflow: ellipsis;
              }
              
              .entry-badge {
                display: inline-block;
                padding: 6px 12px;
                border-radius: 20px;
                background: $accent-transparent-10;
                color: $accent-light;
                font-size: 0.85rem;
                font-weight: 600;
                border: 1px solid $accent-transparent-20;
                transition: all 0.3s ease;
                white-space: nowrap;
                flex-shrink: 0;
                
                &::before {
                  content: "#";
                  margin-right: 2px;
                  opacity: 0.7;
                }
              }
            }
            
            .entry-info {
              display: flex;
              flex-wrap: wrap;
              gap: 16px;
              align-items: center;
              
              .meta-item {
                font-size: 0.9rem;
                color: $white-transparent-70;
                display: flex;
                align-items: center;
                gap: 6px;
                
                &::before {
                  font-size: 1rem;
                  opacity: 0.6;
                }
                
                &:first-child::before {
                  content: "👤";
                }
                
                &:last-child::before {
                  content: "🕒";
                }
              }
            }
          }

          .entry-actions {
            display: flex;
            gap: 12px;
            align-items: center;
            flex-wrap: wrap;
            flex-shrink: 0;

            .like {
              background: transparent;
              border: 1px solid $accent-transparent-30;
              border-radius: 12px;
              display: flex;
              align-items: center;
              gap: 8px;
              cursor: pointer;
              padding: 8px 16px;
              transition: all 0.3s ease;
              min-width: 100px;
              
              &:hover {
                background: $accent-transparent-10;
                transform: translateY(-2px);
                
                .like-count {
                  color: $accent-light;
                }
              }
              
              &.active {
                background: $accent-transparent-20;
                border-color: $accent-transparent-50;
                box-shadow: 0 0 20px $accent-transparent-30;
                
                .like-count {
                  color: $accent-light;
                  font-weight: 700;
                }
                
                img {
                  filter: brightness(1.2) saturate(1.5);
                  animation: heart-beat 0.6s ease;
                }
              }
              
              img {
                width: 20px;
                height: 20px;
                transition: all 0.3s ease;
                filter: brightness(0.8) saturate(0.8);
              }
              
              .like-count {
                font-size: 1rem;
                color: $white-transparent-70;
                font-weight: 600;
                transition: color 0.3s ease;
                min-width: 24px;
                text-align: center;
              }
            }

            .edit-delete {
              display: flex;
              gap: 8px;
              
              button {
                padding: 10px 16px;
                border-radius: 10px;
                border: none;
                font-size: 0.9rem;
                font-weight: 600;
                cursor: pointer;
                transition: all 0.3s ease;
                min-width: 80px;
                
                &.small {
                  background: $gray-dark;
                  color: $white;
                  border: 1px solid $accent-transparent-20;
                  
                  &:hover {
                    background: $accent-transparent-20;
                    transform: translateY(-2px);
                    box-shadow: 0 6px 20px $accent-transparent-20;
                  }
                }
                
                &.danger {
                  background: linear-gradient(145deg, 
                    rgba(198, 40, 40, 0.2) 0%, 
                    rgba(142, 0, 0, 0.15) 100%);
                  color: $accent-light;
                  border: 1px solid $accent-transparent-30;
                  
                  &:hover {
                    background: linear-gradient(145deg, 
                      rgba(198, 40, 40, 0.3) 0%, 
                      rgba(142, 0, 0, 0.25) 100%);
                    box-shadow: 0 6px 20px $accent-transparent-30;
                  }
                }
              }
            }
          }
        }
      }
    }
  }

  /* ====== 模态框样式 ====== */
  .modal-overlay {
    position: fixed;
    inset: 0;
    background: linear-gradient(145deg, 
      rgba(10, 10, 20, 0.9) 0%, 
      rgba(26, 10, 20, 0.95) 100%);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
    padding: 20px;
    backdrop-filter: blur(8px);
    
    .modal {
      width: min(800px, 96%);
      max-height: 90vh;
      overflow-y: auto;
      background: linear-gradient(145deg, 
        rgba(42, 42, 42, 0.95) 0%, 
        rgba(26, 26, 26, 0.98) 100%);
      backdrop-filter: blur(20px);
      border-radius: 20px;
      padding: 30px;
      box-shadow: 
        0 30px 80px $shadow-deep,
        inset 0 1px 0 $white-transparent-10;
      border: 1px solid $accent-transparent-30;
      animation: modal-appear 0.4s cubic-bezier(0.4, 0, 0.2, 1);
      
      /* 模态框装饰 */
      &::before {
        content: "📖";
        position: absolute;
        top: 20px;
        left: 20px;
        font-size: 24px;
        color: $accent-transparent-20;
        opacity: 0.5;
      }
      
      /* 详情模态框 */
      &.detail-modal {
        max-width: 900px;
        
        .modal-body {
          .detail-content {
            white-space: pre-wrap;
            line-height: 1.8;
            font-size: 1.1rem;
            color: $white-transparent-90;
            padding: 20px;
            background: $black-transparent-30;
            border-radius: 12px;
            border: 1px solid $accent-transparent-10;
            max-height: 60vh;
            overflow-y: auto;
            
            &::-webkit-scrollbar {
              width: 8px;
            }
            
            &::-webkit-scrollbar-thumb {
              background: linear-gradient(180deg, 
                $accent-transparent-50 0%, 
                $accent-transparent-30 100%);
              border-radius: 4px;
            }
          }
        }
      }

      .modal-header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin-bottom: 24px;
        padding-bottom: 16px;
        border-bottom: 1px solid $accent-transparent-20;
        
        h3 {
          margin: 0;
          font-size: 1.5rem;
          color: $white;
          font-weight: 700;
          position: relative;
          
          &::after {
            content: "";
            position: absolute;
            bottom: -16px;
            left: 0;
            width: 60px;
            height: 3px;
            background: linear-gradient(90deg, $accent, $accent-dark);
            border-radius: 3px;
          }
        }
        
        .close {
          background: transparent;
          border: none;
          font-size: 1.5rem;
          color: $white-transparent-70;
          cursor: pointer;
          transition: all 0.3s ease;
          width: 40px;
          height: 40px;
          border-radius: 50%;
          display: flex;
          align-items: center;
          justify-content: center;
          
          &:hover {
            color: $accent-light;
            background: $accent-transparent-10;
            transform: rotate(90deg);
          }
        }
      }

      .modal-body {
        color: $white;
        font-size: 1rem;
        line-height: 1.6;
        display: flex;
        flex-direction: column;
        gap: 20px;
        
        label {
          display: flex;
          flex-direction: column;
          gap: 8px;
          font-weight: 600;
          color: $white-transparent-90;
          
          input, textarea {
            padding: 14px 18px;
            border-radius: 12px;
            border: 1px solid $accent-transparent-20;
            background: $black-transparent-50;
            color: $white;
            font-size: 1rem;
            transition: all 0.3s ease;
            
            &::placeholder {
              color: $white-transparent-50;
            }
            
            &:focus {
              outline: none;
              border-color: $accent-transparent-50;
              box-shadow: 0 0 0 3px $accent-transparent-20;
              background: $black-transparent-70;
            }
          }
          
          textarea {
            min-height: 200px;
            resize: vertical;
            font-family: inherit;
            line-height: 1.6;
          }
        }
      }

      .modal-footer {
        display: flex;
        justify-content: flex-end;
        gap: 16px;
        margin-top: 32px;
        padding-top: 20px;
        border-top: 1px solid $accent-transparent-10;
        
        .btn {
          padding: 12px 28px;
          border-radius: 12px;
          border: none;
          font-size: 1rem;
          font-weight: 600;
          cursor: pointer;
          transition: all 0.3s ease;
          min-width: 120px;
          
          &.ghost {
            background: transparent;
            border: 1px solid $accent-transparent-30;
            color: $white-transparent-90;
            
            &:hover {
              background: $accent-transparent-10;
              border-color: $accent-transparent-50;
              transform: translateY(-2px);
            }
          }
          
          &:not(.ghost) {
            background: linear-gradient(145deg, $accent, $accent-dark);
            color: $white;
            box-shadow: 0 8px 28px $accent-transparent-30;
            
            &:hover {
              background: linear-gradient(145deg, $accent-light, $accent);
              transform: translateY(-3px);
              box-shadow: 0 14px 36px $accent-transparent-50;
            }
            
            &:active {
              transform: translateY(-1px);
            }
          }
        }
      }
    }
  }

  /* 模态框动画 */
  .fade-zoom-enter-active,
  .fade-zoom-leave-active {
    transition: all 0.4s cubic-bezier(0.4, 0, 0.2, 1);
  }
  
  .fade-zoom-enter-from,
  .fade-zoom-leave-to {
    opacity: 0;
    transform: scale(0.95);
  }
}

/* ====== 动画定义 ====== */
@keyframes pulse {
  0%, 100% { 
    opacity: 0.2; 
    transform: scale(1); 
  }
  50% { 
    opacity: 0.4; 
    transform: scale(1.1); 
  }
}

@keyframes heart-beat {
  0%, 100% { 
    transform: scale(1); 
  }
  25% { 
    transform: scale(1.2); 
  }
  50% { 
    transform: scale(1); 
  }
  75% { 
    transform: scale(1.1); 
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

/* ====== 响应式设计 ====== */
@media (max-width: 768px) {
  .wiki-page {
    padding: 12px;
    padding-top: 70px;
    
    .wiki-header {
      flex-direction: column;
      align-items: stretch;
      gap: 16px;
      padding: 16px;
      
      .title {
        h1 {
          font-size: 1.5rem;
        }
      }
      
      .actions {
        flex-direction: column;
        align-items: stretch;
        
        .search {
          width: 100%;
          
          &:focus {
            width: 100%;
          }
        }
        
        .btn-new {
          width: 100%;
          text-align: center;
        }
      }
    }
    
    .entry-card {
      .entry-head {
        flex-direction: column;
        gap: 16px;
        
        .entry-actions {
          width: 100%;
          justify-content: space-between;
          
          .like {
            flex: 1;
            justify-content: center;
          }
          
          .edit-delete {
            flex: 1;
            justify-content: flex-end;
          }
        }
      }
    }
    
    .modal-overlay {
      padding: 12px;
      
      .modal {
        padding: 20px;
        
        .modal-footer {
          flex-direction: column;
          
          .btn {
            width: 100%;
          }
        }
      }
    }
  }
}

@media (max-width: 480px) {
  .entry-card {
    padding: 16px !important;
    
    .entry-head {
      .entry-title-wrap {
        flex-direction: column;
        align-items: flex-start !important;
        gap: 8px !important;
      }
      
      .entry-info {
        flex-direction: column;
        align-items: flex-start !important;
        gap: 8px !important;
      }
      
      .entry-actions {
        flex-direction: column;
        align-items: stretch !important;
        
        .like, .edit-delete {
          width: 100%;
        }
        
        .edit-delete {
          button {
            width: 100%;
          }
        }
      }
    }
  }
}
</style>