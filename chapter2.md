# 第 2 章：从五线谱到代码——音高、时值与读谱映射

## 1. 开篇段落：你现在的身份是“指挥家”

欢迎来到 Sonic Pi 的核心逻辑世界。既然你已经具备识读五线谱的能力，并且熟悉基础的和弦进行，那么掌握 Sonic Pi 对你来说将非常快——你只需要完成一次思维上的**“翻译”**。

在传统的五线谱（Staff Notation）中，你是在告诉演奏者“看这里，弹这个音”。但在 Sonic Pi 中，你的角色从“作曲家”变成了**“指挥家兼程序员”**。计算机是世界上最听话但也最死板的演奏者，它不会自己感悟“这里应该慢一点”或“这里应该悲伤一点”，你必须用精确的数字告诉它。

本章的目标是立一种**直觉映射**：
*   当你看到久石让乐谱上的一串**连奏（Legato）**，你知道如何调整包络（Envelope）；
*   当你看到 Hans Zimmer 总谱里的**断奏（Staccato）**弦乐，你知道如何控制 `release` 参数；
*   当你需要华语古风的**散板（Rubato）**，你知道如何打破标准时值。

我们将学会如何精确地控制时间与频率，这是通往“精确配乐”的第一步。

---

## 2. 文字论述

### 2.1 音高与八度 (Pitch & Octave)：垂直轴的映射

在 Sonic Pi 中，音高主要有两种表示法：**MIDI 编号**和**音名符号**。对于习惯读谱的你，**音名符号**是最直观的，但理解 MIDI 编号对后续做“转调算法”非常有用。

#### 2.1.1 核心映射表

Sonic Pi 遵循标准的国际音高记法。中央 C（Middle C）在五线谱上是高音谱表的下加一线，在 Sonic Pi 代码中是 `:c4`。

```text
      五线谱位置            音名(Sonic Pi)    MIDI 编号    备注
-----------------------------------------------------------------------
  (高音谱表上加一线)            :a5             81       人声高音区极限附近
          |
  (高音谱表第三间)              :c5             72       高音区
          |
  (高音谱表下加一线)            :c4             60       中央 C
          |
  (低音谱表第二间)              :c3             48       大提琴/男低音常用区
          |
  (低音谱表下加二线)            :c2             36       低音贝司/电影Sub区
```

#### 2.1.2 升降号、等音与调性逻辑

Sonic Pi 使用 `s` (sharp, $\sharp$) 代表升号，`b` (flat, $\flat$) 代表降号。
虽然 `:ds4` (D$\sharp$4) 和 `:eb4` (E$\flat$4) 在物理频率上完全一样（MIDI 均为 63），但在代码写作中，建议严格遵循**调号逻辑**。

*   **升号写法**：F$\sharp$ $\rightarrow$ `:fs`
*   **降号写法**：E$\flat$ $\rightarrow$ `:eb`

> **Rule of Thumb (经验法则)：调号可读性**
> 如果你在写一首 **c 小调**（3 个降号：Bb, Eb, Ab）的曲子，请务必在代码中使用 `:bb`, `:eb`, `:ab`，而不要混用 `:as` 或 `:ds`。
> 当三个月后你回头看代码时，看到 `:eb` 你会立刻反应出“这是三级音”，看到 `:ds` 你会困惑“这是升二级音？我要离调了吗？”。

#### 2.1.3 微音程 (Microtones) 与“戏腔韵味”

华语戏腔和蓝调音乐中，常出现“不到半音”的微小音高变化。Sonic Pi 的 MIDI 值支持浮点数。
*   `:c4` = 60
*   `60.5` = C4 与 C#4 正中间的音。
这对模拟滑音（Glissando）和老唱片的走调感至关重要（后续章节详述）。

### 2.2 时值体系 (Duration & Sleep)：水平轴的映射

这是 Sonic Pi 与 DAW（数字音频工作站）最大的思维差异点。
*   **DAW 思维**：把音符块画在网格上，块的长度决定发声时长。
*   **Sonic Pi 思维**：触发一个声音，然后**“休眠”**一段时间，再触发下一个。

核心命令是 `sleep`。它控制的是**“音头到下一个音头的时间距离”**（Onset to Onset），而不是音符本身的持续时间。

#### 2.2.1 标准时值映射 (假设 4/4 拍)

默认情况下，我们将 `sleep 1` 视为 **一拍（One Beat / 四分音符）**。

```text
音符名称        五线谱形状         Sonic Pi 代码 (sleep)
-------------------------------------------------------
全音符          空心圆              sleep 4
二分音符        空心加符干          sleep 2
四分音符        实心加符干          sleep 1
八分音符        单符尾              sleep 0.5
十六分音符      双符尾              sleep 0.25
三十二分音符    三符尾              sleep 0.125
```

#### 2.2.2 附点 (Dotted Notes)

附点代表“延长原时值的一半”。
*   附点四分音符 ($1 + 0.5$) $\rightarrow$ `sleep 1.5`
*   附点八分音符 ($0.5 + 0.25$) $\rightarrow$ `sleep 0.75`

