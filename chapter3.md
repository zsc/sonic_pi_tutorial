# 第 3 章：时间、同步与 Live Coding 思维

## 3.1 开篇：时间是你的画布，更是你的算法

在学习五线谱时，我们习惯将时间视作**横向的空间**：小节线将时间切片，音符占据一定的宽度。
在 DAW（如 Cubase/Logic）中，时间是**线性的**：播放头（Playhead）从左向右扫过，触发沿途的事件。

但在 Sonic Pi 中，**时间是代码执行的休止符**。
当你按下 `Run` 时，并非生成了一条完整的音频轨道，而是启动了一系列**并行（Concurrent）**的线。这就像你在指挥一个乐队，你不是在录音带上剪辑，而是给每位乐手（鼓手、古筝手、大提琴手）分别发了一张谱子，并告诉他们：“你们同时开始，每人按自己的循环去演奏。”

本章的核心目标是让你从“剪辑师”思维转变为“指挥家”思维：
1.  **全局控制**：如何用代码定义绝对时间与相对时间。
2.  **并发执行**：如何让鼓和旋律互不干扰地运行。
3.  **相位与对齐**：如何解决“跑拍”问题，以及如何利用“错位”创造 Hans Zimmer 式的复杂织体。
4.  **弹性时间**：如何在严谨的电子节拍中，写入古风戏腔特有的“散板”与呼吸感。

---

## 3.2 节拍网格与全局速度 (`use_bpm`)

在物理世界，时间以秒为单位。在音乐世界，时间以拍（Beat）为单位。Sonic Pi 的 `sleep` 命令默认以**拍**为单位，而不是秒。

### 3.2.1 BPM 的换算逻辑
默认情况下，Sonic Pi 的 BPM（Beats Per Minute）为 60。
此时：
*   `sleep 1` = 1 拍 = 1 秒。

当你设定 `use_bpm 120` 时：
*   `sleep 1` = 1 拍 = 0.5 秒。
*   `sleep 0.5` = 0.5 拍 = 0.25 秒。

**Rule of Thumb：心理映射表**
对于习惯读谱的你，请死记以下对应关系（假设拍号为 4/4）：

| 代码指令 | 音乐术语 (Note Value) | 常见用途 |
| :--- | :--- | :--- |
| `sleep 4` | 全音符 (Whole Note) | Pad 长音、锣声残响 |
| `sleep 2` | 二分音符 (Half Note) | 贝斯根音、人声长腔 |
| `sleep 1` | 四分音符 (Quarter Note) | 底鼓 (Kick)、行进的低音 |
| `sleep 0.5` | 八分音符 (8th Note) | Hi-hat、古筝分解和弦 |
| `sleep 0.25` | 十六分音符 (16th Note) | Zimmer 式弦乐短奏 (Spiccato)、滚奏 |
| `sleep 1.0/3` | 三连音 (Triplet) | 史诗战鼓、爵士摇摆感 |

> **注意**：三连音必须写成 `1.0/3` 而不是 `1/3`。因为在编程中整数除法 `1/3` 会等于 `0`，而 `1.0/3` 会得到 `0.3333...`。

### 3.2.2 风格化的速度选择
速度决定了作品的物理惯性。
*   **华语古风 / 戏腔 (60 - 90 BPM)**：
    较慢的速度允许更复杂的装饰音（倚音、波音）存在。戏腔需要“拖腔”，在 120 BPM 下拖腔会显得急促焦虑。
*   **Hans Zimmer / 史诗管弦 (100 - 140 BPM)**：
    动作场景通常在 120-140 BPM。Zimmer 著名的《Mombasa》或《Dark Knight》配乐利用高速度下的 16 分音符 Ostinato（固定音型）制造极强的压迫感。
*   **久石让 / 治愈系 (80 - 110 BPM)**：
    步行的速度，不急不徐。

---

## 3.3 Live Loop：心脏与血管

`live_loop` 是 Sonic Pi 最伟大的发明。它不仅仅是“循环”，它是**实时可编辑的独立线程**。

### 3.3.1 并发模型图解
想象你在写两行五线谱：
*   上面一行是笛子。
*   下面一行是鼓。

在 Sonic Pi 中，你定义两个 `live_loop`。当你按下 Run，它们**同时**启动，并在各自的轨道上无限奔跑。

```text
时间轴 (Time) --->

Live Loop :drums  (线 1)
|-- Kick -- sleep 1 -- Snare -- sleep 1 --| (回到开头) ↺
|                                         |
v                                         v

Live Loop :flute  (线程 2)
|-- G4 -- sleep 0.5 -- A4 -- sleep 0.5 -- E4 -- sleep 1 --| (回到开头) ↺
```

