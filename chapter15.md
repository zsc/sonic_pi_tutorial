# 第 15 章：实战项目 A——90 秒《古风/戏腔 × 弦乐氛围 × 轻电子》完整制作

## 15.1 项目背景与设计哲学

你接到一个 90 秒的概念视频配乐需求。
**画面描述**：
> 镜头从云端俯瞰山峦（0-15s），穿过云层看到一位戏曲装扮的角色在空旷的现代舞台上独舞（15-45s），动作逐渐激烈，背景出现粒子特效与机械结构（45-75s），最后画面定格在角色的眼部特写，瞬间黑屏（75-90s）。

**音乐关键词**：
- **核心冲突**：传统的柔美（古风/戏腔） vs 现代的冷冽（Zimmer 式脉冲/合成器）。
- **空间感**：巨大的、空旷的，人声必须贴耳。
- **色彩**：D 小调（D Minor）。这是一个既能表现“悲剧感”又能承载“史诗感”的调性。

**技术目标**：
1.  **全局状态机**：不再线性堆砌 `sleep`，而是用“指挥官”模式控制段落。
2.  **织体融合**：如何让五声音阶（Pentatonic）与电子音序（Sequencer）不打架。
3.  **动态自动化**：使用 Sonic Pi 的 `control` 功能绘制音量与滤波器的包络曲线。

---

## 15.2 工程架构：指挥官模式 (The Conductor Pattern)

在 90 秒的项目中，如果还在用 `sleep 90` 这种写法，调试将是噩梦。我们采用**状态驱动（State-Driven）**的架构。

### 15.2.1 时间轴规划

我们将全曲划分为四个状态（State），存储在全局变量 `:stage` 中：

| 时间 | 状态名 (`:stage`) | 能量级 | 配器重点 |
| :--- | :--- | :--- | :--- |
| **00-16s** | `:intro` | 10% | 氛围 Pad，风声，远处的铃声 |
| **16-48s** | `:verse` | 40% | **戏腔进场**，筝流动织体，无打击乐 |
| **48-80s** | `:climax` | 90% | 低音脉冲，重型弦乐，Epic Drums |
| **80-90s** | `:outro` | 5% | 突然抽空，只剩人声尾音与反向混响 |

### 15.2.2 指挥官循环 (The Conductor Loop)
这是一个不发声的 `live_loop`，它的唯一任务是“数拍子”并切换状态。

> **Rule of Thumb (工程法则)**：
> 所有的乐器 Loop 都应该“监听” `:stage` 变量。如果当前状态不需要它，它应该执行 `stop` 或者 `sleep 1`（低功耗待机），而不是被注释掉。

---

## 15.3 氛围层：构建“云端”空间 (Pad & Atmosphere)

古风的“仙气”和 Hans Zimmer 的“史诗感”都依赖于**中低频的铺底（Pad）**。

### 15.3.1 和声选择
我们使用 **D Minor** 调。为了古风感，我们尽量避免三全音（Tritone）。
推荐的和声进行（以 8 小节为一个循环）：
`Dm -> Bb -> F -> C/E`
(i -> VI -> III -> VII)
这是一个非常经典的“英雄史诗”走向，既有东方悲怆（Dm, Bb），又有西方的辉煌（F, C）。

### 15.3.2 音色设计：会呼吸的弦乐
不要直接用 `:prophet` 或 `:saw` 发一个长音就完了。我们需要它“动”起来。

*   **技术手段**：叠加两层。
    1.  **底层 (Sub/Body)**：`:dsaw` (Detuned Saw)，低通滤波器 `cutoff: 70`，提供厚度。
    2.  **上层 (Air/Texture)**：`:hollow` 或 `:blade`，高通滤波器 `hpf: 80`，加上极慢的 `amp` LFO（低频振荡器）。

> **关键参数自动化**：
> 在 `:intro` 阶段，cutoff 设为 60（暗）；
> 随着进入 `:climax`，利用 `control` 将 cutoff 在 16 拍内平滑升至 100（亮）。这模拟了云层散开的光照感。

---

## 15.4 旋律层 A：古筝/竖琴的“流动织体”

在 `:verse` 阶段，我们需要一种乐器来填补人声的空隙。久石让常用钢琴分解，古风常用古筝。

### 15.4.1 五声随机漫步 (Pentatonic Random Walk)
古风不使用半音。我们将音符限制在 `scale(:d4, :minor_pentatonic)` 中（D, F, G, A, C）。

*   **算法思路**：
    不要完全随机（`choose`），那样会很乱。
    要使用**受限随机**：下一个音只能是当前音的“邻居”（+1 或 -1 索引）。
    这样写出来的旋律像水流一样连贯，没有突兀的大跳。

