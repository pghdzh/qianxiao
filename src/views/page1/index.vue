<template>
  <div class="chisaki-home">
    <!-- 背景弦线Canvas -->
    <canvas ref="canvasRef" class="string-canvas"></canvas>
    
    <!-- 主标题区域 -->
    <div class="title-section">
      <div class="subtitle">星炬学院 · 解弦之眼</div>
      <h1 class="main-title">千咲</h1>
      <div class="title-decoration">
        <div class="scissors-left"></div>
        <div class="scissors-right"></div>
      </div>
    </div>
    
    <!-- 中心内容区域 -->
    <div class="content-section">
      <div class="quote-box">
        <div class="quote-mark">"</div>
        <div class="quote-text">{{ currentQuote.text }}</div>
        <div class="quote-mark">"</div>
      </div>
      
 
      
      <div class="philosophy-section">
        <div class="philosophy-item">
          <div class="philosophy-icon">✂</div>
          <div class="philosophy-content">
            <h3>切断</h3>
            <p>世间万物皆可解构，问题终将被剪裁</p>
          </div>
        </div>
        <div class="philosophy-item">
          <div class="philosophy-icon">⟡</div>
          <div class="philosophy-content">
            <h3>相连</h3>
            <p>正因掌握切断之力，才更珍视相连之情</p>
          </div>
        </div>
      </div>
    </div>
    
    <!-- 页脚 -->
    <footer class="footer">
      <div class="footer-content">
        <div class="footer-quote">"普通学生而已。"</div>
        <div class="footer-info">© 星炬学院档案 · 千咲</div>
      </div>
    </footer>
    
    <!-- 浮动元素 -->
    <div class="floating-scissors" v-for="(scissor, index) in floatingScissors" 
         :key="index" :style="getScissorStyle(scissor)">✂</div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

// Canvas相关
const canvasRef = ref<HTMLCanvasElement | null>(null)
let animationFrameId: number
let ctx: CanvasRenderingContext2D | null = null

// 弦线数据
interface StringLine {
  x: number
  y: number
  length: number
  thickness: number
  speed: number
  opacity: number
  sway: number
  swaySpeed: number
  swayPhase: number
  isCut: boolean
  cutProgress: number
  hue: number // 0-360 色调
}

const stringLines = ref<StringLine[]>([])

// 初始化弦线
const initStringLines = () => {
  if (!canvasRef.value) return
  
  const width = canvasRef.value.width
  const height = canvasRef.value.height
  const lineCount = 25
  
  stringLines.value = []
  
  for (let i = 0; i < lineCount; i++) {
    stringLines.value.push({
      x: Math.random() * width,
      y: Math.random() * height,
      length: 40 + Math.random() * 100,
      thickness: 0.5 + Math.random() * 1.5,
      speed: 0.5 + Math.random() * 2,
      opacity: 0.1 + Math.random() * 0.2,
      sway: 5 + Math.random() * 15,
      swaySpeed: 0.5 + Math.random() * 1.5,
      swayPhase: Math.random() * Math.PI * 2,
      isCut: Math.random() > 0.7,
      cutProgress: Math.random(),
      hue: 340 + Math.random() * 20 // 红色调范围
    })
  }
}