### 3.3.2 实时编码（Live Coding）的本质
在其他编程语言中，修改代码通常需要“停止 -> 编译 -> 重启”。
在 Sonic Pi 中：
1.  你正在听一个 `live_loop` 演奏。
2.  你修改了循环内的音符或 sleep 时间。
3.  你再次点击 **Run**。
4.  **旧的循环**会跑完它当前的这一圈。
5.  **新的代码**会在下一圈的起点无缝接管。

**这就是为什么它是“Live”的**。你可以像 DJ 一样，在演出过程中把鼓点从 4/4 拍改成 3/4 拍，而不会出现音乐中断或爆音。

---

## 3.4 同步 (`cue` / `sync`)：指挥棒的作用

这是新手遇到最大的坑：**“为什么跑着跑着，鼓和旋律就错开了？”**

### 3.4.1 漂移的来源
假设你有两个循环：
*   循环 A 长度正好 4 拍。
*   循环 B 你写了很多复杂的切分音，加起来长度是 4.01 拍（可能因为手误，或者浮点数微小误差）。
*   100 遍循环后，两者就会相差 1 拍，完全乱套。

### 3.4.2 核心机制：阻塞与等待
Sonic Pi 的 `sync` 命令是一个**阻塞（Blocking）**命令。
当程序执行到 `sync :name` 时，它会**立刻停止执行，进入休眠状态**，死死地等待，直到它收到了一个名为 `:name` 的 `cue` 信号。

### 3.4.3 工业标准模式：主时钟 (Master Clock)
为了写出像电影配乐一样严谨对齐的作品，**不要**让乐器互相 sync（比如不要让吉他去 sync 鼓，万一你想把鼓静音呢？）。
你需要建立一个看不见的“指挥家”。

**代码范式（请背诵）：**

```ruby
# 1. 建立指挥家 (Metronome)
live_loop :meter do
  sleep 4       # 每 4 拍一个小节
end

# 2. 乐器 A (Kick)
live_loop :kick do
  sync :meter   # 等待指挥挥下！
  sample :bd_haus
  sleep 1       # 即使这里写错了导致没对齐...
  sample :bd_haus
  sleep 1.1     # <--- 故意写错
                # ...下一轮开始前，sync 会强制它重新等待指挥棒
end

# 3. 乐器 B (Piano)
live_loop :piano do
  sync :meter   # 同时也等待指挥棒，确保与 Kick 在同一点启动
  play :c4
  sleep 4
end
```

**解析**：
即使 `:kick` 内部的时间算错了，它跑完一轮后，会回到开头的 `sync :meter`。如果此时 `:meter` 还没走完 4 拍，`:kick` 就会停下来等；如果 `:meter` 已经发出了信号，它就会瞬间对齐。
**这保证了所有乐器永远在小节线（Downbeat）上重合。**

---

## 3.5 摇摆（Swing）与微小偏移：拒绝机械感

电脑的精确是音乐的敌人。
*   **Hans Zimmer**：即使是机械的脉冲，也会通过音色的变化（滤波器打开/关闭）来模拟呼吸。
*   **古风**：弹拨乐器（琵琶/古筝）的轮指，戏腔的转音，绝对不可能是死在 Grid 上的。

### 3.5.1 摇摆 (Swing) 的算法实现
爵士或古风里的“摇曳感”，本质上是把连续的两个八分音符变成了“长-短”组合。
*   标准：0.5 + 0.5
*   Swing：0.6 + 0.4 （温和） 或 0.66 + 0.33 （三连音手感）

在 Sonic Pi 中，我们不需要手动算，可以使用 `use_swing` (新版特性) 或手动构建 helper 函数（经典做法，利于理解）。本阶段建议手动控制：

```ruby
# 手动 Swing 示例
live_loop :guzheng_swing do
  # 第一个音长一点
  play :c5
  sleep 0.3
  # 第二个音短一点
  play :d5
  sleep 0.2
end
```

### 3.5.2 Humanize (人性化微差)
对于古风独奏（Solo），我们不希望它完全对齐。
可以使用 `rtand(min, max)` 在 sleep 时间上增加极微小的随机量。

`sleep 0.25 + rrand(-0.02, 0.02)`

这点微小的误差（+/- 20毫秒）是人耳识别“这是真人”还是“这是 MIDI”的关键阈值。