### 15.4.2 模拟拨弦质感
*   **音色**：`:pluck` 或 `:kalimba`。
*   **Humanize（人性化）**：
    1.  **时值微差**：不要写死 `sleep 0.25`，而是 `sleep [0.24, 0.26].choose`。
    2.  **力度微差**：`amp: rrand(0.6, 0.9)`。
    3.  **回声设计**：加一个 `echo` 效果器，`phase: 0.75`（附点八分音符），这会制造出一种“大珠小珠落玉盘”的错落感。

---

## 15.5 旋律层 B：戏腔人声的处理 (The Opera Vocal)

假设你有一个名为 `opera_dry` 的干声采样。我们需要把它处理得既有传统戏曲的“穿透力”，又有电子音乐的“融合感”。

### 15.5.1 模拟“大青衣”的频响 (Formant Sculpting)
戏曲唱腔的特点是**中高频共振峰**（2kHz - 4kHz）非常突出，而低频（200Hz以下）较少。

*   **EQ 策略**：
    使用 `hpf: 200` 切掉胸腔共鸣的低频，让声音变“薄”且“高”。
    使用 `:bpf` (带通滤波器) 在 3000Hz 附近做一点微弱的提升，增加“金属光泽”。

### 15.5.2 空间的虚实结合
*   **干声位置**：`pan: 0`（正中），`amp: 1.2`（突出）。
*   **湿声位置**：发送到 AUX 轨道（在 Sonic Pi 中通过嵌套 `with_fx` 实现）。
    *   **FX 1**: `reverb`, `room: 100`, `mix: 0.6`（巨大的远景）。
    *   **FX 2**: `ping_pong`, `phase: 0.5`（左右摆动的回声）。

> **Gotcha**：
> 这一层必须在 `climax` 阶段使用**侧链压缩（Sidechain）**。当底鼓（Kick）响的时候，混响层要稍微压低一点，否则整个声场会变成一锅粥。

---

## 15.6 节奏层：Hans Zimmer 式的“隐形脉冲”

Zimmer 风格的精髓不在于“吵”，而在于**持续不断的 16 分音符驱力（Ostinato）**。

### 15.6.1 这里的“鼓”不是鼓
我们不使用 `:bd_haus` 这种动感的舞曲底鼓，而是使用 **Taiko（太鼓）** 或 **Orchestral Percussion**。
如果 Sonic Pi 自带采样不够，可以用 `:tom_lo_hard` 并调低 `rate`。

### 15.6.2 合成器脉冲 (Synth Pulse)
这是核心技巧。
*   **音色**：`:tb303` 或 `:tech_saws`。
*   **写法**：连续演奏 16 分音符的根音（D2）。
*   **关键处理**：
    `cutoff` 必须极低（例如 50），只有一点点“噗噗”声。
    利用 `control` 随着小节推进，慢慢打开 `cutoff` 和 `resonance`（共振）。
    这种**“慢慢逼近”**的感觉是营造紧张感的关键。

---

## 15.7 自动化与转场 (Automation & Transition)

让音乐“活”起来的不是音符，是参数的变化。

### 15.7.1 Riser（过门提升音效）
在 `:verse` 转入 `:climax` 的最后 4 小节（即 44s-48s）：
*   **白噪声扫频**：使用 `:noise` 音色，`attack: 4`, `release: 0`。
*   **音高提升**：如果使用有音高的合成器，使用 `slide` 参数让音高上滑 1-2 个八度。

### 15.7.2 结尾的“抽离” (The Drop-out)
在 89 秒处：
1.  瞬间 `stop` 所有的鼓和低音 Loop。
2.  只留下一个长长的 Reverb 尾音。
3.  或者，播放一个 **Reverse Cymbal (反向镲片)**，在最高点戛然而止。

---

## 15.8 实战代码逻辑演示 (Pseudocode Snippets)

为了让你理解架构，这里提供核心逻辑的代码骨架（不可直接运行，需填充具体音符）：

```ruby
# 1. 全局配置
use_bpm 84
set :stage, :intro  # 初始状态

# 2. 指挥官 Loop (负责结构)
live_loop :conductor do
  # 0-16s Intro
  set :stage, :intro
  sleep 16 * 4
  
  # 16-48s Verse
  set :stage, :verse
  sleep 32 * 4
  
  # 48-80s Climax
  set :stage, :climax
  sleep 32 * 4
  
  # 80-90s Outro
  set :stage, :outro
  sleep 10 * 4
  
  stop # 结束全曲
end

# 3. 乐器层：Pad (监听状态)
live_loop :strings_pad do
  current_stage = get[:stage]
  
  case current_stage
  when :intro
    # 薄薄的铺底
    use_synth :hollow
    play_chord [:d3, :f3, :a3], release: 8, amp: 0.5
    sleep 8
  when :verse, :climax
    # 厚重的铺底
    use_synth :dsaw
    # Climax 时更亮，Verse 时更暗
    brightness = (current_stage == :climax) ? 90 : 60
    play_chord [:d2, :d3, :f3, :a3, :c4], release: 4, cutoff: brightness, amp: 0.8
    sleep 4
  when :outro
    stop
  end
end

# 4. 乐器层：脉冲 (只在 Climax 出现)
live_loop :zimmer_pulse do
  sync :conductor # 与指挥官对齐
  if get[:stage] == :climax
    use_synth :tb303
    # 16分音符推进
    4.times do
      play :d1, release: 0.2, cutoff: rrand(60, 80), amp: 0.7
      sleep 0.25
    end
  else
    sleep 1
  end
end
```

