<template>
  <div class="gallery-container">
    <button class="upload-btn" @click="openUploadModal">上传图片</button>

    <section class="gallery section">
      <div class="sort-controls">
        <button @click="toggleSort" class="sort-btn">
          按 {{ sortBy === "like_count" ? "点赞量" : "最新上传" }} 排序
        </button>
      </div>
      <div class="gallery-grid">
        <div
          v-for="(img, index) in images"
          :key="img.id"
          class="card"
          @click="openLightbox(index)"
          ref="cards"
        >
          <div class="card-inner">
            <img
              :src="img.src"
              :alt="img.alt"
              loading="lazy"
              @load="onImageLoad($event)"
            />
            <div class="overlay">
              <span>查看大图</span>
            </div>
            <button class="like-btn" @click.stop="handleLike(img)">
              <i class="heart" :class="{ liked: img.liked }"></i>
              <span class="like-count">{{ img.likeCount }}</span>
            </button>
          </div>
        </div>
      </div>
      <!-- sentinel：用于触发无限滚动 -->
      <div ref="sentinel" class="sentinel"></div>
      <!-- 可选：加载中/结束提示 -->
      <div class="loading" v-if="loading">加载中...</div>
      <div class="finished" v-if="finished">已全部加载</div>
    </section>
    <aside class="ranking-panel">
      <div class="panel-header" @click="expanded = !expanded">
        <h3 class="ranking-title">排行榜</h3>
        <span>共{{ imgTotal }}张</span>
        <span class="toggle-icon">{{ expanded ? "▾" : "▸" }}</span>
      </div>
      <transition name="fade">
        <ul v-if="expanded" class="ranking-list">
          <li
            v-for="(item, idx) in rankingList"
            :key="idx"
            class="ranking-item"
            :class="`rank-${idx + 1}`"
          >
            <span class="rank">{{ idx + 1 }}</span>
            <span class="name">{{ item.nickname }}</span>
            <span class="count">{{ item.count }} 张</span>
          </li>
        </ul>
      </transition>
    </aside>
    <!-- Lightbox Modal -->
    <div v-if="lightboxOpen" class="lightbox" @click.self="closeLightbox">
      <span class="close" @click="closeLightbox">✕</span>
      <span class="prev" @click.stop="prevImage">‹</span>
      <img :src="images[currentIndex].src" :alt="images[currentIndex].alt" />
      <span class="next" @click.stop="nextImage">›</span>
    </div>

    <!-- 上传弹窗 -->
    <div
      v-if="uploadModalOpen"
      class="upload-modal-overlay"
      @click.self="closeUploadModal"
    >
      <div class="upload-modal">
        <h3>批量上传图片</h3>
        <div class="tip-container">
          <ul class="tips-list">
            <li>
              审核规则： 1.不要色情倾向（不要露三点，我怕被封）
              2.要我能认出是千咲。
            </li>
            <li>
              由于没有用户系统，我这边不好做审核反馈，但只要显示上传成功，我这边肯定能收到。
            </li>
            <li>
              如果图片数量较多请在b站私信联系我给我网盘链接，因为我云服务器比较小一次性上传太多图片可能会导致上传不上，感谢理解。
            </li>
            <li>
              因为审核上传一次比较麻烦，所以审核时间不定，最晚一周，感谢谅解。
            </li>
          </ul>
        </div>
        <p class="stats">
          今日已上传：<strong>{{ uploadedToday }}</strong> 张，
          剩余可上传：<strong>{{ remaining }}</strong> 张
        </p>
        <label>
          昵称：
          <input v-model="nickname" type="text" placeholder="请输入昵称" />
        </label>
        <label>
          选择图片（最多 {{ remaining }} 张）：
          <input
            ref="fileInput"
            type="file"
            multiple
            accept="image/*"
            @change="handleFileSelect"
          />
        </label>
        <p class="tip" v-if="selectedFiles.length">
          已选 {{ selectedFiles.length }} 张
        </p>
        <div class="modal-actions">
          <button :disabled="!canSubmit || isUploading" @click="submitUpload">
            {{ isUploading ? "上传中..." : "立即上传" }}
          </button>
          <button class="cancel" @click="closeUploadModal">取消</button>
        </div>
      </div>
    </div>

    <!-- <div class="floating-chibis">
      <img
        v-for="(pet, i) in chibiList"
        :key="i"
        :src="pet.src"
        :style="{ top: pet.top + 'px', left: pet.left + 'px' }"
        class="chibi-img"
      />
    </div> -->
  </div>
</template>

<script lang="ts" setup>
import { ref, onMounted, computed, nextTick, onBeforeUnmount } from "vue";
import { uploadImages } from "@/api/modules/images"; // 前面封装的上传接口
import { getRankingList } from "@/api/modules/ranking"; // 根据你的实际路径调整
import { gsap } from "gsap"; // ← 本地引入
import { getImagesLikesList, likeImage } from "@/api/modules/imagesLikes";
import { debounce } from "lodash";