---

## 3.6 复杂的节奏错觉：复节奏 (Polyrhythms)

这是 Zimmer / Junkie XL 风格配乐的秘密武器。通过不同长度的循环叠加，制造“一直在变但又有规律”的听感。

**3 对 4 (Hemiola)**：
*   **低音层**：每 4 拍循环一次（代表稳固的大地/秩序）。
*   **中高频层**：每 3 拍循环一次（代表冲突/混乱/紧迫感）。

**视觉化：**
```text
Loop 4: X . . . X . . . X . . . (稳)
Loop 3: X . . X . . X . . X . . (急)
Result: X . . X X . X . X . X . (复杂的复合节奏)
```

在 Sonic Pi 中实现极其简单：
1. 写一个 Loop `sleep 4`。
2. 写另一个 Loop `sleep 3`。
3. **不要让它们互相 sync**（或者只在很长的周期 sync 一次）。让它们自然错开。

这种技法在描写“两军对垒”或“内心纠结”的场景时极为有效。

---

## 3.7 本章小结

1.  **时间观念**：Sonic Pi 的时间是基于代码执行的，`sleep` 控制的是线程挂起的时长。
2.  **BPM 映射**：建立 `sleep 1` = 四分音符的直觉。古风偏慢（60-90），诗偏快（100+）。
3.  **Live Loop**：它们是并发的演员。修改代码后，新指令在下一个循环周期生效。
4.  **Master Clock 模式**：**必须掌握**的技巧。用一个空的 `live_loop` 发送 `cue`，其他所有乐器 `sync` 它，以保证永远对齐。
5.  **人性化**：通过微调 `sleep` 时间制造 Swing 和 Humanize，是打破“机器味”的关键。
6.  **复节奏**：利用循环长度的最小公倍数原理（如 3 vs 4），制造电影感的节奏张力。

---

## 3.8 练习题

### 基础题（熟悉手感）

**习题 1：建立你的第一个指挥家**
编写代码：
1. 设定 BPM 为 90。
2. 创建一个 `:metronome` 循环，每 4 拍 `sleep` 一次（不需要发声，Sonic Pi 的 loop 机制本身就是对齐点，但为了显式控制，可以使用 dummy loop）。
3. 创建一个 `:kick` 循环，`sync` 那个指挥家，并演奏 4 下底鼓。
*提示：思考 `sync` 应该放在 loop 内部的哪里？*

<details>
<summary>参考答案</summary>

```ruby
use_bpm 90

# 指挥家：只负责定义小节长度
live_loop :metronome do
  sleep 4
end

# 鼓手：跟随指挥家
live_loop :kick do
  sync :metronome  # 每次循环开始前，确认对齐指挥家
  4.times do
    sample :bd_haus
    sleep 1
  end
end
```
</details>

**习题 2：古风的“板眼”练习**
设定 BPM 60。
1. “板”（强拍）：在第 1 拍播放一个深沉的鼓声（如 `:bd_tek`）。
2. “眼”（弱拍）：在第 2、3、4 拍播放轻微的木头敲击声（如 `:elec_tick` 或 `:drum_cymbal_closed`，用 `amp` 调小音量）。
3. 将其写在一个循环内。

<details>
<summary>参考答案</summary>

```ruby
use_bpm 60

live_loop :banyan do
  # 板 (强)
  sample :bd_tek, amp: 1
  sleep 1
  
  # 眼 (弱) - 重复3次
  3.times do
    sample :elec_tick, amp: 0.5
    sleep 1
  end
end
```
</details>

**习题 3：简单的 3 对 2 练习**
1. 循环 A：每 2 拍响一下（低音）。
2. 循环 B：每 3 拍响一下（高音）。
3. 它们跑起来，数数看每过多少拍它们会同时响一次？（答案应该是 6 拍）。

<details>
<summary>参考答案</summary>

```ruby
live_loop :rhythm_2 do
  play :c3
  sleep 2
end

live_loop :rhythm_3 do
  play :g5
  sleep 3
end
# 这是一个听感实验，仔细听重合点带来的重音感
```
</details>

---

### 挑战题（进阶思维）

**习题 4：Hans Zimmer 的“行动”模式（16 分音符推进）**
很多动作片配乐都有一个从未停歇的“哒哒哒哒”基底。
1. 设定 BPM 130。
2. 编写一个 `:ostinato` 循环。
3. 让它连续演奏 16 个十六分音符（`sleep 0.25`）。
4. **关键挑战**：让第 1、5、9、13 个音（即每拍的重音）大声一点，其他的音小一点。
*提示：可以使用 `tick` 和取余数 `%` 操作符，或者简单的 `if` 判断，或者把 4 个音写成一组重复 4 次。*

