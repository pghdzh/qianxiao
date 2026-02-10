<template>
  <div class="chisaki-page" aria-live="polite">
    <!-- 弦线动态背景 -->
    <div class="string-background">
      <div
        v-for="i in 24"
        :key="i"
        class="string"
        :style="stringStyle(i)"
      ></div>
    </div>

    <main class="container" ref="mainContainer">
      <!-- HERO 区域 -->
      <header class="hero" role="banner">
        <div class="left-col">
          <div class="avatar-card">
            <div class="avatar-wrap">
              <!-- 使用提供的头像图片 -->
              <img
                src="@/assets/images1/slty (4).webp"
                class="avatar"
                loading="lazy"
                decoding="async"
                alt="千咲角色头像"
              />
            </div>
            <div class="avatar-meta">
              <span class="avatar-badge">共鸣能力：解弦之眼</span>
              <span class="avatar-badge secondary">武器：长刃剪刀</span>
              <span class="avatar-badge accent">属性：湮灭</span>
            </div>
          </div>
        </div>
        <div class="right-col" ref="metaCard">
          <div class="hero-panel">
            <div class="name-section">
              <h1 class="name">
                朽叶千咲
                <span class="jp">Kuchiba Chisa</span>
              </h1>
              <div class="title-decoration">
                <div class="scissors-left"></div>
                <div class="scissors-right"></div>
              </div>
            </div>
            <p class="subtitle">
              "唯真挚的情感永远无法量化和剪裁——正因掌握着「切断」，才更加珍惜「相连」"
            </p>
            <p class="lead">
              "普通学生而已。" 她平静地自我介绍，瞳中隐约散溢出斑斓的光。
              于是万事万物的结构于视野里渐次明晰，她微微眯起眼睛，从中辨认出牵引着性命的那一根弦。
            </p>
            <div class="hero-tags">
              <span class="tag">星炬学院</span>
              <span class="tag">战术开发学科群</span>
              <span class="tag">归墟港市</span>
              <span class="tag">电锯模式</span>
            </div>
          </div>
        </div>
      </header>

      <!-- 共鸣状况 -->
      <section id="resonance-status" class="card resonance-status">
        <div class="card-head">
          <h2><span class="card-icon">⚡</span> 共鸣状况 · 解弦之眼</h2>
          <p class="card-subtitle">
            ＞＞＞权限已确认，关联档案已启封。欢迎，监视官███＞＞＞
          </p>
        </div>
        <div class="status-content">
          <div class="status-grid">
            <div class="status-item">
              <h3>检测报告：NHA-解弦-017</h3>
              <div class="status-details">
                <p><strong>受试者：</strong>朽叶 千咲</p>
                <p><strong>共鸣类型：</strong>突变型</p>
                <p><strong>声痕位置：</strong>右上臂</p>
                <p><strong>检测机构：</strong>苇原-新穗波市-共鸣者监测中心</p>
              </div>
            </div>
            <div class="status-item">
              <h3>能力描述</h3>
              <div class="ability-desc">
                <p>
                  经测试，对象具备超常结构认知能力，可先验地洞悉所见物体的结构、形态以及运动方式。
                </p>
                <p>
                  共鸣状态下，能以目视形式解析物体的脆弱点，并实施精准干涉，达成结构性破坏。
                </p>
                <p class="remark">
                  备注：对象自诉"观察到了令事物崩解的弦线"。初步分析判明，"弦线"并非固有存在的实体，而是仅在其视野中出现的虚像。
                </p>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- 操作手法与核心机制 -->
      <section id="combat-guide" class="card combat-guide">
        <div class="card-head">
          <h2><span class="card-icon">⚙</span> 战斗指南 · 操作手法</h2>
        </div>
        <div class="guide-content">
          <div class="guide-section">
            <h3>核心机制</h3>
            <ul class="mechanic-list">
              <li>千咲处于非电锯模式下，可获得【特殊能量】。</li>
              <li>
                【特殊能量】充满时，可施放强化共鸣技能【齿轨轮回】进入电锯模式。
              </li>
              <li>
                千咲施放电锯普攻需消耗【特殊能量】，消耗越多，下次【锯环·终结】伤害越高。
              </li>
              <li>电锯模式下免疫打断效果，增强生存与输出能力。</li>
            </ul>
          </div>

          <div class="guide-section">
            <h3>推荐输出流程</h3>
            <div class="combo-flow">
              <div class="combo-step">
                <span class="step-number">1</span>
                <span class="step-desc">变奏入场</span>
              </div>
              <div class="combo-arrow">→</div>
              <div class="combo-step">
                <span class="step-number">2</span>
                <span class="step-desc">EAAAAA</span>
              </div>
              <div class="combo-arrow">→</div>
              <div class="combo-step">
                <span class="step-number">3</span>
                <span class="step-desc">R-E【齿轨轮回】</span>
              </div>
              <div class="combo-arrow">→</div>
              <div class="combo-step">
                <span class="step-number">4</span>
                <span class="step-desc">A（长按三段）【锯环·疾攻】</span>
              </div>
              <div class="combo-arrow">→</div>
              <div class="combo-step">
                <span class="step-number">5</span>
                <span class="step-desc">【锯环·终结】</span>
              </div>
              <div class="combo-arrow">→</div>
              <div class="combo-step">
                <span class="step-number">6</span>
                <span class="step-desc">Q（声骸）</span>
              </div>
            </div>
            <div class="skill-legend">
              <p>
                <strong>技能缩写：</strong>普攻 = A，重击 = Z，共鸣技能 =
                E，共鸣解放 = R，声骸 = Q
              </p>
            </div>
          </div>
        </div>
      </section>

      <!-- 剧情时间线 -->
      <section id="story-timeline" class="card story-timeline">
        <div class="card-head">
          <h2><span class="card-icon">📅</span> 剧情时间线 · 千咲的成长轨迹</h2>
        </div>
        <div class="timeline-container">
          <div class="timeline-line"></div>

          <div class="timeline-items">
            <div
              class="timeline-item"
              v-for="(event, index) in timelineEvents"
              :key="index"
              :style="{ '--item-order': index }"
            >
              <div class="timeline-marker">
                <div class="marker-circle"></div>
                <div class="marker-year">{{ event.year }}</div>
              </div>
              <div class="timeline-content">
                <h3 class="event-title">{{ event.title }}</h3>
                <p class="event-desc">{{ event.description }}</p>
                <div class="event-tags">
                  <span
                    class="event-tag"
                    v-for="tag in event.tags"
                    :key="tag"
                    >{{ tag }}</span
                  >
                </div>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- 人物故事 -->
      <section id="character-stories" class="card character-stories">
        <div class="card-head">
          <h2><span class="card-icon">📖</span> 千咲纪事 · 人物故事</h2>
        </div>
        <div class="stories-container">
          <div class="story-navigation">
            <button
              v-for="(story, index) in stories"
              :key="story.id"
              class="story-tab"
              :class="{ active: currentStory === index }"
              @click="selectStory(index)"
            >
              {{ story.title }}
            </button>
          </div>

          <div class="story-content">
            <transition name="story-fade" mode="out-in">
              <div class="story-panel" :key="currentStory">
                <h3 class="story-title">{{ stories[currentStory].title }}</h3>
                <div class="story-text">
                  {{ stories[currentStory].content }}
                </div>
                <div class="story-meta">
                  <span class="story-period">{{
                    stories[currentStory].period
                  }}</span>
                  <div class="story-controls">
                    <button
                      class="story-nav-btn"
                      @click="prevStory"
                      :disabled="currentStory === 0"
                    >
                      ← 上一篇
                    </button>
                    <span class="story-counter"
                      >{{ currentStory + 1 }} / {{ stories.length }}</span
                    >
                    <button
                      class="story-nav-btn"
                      @click="nextStory"
                      :disabled="currentStory === stories.length - 1"
                    >
                      下一篇 →
                    </button>
                  </div>
                </div>
              </div>
            </transition>
          </div>
        </div>
      </section>

      <!-- 共鸣链详解 -->
      <section id="resonance-chain" class="card resonance-chain">
        <div class="card-head">
          <h2><span class="card-icon">⟡</span> 共鸣链详解 · 解弦之力</h2>
        </div>
        <div class="chain-container">
          <div class="chain-visual">
            <div class="chain-line"></div>
            <div class="chain-nodes">
              <div
                class="chain-node"
                v-for="(node, idx) in resonanceChain"
                :key="idx"
                :style="{
                  left: `${(idx / (resonanceChain.length - 1)) * 100}%`,
                }"
                @click="selectChainNode(idx)"
              >
                <div class="node-circle">{{ idx + 1 }}</div>
                <div class="node-label">{{ node.title }}</div>
              </div>
            </div>
          </div>

          <div class="chain-details" v-if="selectedNode !== null">
            <div class="detail-card">
              <div class="detail-header">
                <span class="detail-chain">第{{ selectedNode + 1 }}链</span>
                <h3 class="detail-title">
                  {{ resonanceChain[selectedNode].title }}
                </h3>
              </div>
              <div class="detail-effect">
                {{ resonanceChain[selectedNode].effect }}
              </div>
              <div class="detail-tags">
                <span
                  class="effect-tag"
                  v-for="tag in resonanceChain[selectedNode].tags"
                  :key="tag"
                >
                  {{ tag }}
                </span>
              </div>
            </div>
          </div>

          <div class="chain-list" v-else>
            <div
              class="chain-item"
              v-for="(node, idx) in resonanceChain"
              :key="idx"
              @click="selectChainNode(idx)"
            >
              <div class="chain-item-header">
                <span class="chain-item-number">第{{ idx + 1 }}链</span>
                <h4 class="chain-item-title">{{ node.title }}</h4>
              </div>
              <p class="chain-item-effect">{{ node.effect }}</p>
            </div>
          </div>
        </div>
      </section>

      <!-- 个人物品（使用提供的图片） -->
      <section id="personal-items" class="card personal-items">
        <div class="card-head">
          <h2><span class="card-icon">✂</span> 弦外之物 · 个人物品</h2>
        </div>
        <div class="items-container">
          <div class="item-card" v-for="item in personalItems" :key="item.id">
            <div class="item-image">
              <img
                :src="item.image"
                :alt="item.title"
                loading="lazy"
                decoding="async"
                class="item-img"
              />
              <div class="item-glow"></div>
            </div>
            <div class="item-content">
              <h3 class="item-title">{{ item.title }}</h3>
              <p class="item-desc">{{ item.description }}</p>
            </div>
          </div>
        </div>
      </section>
    </main>

    <!-- 浮动元素 -->
    <div class="floating-elements">
      <div
        class="floating-scissors"
        v-for="i in 8"
        :key="i"
        :style="floatingScissorStyle(i)"
      >
        ✂
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from "vue";