const sortBy = ref<"uploaded_at" | "like_count">("like_count");
const order = ref<"asc" | "desc">("desc");
function toggleSort() {
  if (sortBy.value === "uploaded_at") {
    sortBy.value = "like_count";
    order.value = "desc";
  } else {
    sortBy.value = "uploaded_at";
    order.value = "desc";
  }
  pageImage.value = 1;
  images.value = [];
  finished.value = false;
  window.scrollTo(0, 0);
  loadNextPage();
}
// 获取已点赞 ID 数组
function getLikedIds(): number[] {
  const data = localStorage.getItem("likedImageIds");
  return data ? JSON.parse(data) : [];
}

// 保存已点赞 ID 数组
function setLikedIds(ids: number[]) {
  localStorage.setItem("likedImageIds", JSON.stringify(ids));
}

async function handleLike(img: ImageItem) {
  if (img.liked) return; // 已点过就不重复调用

  try {
    await likeImage(img.id); // 调用后端接口
    img.likeCount += 1; // 本地更新点赞数
    img.liked = true; // 标记已点赞

    // 更新 localStorage
    const likedIds = getLikedIds();
    likedIds.push(img.id);
    setLikedIds(likedIds);
  } catch (error) {
    console.error("点赞失败", error);
    alert("点赞失败，请稍后重试");
  }
}

interface ImageItem {
  src: string;
  alt: string;
  likeCount: number;
  id: number;
  liked: Boolean;
}

interface RankingItem {
  id?: number; // 如果接口返回有 id，可加上
  nickname: string;
  count: number;
}
const rankingList = ref<RankingItem[]>([]);
const expanded = ref(true);

// 默认分页参数（如不分页可省略）
const page = 1;
const pageSize = 99;

const fetchRanking = async () => {
  const res = await getRankingList({
    page,
    pageSize,
    character_key: "qianxiao",
  });
  if (res.success) {
    rankingList.value = res.data;
  } else {
    console.error("获取排行榜失败", res.message);
  }
};

// 响应式存放最终图片列表
const images = ref<ImageItem[]>([]);

const pageImage = ref(1);
const limit = ref(10);
const loading = ref(false);
const finished = ref(false);

const sentinel = ref<HTMLElement | null>(null);

// 1. 在外层创建一个单例 observerCard
const observerCard = new IntersectionObserver(
  (entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        entry.target.classList.add("visible");
        observerCard.unobserve(entry.target);
      }
    });
  },
  { threshold: 0.1 }
);
// 2. 每次有新卡片时，都调用这个方法去挂载观察
async function observeNewCards(startIndex = 0) {
  await nextTick();
  const cards = document.querySelectorAll<HTMLElement>(".card");
  for (let i = startIndex; i < cards.length; i++) {
    observerCard.observe(cards[i]);
  }
}
const imgTotal = ref(0);
const fixImageUrl = (url: string): string => {
  if (url.includes('127.0.0.1')) {
    // 将 127.0.0.1 替换为当前页面的完整源（协议+域名）
    return url.replace('http://127.0.0.1', window.location.origin);
  }
  return url;
};

async function loadNextPage() {
  if (loading.value || finished.value) return;
  loading.value = true;
  try {
    const res = await getImagesLikesList({
      page: pageImage.value,
      limit: limit.value,
      sortBy: sortBy.value,
      character_key: "qianxiao",
      order: order.value,
    });
    imgTotal.value = res.total;
    const likedIds = getLikedIds();
    const list = (
      res.images as Array<{ url: string; like_count: number; id: number }>
    ).map((item) => ({
      src: fixImageUrl(item.url),
      alt: "",
      likeCount: item.like_count,
      id: item.id, // 如果需要的话，方便点赞用
      liked: likedIds.includes(item.id),
    }));
    if (list.length === 0) {
      finished.value = true;
      return;
    }
    // 记录加载前的长度，方便后面找出“新增”节点
    const oldLength = images.value.length;
    const existingIds = new Set(images.value.map((i) => i.id));
    const filtered = list.filter((item) => !existingIds.has(item.id));
    images.value.push(...filtered);
    pageImage.value++;

    observeNewCards(oldLength);
  } catch (err) {
    console.error(err);
  } finally {
    loading.value = false;
  }
}

// 3. 给 loadNextPage 包装一个防抖版
const debouncedLoad = debounce(
  () => {
    loadNextPage();
  },
  200,
  { leading: true, trailing: false }
);

const lightboxOpen = ref(false);
const currentIndex = ref(0);

function openLightbox(index: number) {
  currentIndex.value = index;
  lightboxOpen.value = true;
}
function closeLightbox() {
  lightboxOpen.value = false;
}
function prevImage() {
  currentIndex.value =
    (currentIndex.value + images.value.length - 1) % images.value.length;
}
function nextImage() {
  currentIndex.value = (currentIndex.value + 1) % images.value.length;
}

// 渐显＆Blur‑Up 效果
function onImageLoad(e: Event) {
  const img = e.target as HTMLImageElement;
  const card = img.closest(".card");
  card?.classList.add("loaded");
}

// 上传弹窗逻辑

const uploadModalOpen = ref(false);
const nickname = ref("");
const fileInput = ref<HTMLInputElement>();
const selectedFiles = ref<File[]>([]);

