# 第 1 章：Sonic Pi 快速上手——从 8 小节开始

## 1.1 开篇：代码作为乐器

欢迎来到 Sonic Pi 的世界。作为一个能识谱、懂和弦走向，并对频谱有直觉的创作者，你可能会问：**为什么要放弃直观的钢琴卷帘窗（Piano Roll），转而使用代码？**

在 DAW 中，你是在**描绘**波形；而在 Sonic Pi 中，你是在**描述**产生波形的逻辑。对于我们设定的目标风格，这种思维转换带来了巨大的优势：

1.  **Hans Zimmer 式的“墙” (Wall of Sound)**：Zimmer 的配乐常由数十层精准同步的 16 分音符脉冲构成（如 *The Dark Knight*）。在 DAW 中，稍微量化误差或复制粘贴错误会导致“相位抵消”或律动松散。而在代码中，`sleep 0.25` 是数学上的绝对精确，能产生极具压迫感的“机械整齐度”。
2.  **久石让式的“有机织体”**：久石让的极简主义风格（Minimalism）常包含不断重复但逐渐演变的分解和弦。用代码写一个“每次重复时有 20% 概率把根音提高八度”的函数，比手画 100 个小节的 MIDI 还要生动且高效。
3.  **古风/戏腔的“气韵”**：戏腔讲究“非网格化”的拖腔。通过代码控制微小的随机时间偏移（Randomization），我们可以模拟出人类演奏时的“呼吸感”，而不是死板的机器节奏。

本章的目标是让你建立**“时间=代码行”**的直觉，并写出第一个包含鼓、贝斯、和声的 8 小节循环。

---

## 1.2 安装与音频设置：物理与数字的桥梁

Sonic Pi 的底层是一个强大的音频合成引擎（SuperCollider）。作为熟悉频谱分析的，需要理解这里的**采样率**与**缓冲**设置，因为它们直接关系到声音的**瞬态响应（Transient Response）**。

### 1.2.1 采样率 (Sample Rate)
*   **建议**：保持与你的系统或外置声卡一致，通常为 **44.1kHz** 或 **48kHz**。
*   **坑**：如果 Sonic Pi 的采样率与声卡不匹配，系统会进行实时重采样（Resampling），这会引入**混叠失真（Aliasing）**，导致高频部分变得“脏”且有金属味，破坏古风音乐需要的“清透感”。

### 1.2.2 缓冲区大小 (Buffer Size) 与延迟
这里谈论的是音频硬件的 Buffer，不是代码界面的 Buffer。

*   **原理**：Buffer 是一个蓄水池。CPU 计算出的音频数据先存入池中，填满后再一次性送往声卡。
*   **听觉影响**：
    *   **Buffer 过小 (e.g., 64/128 samples)**：CPU 来不及填满水池，声卡就来取数据，导致“空缺”。这在听感上就是**爆音 (Crackles/Xruns)**，频谱上表现为全频段的宽带声脉冲。
    *   **Buffer 过大 (e.g., 1024/2048 samples)**：水池很大，CPU 很从容，声音很纯净。但延迟（Latency）会增加。
*   **Rule-of-Thumb (经验法则)**：
    *   **创作阶段**：设为 `512` 或 `1024`。我们是在“写作”，不是在“手指击鼓”，100ms 的延迟完全不影响代码生效，且能保证你在叠加几十层 Zimmer 式大动态管弦乐时不会爆音。
    *   **现场 Live Coding**：设为 `256`。

---

## 1.3 界面与运行模型：这不是播放器

Sonic Pi 的操作逻辑不是“播放录音带”，而是**“实时解释执行”**。

### 1.3.1 核心按钮
1.  **Run (运行)**：发送当前代码给合成引擎。
    *   *注意*：如果你在一个 `live_loop` 已经在跑的时候再次点击 Run，Sonic Pi 会**动态更新**那个循环的逻辑，而不会停止它。这就是 Live Coding 的核心。
2.  **Stop (停止)**：类似于“紧急切断”。它会立即停止所有正在发声的合成器节点。
3.  **Rec (录音)**：将总线输出录制为 WAV 文件。

### 1.3.2 示波器 (Scope)
界面右上角的 Scope 显示了输出信号的波形。
*   **用途**：当你使用立体声声像（Pan）或复杂的相位调制（Phasing）效果时，观察 Scope 的李萨如图形（Lissajous）可以帮你判断**相位兼容性**。如果图形是一条垂直线，说明是单声道；如果是一团乱麻，说明立体声宽度很大。

