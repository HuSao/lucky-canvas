# 大转盘"经过区域"功能说明

> 用最简单的话，讲清楚这个功能是什么、怎么用、怎么实现的。

---

## 一、这个功能是干嘛的？

### 1.1 以前的转盘

以前的转盘只有两个事件：

```
点击按钮 → 转盘开始转 → 转盘停下来
    ↓                        ↓
  start()                  end()
```

**问题**：转盘在转的时候，我们不知道它现在指向哪个奖品。

### 1.2 现在的转盘

现在的转盘多了一个事件：

```
点击按钮 → 转盘开始转 → 经过奖品1 → 经过奖品2 → ... → 转盘停下来
    ↓           ↓            ↓           ↓                ↓
  start()   onCurrentChange()  onCurrentChange()      end()
```

**好处**：每次指针进入新的奖品区域，我们都能收到通知！

### 1.3 能用来做什么？

最常见的用途：**播放"哒哒哒"的音效**

你肯定玩过这样的转盘游戏：
- 转盘转起来的时候，会发出"哒、哒、哒、哒"的声音
- 这就是每次指针经过新区域时播放的音效

---

## 二、怎么用？(超简单)

### 2.1 最简单的例子

```html
<div id="my-lucky"></div>
<script src="dist/index.umd.js"></script>
<script>
  const myLucky = new LuckyCanvas.LuckyWheel('#my-lucky', {
    width: '300px',
    height: '300px',
    prizes: [
      { fonts: [{ text: '奖品1' }] },
      { fonts: [{ text: '奖品2' }] },
      { fonts: [{ text: '奖品3' }] },
    ],
    // 👇 新功能在这里！
    onCurrentChange(index, prize) {
      console.log('指针指向了：' + prize.fonts[0].text)
    },
    start() { myLucky.play() },
    end(prize) { alert('恭喜：' + prize.fonts[0].text) }
  })
</script>
```

### 2.2 配合音效使用

```html
<div id="my-lucky"></div>

<!-- 准备一个音效文件 -->
<audio id="tick-sound" src="tick.mp3" preload="auto"></audio>

<script src="dist/index.umd.js"></script>
<script>
  const tickSound = document.getElementById('tick-sound')

  const myLucky = new LuckyCanvas.LuckyWheel('#my-lucky', {
    width: '300px',
    height: '300px',
    prizes: [
      { fonts: [{ text: '一等奖' }] },
      { fonts: [{ text: '二等奖' }] },
      { fonts: [{ text: '三等奖' }] },
      { fonts: [{ text: '谢谢参与' }] },
    ],
    buttons: [{ radius: '30%', fonts: [{ text: '抽奖' }] }],

    // 👇 每次进入新区域，播放"哒"的一声
    onCurrentChange(index, prize) {
      tickSound.currentTime = 0  // 从头开始播放
      tickSound.play()           // 播放音效
    },

    start() {
      myLucky.play()
      setTimeout(() => myLucky.stop(0), 2000)
    },

    end(prize) {
      alert('恭喜获得：' + prize.fonts[0].text)
    }
  })
</script>
```

### 2.3 回调函数的参数

```javascript
onCurrentChange(index, prize) {
  // index = 当前是第几个奖品（从0开始数）
  // prize = 当前奖品的所有信息

  console.log(index)           // 比如：2
  console.log(prize.fonts[0].text)  // 比如："三等奖"
  console.log(prize.background)      // 比如："#ff0000"
}
```

---

## 三、什么时候会触发？

### 3.1 不是每帧都触发！

转盘一秒钟要画 60 次（60帧），但我们**不是每帧都触发回调**。

```
帧1:  转盘角度=0°    → 指向奖品0 → 进入新区域！触发回调 ✅
帧2:  转盘角度=5°    → 还在奖品0 → 不触发 ❌
帧3:  转盘角度=10°   → 还在奖品0 → 不触发 ❌
...
帧12: 转盘角度=55°   → 还在奖品0 → 不触发 ❌
帧13: 转盘角度=60°   → 进入奖品1！触发回调 ✅
帧14: 转盘角度=65°   → 还在奖品1 → 不触发 ❌
...
```