// 弦线背景样式
const stringStyle = (index: number) => {
  const delay = index * 0.15;
  const duration = 2.5 + Math.random() * 2;
  const height = 40 + Math.random() * 60;
  const opacity = 0.05 + Math.random() * 0.1;

  return {
    animationDelay: `${delay}s`,
    animationDuration: `${duration}s`,
    height: `${height}%`,
    opacity: opacity.toString(),
    left: `${(index / 24) * 100}%`,
  };
};

// 浮动剪刀样式
const floatingScissorStyle = (index: number) => {
  const size = 0.8 + Math.random() * 1.2;
  const speed = 8 + Math.random() * 12;
  const delay = Math.random() * 5;
  const x = Math.random() * 100;
  const y = Math.random() * 100;

  return {
    fontSize: `${size}em`,
    animationDuration: `${speed}s`,
    animationDelay: `${delay}s`,
    left: `${x}%`,
    top: `${y}%`,
    opacity: (0.2 + Math.random() * 0.3).toString(),
  };
};

// 剧情时间线数据
const timelineEvents = ref([
  {
    year: "幼年",
    title: "祭典之夜",
    description:
      "六岁的夏天，趴在父亲肩头参加烟火晚会，母亲从神社为她求来御守，许下守护的诺言。",
    tags: ["童年", "家庭", "御守"],
  },
  {
    year: "12岁",
    title: "孤独的成长",
    description:
      "父母的忙碌让家庭变得安静，开始学会独立生活，尝试制作香柠寿喜锅。",
    tags: ["成长", "独立", "料理"],
  },
  {
    year: "15岁",
    title: "能力的觉醒",
    description:
      "为保护被霸凌的同学，第一次使用剪刀对抗他人，同时心因性视觉识别障碍伴随共鸣能力觉醒。",
    tags: ["觉醒", "霸凌", "视觉障碍"],
  },
  {
    year: "16岁",
    title: "监测颈环",
    description:
      "戴上共鸣者监测中心的颈环，遭受社会歧视，但在帮助小男孩找回风筝时感受到温暖。",
    tags: ["监测", "歧视", "善意"],
  },
  {
    year: "17岁",
    title: "星炬学院",
    description: "转学到拉海诺地区的星炬学院，结识新朋友，开始新的学习生活。",
    tags: ["转学", "新起点", "学院"],
  },
  {
    year: "18岁",
    title: "穗波市循环",
    description:
      "意外被困在穗波市索诺拉，与澄夏一起在日复一日的循环中寻找真相。",
    tags: ["循环", "穗波市", "澄夏"],
  },
  {
    year: "现在",
    title: "新的未来",
    description:
      "离开永恒夏日，在拉海洛开始新的生活，重新融入正常的学习与社交。",
    tags: ["新生活", "未来", "希望"],
  },
]);