// 绘制弦线
const drawStringLine = (line: StringLine, time: number) => {
  if (!ctx) return
  
  const swayOffset = Math.sin(time * line.swaySpeed + line.swayPhase) * line.sway
  
  // 计算剪断位置
  const cutY = line.length * line.cutProgress
  
  // 绘制上半部分（如果存在）
  if (cutY > 0) {
    const gradient = ctx.createLinearGradient(
      line.x + swayOffset, line.y,
      line.x + swayOffset, line.y + cutY
    )
    
    // 使用HSV到RGB转换来获得红色调
    const hue = line.hue
    const saturation = 0.7
    const lightness = 0.5 + line.opacity * 0.3
    
    // 简化版的HSL到RGB转换
    const rgb = hslToRgb(hue / 360, saturation, lightness)
    
    gradient.addColorStop(0, `rgba(${rgb[0]}, ${rgb[1]}, ${rgb[2]}, ${line.opacity})`)
    gradient.addColorStop(1, `rgba(${rgb[0]}, ${rgb[1]}, ${rgb[2]}, ${line.opacity * 0.3})`)
    
    ctx.strokeStyle = gradient
    ctx.lineWidth = line.thickness
    ctx.lineCap = 'round'
    
    ctx.beginPath()
    ctx.moveTo(line.x + swayOffset, line.y)
    ctx.lineTo(line.x + swayOffset, line.y + cutY)
    ctx.stroke()
  }
  
  // 绘制下半部分（如果未被完全剪断）
  if (cutY < line.length) {
    const gradient2 = ctx.createLinearGradient(
      line.x + swayOffset, line.y + cutY,
      line.x + swayOffset, line.y + line.length
    )
    
    const hue = line.hue
    const saturation = 0.5
    const lightness = 0.3
    
    const rgb = hslToRgb(hue / 360, saturation, lightness)
    
    gradient2.addColorStop(0, `rgba(${rgb[0]}, ${rgb[1]}, ${rgb[2]}, ${line.opacity * 0.5})`)
    gradient2.addColorStop(1, `rgba(${rgb[0]}, ${rgb[1]}, ${rgb[2]}, 0)`)
    
    ctx.strokeStyle = gradient2
    ctx.lineWidth = line.thickness * 0.7
    
    ctx.beginPath()
    ctx.moveTo(line.x + swayOffset, line.y + cutY)
    ctx.lineTo(line.x + swayOffset, line.y + line.length)
    ctx.stroke()
  }
  
  // 在剪断点绘制金色光点
  if (line.isCut && line.cutProgress > 0.1 && line.cutProgress < 0.9) {
    const glowSize = 3 + Math.sin(time * 5) * 2
    const gradient3 = ctx.createRadialGradient(
      line.x + swayOffset, line.y + cutY, 0,
      line.x + swayOffset, line.y + cutY, glowSize
    )
    gradient3.addColorStop(0, 'rgba(255, 215, 0, 0.8)')
    gradient3.addColorStop(1, 'rgba(255, 215, 0, 0)')
    
    ctx.fillStyle = gradient3
    ctx.beginPath()
    ctx.arc(line.x + swayOffset, line.y + cutY, glowSize, 0, Math.PI * 2)
    ctx.fill()
  }
}

// HSL转RGB辅助函数
const hslToRgb = (h: number, s: number, l: number): [number, number, number] => {
  let r, g, b
  
  if (s === 0) {
    r = g = b = l
  } else {
    const hue2rgb = (p: number, q: number, t: number) => {
      if (t < 0) t += 1
      if (t > 1) t -= 1
      if (t < 1/6) return p + (q - p) * 6 * t
      if (t < 1/2) return q
      if (t < 2/3) return p + (q - p) * (2/3 - t) * 6
      return p
    }
    
    const q = l < 0.5 ? l * (1 + s) : l + s - l * s
    const p = 2 * l - q
    
    r = hue2rgb(p, q, h + 1/3)
    g = hue2rgb(p, q, h)
    b = hue2rgb(p, q, h - 1/3)
  }
  
  return [Math.round(r * 255), Math.round(g * 255), Math.round(b * 255)]
}

// 更新弦线位置
const updateStringLines = (time: number) => {
  if (!canvasRef.value) return
  
  stringLines.value.forEach(line => {
    // 移动弦线
    line.y += line.speed
    
    // 如果弦线移出画布，重置到顶部
    if (line.y > canvasRef.value!.height + line.length) {
      line.y = -line.length
      line.x = Math.random() * canvasRef.value!.width
      line.isCut = Math.random() > 0.7
      line.cutProgress = Math.random()
    }
    
    // 更新剪断动画
    if (line.isCut && line.cutProgress < 1) {
      line.cutProgress += 0.005
    }
    
    // 随机剪断弦线
    if (!line.isCut && Math.random() < 0.001) {
      line.isCut = true
      line.cutProgress = 0
    }
  })
}

// Canvas动画循环
const startCanvasAnimation = () => {
  if (!canvasRef.value) return
  
  const canvas = canvasRef.value
  ctx = canvas.getContext('2d')
  
  if (!ctx) return
  
  const resizeCanvas = () => {
    const dpr = window.devicePixelRatio || 1
    const rect = canvas.getBoundingClientRect()
    
    canvas.width = rect.width * dpr
    canvas.height = rect.height * dpr
    
    ctx!.scale(dpr, dpr)
    
    initStringLines()
  }
  
  resizeCanvas()
  window.addEventListener('resize', resizeCanvas)
  
  let lastTime = 0
  const animate = (time: number) => {
    if (!ctx || !canvasRef.value) return
    
    const deltaTime = lastTime ? (time - lastTime) / 1000 : 0
    lastTime = time
    
    // 清除画布（使用半透明黑色实现拖尾效果）
    ctx.fillStyle = 'rgba(10, 10, 20, 0.05)'
    ctx.fillRect(0, 0, canvasRef.value.width, canvasRef.value.height)
    
    // 更新并绘制弦线
    updateStringLines(time / 1000)
    stringLines.value.forEach(line => {
      drawStringLine(line, time / 1000)
    })
    
    animationFrameId = requestAnimationFrame(animate)
  }
  
  animationFrameId = requestAnimationFrame(animate)
  
  onUnmounted(() => {
    window.removeEventListener('resize', resizeCanvas)
    cancelAnimationFrame(animationFrameId)
  })
}