### 3.2 为什么这样设计？

想象一下，如果每帧都触发：
- 一秒钟触发 60 次
- 音效会被播放 60 次
- 声音会变成一坨噪音 😱

只在**进入新区域时触发**，刚刚好！

---

## 四、是怎么实现的？

### 4.1 核心问题：怎么知道指针指向哪个奖品？

想象一个时钟：

```
        12点 (0°)
          ↑
     ┌─────────┐
   9点│    ●    │3点
 (270°)│         │(90°)
     └─────────┘
          ↓
        6点 (180°)
```

转盘也是一样的：
- 指针固定在最上面（12点的位置，0°）
- 转盘在转动，角度在不断变化
- 我们要算出：在当前角度下，指针指向哪个区域

### 4.2 计算方法

假设：
- 有 6 个奖品
- 每个奖品占 60°（因为 360° ÷ 6 = 60°）
- 转盘转了 150°

```
步骤1：算出指针相对于转盘的位置
      指针角度 = 360° - 150° = 210°

步骤2：算出落在第几个区域
      第几个 = 210° ÷ 60° = 3.5 → 取整 = 3

答案：指针指向第 3 个奖品（从0开始数）
```

### 4.3 代码实现

```typescript
// 计算当前指针指向第几个奖品
private getCurrentPrizeIndex(): number {
  const { prizes, prizeDeg, rotateDeg, _defaultConfig } = this

  // 如果没有奖品，返回 -1
  if (!prizes.length) return -1

  // 算出指针相对于转盘的角度
  let pointerAngle = (360 - (rotateDeg % 360) + 360) % 360

  // 算出落在第几个区域
  const index = Math.floor(pointerAngle / prizeDeg) % prizes.length

  return index
}
```

### 4.4 在动画循环中检测变化

```typescript
private run(num: number = 0): void {
  // ... 计算转盘角度 ...

  this.rotateDeg = rotateDeg  // 更新角度

  // 👇 检测是否进入了新区域
  if (this.onCurrentChangeCallback) {
    const newIndex = this.getCurrentPrizeIndex()  // 算出当前指向哪个

    if (newIndex !== this.currentPrizeIndex) {    // 如果和上次不一样
      this.currentPrizeIndex = newIndex           // 记住这次的值
      this.onCurrentChangeCallback(newIndex, this.prizes[newIndex])  // 触发回调！
    }
  }

  this.draw()  // 重新画
  rAF(this.run.bind(this, num + 1))  // 下一帧继续
}
```

---

## 五、改了哪些文件？

### 5.1 文件清单

一共改了 **2 个文件**：

```
packages/core/src/
├── types/wheel.ts  ← 类型定义文件
└── lib/wheel.ts    ← 核心实现文件
```

### 5.2 types/wheel.ts 的改动

**加了什么：**

```typescript
// 新增：定义回调函数的类型
export type OnCurrentChangeCallbackType = (index: number, prize: PrizeType) => void

// 新增：在配置里加上这个选项
export default interface LuckyWheelConfig {
  // ... 原来的配置 ...
  onCurrentChange?: OnCurrentChangeCallbackType  // 👈 新增这行
}
```

**为什么要加：**
- TypeScript 需要知道 `onCurrentChange` 是什么类型
- 不然写代码的时候会报错

### 5.3 lib/wheel.ts 的改动

**改动一：加两个属性**

```typescript
// 存储用户传入的回调函数
private onCurrentChangeCallback?: OnCurrentChangeCallbackType

// 记录上次指向的奖品索引（用来判断是否变化）
private currentPrizeIndex = -1
```

**改动二：初始化时保存回调**

```typescript
private initData(data: LuckyWheelConfig): void {
  // ... 原来的代码 ...
  this.$set(this, 'onCurrentChangeCallback', data.onCurrentChange)  // 👈 新增
}
```

**改动三：重置时清空索引**

```typescript
protected initLucky(): void {
  // ... 原来的代码 ...
  this.currentPrizeIndex = -1  // 👈 新增
}
```

**改动四：新增计算方法**

```typescript
// 计算当前指向哪个奖品
private getCurrentPrizeIndex(): number {
  // ... 详见 4.3 ...
}
```