### 1.3.3 日志 (Logs)
右侧面板。它会显示：
*   `{run: 1, time: 0.0}`：当前运行的 ID 和节拍时间。
*   `synth :prophet, {note: 60, ...}`：具体触发了什么音。
*   **调试技巧**：如果声音不响但 Log 在滚动，说明代码逻辑是对的，问题出在音频路由或合成器参数（如 `amp: 0`）。

---

## 1.4 代码最小语法：从乐理到字符

我们需要将五线谱上的概念精准翻译成 Ruby 语法。

### 1.4.1 音高 (Pitch)：频率的抽象
Sonic Pi 接受三种音高表示法，它们可以混用：
1.  **音名符号**：`:c4` (中央C), `:fs3` (F#3), `:eb5` (Eb5)。这是最直观的。
2.  **MIDI 编号**：`60` (中央C)。适合数学运算（例如：`60 + 12` 升八度）。
3.  **赫兹 (Hz)**：可以直接输入频率 `440`。这对于做**微分音（Microtonal）**或模拟古琴不标准的定弦非常有用。

### 1.4.2 时值 (Duration)：`sleep` 的哲学
在 Sonic Pi 中，**时间是阻塞的**。程序读到 `sleep 1` 会真的“睡” 1 拍，然后再执行下一行。

$$ \text{实际秒数} = \frac{60}{\text{BPM}} \times \text{sleep数值} $$

| 五线谱 | 代码 (默认 BPM) | 说明 |
| :--- | :--- | :--- |
| **四分音符** (Quarter) | `sleep 1` | 一拍 |
| **八分音符** (Eighth) | `sleep 0.5` | 半拍 |
| **十六分音符** (16th) | `sleep 0.25` | Zimmer 动作类配乐的基础网格 |
| **三连音** (Triplet) | `sleep 1.0/3` | 注意要写 `1.0` 这种浮点数，写 `1/3` 会变成 0 |

### 1.4.3 容器：`do ... end` 块
在 Sonic Pi 中，几乎所有的结构（循环、函数、随机选择）都包裹在 `do ... end` 之间。
```ruby
# 这是一个代码块的示例
live_loop :my_loop do  # 开始
  play 60
  sleep 1
end                    # 结束
```

---

## 1.5 第一个 8 小节：构建“电影感”织体

我们将直接跳过单调的“滴滴嘟嘟”，构建一个包含 **Zimmer 式低频脉冲**、**久石让式清澈和弦** 和 **古风旋律动机** 的 8 小节片段。

请将以下代码复制到 Buffer 0 中：

```ruby
# 全局设定：速度设为 80 BPM，适合史诗感或舒缓的古风
use_bpm 80

# === 层级 1: 节奏与脉冲 (Rhythm & Pulse) ===
# 模拟 Hans Zimmer 的 Action Strings 或合成器 Bass 脉冲
# 这种 16 分音符的持续驱动是现代配乐的骨架
live_loop :pulse do
  # 选用一个模拟合成器贝斯音色
  use_synth :tb303
  
  # tick 命令：每次循环让计数器 +1
  # look 命令：查看当前计数器的值
  # 这是一个简单的逻辑：每 4 个音符(一拍)，第一个音大声(0.3)，后个小声(0.15)
  # 这种强弱规律创造了“推进感”
  if (look % 4) == 0
    amp_val = 0.3
  else
    amp_val = 0.15
  end
  
  # cutoff: 滤波器截止频率。设低一点(70)让声音闷在下面，不抢高频旋律
  play :c2, release: 0.2, cutoff: 70, amp: amp_val
  
  sleep 0.25 # 16分音符网格
  tick       # 计数器推进
end

# === 层级 2: 氛围和声 (Atmospheric Chords) ===
# 模拟久石让风格的铺底，使用长音 Pad
live_loop :chords do
  # :hollow 是一种带有风声特性的合成器，非常适合古风留白
  use_synth :hollow
  
  # 定义和弦进行：C大调 I - V - vi - IV (C - G - Am - F)
  # 这里的 ring 是环形列表，tick 会自动循环读取
  current_chord = (ring 
    [:c3, :e3, :g3, :b3],  # Cmaj7
    [:g2, :d3, :g3, :b3],  # G/B (转位)
    [:a2, :c3, :e3, :g3],  # Am7
    [:f2, :c3, :e3, :a3]   # Fmaj7
  ).tick
  
  # attack: 起音时间。4秒的缓慢淡入，制造像云雾一样的感觉
  # sustain: 保持时间。
  # release: 释放时间。
  play current_chord, attack: 4, sustain: 2, release: 2, amp: 1.2
  
  sleep 8 # 每个和弦持续 2 小节 (8拍)
end

# === 层级 3: 旋律 (Melody) ===
# 简单的五声音阶动机，模拟钢琴或钟琴
live_loop :melody do
  use_synth :piano
  
  # sync 极其重要！它强制让旋律层等待 :pulse 层完成一轮
  # 这保证了无论你何时点击 Run，鼓和旋律永远对齐，不会乱拍
  sync :pulse 
  
  # 五声音阶：C D E G A
  play_pattern_timed [:c5, :d5, :e5, :g5], [0.5, 0.5, 1, 2]
  
  play_pattern_timed [:a5, :g5, :e5, :d5], [0.5, 0.5, 1, 2]
end
```

### 代码解析与声学直觉
1.  **滤波器 (`cutoff: 70`)**：在 `:pulse` 层，我们使用了低通滤波器。作为懂频谱的你，知道这能把 1000Hz 以上的空间留给 `:melody` 层，避免低频乐器和高频旋律“打架”。这就在代码里完成了初步的**混音（Mixing）**。
2.  **包络 (`attack: 4`)**：在 `:chords` 层，长 Attack 意味着声音慢慢浮现，这是营“电影感”和“古风意境”的关键——不要让所有声音都瞬间冲出来。
3.  **同步 (`sync`)**：这是 Sonic Pi 最强大的功能。它解决了传统编程很难处理的“对齐”问题。

---

## 1.6 如何“录下来”：内录与分轨 (Stems)

如果你只是点击顶部的 `Rec`，你会得到一个混合好的 WAV。但这不够专业。为了配合人声处理，我们需要**分轨导出 (Stems)**。

### 1.6.1 为什么要分轨？
人声（Vocal）通常占据中频（500Hz - 2kHz）。如果你把伴奏录成了一轨，当人声切入时，你无法单独把伴奏里的钢琴中频挖掉，也无法只给鼓组加侧链压缩（Sidechain Compression）。

### 1.6.2 Sonic Pi 的分轨流程
没有“一键导出所有分轨”的功能，我们通常这样做：

1.  **独奏 (Solo) 鼓组**：
    *   在 `:chords` 和 `:melody` 的 `live_loop` 前面加上 `stop` 命令，或者注释掉它们。
    *   点击 `Run`，现在只听到鼓。
    *   点击 `Rec`，录制 8-16 小节。保存为 `Drums.wav`。
2.  **独奏和声**：
    *   恢复 `:chords`，注释掉其他。
    *   点击 `Rec`。保存为 `Pad.wav`。
3.  **独奏旋律**：
    *   同理。保存为 `Melody.wav`。

> **💡 经验法则 - 头部静音**
>
> 录音时，**先点 Rec，再点 Run**。让音频文件开头有一段空白。
> 拖入 DAW (Logic/Cubase) 后，你只需要把所有波形的**第一个声音瞬态（Transient）** 对齐到网格的第 2 小节或第 3 小节开头，所有轨道就会完美同步。

---

## 1.7 本书约定

为了后续章节的学习，我们约定以下代码规范：

*   **命名**：`live_loop` 的名字要有意义，如 `:drums_kick`、`:piano_arp`。
*   **版本管理**：如果你想试一个新的旋律，不要删掉旧的。复制那个 loop，把旧的改名为 `:melody_v1` 并注释掉，新的叫 `:melody_v2`。
*   **注释**：用 `#` 写下你的乐理意图。例如 `# 这里的 sleep 是为了让出人声的空间`。

---

## 1.8 练习题

### 基础题 (50%)

**练习 1：音高听写与修正**
下面的代码试图播放 C 大调音阶（C D E F G A B C），但听起来很怪。请找出错误的音符并修正它。
```ruby
live_loop :scale_test do
  play :c4
  sleep 0.5
  play :d4
  sleep 0.5
  play :e4
  sleep 0.5
  play :fs4  # <--- 听听这里是不是怪怪的？
  sleep 0.5
  play :g4
  sleep 0.5
  play :a4
  sleep 0.5
  play :bb4 # <--- 这里呢？
  sleep 0.5
  play :c5
  sleep 0.5
end
```
<details>
<summary>点击查看提示</summary>
C 大调没有升降号。`fs` 是 F Sharp (#F)，`bb` 是 B Flat (bG)。你需要把它们改成自然音。
</details>
<details>
<summary>点击查看参考答案</summary>
将 `:fs4` 改为 `:f4`，将 `:bb4` 改为 `:b4`。
</details>

**练习 2：呼吸节奏**
使用 `sample :bd_tek` 写一个心跳节奏。要求：
*   不是机械的 `sleep 1`。
*   而是“咚(0.5s) -> 咚(1s) -> 咚(0.5s) -> 咚(1s)”的循环。
<details>
<summary>点击查看提示</summary>
你需要两个 sample 命令和两个 sleep 命令交替。
</details>
<details>
<summary>点击查看参考答案</summary>

```ruby
live_loop :heartbeat do
  sample :bd_tek
  sleep 0.5
  sample :bd_tek
  sleep 1
end
```
</details>

**练习 3：和弦的声音设计**
复制 1.5 节中的 `:chords` 循环。尝试修改 `attack` 和 `release` 参数。
*   任务：让声音变成短促的“拨弦感”，而不是长音 Pad。
<details>
<summary>点击查看提示</summary>
拨弦乐器（如吉他、竖琴）的特性是：Attack 极短（接近 0），Release 较短（1 左右）。同时尝试把 synth 改为 `:pluck`。
</details>

---

### 挑战题 (50%)

**练习 4：Hans Zimmer 的 5/4 拍推进**
Zimmer 喜欢用非 4/4 拍的奇数拍子制造不稳定感。
请写一个 Loop：
*   每小节 5 拍。
*   节奏型是：**3 + 2** (即 `强 弱 弱 | 强 弱`)。
*   使用 `:drum_heavy_kick`。
*   **关键**：强拍 `amp: 1`，弱拍 `amp: 0.5`。
<details>
<summary>点击查看提示</summary>
这需要 5 个 `sample` 命令。前 3 个一组，后 2 个一组。第 1 个和第 4 个是强拍。
或者，使用 `ring` 和 `tick` 来控制 amp 值。
</details>
<details>
<summary>点击查看参考答案</summary>

```ruby
live_loop :cinematic_5_4 do
  # 定义 5 拍的重音模式: 1=强, 0.5=弱
  # 模式: 强 弱 弱 强 弱 (3+2 结构)
  amp_pattern = (ring 1, 0.5, 0.5, 1, 0.5)
  
  sample :drum_heavy_kick, amp: amp_pattern.tick
  sleep 1 # 每拍一下
end
```
</details>

**练习 5：古风的“留白”概率**
写一个风铃或古筝的高音 Loop。
*   要求：每 0.25 拍执行一次代码，但**只有 20% 的概率**真正发出声音。
*   提示：使用 `one_in(5)` 函数。如果是 true 则 play，否则不 play。
<details>
<summary>点击查看提示</summary>
`if one_in(5)` ... `play ...` `end`。这是生成稀疏织体的核心技巧。
</details>
<details>
<summary>点击查看参考答案</summary>

```ruby
live_loop :guzheng_random do
  use_synth :pluck
  
  # 每次都有 1/5 的概率响，制造稀疏感
  if one_in(5)
    # 在五声音阶中随机选一个高音
    play (choose [:c6, :d6, :e6, :g6, :a6]), amp: 0.5
  end
  
  sleep 0.25
end
```
</details>

---

## 1.9 常见陷阱与错误 (Gotchas)

### 陷阱 1：`live_loop` 命名冲突
**错误**：复制了两个 `live_loop` 但忘记改名，两个都叫 `:beat`。
**后果**：Sonic Pi 会认为你是在修改同一个 Loop。只有最后一个被定义的 `:beat` 会运行，前一个会消失或被覆盖。
**调试**：确保每个 `live_loop` 的名字（冒号后面的部分）在整个文件中是唯一的。

### 陷阱 2：休止符的误解
**错误**：想写一个休止符，于是写了 `play 0` 或 `amp: 0`。
**解释**：虽然听不到声音，但这实际上还是消耗了 CPU 去合成一个静音。
**正确做法**：直接 `sleep` 即可。`sleep` 本身就是“不发声的时间流逝”，也就是休止符。

### 陷阱 3：中文标点
这是新手最容易遇到的 Syntax Error。
*   ❌ `play :c4， amp: 1` (中文逗号)
*   ✅ `play :c4, amp: 1` (英文逗号)
*   Sonic Pi 编辑器对于无法识别的字符通常不会变色（保持黑色或灰色），而合法的代码会有颜色（粉色、蓝色等）。**看颜色**是快速排错的方法。
