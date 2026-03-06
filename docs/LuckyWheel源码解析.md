# LuckyWheel 大转盘源码解析

> 从 HTML 使用到源码实现的完整讲解，适合初学者阅读。

---

## 目录

1. [从 HTML 使用开始](#一从-html-使用开始)
2. [构造函数做了什么](#二构造函数做了什么)
3. [初始化过程 init](#三初始化过程-init)
4. [绘制原理 draw](#四绘制原理-draw)
5. [动画机制](#五动画机制)
6. [点击事件处理](#六点击事件处理)
7. [完整流程总结](#七完整流程总结)
8. [关键代码速查表](#八关键代码速查表)

---

## 一、从 HTML 使用开始

### 1.1 最简单的使用方式

```html
<!DOCTYPE html>
<html>
<body>
  <!-- 第1步：准备一个容器 -->
  <div id="my-lucky"></div>

  <!-- 第2步：引入库 -->
  <script src="dist/index.umd.js"></script>

  <!-- 第3步：创建实例 -->
  <script>
    const myLucky = new LuckyCanvas.LuckyWheel('#my-lucky', {
      width: '200px',
      height: '200px',
      prizes: [
        { fonts: [{ text: '奖品1' }] },
        { fonts: [{ text: '奖品2' }] },
        { fonts: [{ text: '奖品3' }] },
      ],
      start() {
        myLucky.play()
        setTimeout(() => myLucky.stop(0), 2000)
      }
    })
  </script>
</body>
</html>
```

### 1.2 这行代码发生了什么？

```
new LuckyCanvas.LuckyWheel('#my-lucky', config)
         │
         ▼
    ┌─────────────────────────────────────────┐
    │  1. 找到 #my-lucky 这个 div             │
    │  2. 在里面创建一个 <canvas>              │
    │  3. 获取 canvas 的绑定上下文 ctx         │
    │  4. 根据 config 绘制转盘                │
    └─────────────────────────────────────────┘
```

---

## 二、构造函数做了什么

### 2.1 代码流程图

```
new LuckyWheel('#my-lucky', config)
        │
        ▼
┌───────────────────┐
│ super(config)     │  ← 调用父类 Lucky 构造函数
└───────────────────┘
        │
        ▼
┌───────────────────┐
│ initData(config)  │  ← 保存配置数据
└───────────────────┘
        │
        ▼
┌───────────────────┐
│ initWatch()       │  ← 设置数据监听
└───────────────────┘
        │
        ▼
┌───────────────────┐
│ initComputed()    │  ← 计算默认配置
└───────────────────┘
        │
        ▼
┌───────────────────┐
│ init()            │  ← 初始化并绘制
└───────────────────┘
```

### 2.2 父类构造函数详解

```typescript
// lucky.ts
constructor(config, data) {
  // ━━━ 第1步：处理参数 ━━━
  // 用户可能传入三种格式：
  // 1. 字符串：'#my-lucky'
  // 2. DOM元素：document.getElementById('my-lucky')
  // 3. 配置对象：{ el: '#my-lucky', ... }

  if (typeof config === 'string') {
    config = { el: config }  // 统一转成对象
  }

  // ━━━ 第2步：获取 DOM 元素 ━━━
  if (config.el) {
    // document.querySelector('#my-lucky')
    config.divElement = document.querySelector(config.el)
  }

  // ━━━ 第3步：创建 Canvas ━━━
  if (config.divElement) {
    // 创建 <canvas> 标签
    config.canvasElement = document.createElement('canvas')
    // 放到 <div id="my-lucky"> 里面
    config.divElement.appendChild(config.canvasElement)
  }

  // ━━━ 第4步：获取绑定上下文 ━━━
  if (config.canvasElement) {
    // ctx 是绑定的核心，所有绑制都靠它
    config.ctx = config.canvasElement.getContext('2d')

    // 绑定点击事件
    config.canvasElement.addEventListener('click', e => this.handleClick(e))
  }

  // 保存到实例
  this.ctx = config.ctx

  // ━━━ 第5步：监听窗口变化 ━━━
  // 当浏览器窗口大小改变时，重新绘制
  window.addEventListener('resize', () => {
    this.resize()  // 重新计算尺寸并重绘
  })
}
```

**此时的 DOM 结构：**

```html
<div id="my-lucky">
  <canvas package="lucky-canvas@1.7.x"></canvas>
</div>
```

---

## 三、初始化过程 init()

### 3.1 init() 方法

```typescript
public async init(): Promise<void> {
  // 1. 重置所有状态
  this.initLucky()

  // 2. 触发初始化前回调
  config.beforeInit?.call(this)

  // 3. 先绘制一次（防止闪烁）
  this.draw()

  // 4. 异步加载图片
  await this.initImageCache()

  // 5. 触发初始化后回调
  config.afterInit?.call(this)
}
```

### 3.2 initLucky() 重置状态

```typescript
protected initLucky(): void {
  this.Radius = 0          // 转盘半径
  this.prizeRadius = 0     // 奖品区域半径
  this.prizeDeg = 0        // 每个奖品的角度
  this.rotateDeg = 0       // 当前旋转角度
  this.step = 0            // 动画阶段
  this.prizeFlag = -1      // 中奖索引
  // ...
  super.initLucky()        // 调用父类
}
```

### 3.3 resize() 计算尺寸

```typescript
protected resize(): void {
  // 1. 获取 rem 基准值（用于单位转换）
  this.setHTMLFontSize()

  // 2. 设置设备像素比（高清屏适配）
  this.setDpr()

  // 3. 计算宽高
  this.resetWidthAndHeight()

  // 4. 缩放 Canvas
  this.zoomCanvas()

  // 5. 计算转盘半径
  this.Radius = Math.min(this.boxWidth, this.boxHeight) / 2

  // 6. 移动坐标原点到中心
  this.ctx.translate(this.Radius, this.Radius)

  // 7. 绘制
  this.draw()
}
```

### 3.4 zoomCanvas() 高清屏适配

这是个重要概念，让我详细解释：

```
普通屏幕：
  1个 CSS 像素 = 1个物理像素
  canvas 200px × 200px = 200×200 物理像素

Retina 高清屏：
  1个 CSS 像素 = 2个物理像素（dpr=2）
  canvas 200px × 200px = 400×400 物理像素

如果不处理，在 Retina 屏上会模糊！
```

**解决方案：**

```typescript
protected zoomCanvas(): void {
  const dpr = window.devicePixelRatio || 1  // 获取像素比

  // 实际像素 = CSS像素 × 像素比
  canvas.width = this.boxWidth * dpr    // 如 200 × 2 = 400
  canvas.height = this.boxHeight * dpr

  // CSS 样式保持不变
  canvas.style.width = `${this.boxWidth}px`   // 200px
  canvas.style.height = `${this.boxHeight}px`

  // 用 CSS 缩小显示
  canvas.style.transform = `scale(${1 / dpr})`  // scale(0.5)

  // 画布内部也缩放，这样绑制时坐标不用变
  ctx.scale(dpr, dpr)
}
```

---

## 四、绘制原理 draw()

### 4.1 Canvas 坐标系

**初始状态（原点在左上角）：**

```
          ↓ y
          │
          │
          │
(0,0)─────┼─────────→ x
          │
          │
          │
```

**执行 `ctx.translate(Radius, Radius)` 后（原点移到中心）：**

```
          ↓ y
          │
    (-x,+y)│    (+x,+y)
          │
──────────┼──────────→ x
          │
    (-x,-y)│    (+x,-y)
          │
          │

原点移动到中心，绘制转盘就方便多了！
```

### 4.2 draw() 主流程

```typescript
protected draw(): void {
  // ━━━ 第1步：清空画布 ━━━
  ctx.clearRect(-Radius, -Radius, Radius*2, Radius*2)

  // ━━━ 第2步：绘制背景圆环 ━━━
  this.prizeRadius = this.blocks.reduce((radius, block) => {
    this.drawBlock(radius, block)  // 画一个圆
    return radius - padding        // 向内缩进
  }, this.Radius)

  // ━━━ 第3步：计算奖品角度 ━━━
  this.prizeDeg = 360 / this.prizes.length  // 如3个奖品，每个120°
  this.prizeAng = prizeDeg * Math.PI / 180  // 转成弧度

  // ━━━ 第4步：绘制奖品扇形 ━━━
  this.prizes.forEach((prize, index) => {
    // 绘制扇形背景
    // 绘制图片
    // 绘制文字
  })

  // ━━━ 第5步：绘制中心按钮 ━━━
  this.buttons.forEach(btn => {
    // 绘制圆形按钮
    // 绘制按钮图片/文字
  })
}
```

### 4.3 绘制背景圆环详解

假设配置是：

```javascript
blocks: [
  { padding: '10px', background: '#ccc' },  // 外圈
  { padding: '5px', background: '#fff' },   // 内圈
]
```

**绘制过程：**

```
第1次循环：radius = 100 (初始半径)
  ┌─────────────────────┐
  │  绘制半径100的圆     │  背景 #ccc
  │  radius = 100-10=90 │
  └─────────────────────┘

第2次循环：radius = 90
  ┌─────────────────────┐
  │  绘制半径90的圆      │  背景 #fff
  │  radius = 90-5=85   │
  └─────────────────────┘

最终 prizeRadius = 85（奖品区域半径）
```

**代码：**

```typescript
private drawBlock(radius, block, blockIndex): void {
  // 绘制圆形背景
  if (block.background) {
    ctx.beginPath()
    ctx.fillStyle = block.background
    ctx.arc(0, 0, radius, 0, Math.PI * 2)  // 画圆
    ctx.fill()
  }

  // 如果有图片，绘制图片
  block.imgs && block.imgs.forEach(imgInfo => {
    const img = this.ImageCache.get(imgInfo.src)
    // 计算图片位置和大小
    ctx.drawImage(img, x, y, width, height)
  })
}
```

### 4.4 绘制奖品扇形详解

**角度计算：**

```
假设有6个奖品，offsetDegree=0

奖品0：从 -60° 到 0°   (中心在 -30°)
奖品1：从 0° 到 60°    (中心在 30°)
奖品2：从 60° 到 120°  (中心在 90°)
...

起始角度 = -90° - prizeDeg/2 + rotateDeg
         = -90° - 60°/2 + 0°
         = -120°

这样第一个奖品就在正上方
```

**示意图：**

```
        奖品0
       ╱     ╲
     ╱         ╲
   ╱             ╲
  │      ●        │  ← 中心按钮
   ╲             ╱
     ╲         ╱
       ╲     ╱
        奖品3
```

**代码：**

```typescript
// 计算起始角度（从正上方开始）
let start = getAngle(rotateDeg - 90 + prizeDeg/2 + offsetDegree)

this.prizes.forEach((prize, prizeIndex) => {
  // 当前奖品的中心角度
  const currMiddleDeg = start + prizeIndex * prizeAng

  // 扇形起始和结束角度
  const startAngle = currMiddleDeg - prizeAng/2
  const endAngle = currMiddleDeg + prizeAng/2

  // 绘制扇形背景
  if (prize.background) {
    ctx.beginPath()
    ctx.fillStyle = prize.background
    ctx.moveTo(0, 0)
    ctx.arc(0, 0, prizeRadius, startAngle, endAngle)
    ctx.fill()
  }

  // ━━━ 绘制文字 ━━━
  // 先移动坐标到奖品中心位置
  const x = Math.cos(currMiddleDeg) * prizeRadius
  const y = Math.sin(currMiddleDeg) * prizeRadius
  ctx.translate(x, y)

  // 旋转文字，让它沿着扇形方向
  ctx.rotate(currMiddleDeg + Math.PI/2)

  // 绘制文字
  ctx.fillStyle = fontColor
  ctx.font = `${fontSize}px ${fontStyle}`
  ctx.fillText(text, x, y)

  // 恢复变换
  ctx.rotate(-(currMiddleDeg + Math.PI/2))
  ctx.translate(-x, -y)
})
```

**坐标变换示意图：**

```
原始状态：              translate后：           rotate后：

      y                      y                      y
      ↑                      ↑                      ↑
      │                      │    *奖品中心          │
      │                      │   /                   │
      │         →           │  /         →         ● → 文字正向
──────┼────── x        ──────┼────── x        ──────┼────── x
      │                      │                      │

1. 原点在中心            2. 移到奖品位置        3. 旋转让文字正向
```

---

## 五、动画机制

### 5.1 动画状态（step）

```typescript
// step 的4个状态
step = 0  // 静止状态，等待开始
step = 1  // 加速阶段
step = 2  // 匀速阶段
step = 3  // 减速阶段
```

**状态转换图：**

```
          play()
  step=0 ───────→ step=1
 (静止)          (加速)
                    │
                    │ 加速完成
                    ▼
                step=2
                (匀速)
                    │
                    │ stop(index)
                    ▼
                step=3
                (减速)
                    │
                    │ 减速完成
                    ▼
  step=0 ←───────────
 (静止)              endCallback()
```

### 5.2 play() 开始旋转

```typescript
public play(): void {
  if (this.step !== 0) return  // 如果已经在转，忽略

  // 记录开始时间
  this.startTime = Date.now()

  // 重置中奖索引
  this.prizeFlag = undefined

  // 进入加速阶段
  this.step = 1

  // 开始动画循环
  this.run()
}
```

### 5.3 stop() 停止旋转

```typescript
public stop(index): void {
  // 不能在静止或减速时调用
  if (this.step === 0 || this.step === 3) return

  // 处理中奖索引
  if (index < 0) {
    // 负数表示无效抽奖
    this.step = 0
    this.prizeFlag = -1
  } else {
    // 进入减速阶段（先经过匀速）
    this.step = 2
    // 记录要停在哪个奖品
    this.prizeFlag = index % this.prizes.length
  }
}
```

### 5.4 run() 动画循环

这是核心的动画方法：

```typescript
private run(num = 0): void {
  const { step, prizeFlag, _defaultConfig } = this
  const { accelerationTime, decelerationTime, speed } = _defaultConfig

  // ━━━ 检查是否结束 ━━━
  if (step === 0) {
    // 调用结束回调
    this.endCallback?.(this.prizes[prizeFlag])
    return
  }

  // 如果是无效抽奖，直接停止
  if (prizeFlag === -1) return

  // ━━━ 计算时间间隔 ━━━
  const startInterval = Date.now() - this.startTime  // 从开始到现在
  const endInterval = Date.now() - this.endTime      // 从减速开始到现在

  // ━━━ 加速阶段 ━━━
  if (step === 1 || startInterval < accelerationTime) {
    // 记录帧率
    this.FPS = startInterval / num

    // 使用 easeIn 缓动函数计算当前速度
    const currSpeed = quad.easeIn(startInterval, 0, speed, accelerationTime)

    // 如果达到最大速度，进入匀速
    if (currSpeed >= speed) {
      this.step = 2
    }

    // 累加旋转角度
    this.rotateDeg += currSpeed
  }
  // ━━━ 匀速阶段 ━━━
  else if (step === 2) {
    // 以恒定速度旋转
    this.rotateDeg += speed

    // 如果 prizeFlag 有值，进入减速阶段
    if (prizeFlag !== undefined && prizeFlag >= 0) {
      this.step = 3
      // 清空上次的位置信息
      this.stopDeg = 0
      this.endDeg = 0
    }
  }
  // ━━━ 减速阶段 ━━━
  else if (step === 3) {
    // 第一次进入减速阶段，计算要停在哪里
    if (!this.endDeg) {
      this.carveOnGunwaleOfAMovingBoat()
    }

    // 使用 easeOut 缓动函数计算当前角度
    this.rotateDeg = quad.easeOut(endInterval, this.stopDeg, this.endDeg, decelerationTime)

    // 时间到了，停止
    if (endInterval >= decelerationTime) {
      this.step = 0
    }
  }

  // 重绘
  this.draw()

  // 下一帧继续
  requestAnimationFrame(() => this.run(num + 1))
}
```

### 5.5 缓动函数详解

```typescript
// utils/tween.ts
export const quad = {
  // easeIn：开始慢，越来越快
  easeIn(t, b, c, d) {
    // t: 当前时间
    // b: 初始值
    // c: 变化量
    // d: 总时长
    t /= d
    return c * t * t + b
  },

  // easeOut：开始快，越来越慢
  easeOut(t, b, c, d) {
    t /= d
    return -c * t * (t - 2) + b
  }
}
```

**曲线图：**

```
easeIn（加速曲线）：        easeOut（减速曲线）：

速度 │                        速度 │
  ↑  │     /                    ↑  │ \
  │  │    /                     │  │  \
  │  │   /                      │  │   \
  │  │  /                       │  │    \____
  └──┴─────────→ 时间          └──┴─────────→ 时间
      开始慢，越来越快              开始快，越来越慢
```

### 5.6 刻舟求剑算法

这个方法用于计算转盘最终要停在哪个角度：

```typescript
private carveOnGunwaleOfAMovingBoat(): void {
  // 记录开始减速的时间
  this.endTime = Date.now()

  // 记录当前角度
  const stopDeg = this.stopDeg = this.rotateDeg

  const speed = _defaultConfig.speed
  const decelerationTime = _defaultConfig.decelerationTime

  let i = 0
  while (++i) {
    // 计算转 i 圈后停在目标奖品需要的角度
    const endDeg = 360 * i                // 转i圈
                  - prizeFlag * prizeDeg  // 减去目标奖品的角度
                  - rotateDeg              // 减去当前角度
                  - offsetDegree           // 减去偏移角度
                  + stopRange              // 加上随机范围
                  - prizeDeg / 2           // 调整到奖品中心

    // 模拟减速，看这个角度的"速度"是多少
    const currSpeed = quad.easeOut(this.FPS, stopDeg, endDeg, decelerationTime) - stopDeg

    // 如果速度超过了最大速度，说明找到了合适的圈数
    if (currSpeed > speed) {
      this.endDeg = endDeg
      break
    }
  }
}
```

**示意图：**

```
假设 prizeFlag = 1（停在奖品1），当前 rotateDeg = 45°

                ┌──────────────────────────────────┐
                │         计算 endDeg              │
                └──────────────────────────────────┘

  i=1: endDeg = 360×1 - 1×60 - 45 = 255°     ← 速度不够
  i=2: endDeg = 360×2 - 1×60 - 45 = 615°     ← 速度合适！

  最终 endDeg = 615°

  转盘会转动：从 45° → 615°（转1圈多）
  停在奖品1的位置
```

---

## 六、点击事件处理

### 6.1 handleClick()

```typescript
protected handleClick(e: MouseEvent): void {
  const { ctx } = this

  // 创建一个圆形路径（按钮的大小）
  ctx.beginPath()
  ctx.arc(0, 0, this.maxBtnRadius, 0, Math.PI * 2)

  // 判断点击是否在圆形区域内
  if (!ctx.isPointInPath(e.offsetX, e.offsetY)) {
    return  // 点击的不是按钮，忽略
  }

  // 如果正在旋转，忽略
  if (this.step !== 0) return

  // 调用用户的 start 回调
  this.startCallback?.(e)
}
```

### 6.2 isPointInPath 原理

```typescript
// Canvas 的 isPointInPath 方法可以判断点是否在路径内

ctx.beginPath()
ctx.arc(0, 0, 50, 0, Math.PI * 2)  // 圆形路径

ctx.isPointInPath(30, 30)   // true，点在圆内
ctx.isPointInPath(100, 100) // false，点在圆外
```

---

## 七、完整流程总结

### 7.1 初始化流程

```
new LuckyWheel()
    │
    ├─→ super() 创建 Canvas
    │
    ├─→ initData() 保存配置
    │
    ├─→ initWatch() 设置监听
    │
    └─→ init()
         │
         ├─→ initLucky() 重置状态
         │
         ├─→ draw() 首次绘制
         │
         └─→ initImageCache() 加载图片
```

### 7.2 绘制流程

```
draw()
    │
    ├─→ 清空画布
    │
    ├─→ 绘制 blocks（背景圆环）
    │      │
    │      └─→ for each block:
    │              画圆，缩小半径
    │
    ├─→ 绘制 prizes（奖品扇形）
    │      │
    │      └─→ for each prize:
    │              画扇形背景
    │              画图片
    │              画文字
    │
    └─→ 绘制 buttons（中心按钮）
           │
           └─→ for each button:
                   画圆形背景
                   画图片/文字
```

### 7.3 动画流程

```
play() ──→ step=1 (加速)
              │
              ▼
         step=2 (匀速) ← stop(index) 触发
              │
              ▼
         step=3 (减速)
              │
              ▼
         step=0 (停止)
              │
              ▼
         endCallback(中奖奖品)
```

### 7.4 用户交互流程

```
点击按钮
    │
    ▼
handleClick() ──→ startCallback()
                      │
                      ▼
                  用户调用 play()
                      │
                      ▼
                  开始旋转动画
                      │
                      ▼
                  用户调用 stop(index)
                      │
                      ▼
                  减速停在目标奖品
                      │
                      ▼
                  endCallback(奖品信息)
```

---

## 八、关键代码速查表

| 方法 | 作用 | 关键代码 |
|------|------|----------|
| `constructor` | 初始化 | 创建 canvas, 获取 ctx |
| `init` | 首次绘制 | `initLucky()` → `draw()` |
| `resize` | 尺寸变化 | 计算尺寸, `zoomCanvas()` |
| `draw` | 绘制转盘 | blocks → prizes → buttons |
| `play` | 开始旋转 | `step=1`, `run()` |
| `stop` | 停止旋转 | `step=2`, `prizeFlag=index` |
| `run` | 动画循环 | 根据 step 计算 rotateDeg |
| `handleClick` | 点击事件 | 判断是否点击按钮 |

---

## 附录：常用配置说明

```javascript
new LuckyCanvas.LuckyWheel('#my-lucky', {
  // 尺寸
  width: '300px',
  height: '300px',

  // 背景圆环（从外到内）
  blocks: [
    { padding: '10px', background: '#ffc371' },
    { padding: '10px', background: '#ff5f6d' },
  ],

  // 奖品列表
  prizes: [
    {
      background: '#fff',
      fonts: [{ text: '奖品1', top: '10px', fontColor: '#333' }],
      imgs: [{ src: 'prize.png', width: '30%', top: '20px' }]
    },
    // ... 更多奖品
  ],

  // 中心按钮
  buttons: [
    {
      radius: '30%',
      background: '#fff',
      pointer: true,  // 显示指针
      fonts: [{ text: '开始' }],
    }
  ],

  // 默认配置
  defaultConfig: {
    gutter: '5px',           // 奖品间隔
    offsetDegree: 0,         // 起始偏移角度
    speed: 20,               // 旋转速度
    accelerationTime: 2500,  // 加速时间(ms)
    decelerationTime: 2500,  // 减速时间(ms)
  },

  // 默认样式
  defaultStyle: {
    fontSize: '18px',
    fontColor: '#333',
    background: '#fff',
  },

  // 回调函数
  start() {
    // 点击按钮时触发
    myLucky.play()
    // 请求后端获取中奖结果
    api.getPrize().then(index => {
      myLucky.stop(index)
    })
  },

  end(prize) {
    // 动画结束时触发
    alert(`恭喜获得: ${prize.fonts[0].text}`)
  }
})
```