// 人物故事数据
const stories = ref([
  {
    id: 1,
    title: "故事开始的夏天",
    period: "六岁 · 祭典",
    content: `六岁的夏天是在父亲肩头度过的。

夏夜的风里混杂着烟火的气息与摊位上烤鱿鱼的酱香，小千咲一手握着棉花糖，一手伸向遥遥夜空——金鱼形状的灯笼在晚风中游动，斑斓花火猝然绽放，她仿佛看到人们手腕上所系的无数红线游曳延伸、继而交汇，于喧闹祭典上空织成温暖的网。

“小千刚刚许了什么愿望？”母亲仰头问她，眼中温柔地倒映着花火的影子。她鼓着沾了苹果糖浆的腮帮自豪宣布：“要像妈妈一样……剪最漂亮的剪纸！要把所有珍贵的、幸福的东西都框起来——”

十二岁的那个夏天，世界忽然安静下来。

父亲隔三岔五拖着行李箱出差，母亲的值班表贴满了冰箱门，久而久之，千咲已然习惯电话那端以忙音收场的冷淡回应。

“没关系，我自己也能照顾好自己。”小小的千咲在心里轻声自语。她郑重地打开锅盖，决定尝试做一份香柠寿喜锅。一张料理宣传页正静静摊在桌上，上边印着行大大的字：适用于全家围坐的治愈料理。

锅中渐渐沸腾，咕嘟咕嘟翻滚着牛肉与野蔬的香气，弥散至整个屋内。热气氤氲，模糊了窗上的玻璃，千咲侧过脸，雾气中只映出自己模糊单薄的身影，身边没有笑语，也没有清脆的碰杯声。

“全家围坐”。
她想，多么温柔而遥远的词。`,
  },
  {
    id: 2,
    title: "漫长梅雨季",
    period: "十五岁 · 觉醒",
    content: `十五岁的夏天，充斥着铅笔屑与孤独。

事情发生在梅雨暂歇的午后。她穿过学校中庭去图书馆还书，路过器材室时，听到门后传来的压抑呜咽。透过门缝，她看见同班同学律子被三个女生围在墙角，衬衫上溅满墨水，眼镜摔在地上，为首者正用火烧灼律子制服上的姓名绣线，空气中弥漫着化纤灼烧的酸味。

千咲想起实验课上被钉在解剖盘里的青蛙，四肢被固定，胸腔里的心脏却仍在徒劳跳动。身体先于思考行动了——她推开虚掩的门，翻出随身携带的那柄小剪刀。

曾经伴随母亲度过整个职业生涯、并最终被她送至自己手里的剪刀，第一次指向了某个人的面孔。
“放开她。”千咲说。

报复来得很快。第二天她的课桌上就刻满了“蠢货”，储物柜里塞满被撕碎的教科书。最残忍的是那些模糊的脸——当试图辨认是谁伸脚绊倒自己时，她才后知后觉所有人的五官都融化成了蜡像般的色块。医生的诊断报告上写着“心因性视觉识别障碍”，伴随着共鸣能力的被动觉醒，大脑也擅自剪断了辨认人脸的那根线。

在那个瞬间，她确实听到了某种纤细的破裂声——甚至不像是神经断裂的动静，而是更接近童年祭典上，金鱼灯笼被树枝割断丝线时的轻响。那些曾经鲜活的、连接着人与人之间牵绊的温暖丝线，在此刻骤然断裂，崩落一地不堪的碎片。

最后一张写满恶毒咒话的纸条在剪刀下粉碎时，暴雨正冲刷着教学楼的玻璃窗。千咲凝视着窗面上自己模糊晃动的倒影，忽然明白所谓成长，不过就是学会独自修剪生命中所有错位的线：无论是祭典上缠绕枝头的灯笼线，父母逐渐松手令其远行的风筝线，还是如今凝固在霸凌者嘴角的恶意曲线。

梅雨季很长，蝉声同夏日热浪一齐涌来。她握紧剪刀，开始设想为自己裁剪出一条新的道路：一条或许孤独，但无愧于心、亦无需回头的路。`,
  },
  {
    id: 3,
    title: "看不清的轮廓",
    period: "十六岁 · 监测",
    content: `十六岁的夏天，天边铺开长达数里的云翳，风雨交加。千咲站在共鸣者监测中心的落地玻璃下，闷重雷鸣裹住她的鼓膜，耳中断断续续地嗡鸣。
“听清楚了吗？”那人似乎不满她的态度。千咲抬起头，对方的五官在她视野中诡异地模糊成一片虚无。
“听清楚了。”她对着这片虚无说。

颈环扣上脖颈，带着监控器械特有的冰凉。

等雨势终于变小了一些，她启程回家。拐进街心公园时，千咲撞见滑梯旁有个小男孩正对着断裂的风筝线哭泣，在她眼中，那风筝卡在树梢的形态清晰得惊人：三根主枝交汇处形成天然锁扣，只要切断最右侧那根细枝——

“别动。”她轻声制止男孩试图拉扯风筝线的动作。男孩瞪大眼睛，随着“咔擦”一声轻响，他眼看着树枝被裁断，风筝轻飘飘落下，坠入自己怀中。
“谢谢姐……”男孩的欢呼被尖锐的女声打断，“怪……怪物！离我家孩子远点！”

循着女人惊恐的目光，千咲这才后知后觉刚扣上脖颈的那个装置。

……没什么。该习惯了。
她看得清风筝的骨架、雨滴下落的轨迹、甚至云层中电荷流动的脉络，却并不能看清人脸与人心。

就在要转身走向雨幕时，衣角突然被拽住。男孩挣脱母亲跑来，将一包星星形状的糖果塞进她手心：“姐姐这个给你！老师说勇敢的人才会被奖励星花糖，所以……”

塑料糖纸在她掌心发出细碎响声。千咲凝视着男孩——虽然五官依旧模糊，但她像是看见周遭晕染开暖色的光晕，像小时候祭典上环绕灯笼的流萤。
“妈妈说戴这个颈环的都是坏人，”男孩凑近小声说，“但我不相信，我觉得姐姐刚才剪下树枝的样子超级酷！”

千咲顿了顿。她蹲下身，摸了摸孩子的头。

颈间的装置平静无声，此刻她的共鸣波谱正呈现出罕见的稳定曲线。就像被雨水洗过的夏日天空，终于透出一丝光亮。`,
  },
  {
    id: 4,
    title: "第二个月的梦魇",
    period: "穗波市 · 循环",
    content: `“……！”
千咲从梦魇中猛然苏醒。冷汗浸透了后背，恍惚中腰侧仍尖锐发痛，撕扯着神经。
但皮肉并没有如梦中那般被割开。她深吸一口气，试图摁下几欲跳出胸腔的心脏。

被困在穗波市的第二个月。肉身已渐渐习惯这样日复一日的循环，精神却被噩梦愈发拽下深渊。
千咲侧过脸，看向沙发上仍在熟睡的澄夏。对方的手微微滑落，手边还摊开着写满数字与文字的笔记本。
外边大雨倾盆。

等澄夏醒来，天色已趋近夜晚。雨势渐弱，墨蓝色的天幕上散落着几粒疏星，像谁随手撒下的银钉，遥远而冰冷。澄夏打着哈欠，睡眼惺忪地跟在千咲后面踏入便利店，千咲在货架间仔细挑选着日用品和食物，一转头，却发现澄夏已经聚精会神地研究起头顶的广告灯牌，笔记本和笔都不知何时掏了出来。

“千咲，你看这个——”澄夏头也不回，声音因兴奋而压得很低，“和我们上次来的时候一模一样，连灯牌坏掉的时间、闪烁的频率都分毫不差！只要再多收集几次样本，记下这些数据……我们绝对能摸清这些'循环'背后的规律。”

千咲看着她转过头来时亮亮的眼睛，下意识攥紧掌心。那里仿佛还残留着昨日剪断残象弦线时的触感，一种冰凉的、如同撕裂丝绸般的震颤。

每当剪刀出鞘、共鸣能力催动之时，绯色丝线便从视野中悉数浮现——那是只有她的眼睛能窥见的生命脉络。不同于人类温热的血管，这些源于残象身上的线散发着腐朽的味道，他们追逐和撕扯穗波市中的「往人」，与悲鸣一同将这些鲜活性命送入新一轮“循环”——

千咲凝视着玻璃中的倒影。少女的脸庞冷静得近乎透明，大概唯有眼底燃着一丝未熄灭的火星。

昨天，也是在这个便利店，她又遇到了那个小女孩——那个每次循环都会与她相遇、并开心地把零食分享给自己的小女孩，浑然不觉自己身上已经烙下了那代表「往人」身份的虹彩色。或许正是这些微小却固执的善意，让她始终不愿接受这座城已经永远滞留过去的结论。

夜很漫长，但天总会亮。
在光芒终于穿过铅云的刹那，她仿佛听到弦线震颤的清鸣。千咲轻轻按住胸口，知道当下一个循环开始时，一切又会重归原状。

但没关系。
只要剪刀还在手中，只要那些破碎的温度尚未被完全遗忘……
她就会继续走下去，直到某天彻底剪开这个梦魇，与困局。`,
  },
  {
    id: 5,
    title: "穿过夏日尽头",
    period: "十八岁 · 拉海洛",
    content: `午后，窗外响起淅淅沥沥的小雨声。
千咲坐在深空文献中心的落地窗边。十八岁的这个午后，她终于有闲暇去分辨雨声的韵律——不再是穗波市那种日复一日永恒循环的雨滴频率，拉海洛真正的、带着清甜气息与生机的雨，会随意打湿晾在阳台的校服，也会在放课路上猝不及防地浇透路过的学生。

托盘上的咖啡升起袅袅白雾，她剪开方糖包装袋……是的，她依然随身带着那把剪刀，如今更多用来修剪盆栽或拆快递。糖粒坠入棕褐色的咖啡中，窗外有几个穿明黄色雨衣的学生踩着水洼跑过，那抹鲜亮刺破灰蒙雨幕，让她回想起那些无数次直面轮回与起点的日子——有人曾与她共步迈出无形回廊的终末，一同劈开漫漫长夜；有人曾只身趟过不计其数的错路与谬误，将自己变成验证真相的砝码。

雨势渐歇，千咲抱着书本走出文献中心。湿润的晚风撩起她新剪的头发，眼前豁然开朗，学院在雨后流转着与生气交融的微光，有几个工学部的学生正围着一个悬浮机械装置激烈讨论，有人认出她来，隔着雨霁的薄雾欢快招手，邀她加入讨论——

她望着这些年轻而炽热的生命，这些与她共享同一片天空、要奔向同一片星海的人，忽然再次清晰意识到她不再是那个被困在循环中的独行者，她的人生在短暂停滞后，已经重新汇入这条奔涌向前的河流。

原来真的已经离开那个漫无尽头的夏天了。
她向前一步，走向新的未来。`,
  },
]);