// 从 localStorage 读取“今天”已上传数量
function getTodayKey() {
  return `uploaded_${new Date().toISOString().slice(0, 10)}`;
}
const uploadedToday = ref<number>(
  Number(localStorage.getItem(getTodayKey()) || 0)
);
const remaining = computed(() => Math.max(27 - uploadedToday.value, 0));

// 控制提交按钮
const canSubmit = computed(() => {
  return (
    nickname.value.trim().length > 0 &&
    selectedFiles.value.length > 0 &&
    selectedFiles.value.length <= remaining.value
  );
});

// 放在 script 顶部，或者 utils 里
function clearOldUploadRecords() {
  const today = new Date();
  const storage = window.localStorage;
  for (const key of Object.keys(storage)) {
    if (!key.startsWith("uploaded_")) continue;

    // key 格式 uploaded_YYYY-MM-DD
    const dateStr = key.slice("uploaded_".length);
    const recordDate = new Date(dateStr);
    if (isNaN(recordDate.getTime())) continue;

    // 计算相差天数
    const diffMs = today.getTime() - recordDate.getTime();
    const diffDays = diffMs / (1000 * 60 * 60 * 24);

    // 如果超过 2 天，就删掉
    if (diffDays > 2) {
      storage.removeItem(key);
    }
  }
}

function openUploadModal() {
  clearOldUploadRecords();
  nickname.value = "";
  selectedFiles.value = [];
  if (fileInput.value) fileInput.value.value = "";
  // 每次打开重新刷新已上传数
  uploadedToday.value = Number(localStorage.getItem(getTodayKey()) || 0);
  uploadModalOpen.value = true;
}
function closeUploadModal() {
  uploadModalOpen.value = false;
}

// 本地截断到剩余数量
function handleFileSelect(e: Event) {
  const files = Array.from((e.target as HTMLInputElement).files || []);

  if (!files) return;

  const validFiles: File[] = [];
  for (const file of files) {
    if (file.size > 20 * 1024 * 1024) {
      alert(`文件太大：${file.name}，请控制在 20MB 内`);
      continue;
    }
    validFiles.push(file);
  }

  if (validFiles.length === 0) return;

  if (validFiles.length > remaining.value) {
    alert(
      `今天最多还能上传 ${remaining.value} 张，已为你截取前 ${remaining.value} 张`
    );
    selectedFiles.value = files.slice(0, remaining.value);
  } else {
    selectedFiles.value = files;
  }
}
const isUploading = ref(false);
async function submitUpload() {
  if (!canSubmit.value) return;
  isUploading.value = true;
  try {
    const res = await uploadImages(
      selectedFiles.value,
      nickname.value.trim(),
      "qianxiao"
    );
    const uploadedCount = res.data.length;
    // 更新 localStorage
    uploadedToday.value += uploadedCount;
    localStorage.setItem(getTodayKey(), String(uploadedToday.value));

    alert(`成功上传 ${uploadedCount} 张图片`);
    closeUploadModal();
    // …可选：刷新画廊列表或把新图片追加到 images …
  } catch (err: any) {
    console.error(err);
    alert(err.message || "上传失败");
  } finally {
    isUploading.value = false;
  }
}

interface Chibi {
  src: string;
  top: number;
  left: number;
}