// 浮动剪刀
interface FloatingScissor {
  x: number
  y: number
  size: number
  speed: number
  rotation: number
  rotationSpeed: number
  opacity: number
  delay: number
}

const floatingScissors = ref<FloatingScissor[]>([])

const initFloatingScissors = () => {
  floatingScissors.value = []
  
  for (let i = 0; i < 8; i++) {
    floatingScissors.value.push({
      x: Math.random() * 100,
      y: Math.random() * 100,
      size: 0.8 + Math.random() * 1.2,
      speed: 0.5 + Math.random() * 1,
      rotation: Math.random() * 360,
      rotationSpeed: (Math.random() - 0.5) * 0.5,
      opacity: 0.1 + Math.random() * 0.3,
      delay: Math.random() * 5
    })
  }
}

const getScissorStyle = (scissor: FloatingScissor) => {
  const time = Date.now() / 1000 + scissor.delay
  const x = scissor.x + Math.sin(time * scissor.speed) * 10
  const y = scissor.y + Math.cos(time * scissor.speed * 0.7) * 8
  const rotation = scissor.rotation + time * scissor.rotationSpeed * 20
  
  return {
    left: `${x}%`,
    top: `${y}%`,
    transform: `translate(-50%, -50%) rotate(${rotation}deg) scale(${scissor.size})`,
    opacity: scissor.opacity.toString(),
    fontSize: `${scissor.size * 1.5}em`,
    animationDelay: `${scissor.delay}s`
  }
}

// 名言警句
const quotes = [
  { text: "我会……切开这个死局。" },
  { text: "洞察、解析、切断——这就是我的战斗方式。" },
  { text: "大部分问题对我而言都是可以解构的数理问题。" },
  { text: "唯真挚的情感永远无法量化和剪裁。" },
  { text: "正因掌握着「切断」，才更加珍惜「相连」。" },
  { text: "严谨认真，冷静凛然的星炬学院学生。" }
]

const currentQuote = ref(quotes[0])
let quoteIndex = 0

const rotateQuote = () => {
  quoteIndex = (quoteIndex + 1) % quotes.length
  currentQuote.value = quotes[quoteIndex]
}

// 初始化
onMounted(() => {
  startCanvasAnimation()
  initFloatingScissors()
  
  // 每5秒切换一次名言
  setInterval(rotateQuote, 5000)
})
</script>