const currentStory = ref(0);

const selectStory = (index: number) => {
  currentStory.value = index;
};

const prevStory = () => {
  if (currentStory.value > 0) {
    currentStory.value--;
  }
};

const nextStory = () => {
  if (currentStory.value < stories.value.length - 1) {
    currentStory.value++;
  }
};

// 共鸣链数据
const resonanceChain = ref([
  {
    title: "穿行于荒芜长廊",
    effect: `千咲施放锯环·疾攻、锯环·终结、电锯模式·闪避反击期间，免疫打断。

千咲施加虚无绞痕时，触发下述效果：
·自身攻击提升30%，持续15秒。
·对目标造成61803点固定湮灭伤害，最多可将目标生命扣除至61.80%，同一目标最多可受到1次该伤害。此次伤害为普攻伤害，不受伤害加成影响。`,
    tags: ["免疫打断", "攻击提升", "固定伤害"],
  },
  {
    title: "织作牵绊的弦网",
    effect: `造成伤害无视目标10%湮灭伤害抗性。
附近队伍中的角色处于虚湮之线状态时，全属性伤害加成提升50%。`,
    tags: ["无视抗性", "全属性加成", "团队增益"],
  },
  {
    title: "踱过长夜的迷惘",
    effect: `锯环·疾攻、电锯模式·闪避反击、锯环·终结的伤害倍率提升120%，该倍率提升效果与万缕·汇终的倍率提升效果相互叠加。
锯环·疾攻、电锯模式·闪避反击消耗【锯环残响】提供的锯环·终结倍率增加效果提升120%，该倍率提升效果与万缕·汇终的倍率提升效果相互叠加。
锯环·疾攻、电锯模式·闪避反击、锯环·终结破共振度能力提升50%。`,
    tags: ["伤害倍率", "破盾增强", "输出提升"],
  },
  {
    title: "斩断循环的劫章",
    effect: `虚无绞痕效果替换为：
拥有虚无绞痕的目标受到角色直接造成的伤害时，千咲为目标附加1层【虚湮效应】，该效果每1秒最多触发1次。`,
    tags: ["效果替换", "虚湮效应", "持续伤害"],
  },
  {
    title: "万盏灯火将照亮归途所向",
    effect: `共鸣解放即刻·归无的伤害加成提升100%。
命弦·掠行消耗的【命弦·本流】降低50%。`,
    tags: ["伤害加成", "消耗降低", "资源优化"],
  },
  {
    title: "由此重铸希望，与天光",
    effect: `千咲施放锯环·疾攻、锯环·终结、电锯模式·闪避反击期间，受到致死伤害时不会倒下，最少保留1点生命。
虚无绞痕强化为虚无绞痕·终焉，获得下述效果：
·虚无绞痕·终焉拥有虚无绞痕的所有效果。
·拥有虚无绞痕·终焉的目标受到异常效应伤害加深30%。
·拥有虚无绞痕·终焉的目标受到千咲伤害提升40%。`,
    tags: ["免死", "效果强化", "生存增强"],
  },
]);