<details>
<summary>参考答案（简单版与进阶版）</summary>

```ruby
use_bpm 130

# 简单版写法：暴力罗列
live_loop :ostinato_simple do
  4.times do
    sample :elec_blip, amp: 1.0  # 重音
    sleep 0.25
    sample :elec_blip, amp: 0.5  # 弱
    sleep 0.25
    sample :elec_blip, amp: 0.5  # 弱
    sleep 0.25
    sample :elec_blip, amp: 0.5  # 弱
    sleep 0.25
  end
end

# 进阶版写法：使用 Ring 和 Tick (后续章节会详解，这里先体验)
live_loop :ostinato_pro do
  # 定义强弱规律
  vol = (ring 1.0, 0.5, 0.5, 0.5).tick
  sample :bass_hit_c, amp: vol, cutoff: 80
  sleep 0.25
end
```
</details>

**习题 5：古风“散板”前奏模拟（随机时值与留白）**
我们要模拟一个古琴乐手在酒醉后的即兴。
1. 创建一个循环，使用 `:pluck` 或 `:guit_em9` 音色。
2. 音高限制在五声音阶内（`scale(:c4, :pentatonic)`）。
3. 每次演奏后，休止的时间是**随机的**，范围在 0.2 到 3 秒之间。
4. 加上长混响，让声音在留白中回荡。
*提示：这是制造“意境”最便宜的方法。*

<details>
<summary>参考答案</summary>

```ruby
with_fx :reverb, room: 0.9, mix: 0.7 do # 大混响
  live_loop :drunk_guzheng do
    # 随机挑一个音
    n = scale(:c4, :pentatonic).choose
    
    # 随机力度和释放时间
    play n, amp: rrand(0.5, 0.9), release: rrand(2, 5)
    
    # 关键：随机的留白时间 (Rubato)
    sleep rrand(0.2, 3) 
  end
end
```
</details>

**习题 6：同步灾难修复**
（这是一个脑力调试题）
阅读以下代码，找出为什么 `:melody` 永远不会发声？

```ruby
live_loop :beat do
  sample :bd_haus
  sleep 1
end

live_loop :melody do
  sync :beats    # <--- 注意这里
  play :c4
  sleep 1
end
```
*提示：检查名字的拼写。*

<details>
<summary>参考答案</summary>
错误在于 `sync :beats` 多了一个 `s`。
`:beat` 循环的名字是 `:beat`。Sonic Pi 在等待一个名为 `:beats` 的信号，但永远等不到，所以 `:melody` 线程被永久阻塞（死锁）。
</details>

---

## 3.9 常见陷阱与错误 (Gotchas)

### 1. 死亡静默 (The Deadlock)
**象**：点击 Run，没有任何报错，但某个乐器就是不响。
**原因**：通常是因为 `sync` 的目标拼写错误（如习题 6），或者 sync 的目标循环本身崩溃/未运行。
**调试**：在 `sync` 语句前加一行 `print "Waiting..."`，在之后加一行 `print "Resumed!"`。如果你只看到 Waiting 却没看到 Resumed，那就是卡在 sync 上了。

### 2. 无限循环崩溃 (Zero Sleep)
**现象**：Sonic Pi 界面卡死，甚至需要强制关闭软件。
**原因**：`live_loop` 中没有 `sleep`，或者 `sleep 0`。
**原理**：计算机试图在 0 秒内执行无限次指令，CPU 瞬间 100%。
**防范**：养成习惯，写 `live_loop` 先写 `sleep 1`，再写其他的。

### 3. 这里的 1 不是那里的 1 (Global vs Local BPM)
**现象**：你以为你改了速度，但有些循环没变。
**原因**：`use_bpm` 如果写在 `live_loop` **内部**，它只影响该循环。如果写在**外部**，它通常只影响后续读取的代码。
**最佳实**：尽量在文件最顶端设定全局 BPM。如果需要变速，使用 `set_bpm`（后续章节讲解）或在所有循环内引用同一个变量。

### 4. 为什么我的古风像儿歌？
**原因**：节奏太整齐（Quantized）。
**解决**：这是“代码音乐”的通病。在第 3.5 节学到的 Humanize 技巧（微小的随机 sleep）和长混响（Reverb）是解决这个问题的核心手段。不要让每个音都精准落在 1.000 秒上。