const chibiList = ref<Chibi[]>([]);
let sentinelObserver: IntersectionObserver;
// Scroll-triggered lazy animation
onMounted(async () => {
  // 1. 拉排行榜
  await fetchRanking();

  // 2. 拉第一页图片并挂载动画 observer
  await loadNextPage(); // 内部会调用 observeNewCards(oldLen)
  // 对首次卡片做一次完整 observe
  observeNewCards(0);

  // 3. 初始化 sentinelObserver，再 observe
  sentinelObserver = new IntersectionObserver(
    (entries) => {
      if (entries[0].isIntersecting) debouncedLoad();
    },
    { rootMargin: "0px", threshold: 0.1 }
  );
  if (sentinel.value) {
    sentinelObserver.observe(sentinel.value);
  }
  // 1. 基础配置信息
  const total = 6;
  let pickCount = 3; // 每次抽取 3 张
  const vw = window.innerWidth;
  const vh = window.innerHeight;
  const isMobile = window.innerWidth <= 768;
  // 如果已知单张小人图片的宽高，可避免超出边界；
  // 假设小人图片宽 100px、高 100px，按需替换：
  const imgWidth = 100;
  const imgHeight = 100;

  // 2. Fisher–Yates 洗牌函数
  function shuffle(array) {
    for (let i = array.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [array[i], array[j]] = [array[j], array[i]];
    }
    return array;
  }

  // 3. 随机选出 3 个编号
  if (isMobile) {
    pickCount = 1;
  }
  const nums = shuffle(Array.from({ length: total }, (_, k) => k + 1));
  const picks = nums.slice(0, pickCount);

  // 4. 生成随机位置并填充 chibiList
  chibiList.value = []; // 先清空
  picks.forEach((i) => {
    chibiList.value.push({
      src: `/QImages/1 (${i}).png`,
      left: Math.random() * (vw - imgWidth), // 保证不超出左右边界
      top: Math.random() * (vh - imgHeight), // 保证不超出上下边界
    });
  });

  // 2. 等 img 渲染到 DOM
  await nextTick();

  // 3. 给每个小人绑定 GSAP 动画
  const imgs = document.querySelectorAll<HTMLImageElement>(".chibi-img");
  imgs.forEach((img, index) => {
    const padding = 200; // 边缘预留空间
    // ✅ 初始出场动画（闪现）
    gsap.fromTo(
      img,
      { opacity: 0, scale: 0.5 },
      {
        opacity: 1,
        scale: 1,
        duration: 0.8,
        ease: "back.out(2)",
        delay: 0.2 * index,
      }
    );

    // ✅ 鼠标靠近闪避
    img.addEventListener("mouseenter", () => {
      gsap.killTweensOf(img);

      gsap.to(img, {
        x: "+=" + ((Math.random() - 0.5) * 400).toFixed(0),
        y: "+=" + ((Math.random() - 0.5) * 400).toFixed(0),
        duration: 1.2,
        ease: "back.out(2)",
        onComplete: () => {
          // 闪避完成后，再重新启用动画
          animate(img);
        },
      });
    });

    const animate = (img: HTMLImageElement) => {
      let { x, y } = img.getBoundingClientRect();
      let deltaX = (Math.random() - 0.5) * 200;
      let deltaY = (Math.random() - 0.5) * 200;

      // 预测一下偏移后的位置
      let nextX = x + deltaX;
      let nextY = y + deltaY;

      // 校正：防漂出左、右、上、下边界
      if (nextX < padding) deltaX = padding - x;
      if (nextX + img.width > window.innerWidth - padding)
        deltaX = window.innerWidth - padding - (x + img.width);
      if (nextY < padding) deltaY = padding - y;
      if (nextY + img.height > window.innerHeight - padding)
        deltaY = window.innerHeight - padding - (y + img.height);

      gsap.to(img, {
        x: `+=${deltaX.toFixed(0)}`,
        y: `+=${deltaY.toFixed(0)}`,
        rotation: `+=${((Math.random() - 0.5) * 60).toFixed(0)}`,
        duration: 2 + Math.random() * 2,
        ease: "power1.inOut",
        onComplete: () => animate(img),
      });
    };
    animate(img);
  });
});

onBeforeUnmount(() => {
  observerCard.disconnect();
  sentinelObserver.disconnect();
  // 以及你在 onMounted 里新建的其它 Observer
});
</script>


<style lang="scss" scoped>
/* 千咲配色变量 */