const selectedNode = ref<number | null>(null);

const selectChainNode = (index: number) => {
  selectedNode.value = selectedNode.value === index ? null : index;
};

import item1 from "./img/item1.png";
import item2 from "./img/item2.png";
import item3 from "./img/item3.png";
import food from "./img/food.png";
// 个人物品数据（使用提供的图片路径）
const personalItems = ref([
  {
    id: 1,
    image: item1,
    title: "旧制剪刀",
    description: `一柄款式稍显古旧的剪刀，数十年前在市场上还颇为常见，近年已渐渐消匿了影子。

它曾剪开过校服上的松线，也剪开过包裹生日礼物的缎带。命运总喜欢在微不足道之物中暗暗埋下伏笔，每一次开合，都重复着她生命里无数次的“结束”与“伊始”——剪开松线、剪裂缎带、剪碎结构、剪断循环。

所有的割舍与崩毁，并非终结，而是新的开端。`,
  },
  {
    id: 2,
    image: item2,
    title: "红绳发带",
    description: `有人说红色绳线象征着守护，也有人相信它是驱除邪祟的符物，但在更古老的传说里，红线意味着与人结缘。

无形的羁绊，被牵系于命途之上，悄然把她与他人、与某个未知的“未来”，牢牢系在一起。

自此，旅途纵是漫长孤寂，也将伴随着并肩共行的回响。`,
  },
  {
    id: 3,
    image: item3,
    title: "「结」",
    description: `她还记得那一天。雨后天色初霁，一碧如洗，父母握着她的手轻轻摇动神社内的钟绳，在杳杳钟声中，母亲将一个御守结系在她手腕上。

“它会一直护佑着你，小千。”

年幼的千咲晃了晃手腕，看结穗随风轻荡，仿佛在回应她尚未懂得的誓言。

“不论你走去多远的地方，不论何时，你都可以回到我们身边。”`,
  },
  {
    id: 4,
    image: food,
    title: "香柠寿喜锅",
    description:
      "清香柠檬融入浓郁汤底，酸甜解腻，鲜嫩回甘，是千咲擅长的料理。每逢冬日，窝在被炉内看着歌会，香味自锅中溢满房间，那是她记忆中最舒适的时间。",
  },
]);

onMounted(() => {
  // 初始化动画效果
  const observerOptions = {
    threshold: 0.1,
    rootMargin: "0px 0px -50px 0px",
  };

  const observer = new IntersectionObserver((entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        entry.target.classList.add("animated");
      }
    });
  }, observerOptions);

  // 观察所有卡片
  document.querySelectorAll(".card").forEach((el) => {
    observer.observe(el);
  });
});
</script>

<style scoped lang="scss">
/* 千咲风格配色 */
$deep-bg: #0a0a14;
$deep-2: #1a0a14;
$accent-1: #c62828;
$accent-2: #f44336;
$accent-dark: #8e0000;
$black: #0a0a0a;
$gray-dark: #1a1a1a;
$gray-light: #2a2a2a;
$white: #f5f5f5;
$gold: #ffd700;

$shadow-core: rgba(198, 40, 40, 0.15);
$shadow-deep: rgba(10, 10, 20, 0.8);

.chisaki-page {
  position: relative;
  min-height: 100vh;
  background: $deep-bg;
  color: $white;
  font-family: "Segoe UI", "Microsoft YaHei", sans-serif;
  overflow-x: hidden;
  padding: 20px;

  /* 弦线背景 */
  .string-background {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    z-index: 1;

    .string {
      position: absolute;
      bottom: 0;
      width: 1px;
      background: linear-gradient(to top, transparent, $accent-1, transparent);
      animation: stringFloat ease-in-out infinite alternate;

      &::before {
        content: "";
        position: absolute;
        top: 0;
        left: -1px;
        width: 3px;
        height: 3px;
        background: $accent-2;
        border-radius: 50%;
        box-shadow: 0 0 8px $accent-2;
      }
    }

    @keyframes stringFloat {
      0% {
        transform: translateY(0) translateX(0);
        opacity: 0.1;
      }
      100% {
        transform: translateY(-20px) translateX(5px);
        opacity: 0.3;
      }
    }
  }

  .container {
    position: relative;
    z-index: 2;
    max-width: 1200px;
    margin: 0 auto;
    padding-top: 40px;
  }
}