---

## 本章小结

1.  **架构先行**：不要在没有地图的情况下进入丛林。`:stage` 状态机是管理长篇结构的最佳工具。
2.  **减法美学**：古风需要留白。当戏腔出现时，复杂的琶音和 Lead 旋律必须让位静音或降低音量）。
3.  **情绪曲线**：Hans Zimmer 风格的精髓是 Dynamic Range（动态范围）。从 Intro 的极弱到 Climax 的极强，cutoff 和 amp 的自动化控制是关键。
4.  **音色融合**：通过 LPF（低通）让电子音色变暗以融入弦乐，通过 HPF（高通）让戏腔变薄以突显穿透力。

---

## 练习题

### 基础题
1.  **状态机练习**：编写一个简单的脚本，包含 `:intro` (只有鼓), `:main` (鼓+贝斯), `:break` (静音) 三个状态，每 4 小节自动切换一次。
    <details>
    <summary>提示 (Hint)</summary>
    使用 `case` 或 `if` 语句检查 `get[:stage]`。
    </details>

2.  **古筝琶音器**：使用 `:pluck` 音色，编写一个函数，输入一个根音，就能自动演奏出该根音对应的五声音阶上行琶音（如 D, F, G, A）。
    <details>
    <summary>提示 (Hint)</summary>
    使用 `scale(note, :minor_pentatonic)` 获取音符列表，然后用 `.tick` 遍历它。
    </details>

### 挑战题 (Open-ended)
3.  **90秒限制挑战**：
    限定只使用 Sonic Pi 自带的 `:ambi_choir`（模拟人声）和 `:piano` 两种音色。制作一段 90 秒的配乐，要求表现出“从孤独到宏大”的过程。
    *   **思考**：如何只用钢琴表现宏大？（提示：极低音区的八度轰鸣 + 极高音区的快速琶音）。
    *   **思考**：如何让 Choir 不像廉价合成器？（提示：极慢的 Attack 和大量的 Reverb）。

4.  **制作“空气感”底噪**：
    不使用外部采样，仅使用 `:noise` 或 `:pink_noise` 合成器，配合滤波器和声像移动（Pan），模拟出“山谷中的风声”。
    <details>
    <summary>提示 (Hint)</summary>
    风声不是静止的。使用 `bpf` (带通滤波器) 并用 `control` 随机移动它的中心频率 `centre`，模拟风速的变化。
    </details>

---

## 常见陷阱与错误 (Gotchas)

### 1. 爆音与 CPU 过载 (Crackling & Overload)
*   **现象**：在 `:climax` 阶段，声音出现噼里啪啦的杂音。
*   **原因**：太多的 `live_loop` 同时运行，或者 Reverb 效果器嵌套层数太多。
*   **解决**：
    *   **减少发声数**：检查是否有 Release 时间过长的音符堆叠在一起。
    *   **全局效果器**：不要在每个 loop 里都写 `with_fx :reverb`。在最外层包裹一个主 Reverb。
    *   **增加延迟**：在 Audio 设置中增加缓冲大小（Latency）。

### 2. 节奏不同步 (Timing Drift)
*   **现象**：鼓和琶音跑着跑着就对不齐了。
*   **原因**：使用了不规范的 `sleep` 时间（如 `sleep 0.33` 而不是 `sleep 1.0/3`），或者没有使用 `sync`。
*   **解决**：所有的 Loop 最好都显式地 `sync :conductor`（或者是鼓组 Loop）。确保所有人都在同一个小节的第一拍起跑。

### 3. "塑料"古风感 (Cheesy Sound)
*   **现象**：听起来像 90 年代的低质 MIDI 铃声。
*   **原因**：完全没有动态变化（Velocity/Amp 都是 1），音符完全化（死板的节奏）。
*   **解决**：**Humanize** 是核心。微调 `amp`，微调 `cutoff`，甚至微调 `sleep` 的时间（Swing 感）。古风的韵味在于“不精确”。