> **古风节奏 Tip：**
> 古风旋律中极常用 `sleep 1.5` 紧接 `sleep 0.5`（附点），或者 `sleep 0.75` 紧接 `sleep 0.25`（小附点）。这种“长短交替”产生了独特的吟唱感。

### 2.3 连音与复杂比例 (Tuplets)：Zimmer 的数学

Hans Zimmer 的动作配乐（如《蝙蝠侠》《盗梦空间》）大量依赖 **三连音 (Triplets)** 和 **五连音 (Quintuplets)** 来制造这种“一直在奔跑”的滚动感。

在数学上，三连音将标准时值平均分为 3 份。

*   **警告**：不要写 `sleep 0.33`！
    $0.33 \times 3 = 0.99$。每过一个小节，你的节奏就会快 0.01 拍。100 个小节后，你的鼓点就会和旋律完全错开（Phase Drift）。
*   **正确写法**：利用计算机的除法。
    `sleep 1.0 / 3` （注意 `1.0` 这种写法强制使用浮点运算）。

```ruby
# Zimmer 式 12/8 拍感觉的 Action Strings
3.times do
  play :c3
  sleep 1.0 / 3  # 极其精确的三分这一拍
end
```

### 2.4 声音的“形状”：Staccato vs. Legato

在五线谱上，音符的时值决定了多长；但在 Sonic Pi 里，`sleep` 只决定什么时候开始下一个音，**`release:` 参数决定这个音响多久**。

这是新手最容易混淆的地方：

1.  **断奏 (Staccato)**：声音很短，但空隙很长。
    *   代码：`play :c4, release: 0.1` 接着 `sleep 1`
    *   听感：短促的“得！”，然后长长的沉默。
2.  **连奏 (Legato)**：声音持续直到（甚至重叠）下一个音。
    *   代码：`play :c4, release: 1.0` 接着 `sleep 1`
    *   听感：无缝连接。

> **配器直觉**：
> *   **古风笛子**：通常是 Legato，`release` 甚至要比 `sleep` 稍微长一点点（如 `release: 1.1`），制造混响和重叠的连绵感。
> *   **动作片弦乐**：通常是 Staccato，`release` 极短（`0.1` - `0.2`），哪怕 `sleep` 是 0.5，也要留出空气感，这样才“有劲”。

### 2.5 拍号与小节感 (Meter & Bar)

Sonic Pi 默认没有“小节”的概念。如果你写 6/8 拍，你需要自己用代码结构来体现。

**4/4 拍结：**
```ruby
live_loop :beat_44 do
  4.times do # 明确的 4 拍结构
    play :c2
    sleep 1
  end
end
```

**6/8 拍结构（两组 3 拍）：**
```ruby
live_loop :beat_68 do
  2.times do # 两个大拍
    play :c2, amp: 1.0 # 强拍
    sleep 0.5
    play :c2, amp: 0.5 # 弱
    sleep 0.5
    play :c2, amp: 0.5 # 弱
    sleep 0.5
  end
end
```

---

## 3. 本章小结

*   **翻译思维**：你是给计算机写乐谱的指挥家。
*   **音高**：使用 `:c4` 格式，遵循调号逻辑选用 `:fs` 或 `:gb`。
*   **时间**：`sleep` 是“间隔”，不是“音长”。
*   **音长**：`release` 是“音长”。`release` 短于 `sleep` = 断奏；`release` 等于或长于 `sleep` = 连奏。
*   **精度**：永远使用 `1.0/3` 处理三连音，拒绝 `0.33`。
*   **节奏型**：附点是古风的灵魂，三连音是 Zimmer 的引擎。

---

## 4. 练习题

### 基础题 (Basic)

#### 练习 2.1：C 小调音阶与调号
**目标**：熟悉 MIDI 音名与降号。
**题目**C 小调（自然小调）的音是 C, D, Eb, F, G, Ab, Bb, C。请写出上行音阶，速度为四分音符。
**要求**：必须使用降号标记（`:eb`, `:ab`...），不可以使用升号。

<details>
<summary>参考答案</summary>

```ruby
# 推荐写法：使用列表和循环（更简洁）
play_pattern_timed [:c4, :d4, :eb4, :f4, :g4, :ab4, :bb4, :c5], 1

# 或者笨办法（便于理解）：
play :c4; sleep 1
play :d4; sleep 1
play :eb4; sleep 1 # 注意这里不用 ds4
play :f4; sleep 1
play :g4; sleep 1
play :ab4; sleep 1
play :bb4; sleep 1
play :c5; sleep 1
```
</details>

#### 练习 2.2：大附点节奏 (Grand Dotted Rhythm)
**目标**：构建进行曲或史诗感的骨架。
**题目**：模拟《星球大战》或类似进行曲的开头节奏：| 强(1.5拍) - 弱(0.5拍) | 强(1.5拍) - 弱(0.5拍) |。
**提示**：使用 `amp:` 区分强弱。

<details>
<summary>参考答案</summary>

```ruby
2.times do
  play :c4, amp: 1.0  # 强
  sleep 1.5           # 附点四分
  play :g3, amp: 0.6  # 弱
  sleep 0.5           # 八分
end
```
</details>