/* HERO 区域 */
.hero {
  display: grid;
  grid-template-columns: 280px 1fr;
  gap: 40px;
  margin-bottom: 60px;
  padding: 30px;
  background: linear-gradient(
    145deg,
    rgba(26, 10, 20, 0.7) 0%,
    rgba(10, 10, 20, 0.9) 100%
  );
  border-radius: 20px;
  border: 1px solid rgba(198, 40, 40, 0.2);
  box-shadow: 0 20px 60px $shadow-deep;
  backdrop-filter: blur(10px);

  .left-col {
    .avatar-card {
      background: rgba(26, 26, 26, 0.6);
      border-radius: 16px;
      padding: 20px;
      border: 1px solid rgba(198, 40, 40, 0.3);
      text-align: center;

      .avatar-wrap {
        width: 100%;
        margin-bottom: 20px;

        .avatar {
          width: 100%;
          height: auto;
          border-radius: 12px;
          border: 2px solid rgba(198, 40, 40, 0.4);
          box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
          transition: transform 0.3s ease;

          &:hover {
            transform: scale(1.02);
          }
        }
      }
    }

    .avatar-meta {
      display: flex;
      flex-direction: column;
      gap: 10px;

      .avatar-badge {
        background: rgba(198, 40, 40, 0.2);
        color: $white;
        padding: 8px 16px;
        border-radius: 20px;
        font-size: 0.9rem;
        border: 1px solid rgba(244, 67, 54, 0.3);

        &.secondary {
          background: rgba(26, 26, 26, 0.8);
        }

        &.accent {
          background: rgba(244, 67, 54, 0.25);
        }
      }
    }
  }

  .right-col {
    .hero-panel {
      .name-section {
        position: relative;
        margin-bottom: 20px;

        .name {
          font-size: 3.5rem;
          font-weight: 300;
          letter-spacing: 2px;
          margin: 0 0 10px 0;
          background: linear-gradient(135deg, $white 30%, $accent-2 70%);
          -webkit-background-clip: text;
          background-clip: text;
          -webkit-text-fill-color: transparent;
          line-height: 1.2;

          .jp {
            display: block;
            font-size: 1.2rem;
            color: rgba(244, 67, 54, 0.8);
            margin-top: 5px;
            font-weight: 400;
          }
        }

        .title-decoration {
          position: relative;
          height: 2px;
          margin: 20px 0;

          .scissors-left,
          .scissors-right {
            position: absolute;
            top: 50%;
            width: 60px;
            height: 2px;
            background: linear-gradient(90deg, $accent-1, transparent);

            &::before,
            &::after {
              content: "";
              position: absolute;
              width: 20px;
              height: 2px;
              background: $accent-1;
              transform-origin: left center;
            }
          }

          .scissors-left {
            left: 0;
            transform: translateY(-50%);

            &::before {
              top: -8px;
              transform: rotate(45deg);
            }

            &::after {
              top: 8px;
              transform: rotate(-45deg);
            }
          }

          .scissors-right {
            right: 0;
            transform: translateY(-50%);

            &::before {
              top: -8px;
              right: 0;
              transform-origin: right center;
              transform: rotate(-45deg);
            }

            &::after {
              top: 8px;
              right: 0;
              transform-origin: right center;
              transform: rotate(45deg);
            }
          }
        }
      }

      .subtitle {
        font-size: 1.3rem;
        color: $accent-2;
        font-style: italic;
        margin: 0 0 20px 0;
        line-height: 1.6;
        text-shadow: 0 0 10px rgba(244, 67, 54, 0.3);
      }

      .lead {
        font-size: 1.1rem;
        color: rgba(245, 245, 245, 0.9);
        line-height: 1.8;
        margin-bottom: 25px;
      }

      .hero-tags {
        display: flex;
        flex-wrap: wrap;
        gap: 10px;

        .tag {
          background: rgba(26, 26, 26, 0.6);
          color: rgba(245, 245, 245, 0.9);
          padding: 6px 15px;
          border-radius: 15px;
          font-size: 0.9rem;
          border: 1px solid rgba(42, 42, 42, 0.5);
        }
      }
    }
  }
}

/* 通用卡片样式 */
.card {
  background: linear-gradient(
    145deg,
    rgba(26, 10, 20, 0.6) 0%,
    rgba(10, 10, 20, 0.8) 100%
  );
  border-radius: 16px;
  padding: 30px;
  margin-bottom: 40px;
  border: 1px solid rgba(198, 40, 40, 0.2);
  box-shadow: 0 15px 40px $shadow-deep;
  backdrop-filter: blur(10px);
  opacity: 0;
  transform: translateY(20px);
  transition: all 0.6s ease;

  &.animated {
    opacity: 1;
    transform: translateY(0);
  }

  .card-head {
    margin-bottom: 30px;
    padding-bottom: 15px;
    border-bottom: 1px solid rgba(244, 67, 54, 0.2);

    h2 {
      font-size: 1.8rem;
      font-weight: 600;
      margin: 0;
      color: $white;
      display: flex;
      align-items: center;
      gap: 10px;

      .card-icon {
        color: $accent-2;
      }
    }

    .card-subtitle {
      color: rgba(244, 67, 54, 0.8);
      margin: 10px 0 0 0;
      font-size: 1.1rem;
    }
  }
}

/* 共鸣状况样式 */
.resonance-status {
  .status-content {
    .status-grid {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
      gap: 30px;

      .status-item {
        h3 {
          color: $accent-2;
          margin-bottom: 15px;
          font-size: 1.4rem;
        }

        .status-details {
          p {
            margin-bottom: 8px;
            color: rgba(245, 245, 245, 0.9);

            strong {
              color: $accent-2;
            }
          }
        }

        .ability-desc {
          p {
            margin-bottom: 12px;
            line-height: 1.7;
            color: rgba(245, 245, 245, 0.9);

            &.remark {
              font-style: italic;
              color: rgba(244, 67, 54, 0.8);
              border-left: 2px solid rgba(244, 67, 54, 0.3);
              padding-left: 15px;
              margin-top: 20px;
            }
          }
        }
      }
    }
  }
}

/* 操作手法样式 */
.combat-guide {
  .guide-content {
    .guide-section {
      margin-bottom: 35px;

      &:last-child {
        margin-bottom: 0;
      }

      h3 {
        color: $accent-2;
        margin-bottom: 15px;
        font-size: 1.4rem;
        padding-left: 10px;
        border-left: 3px solid $accent-2;
      }

      .mechanic-list {
        list-style: none;
        padding: 0;

        li {
          padding: 10px 0 10px 25px;
          position: relative;
          color: rgba(245, 245, 245, 0.9);
          line-height: 1.6;
          border-bottom: 1px dashed rgba(42, 42, 42, 0.5);

          &:last-child {
            border-bottom: none;
          }

          &::before {
            content: "✂";
            position: absolute;
            left: 0;
            color: $accent-2;
          }
        }
      }

      .combo-flow {
        display: flex;
        flex-wrap: wrap;
        align-items: center;
        gap: 10px;
        margin: 20px 0;

        .combo-step {
          display: flex;
          flex-direction: column;
          align-items: center;
          background: rgba(26, 26, 26, 0.6);
          padding: 12px 15px;
          border-radius: 10px;
          border: 1px solid rgba(244, 67, 54, 0.2);
          min-width: 100px;

          .step-number {
            color: $accent-2;
            font-weight: bold;
            margin-bottom: 5px;
          }

          .step-desc {
            font-size: 0.9rem;
            color: rgba(245, 245, 245, 0.9);
          }
        }

        .combo-arrow {
          color: $accent-2;
          font-weight: bold;
        }
      }

      .skill-legend {
        background: rgba(26, 26, 26, 0.4);
        padding: 15px;
        border-radius: 8px;
        margin-top: 15px;

        p {
          margin: 0;
          color: rgba(245, 245, 245, 0.8);

          strong {
            color: $accent-2;
          }
        }
      }
    }
  }
}