<style scoped lang="scss">
.chisaki-home {
  min-height: 100vh;
  background: linear-gradient(145deg, #0a0a14 0%, #1a0a14 35%, #0a0a1a 100%);
  color: #f5f5f5;
  font-family: 'Segoe UI', 'Microsoft YaHei', sans-serif;
  position: relative;
  overflow: hidden;
  
  .string-canvas {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 1;
    pointer-events: none;
  }
  
  .title-section {
    position: relative;
    z-index: 2;
    text-align: center;
    padding: 80px 20px 40px;
    padding-top: 180px;
    .subtitle {
      font-size: 1.2rem;
      letter-spacing: 4px;
      color: #f44336;
      margin-bottom: 10px;
      text-transform: uppercase;
      text-shadow: 0 0 10px rgba(244, 67, 54, 0.5);
    }
    
    .main-title {
      font-size: 6rem;
      font-weight: 300;
      letter-spacing: 8px;
      margin: 0;
      background: linear-gradient(135deg, #f5f5f5 30%, #f44336 70%);
      -webkit-background-clip: text;
      -webkit-text-fill-color: transparent;
      background-clip: text;
      position: relative;
      line-height: 1;
    }
    
    .title-decoration {
      position: relative;
      margin-top: 20px;
      
      .scissors-left, .scissors-right {
        position: absolute;
        top: 50%;
        width: 60px;
        height: 2px;
        background: linear-gradient(90deg, #c62828, transparent);
        
        &::before, &::after {
          content: '';
          position: absolute;
          width: 20px;
          height: 2px;
          background: #c62828;
          transform-origin: left center;
        }
      }
      
      .scissors-left {
        left: calc(50% - 100px);
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
        right: calc(50% - 100px);
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
  
  .content-section {
    position: relative;
    z-index: 2;
    max-width: 900px;
    margin: 0 auto;
    padding: 0 20px 100px;
    
    .quote-box {
      background: rgba(26, 26, 26, 0.5);
      border-radius: 12px;
      padding: 30px 40px;
      margin-bottom: 50px;
      border: 1px solid rgba(198, 40, 40, 0.3);
      backdrop-filter: blur(10px);
      display: flex;
      align-items: center;
      justify-content: center;
      gap: 20px;
      
      .quote-mark {
        font-size: 4rem;
        color: #f44336;
        line-height: 1;
      }
      
      .quote-text {
        font-size: 1.8rem;
        text-align: center;
        line-height: 1.4;
        min-height: 2.5em;
        display: flex;
        align-items: center;
        transition: opacity 0.5s ease;
      }
    }
    
   
    
    .philosophy-section {
      display: flex;
      flex-direction: column;
      gap: 30px;
      
      @media (min-width: 768px) {
        flex-direction: row;
      }
      
      .philosophy-item {
        flex: 1;
        background: rgba(26, 26, 26, 0.5);
        border-radius: 12px;
        padding: 25px;
        border: 1px solid rgba(198, 40, 40, 0.2);
        backdrop-filter: blur(5px);
        display: flex;
        align-items: flex-start;
        gap: 20px;
        transition: all 0.3s ease;
        
        &:hover {
          transform: translateY(-5px);
          border-color: rgba(244, 67, 54, 0.4);
        }
        
        .philosophy-icon {
          font-size: 2.5rem;
          color: #f44336;
          flex-shrink: 0;
        }
        
        .philosophy-content {
          h3 {
            font-size: 1.5rem;
            margin: 0 0 10px;
            color: #f5f5f5;
          }
          
          p {
            font-size: 1rem;
            line-height: 1.6;
            color: rgba(245, 245, 245, 0.8);
            margin: 0;
          }
        }
      }
    }
  }
  
  .footer {
    position: fixed;
    z-index: 2;
    background: rgba(10, 10, 20, 0.8);
    border-top: 1px solid rgba(42, 42, 42, 0.5);
    padding: 30px 20px;
    text-align: center;
    bottom: 0;
    left: 0;
    width: 100vw;
    .footer-content {
      max-width: 900px;
      margin: 0 auto;
      
      .footer-quote {
        font-size: 1.2rem;
        font-style: italic;
        margin-bottom: 10px;
        color: rgba(245, 245, 245, 0.9);
      }
      
      .footer-info {
        font-size: 0.9rem;
        color: rgba(245, 245, 245, 0.6);
      }
    }
  }
  
  .floating-scissors {
    position: absolute;
    z-index: 1;
    color: #f44336;
    pointer-events: none;
    animation: float 8s ease-in-out infinite;
    filter: drop-shadow(0 0 5px rgba(244, 67, 54, 0.3));
    
    @keyframes float {
      0%, 100% {
        transform: translate(-50%, -50%) rotate(0deg);
      }
      25% {
        transform: translate(-50%, -50%) rotate(90deg) translateY(-10px);
      }
      50% {
        transform: translate(-50%, -50%) rotate(180deg) translateY(-20px);
      }
      75% {
        transform: translate(-50%, -50%) rotate(270deg) translateY(-10px);
      }
    }
  }
}

/* 响应式调整 */
@media (max-width: 768px) {
  .chisaki-home {
    .title-section {
     
      .subtitle {
        font-size: 1rem;
        letter-spacing: 3px;
      }
      
      .main-title {
        font-size: 4rem;
        letter-spacing: 6px;
      }
      
      .title-decoration {
        .scissors-left, .scissors-right {
          width: 40px;
          
          &::before, &::after {
            width: 15px;
          }
        }
        
        .scissors-left {
          left: calc(50% - 70px);
        }
        
        .scissors-right {
          right: calc(50% - 70px);
        }
      }
    }
    
    .content-section {
      .quote-box {
        padding: 20px 25px;
        
        .quote-mark {
          font-size: 3rem;
        }
        
        .quote-text {
          font-size: 1.4rem;
        }
      }
      
      .stats-grid {
        grid-template-columns: 1fr;
      }
      
      .philosophy-section {
        flex-direction: column;
        
        .philosophy-item {
          padding: 20px;
          
          .philosophy-icon {
            font-size: 2rem;
          }
          
          .philosophy-content {
            h3 {
              font-size: 1.3rem;
            }
          }
        }
      }
    }
  }
}

@media (max-width: 480px) {
  .chisaki-home {
    .title-section {
      .main-title {
        font-size: 3rem;
        letter-spacing: 4px;
      }
    }
    
    .content-section {
      .quote-box {
        padding: 15px 20px;
        gap: 10px;
        
        .quote-mark {
          font-size: 2.5rem;
        }
        
        .quote-text {
          font-size: 1.2rem;
        }
      }
    }
  }
}
</style>