#### 练习 2.3：理解休止符
**目标**：学会“留白”。
**题目**：写出以下谱面：| C4(1拍) | (休止1拍) | E4(0.5拍) | G4(0.5拍) | (休止1拍) |
总共 4 拍。

<details>
<summary>参考答案</summary>

```ruby
play :c4
sleep 1    # 第一拍

sleep 1    # 第二拍（休止）

play :e4
sleep 0.5  # 第三拍前半
play :g4
sleep 0.5  # 第三拍后半

sleep 1    # 第四拍（休止）
```
</details>

### 挑战题 (Challenge)

#### 练习 2.4：Zimmer 式 5/4 拍推进 (Odd Meter & Tuplets)
**目标**：体验 Hans Zimmer 在《邓刻尔克》或《沙丘》中常用的非常规拍号带来的焦虑感。
**题目**：构造一个 5/4 拍的循环（总时长 5）。
结构：3 个四分音符 + 4 个八分音符。
**挑战**：在最后一个八分音符上加上重音（amp: 1.2），其余为 amp: 0.6。

<details>
<summary>参考答案</summary>

```ruby
live_loop :five_four_tension do
  # 前3拍：稳健的四分音符
  3.times do
    play :c3, amp: 0.6
    sleep 1
  end
  
  # 后2拍：密集的八分音符，最后一下重击
  3.times do
    play :c3, amp: 0.6
    sleep 0.5
  end
  # 第4个八分音符（全小节最后一击）
  play :c3, amp: 1.2, release: 0.3 # 短促重击
  sleep 0.5
end
```
</details>

#### 练习 2.5：古风“散板” (Rubato) 模拟
**目标**：古风引子（Intro）通常没有严格节拍，称为“散板”。
**题目**：写一句笛子旋律，要求音符之间的 `sleep` 不是固定的，而是具有微小的随机性，模拟人的随意吹奏。
**提示**：使用 `rrand(min, max)` 函数来生成随机时值。例如 `sleep rrand(0.8, 1.2)`。

<details>
<summary>参考答案</summary>

```ruby
# 模拟笛子散板
use_synth :fm # 权且用 fm 模拟笛子
live_loop :sanban_flute do
  # 几个长音，时长随意
  play :e5, release: 3, amp: 0.8
  sleep rrand(2.0, 3.5) # 随意的长停顿
  
  # 几个快速经过音（装饰音），也不要太齐
  play :g5, release: 0.2
  sleep rrand(0.1, 0.2)
  play :a5, release: 0.2
  sleep rrand(0.1, 0.2)
  
  # 落音
  play :c6, release: 4, amp: 0.9
  sleep 4
end
```
</details>

#### 练习 2.6：断奏与连奏的对比 (Articulation Study)
**目标**：用代码控制“情感”。
**题目**：在一个循环中，先播放 4 个断奏（Staccato）的 C4，再播放 4 个连奏（Legato）的 C4。
**要求**：
*   断奏：`release` 时间必须小于 0.2。
*   连奏：`release` 时间必须大于等于 1.0。
*   节奏：所有音符都是四分音符 (`sleep 1`)。

<details>
<summary>参考答案</summary>

```ruby
live_loop :articulation_test do
  # Staccato Section (紧张、机械)
  4.times do
    play :c4, release: 0.1 # 声音很短
    sleep 1                # 间隔依然是一拍
  end

  # Legato Section (抒情、连绵)
  4.times do
    play :c4, release: 1.2 # 声音稍微重叠
    sleep 1
  end
end
```
</details>

---

## 5. 常见陷阱与错误 (Gotchas)

### 5.1 整数除的诅咒 (Integer Division)
这是程序员最熟悉但音乐人最易崩溃的坑。
在 Ruby (Sonic Pi 的底层语言) 中：
*   `1 / 3` 等于 `0` （整数除以整数，结果只保留整数部分）。
*   `1.0 / 3` 等于 `0.333333...`。

**现象**：你写了 `sleep 1/2`，想表达八分音符，结果声音像机关枪一样重叠爆发（因为 sleep 变成了 0）。
**解决**：**永远在分子或分母上加个 `.0`**。习惯写 `sleep 0.5` 而不是 `1/2`，写 `1.0/3` 而不是 `1/3`。

### 5.2 `release` 导致的“糊成一团”
**现象**：你的古风旋律听起来很浑浊，像是在混响巨大的浴室里弹快板。
**原因**：你可能设置了很长的 `release`（例如默认的 release 可能是 1），但你在写快速的十六分音符（`sleep 0.25`）。前一个音还没消失，后三个音就叠上去了。
**解决**：**快节奏配短 release，慢节奏配长 release**。

### 5.3 复制粘贴的错位
**现象**：你从外部复制了一段代码，行时报错 `Unexpected end` 或缩进极其混乱。
**原因**：Sonic Pi 对代码块 `do ... end` 的配对要求严格。
**解决**：善用快捷键 **Alt + Shift + F** (Windows) 或 **Cmd + Shift + F** (Mac) 自动美化/对齐代码。如果对齐后发现最后少了一个 `end`，那就是因为你少复制了。