/* 剧情时间线样式 */
.story-timeline {
  .timeline-container {
    position: relative;
    padding: 20px 0 40px;

    .timeline-line {
      position: absolute;
      left: 100px;
      top: 0;
      bottom: 0;
      width: 2px;
      background: linear-gradient(
        to bottom,
        transparent,
        $accent-2,
        $accent-1,
        transparent
      );

      @media (max-width: 768px) {
        left: 30px;
      }
    }

    .timeline-items {
      .timeline-item {
        display: flex;
        margin-bottom: 40px;
        position: relative;

        transition: all 0.6s ease;
        transition-delay: calc(var(--item-order, 0) * 0.1s);

        &:last-child {
          margin-bottom: 0;
        }

        .timeline-marker {
          width: 100px;
          flex-shrink: 0;
          display: flex;
          flex-direction: column;
          align-items: center;
          position: relative;

          .marker-circle {
            width: 20px;
            height: 20px;
            background: $accent-2;
            border-radius: 50%;
            border: 3px solid $gray-dark;
            box-shadow: 0 0 15px rgba(244, 67, 54, 0.5);
            position: relative;
            z-index: 2;
          }

          .marker-year {
            margin-top: 10px;
            color: $accent-2;
            font-weight: bold;
            font-size: 1.1rem;
          }
        }

        .timeline-content {
          flex: 1;
          padding-left: 30px;
          background: rgba(26, 26, 26, 0.5);
          border-radius: 12px;
          padding: 20px;
          border: 1px solid rgba(42, 42, 42, 0.5);
          transition: all 0.3s ease;

          &:hover {
            border-color: rgba(244, 67, 54, 0.3);
            transform: translateY(-5px);
          }

          .event-title {
            color: $white;
            margin: 0 0 10px 0;
            font-size: 1.3rem;
          }

          .event-desc {
            color: rgba(245, 245, 245, 0.8);
            line-height: 1.6;
            margin-bottom: 15px;
          }

          .event-tags {
            display: flex;
            flex-wrap: wrap;
            gap: 8px;

            .event-tag {
              background: rgba(198, 40, 40, 0.2);
              color: rgba(245, 245, 245, 0.9);
              padding: 4px 12px;
              border-radius: 12px;
              font-size: 0.85rem;
              border: 1px solid rgba(244, 67, 54, 0.3);
            }
          }
        }

        @media (max-width: 768px) {
          .timeline-marker {
            width: 60px;
          }

          .timeline-content {
            padding-left: 20px;
          }
        }
      }
    }
  }
}

/* 人物故事样式 */
.character-stories {
  .stories-container {
    .story-navigation {
      display: flex;
      flex-wrap: wrap;
      gap: 10px;
      margin-bottom: 30px;

      .story-tab {
        background: rgba(26, 26, 26, 0.5);
        border: 1px solid rgba(42, 42, 42, 0.5);
        color: rgba(245, 245, 245, 0.8);
        padding: 10px 20px;
        border-radius: 20px;
        cursor: pointer;
        transition: all 0.3s ease;
        font-size: 0.95rem;

        &:hover {
          background: rgba(198, 40, 40, 0.2);
          border-color: rgba(244, 67, 54, 0.3);
        }

        &.active {
          background: rgba(244, 67, 54, 0.25);
          border-color: $accent-2;
          color: $white;
        }
      }
    }

    .story-content {
      min-height: 400px;

      .story-panel {
        .story-title {
          color: $accent-2;
          margin-bottom: 20px;
          font-size: 1.6rem;
        }

        .story-text {
          white-space: pre-wrap;
        }

        .story-meta {
          display: flex;
          justify-content: space-between;
          align-items: center;
          margin-top: 30px;
          padding-top: 20px;
          border-top: 1px solid rgba(42, 42, 42, 0.5);

          .story-period {
            color: rgba(244, 67, 54, 0.8);
            font-style: italic;
          }

          .story-controls {
            display: flex;
            align-items: center;
            gap: 20px;

            .story-nav-btn {
              background: rgba(26, 26, 26, 0.6);
              border: 1px solid rgba(244, 67, 54, 0.3);
              color: rgba(245, 245, 245, 0.9);
              padding: 8px 16px;
              border-radius: 6px;
              cursor: pointer;
              transition: all 0.3s ease;

              &:hover:not(:disabled) {
                background: rgba(244, 67, 54, 0.2);
              }

              &:disabled {
                opacity: 0.5;
                cursor: not-allowed;
              }
            }

            .story-counter {
              color: rgba(245, 245, 245, 0.7);
              font-size: 0.9rem;
            }
          }
        }
      }
    }
  }
}

/* 故事切换动画 */
.story-fade-enter-active,
.story-fade-leave-active {
  transition: all 0.4s ease;
}

.story-fade-enter-from {
  opacity: 0;
  transform: translateY(10px);
}

.story-fade-enter-to {
  opacity: 1;
  transform: translateY(0);
}

.story-fade-leave-from {
  opacity: 1;
  transform: translateY(0);
}

.story-fade-leave-to {
  opacity: 0;
  transform: translateY(-10px);
}