@keyframes fadeInUp {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* 固定浮动小 chibi 层（保留） */
.floating-chibis {
  position: fixed;
  inset: 0;
  pointer-events: none;
  z-index: 1;
}

.chibi-img {
  position: absolute;
  width: 80px;
  user-select: none;
  transform-origin: center center;
  pointer-events: auto;
  z-index: 10;
  filter: drop-shadow(0 0 8px rgba(198, 40, 40, 0.3));
}

/* 主容器使用千咲深色渐变背景 */
.gallery-container {
  background: var(--deep-bg);
  color: var(--white);
  min-height: 100vh;
  padding-bottom: 60px;
  padding-top: 20px;
  position: relative;
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
  /* 添加深红色微弱光斑效果 */
  &::before {
    content: "";
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    height: 100vh;
    background: radial-gradient(
        circle at 20% 30%,
        rgba(198, 40, 40, 0.03) 0%,
        transparent 50%
      ),
      radial-gradient(
        circle at 80% 70%,
        rgba(255, 215, 0, 0.02) 0%,
        transparent 50%
      );
    pointer-events: none;
    z-index: 0;
  }

  .section {
    padding: 80px 20px;
    max-width: 1200px;
    margin: 0 auto;
    position: relative;
    z-index: 1;

    .sort-controls {
      margin: 24px 0 32px;

      .sort-btn {
        display: inline-flex;
        align-items: center;
        gap: 8px;
        padding: 12px 32px 12px 60px;
        font-size: 1.05rem;
        line-height: 1;
        font-family: "Noto Serif SC", serif;
        cursor: pointer;
        border-radius: 30px;
        position: relative;
        overflow: hidden;
        border: 1px solid rgba(255, 215, 0, 0.1);

        /* 暗黑底 + 深红渐变边框 */
        background: linear-gradient(
          145deg,
          rgba(10, 10, 20, 0.9),
          rgba(20, 10, 15, 0.95)
        );
        color: var(--white);
        box-shadow: 0 8px 32px rgba(0, 0, 0, 0.6),
          0 0 0 1px rgba(255, 215, 0, 0.05),
          inset 0 1px 0 rgba(255, 255, 255, 0.1);
        transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
        -webkit-tap-highlight-color: transparent;

        /* 左侧深红剪刀徽记 */
        &::after {
          content: "✂";
          position: absolute;
          left: 20px;
          top: 50%;
          transform: translateY(-50%);
          width: 24px;
          height: 24px;
          display: flex;
          align-items: center;
          justify-content: center;
          font-size: 1.2rem;
          color: var(--accent);
          text-shadow: 0 0 10px rgba(198, 40, 40, 0.5);
          transition: all 0.3s ease;
        }

        /* 左侧深红光晕 */
        &::before {
          content: "";
          position: absolute;
          left: 10px;
          top: 50%;
          transform: translateY(-50%);
          width: 44px;
          height: 44px;
          border-radius: 50%;
          background: radial-gradient(
            circle,
            rgba(198, 40, 40, 0.15),
            transparent 70%
          );
          filter: blur(8px);
          pointer-events: none;
          opacity: 0.7;
          z-index: 1;
        }

        &:hover {
          transform: translateY(-4px);
          box-shadow: 0 12px 40px rgba(198, 40, 40, 0.25),
            0 0 0 1px rgba(255, 215, 0, 0.15),
            inset 0 1px 0 rgba(255, 255, 255, 0.15);
          color: #fff;
          background: linear-gradient(
            145deg,
            rgba(15, 10, 20, 0.95),
            rgba(25, 10, 18, 0.98)
          );
        }

        &:hover::after {
          color: var(--accent-light);
          text-shadow: 0 0 15px rgba(244, 67, 54, 0.7);
          transform: translateY(-50%) rotate(15deg);
        }

        &:active {
          transform: translateY(-2px);
          box-shadow: 0 6px 24px rgba(198, 40, 40, 0.2),
            0 0 0 1px rgba(255, 215, 0, 0.1),
            inset 0 1px 0 rgba(255, 255, 255, 0.1);
        }
      }
    }

    .gallery-grid {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
      gap: 28px;
      padding: 0 4px;

      .card {
        perspective: 1000px;
        opacity: 0;
        transform: translateY(20px);
        border-radius: 12px;
        overflow: hidden;
        position: relative;

        /* 卡片边缘深红微光 */
        &::before {
          content: "";
          position: absolute;
          inset: 0;
          border-radius: 12px;
          padding: 2px;
          background: linear-gradient(
            145deg,
            transparent,
            rgba(198, 40, 40, 0.1)
          );
          -webkit-mask: linear-gradient(#fff 0 0) content-box,
            linear-gradient(#fff 0 0);
          mask: linear-gradient(#fff 0 0) content-box, linear-gradient(#fff 0 0);
          -webkit-mask-composite: xor;
          mask-composite: exclude;
          pointer-events: none;
          z-index: 2;
        }

        &.visible {
          animation: fadeInUp 0.6s ease forwards;
        }

        &.loaded {
          .card-inner img {
            filter: none;
            opacity: 1;
          }
        }

        .card-inner {
          position: relative;
          border-radius: 10px;
          overflow: hidden;
          box-shadow: 0 8px 32px rgba(10, 10, 20, 0.6),
            0 0 0 1px rgba(42, 42, 42, 0.2);
          transform-style: preserve-3d;
          transition: all 0.5s cubic-bezier(0.4, 0, 0.2, 1);
          height: 320px;

          &:hover {
            transform: rotateY(5deg) rotateX(2deg) scale(1.03);
            box-shadow: 0 16px 48px rgba(198, 40, 40, 0.3),
              0 0 0 1px rgba(255, 215, 0, 0.1);
          }

          img {
            width: 100%;
            height: 100%;
            object-fit: cover;
            display: block;
            filter: blur(15px) grayscale(20%);
            opacity: 0.9;
            transition: filter 0.7s ease, opacity 0.7s ease;
          }

          .overlay {
            position: absolute;
            bottom: 0;
            left: 0;
            right: 0;
            padding: 20px;
            background: linear-gradient(
              to top,
              rgba(10, 10, 20, 0.9) 0%,
              transparent 100%
            );
            text-align: center;
            opacity: 0;
            transform: translateY(10px);
            transition: all 0.4s ease;

            span {
              color: var(--white);
              font-family: "Noto Serif SC", serif;
              font-size: 1.1rem;
              letter-spacing: 1px;
              padding: 8px 20px;
              border-radius: 20px;
              background: rgba(10, 10, 20, 0.7);
              border: 1px solid rgba(255, 215, 0, 0.2);
              display: inline-block;
            }
          }

          &:hover .overlay {
            opacity: 1;
            transform: translateY(0);
          }

          .like-btn {
            position: absolute;
            top: 16px;
            right: 16px;
            background: rgba(10, 10, 20, 0.85);
            border: 1px solid rgba(255, 215, 0, 0.3);
            border-radius: 50%;
            cursor: pointer;
            z-index: 2;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            width: 52px;
            height: 52px;
            transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
            backdrop-filter: blur(6px);
            padding: 4px;
            box-shadow: 0 4px 12px rgba(0, 0, 0, 0.4),
              0 0 0 1px rgba(255, 215, 0, 0.1),
              inset 0 1px 0 rgba(255, 255, 255, 0.1);

            &:hover {
              transform: scale(1.08);
              background: rgba(198, 40, 40, 0.4);
              border-color: rgba(244, 67, 54, 0.5);
              box-shadow: 0 6px 20px rgba(198, 40, 40, 0.3),
                0 0 0 1px rgba(255, 215, 0, 0.2),
                inset 0 1px 0 rgba(255, 255, 255, 0.15);
            }

            .heart {
              width: 24px;
              height: 24px;
              position: relative;
              display: flex;
              align-items: center;
              justify-content: center;
              margin-bottom: 2px;

              &::before {
                content: "♡";
                position: absolute;
                font-size: 1.6rem;
                color: rgba(255, 255, 255, 0.9);
                text-shadow: 0 0 8px rgba(255, 255, 255, 0.3);
                transition: all 0.3s ease;
              }

              /* 未点赞时的微光效果 */
              &::after {
                content: "";
                position: absolute;
                top: 50%;
                left: 50%;
                width: 32px;
                height: 32px;
                background: radial-gradient(
                  circle,
                  rgba(255, 215, 0, 0.15),
                  transparent 70%
                );
                transform: translate(-50%, -50%);
                opacity: 0.5;
                filter: blur(4px);
                transition: all 0.3s ease;
              }
            }

            .liked {
              &::before {
                content: "♥";
                color: var(--accent-light);
                text-shadow: 0 0 12px rgba(244, 67, 54, 0.7),
                  0 0 20px rgba(244, 67, 54, 0.4);
                animation: heartBeat 0.8s cubic-bezier(0.4, 0, 0.2, 1);
              }

              /* 点赞时的光晕效果 */
              &::after {
                background: radial-gradient(
                  circle,
                  rgba(244, 67, 54, 0.3),
                  transparent 70%
                );
                opacity: 0.8;
                filter: blur(6px);
              }
            }

            .like-count {
              font-size: 0.75rem;
              font-weight: 600;
              color: var(--white);
              text-shadow: 0 1px 2px rgba(0, 0, 0, 0.8),
                0 0 4px rgba(0, 0, 0, 0.6);
              line-height: 1;
              margin-top: -2px;
              letter-spacing: 0.5px;
              white-space: nowrap;

              /* 数字背景增强 */
              padding: 1px 4px;
              border-radius: 4px;
              background: rgba(10, 10, 20, 0.6);
              backdrop-filter: blur(2px);
              border: 0.5px solid rgba(255, 215, 0, 0.1);
            }

            /* 点赞时的数字样式 */
            .liked + .like-count {
              color: var(--accent-light);
              text-shadow: 0 1px 2px rgba(0, 0, 0, 0.8),
                0 0 6px rgba(244, 67, 54, 0.5);
              background: rgba(142, 0, 0, 0.4);
              border-color: rgba(244, 67, 54, 0.2);
            }

            @keyframes heartBeat {
              0% {
                transform: scale(1);
                text-shadow: 0 0 8px rgba(244, 67, 54, 0.5);
              }
              20% {
                transform: scale(1.4);
                text-shadow: 0 0 20px rgba(244, 67, 54, 0.9);
              }
              40% {
                transform: scale(1.2);
                text-shadow: 0 0 15px rgba(244, 67, 54, 0.7);
              }
              60% {
                transform: scale(1.35);
                text-shadow: 0 0 18px rgba(244, 67, 54, 0.8);
              }
              80% {
                transform: scale(1.25);
                text-shadow: 0 0 16px rgba(244, 67, 54, 0.7);
              }
              100% {
                transform: scale(1.3);
                text-shadow: 0 0 12px rgba(244, 67, 54, 0.7),
                  0 0 20px rgba(244, 67, 54, 0.4);
              }
            }
          }
        }
      }
    }
  }

  .lightbox {
    position: fixed;
    inset: 0;
    background: rgba(10, 10, 20, 0.98);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;

    img {
      max-width: 85%;
      max-height: 85%;
      border-radius: 8px;
      box-shadow: 0 20px 60px rgba(0, 0, 0, 0.8),
        0 0 0 1px rgba(255, 215, 0, 0.1);
      animation: fadeInUp 0.4s ease;
    }

    .close,
    .prev,
    .next {
      position: absolute;
      color: var(--white);
      font-size: 2.5rem;
      cursor: pointer;
      user-select: none;
      padding: 16px;
      border-radius: 50%;
      transition: all 0.3s ease;
      background: rgba(10, 10, 20, 0.5);
      backdrop-filter: blur(4px);

      &:hover {
        color: var(--accent-light);
        background: rgba(198, 40, 40, 0.3);
        transform: scale(1.1);
      }
    }

    .close {
      top: 32px;
      right: 32px;
    }
    .prev {
      left: 32px;
      top: 50%;
      transform: translateY(-50%);
    }
    .next {
      right: 32px;
      top: 50%;
      transform: translateY(-50%);
    }
  }

  .upload-btn {
    position: fixed;
    bottom: 32px;
    left: 32px;
    display: inline-flex;
    align-items: center;
    gap: 12px;
    padding: 14px 28px;
    font-size: 1.05rem;
    z-index: 10;
    cursor: pointer;
    user-select: none;
    font-family: "Noto Serif SC", serif;

    color: var(--white);
    background: linear-gradient(145deg, var(--accent-dark), var(--accent));
    border-radius: 30px;
    border: 1px solid rgba(255, 215, 0, 0.2);
    box-shadow: 0 8px 32px rgba(198, 40, 40, 0.3),
      0 0 20px rgba(255, 215, 0, 0.1);
    transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);

    &::before {
      content: "↑";
      font-size: 1.2rem;
    }

    &:hover {
      transform: translateY(-4px);
      box-shadow: 0 12px 40px rgba(244, 67, 54, 0.4),
        0 0 30px rgba(255, 215, 0, 0.15);
      background: linear-gradient(145deg, var(--accent), var(--accent-light));
    }

    &:active {
      transform: translateY(-2px);
    }
  }

  /* 上传弹窗 - 千咲风格 */
  .upload-modal-overlay {
    position: fixed;
    inset: 0;
    z-index: 2000;
    display: flex;
    align-items: center;
    justify-content: center;
    background: rgba(10, 10, 20, 0.92);
    backdrop-filter: blur(8px);

    &::before {
      content: "";
      position: absolute;
      inset: 0;
      background: radial-gradient(
          circle at 20% 30%,
          rgba(198, 40, 40, 0.05) 0%,
          transparent 50%
        ),
        radial-gradient(
          circle at 80% 70%,
          rgba(255, 215, 0, 0.03) 0%,
          transparent 50%
        );
      pointer-events: none;
    }
  }

  .upload-modal {
    position: relative;
    width: 680px;
    max-width: calc(100% - 40px);
    padding: 36px;
    border-radius: 16px;
    background: linear-gradient(
      145deg,
      rgba(20, 15, 25, 0.95),
      rgba(15, 10, 20, 0.98)
    );
    border: 1px solid rgba(255, 215, 0, 0.1);
    box-shadow: 0 20px 60px rgba(0, 0, 0, 0.8), 0 0 0 1px rgba(198, 40, 40, 0.2);
    color: var(--white);
    font-family: "Noto Sans SC", sans-serif;

    h3 {
      margin: 0 0 24px 0;
      font-size: 1.6rem;
      color: var(--white);
      font-weight: 600;
      text-align: center;
      font-family: "Noto Serif SC", serif;
      position: relative;

      &::after {
        content: "";
        position: absolute;
        bottom: -8px;
        left: 50%;
        transform: translateX(-50%);
        width: 100px;
        height: 2px;
        background: linear-gradient(
          90deg,
          transparent,
          var(--accent),
          transparent
        );
      }
    }

    .tip-container {
      margin: 24px 0;
      padding: 20px;
      background: rgba(10, 10, 20, 0.5);
      border-radius: 12px;
      border-left: 4px solid var(--accent);

      .tips-list {
        list-style: none;
        margin: 0;
        padding: 0;

        li {
          position: relative;
          padding-left: 28px;
          margin-bottom: 12px;
          font-size: 0.95rem;
          color: rgba(245, 245, 245, 0.9);
          line-height: 1.5;

          &::before {
            content: "✦";
            position: absolute;
            left: 0;
            color: var(--accent-light);
            font-size: 0.9rem;
          }

          &:last-child {
            margin-bottom: 0;
          }
        }
      }
    }

    .stats {
      margin: 20px 0;
      font-size: 1rem;
      text-align: center;
      color: var(--white);

      strong {
        color: var(--accent-light);
        font-weight: 700;
      }
    }

    .tip {
      margin-top: 8px;
      text-align: right;
      font-size: 0.9rem;
      color: rgba(245, 245, 245, 0.7);
    }

    label {
      display: block;
      margin-bottom: 20px;
      font-size: 0.95rem;
      color: var(--white);

      input[type="text"],
      input[type="file"] {
        width: 100%;
        margin-top: 8px;
        padding: 14px 16px;
        border-radius: 10px;
        border: 1px solid rgba(42, 42, 42, 0.5);
        background: rgba(10, 10, 20, 0.7);
        color: var(--white);
        font-size: 0.95rem;
        outline: none;
        transition: all 0.3s ease;

        &::placeholder {
          color: rgba(245, 245, 245, 0.4);
        }

        &:focus {
          border-color: var(--accent);
          box-shadow: 0 0 0 2px rgba(198, 40, 40, 0.2);
        }
      }
    }

    .modal-actions {
      display: flex;
      justify-content: flex-end;
      gap: 16px;
      margin-top: 32px;

      button {
        padding: 12px 28px;
        border: none;
        border-radius: 10px;
        cursor: pointer;
        font-weight: 600;
        font-size: 0.95rem;
        transition: all 0.3s ease;
        min-width: 100px;
      }

      button:not(.cancel) {
        background: linear-gradient(145deg, var(--accent), var(--accent-light));
        color: var(--white);
        border: 1px solid rgba(255, 215, 0, 0.2);

        &:hover:not(:disabled) {
          transform: translateY(-2px);
          box-shadow: 0 8px 24px rgba(198, 40, 40, 0.4);
        }

        &:disabled {
          opacity: 0.5;
          cursor: not-allowed;
        }
      }

      button.cancel {
        background: transparent;
        border: 1px solid rgba(245, 245, 245, 0.3);
        color: var(--white);

        &:hover {
          background: rgba(245, 245, 245, 0.1);
        }
      }
    }
  }

  /* 右侧排行面板 - 千咲风格 */
  .ranking-panel {
    width: 240px;
    position: fixed;
    top: 82px;
    right: 32px;
    z-index: 1200;
    color: var(--white);
    font-family: "Noto Serif SC", serif;

    background: linear-gradient(
      145deg,
      rgba(15, 10, 20, 0.95),
      rgba(10, 10, 20, 0.98)
    );
    border-radius: 16px;
    border: 1px solid rgba(255, 215, 0, 0.1);
    backdrop-filter: blur(10px);
    box-shadow: 0 12px 48px rgba(0, 0, 0, 0.6), 0 0 0 1px rgba(198, 40, 40, 0.2);

    .panel-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      padding: 20px;
      cursor: pointer;
      user-select: none;
      border-bottom: 1px solid rgba(255, 215, 0, 0.1);

      .ranking-title {
        margin: 0;
        font-size: 1.2rem;
        font-weight: 600;
        color: var(--white);
        position: relative;

        &::after {
          content: "⚔";
          margin-left: 8px;
          color: var(--accent);
          font-size: 1rem;
        }
      }

      span {
        font-size: 0.9rem;
        color: rgba(245, 245, 245, 0.7);
      }

      .toggle-icon {
        font-size: 1.2rem;
        color: var(--accent-light);
        transition: transform 0.3s ease;
      }
    }

    .ranking-list {
      list-style: none;
      padding: 16px;
      margin: 0;
      max-height: 60vh;
      overflow-y: auto;

      &::-webkit-scrollbar {
        width: 6px;
      }
      &::-webkit-scrollbar-thumb {
        background: var(--accent);
        border-radius: 3px;
      }

      .ranking-item {
        display: flex;
        align-items: center;
        justify-content: space-between;
        padding: 14px;
        margin-bottom: 12px;
        border-radius: 10px;
        background: rgba(10, 10, 20, 0.5);
        border: 1px solid rgba(42, 42, 42, 0.3);
        transition: all 0.3s ease;
        position: relative;

        &:hover {
          transform: translateX(-4px);
          background: rgba(20, 15, 25, 0.7);
          border-color: rgba(198, 40, 40, 0.3);
        }

        .rank {
          width: 32px;
          height: 32px;
          display: flex;
          align-items: center;
          justify-content: center;
          border-radius: 50%;
          font-weight: 700;
          font-size: 1rem;
          flex-shrink: 0;
          margin-right: 12px;
        }

        .name {
          flex: 1;
          font-size: 0.95rem;
          color: var(--white);
          overflow: hidden;
          text-overflow: ellipsis;
          white-space: nowrap;
        }

        .count {
          font-size: 0.9rem;
          color: var(--accent-light);
          font-weight: 600;
          flex-shrink: 0;
          margin-left: 12px;
        }

        /* 冠军样式 */
        &.rank-1 {
          background: linear-gradient(
            145deg,
            rgba(142, 0, 0, 0.3),
            rgba(198, 40, 40, 0.2)
          );
          border: 1px solid rgba(255, 215, 0, 0.3);

          .rank {
            background: var(--gold);
            color: #000;
            box-shadow: 0 0 15px rgba(255, 215, 0, 0.5);
          }

          &::before {
            content: "👑";
            position: absolute;
            left: -10px;
            top: 50%;
            transform: translateY(-50%);
            font-size: 1.2rem;
            filter: drop-shadow(0 0 4px var(--gold));
          }
        }

        &.rank-2 {
          background: linear-gradient(
            145deg,
            rgba(42, 42, 42, 0.3),
            rgba(26, 26, 26, 0.2)
          );
          border: 1px solid rgba(244, 67, 54, 0.2);

          .rank {
            background: var(--accent-light);
            color: var(--white);
          }
        }

        &.rank-3 {
          background: linear-gradient(
            145deg,
            rgba(42, 42, 42, 0.25),
            rgba(26, 26, 26, 0.15)
          );
          border: 1px solid rgba(244, 67, 54, 0.1);

          .rank {
            background: var(--accent);
            color: var(--white);
          }
        }
      }
    }
  }
}

/* 响应式调整 */
@media (max-width: 1200px) {
  .ranking-panel {
    position: static;
    width: 100%;
    max-width: 400px;
   
  }

  .upload-btn {
    left: 50%;
    transform: translateX(-50%);
    bottom: 20px;
  }
}

@media (max-width: 768px) {
  .gallery-container .section {
    padding: 40px 16px;
  }

  .gallery-grid {
    grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
    gap: 20px;
  }

  .upload-modal {
    padding: 24px;
  }

  .modal-actions {
    flex-direction: column;

    button {
      width: 100%;
    }
  }
}

@media (max-width: 480px) {
  .gallery-grid {
    grid-template-columns: 1fr;
  }
}
</style>