**改动五：动画循环中检测变化**

```typescript
private run(num: number = 0): void {
  // ... 原来的代码 ...

  // 👇 新增这段
  if (this.onCurrentChangeCallback) {
    const newIndex = this.getCurrentPrizeIndex()
    if (newIndex !== this.currentPrizeIndex) {
      this.currentPrizeIndex = newIndex
      this.onCurrentChangeCallback(newIndex, this.prizes[newIndex])
    }
  }

  // ... 原来的代码 ...
}
```

---

## 六、完整代码对照

### 改动前 vs 改动后

#### types/wheel.ts

```diff
  export type StartCallbackType = (e: MouseEvent) => void
  export type EndCallbackType = (prize: object) => void
+ export type OnCurrentChangeCallbackType = (index: number, prize: PrizeType) => void

  export default interface LuckyWheelConfig {
    // ... 其他配置 ...
    start?: StartCallbackType
    end?: EndCallbackType
+   onCurrentChange?: OnCurrentChangeCallbackType
  }
```

#### lib/wheel.ts

```diff
  import LuckyWheelConfig, {
    // ... 其他导入 ...
+   OnCurrentChangeCallbackType
  } from '../types/wheel'

  export default class LuckyWheel extends Lucky {
    private startCallback?: StartCallbackType
    private endCallback?: EndCallbackType
+   private onCurrentChangeCallback?: OnCurrentChangeCallbackType
+   private currentPrizeIndex = -1
    private Radius = 0
    // ... 其他属性 ...

+   private getCurrentPrizeIndex(): number {
+     const { prizes, prizeDeg, rotateDeg, _defaultConfig } = this
+     if (!prizes.length) return -1
+     let pointerAngle = (360 - (rotateDeg % 360) + _defaultConfig.offsetDegree + 360) % 360
+     const index = Math.floor(pointerAngle / prizeDeg) % prizes.length
+     return index
+   }

    private initData(data: LuckyWheelConfig): void {
      // ... 原来的代码 ...
+     this.$set(this, 'onCurrentChangeCallback', data.onCurrentChange)
    }

    protected initLucky(): void {
      // ... 原来的代码 ...
+     this.currentPrizeIndex = -1
    }

    private run(num: number = 0): void {
      // ... 原来的动画计算 ...

      this.rotateDeg = rotateDeg
+     if (this.onCurrentChangeCallback) {
+       const newIndex = this.getCurrentPrizeIndex()
+       if (newIndex !== this.currentPrizeIndex) {
+         this.currentPrizeIndex = newIndex
+         this.onCurrentChangeCallback(newIndex, this.prizes[newIndex])
+       }
+     }

      this.draw()
      rAF(this.run.bind(this, num + 1))
    }
  }
```

---

## 七、常见问题

### Q1：为什么回调有时候触发得很慢？

A：转盘在减速阶段，每个奖品区域停留的时间会变长，这是正常的。

### Q2：能在回调里修改转盘吗？

A：可以，但建议只做简单的事情（比如播放音效），不要做耗时操作。

### Q3：如果不需要这个功能，要怎么写？

A：不写 `onCurrentChange` 就行，不会影响其他功能：

```javascript
const myLucky = new LuckyCanvas.LuckyWheel('#my-lucky', {
  // 不写 onCurrentChange，完全没问题
  start() { myLucky.play() },
  end(prize) { alert('恭喜') }
})
```

---

## 八、总结

### 一句话概括

**转盘每转到一个新的奖品区域，就告诉你一声。**

### 三个关键点

1. **触发时机**：进入新区域时触发一次
2. **回调参数**：告诉你索引和奖品信息
3. **常见用途**：播放"哒哒哒"音效

### 改动总结表

| 文件 | 改了什么 | 为什么 |
|------|----------|--------|
| types/wheel.ts | 加了类型定义 | TypeScript 需要 |
| lib/wheel.ts | 加了属性 | 存储状态 |
| lib/wheel.ts | 加了方法 | 计算当前指向 |
| lib/wheel.ts | 改了动画循环 | 检测变化并触发回调 |