/* 共鸣链样式 */
.resonance-chain {
  .chain-container {
    .chain-visual {
      position: relative;
      height: 120px;
      margin-bottom: 40px;

      .chain-line {
        position: absolute;
        top: 50%;
        left: 0;
        right: 0;
        height: 2px;
        background: linear-gradient(
          90deg,
          transparent,
          $accent-1,
          $accent-2,
          $accent-1,
          transparent
        );
        transform: translateY(-50%);
      }

      .chain-nodes {
        position: relative;
        height: 100%;

        .chain-node {
          position: absolute;
          top: 50%;
          transform: translate(-50%, -50%);
          cursor: pointer;
          text-align: center;

          .node-circle {
            width: 50px;
            height: 50px;
            background: $gray-dark;
            border: 2px solid $accent-2;
            border-radius: 50%;
            display: flex;
            align-items: center;
            justify-content: center;
            color: $white;
            font-weight: 600;
            font-size: 1.2rem;
            transition: all 0.3s ease;
            margin: 0 auto 10px;

            &:hover {
              background: $accent-2;
              transform: scale(1.1);
              box-shadow: 0 0 25px rgba(244, 67, 54, 0.5);
            }
          }

          .node-label {
            font-size: 0.9rem;
            color: rgba(245, 245, 245, 0.8);
            white-space: nowrap;
            max-width: 120px;
            overflow: hidden;
            text-overflow: ellipsis;
          }
        }
      }
    }

    .chain-details {
      .detail-card {
        background: rgba(26, 26, 26, 0.6);
        border-radius: 12px;
        padding: 25px;
        border: 1px solid rgba(244, 67, 54, 0.3);

        .detail-header {
          display: flex;
          align-items: center;
          gap: 15px;
          margin-bottom: 20px;

          .detail-chain {
            background: rgba(198, 40, 40, 0.25);
            color: $white;
            padding: 6px 15px;
            border-radius: 15px;
            font-size: 0.9rem;
            border: 1px solid rgba(244, 67, 54, 0.3);
          }

          .detail-title {
            font-size: 1.4rem;
            color: $accent-2;
            margin: 0;
          }
        }

        .detail-effect {
          color: rgba(245, 245, 245, 0.9);
          line-height: 1.7;
          font-size: 1.05rem;
          margin-bottom: 20px;
          white-space: pre-wrap;
        }

        .detail-tags {
          display: flex;
          flex-wrap: wrap;
          gap: 10px;

          .effect-tag {
            background: rgba(26, 26, 26, 0.8);
            color: rgba(244, 67, 54, 0.9);
            padding: 5px 12px;
            border-radius: 12px;
            font-size: 0.85rem;
            border: 1px solid rgba(244, 67, 54, 0.2);
          }
        }
      }
    }

    .chain-list {
      .chain-item {
        background: rgba(26, 26, 26, 0.5);
        border-radius: 12px;
        padding: 20px;
        margin-bottom: 15px;
        border-left: 4px solid $accent-2;
        cursor: pointer;
        transition: all 0.3s ease;

        &:hover {
          transform: translateX(5px);
          background: rgba(26, 26, 26, 0.7);
        }

        .chain-item-header {
          display: flex;
          align-items: center;
          gap: 15px;
          margin-bottom: 10px;

          .chain-item-number {
            background: rgba(198, 40, 40, 0.2);
            color: $white;
            padding: 4px 12px;
            border-radius: 12px;
            font-size: 0.9rem;
          }

          .chain-item-title {
            font-size: 1.2rem;
            color: $white;
            margin: 0;
          }
        }

        .chain-item-effect {
          color: rgba(245, 245, 245, 0.8);
          line-height: 1.6;
          margin: 0;
          padding-left: 10px;
          white-space: pre-wrap;
        }
      }
    }
  }
}

/* 个人物品样式（使用图片） */
.personal-items {
  .items-container {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
    gap: 30px;

    .item-card {
      background: rgba(26, 26, 26, 0.5);
      border-radius: 16px;
      overflow: hidden;
      border: 1px solid rgba(42, 42, 42, 0.5);
      transition: all 0.3s ease;
      display: flex;
      flex-direction: column;

      &:hover {
        transform: translateY(-8px);
        border-color: rgba(244, 67, 54, 0.3);
        box-shadow: 0 20px 40px rgba(0, 0, 0, 0.4);

        .item-img {
          transform: scale(1.05);
        }

        .item-glow {
          opacity: 1;
        }
      }

      .item-image {
        position: relative;
        height: 200px;
        overflow: hidden;

        .item-img {
          width: 100%;
          height: 100%;
          object-fit: cover;
          transition: transform 0.5s ease;
        }

        .item-glow {
          position: absolute;
          top: 0;
          left: 0;
          right: 0;
          bottom: 0;
          background: radial-gradient(
            circle at center,
            rgba(244, 67, 54, 0.2) 0%,
            transparent 70%
          );
          opacity: 0;
          transition: opacity 0.3s ease;
          pointer-events: none;
        }
      }

      .item-content {
        padding: 20px;
        flex: 1;
        display: flex;
        flex-direction: column;

        .item-title {
          font-size: 1.4rem;
          color: $white;
          margin: 0 0 12px 0;
          font-weight: 600;
          border-bottom: 1px solid rgba(244, 67, 54, 0.2);
          padding-bottom: 10px;
        }

        .item-desc {
          color: rgba(245, 245, 245, 0.8);
          line-height: 1.6;
          margin: 0;
          flex: 1;
          white-space: pre-wrap;
        }
      }
    }
  }
}

/* 浮动元素 */
.floating-elements {
  position: fixed;
  inset: 0;
  pointer-events: none;
  z-index: 3;

  .floating-scissors {
    position: absolute;
    color: $accent-2;
    animation: floatScissors linear infinite;
    filter: drop-shadow(0 0 8px rgba(244, 67, 54, 0.3));
    opacity: 0.3;
  }

  @keyframes floatScissors {
    0% {
      transform: translateY(0) rotate(0deg) scale(1);
    }
    25% {
      transform: translateY(-20px) rotate(90deg) scale(1.1);
    }
    50% {
      transform: translateY(-40px) rotate(180deg) scale(1);
    }
    75% {
      transform: translateY(-20px) rotate(270deg) scale(0.9);
    }
    100% {
      transform: translateY(0) rotate(360deg) scale(1);
    }
  }
}

/* 响应式设计 */
@media (max-width: 1024px) {
  .hero {
    grid-template-columns: 1fr;
    gap: 30px;
  }

  .chain-visual {
    display: none;
  }

  .combo-flow {
    flex-direction: column !important;
    align-items: stretch !important;

    .combo-arrow {
      transform: rotate(90deg);
    }
  }

  .items-container {
    grid-template-columns: repeat(2, 1fr) !important;
  }
}

@media (max-width: 768px) {
  .chisaki-page {
    padding: 10px;
  }

  .hero {
    padding: 20px;

    .right-col .hero-panel .name-section .name {
      font-size: 2.5rem;
    }
  }

  .card {
    padding: 20px;
  }

  .story-navigation {
    overflow-x: auto;
    flex-wrap: nowrap !important;
    padding-bottom: 10px;

    .story-tab {
      white-space: nowrap;
    }
  }

  .story-controls {
    flex-direction: column;
    gap: 10px !important;
  }

  .items-container {
    grid-template-columns: 1fr !important;
  }

  .timeline-item {
    flex-direction: column !important;

    .timeline-content {
      margin-top: 15px;
      padding-left: 20px !important;
    }
  }

  .timeline-line {
    left: 30px !important;
  }
}

@media (max-width: 480px) {
  .name-section .name {
    font-size: 2rem !important;
  }

  .card-head h2 {
    font-size: 1.5rem !important;
  }

  .status-grid {
    grid-template-columns: 1fr !important;
  }
}
</style>