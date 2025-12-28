# 第 17 章：代码片段速查——可直接复制的“写作积木” (Cheat Sheet)

## 17.1 开篇：你的 Sonic Pi 军火库

本章不讲原理，只讲**效率**。

当你灵感迸发，或者需要在短时间内搭建出一段像样的 Demo 时，不要从零开始敲每一个 `sleep` 和 `play`。本章提供了经过参数优化的代码模板，涵盖了**华语古风的留白**、**久石让式的透明感**以及 **Hans Zimmer 式的声压级**。

**使用本章的“复制-粘贴-修改”三部曲：**
1.  **Copy (复制框架)**：选中你需要的 `live_loop` 或 `define` 函数。
2.  **Tweak (微调参数)**：修改代码中标记为 `# <--- 调节这里` 的关键参数（如 cutoff, amp, sleep）。
3.  **Layer (分层堆叠)**：不要在一个循环里写完所有东西。将低频（Bass）、律动（Rhythm）、和声（Pad）和旋律（Lead）分拆到不同的 `live_loop` 中。

---

## 17.2 节奏与律动 (Rhythm & Groove)

### 17.2.1 [Zimmer] 电影感史诗脉冲 (The Action Pulse)
这是好莱坞动作片标准的“心跳”低音。核心在于 16 分音符的驱动力与滤波器的动态变化。

```ruby
# 适合场景：紧张的追逐戏、战斗前夕、史诗铺垫
live_loop :zimmer_pulse do
  use_bpm 110
  
  # --- 关键参数 ---
  # 强弱逻辑：模拟大提琴/低音合成器的运弓力度 (3+3+2 节奏型)
  accents = (ring 1.3, 0.6, 0.6,  1.3, 0.6, 0.6,  1.5, 0.6)
  
  # 滤波器自动化：让声音像呼吸一样张开又闭合
  # range(60, 110) 决定了声音是从“闷”到“亮”的变化范围
  co_env = (line 60, 110, steps: 64).mirror
  
  use_synth :saw
  use_synth_defaults attack: 0.05, release: 0.25, env_curve: 3
  
  tick
  # 稍微错开一点 pan (相)，制造宽广感
  with_fx :bitcrusher, bits: 12, mix: 0.1 do # 加一点点失真颗粒
    play :c2, cutoff: co_env.look, amp: accents.look * 0.8, pan: rrand(-0.1, 0.1)
    play :c3, cutoff: co_env.look, amp: accents.look * 0.4, pan: rrand(-0.2, 0.2) # 叠高八度增加咬合感
  end
  
  sleep 0.25
end
```

### 17.2.2 [古风] 板鼓与留白 (Chinese Percussion)
戏曲和古风不强调“满”，而强调“点”。利用 `sleep` 的长短变化制造呼吸感。

```ruby
# 适合场景：戏腔独唱背景、古风叙事段落
live_loop :chinese_beat do
  use_bpm 70
  
  # --- 节奏网格 ---
  # 1 = 板 (强拍), 2 = 眼 (弱拍), 0 = 休止
  # 这种不规则的稀疏感是古风的关键
  grid = (ring 1, 0, 0, 2,  0, 0, 2, 0,  1, 0, 2, 0,  0, 0, 0, 0)
  
  t = grid.tick
  
  case t
  when 1 # "板" - 响亮清脆的木质声
    sample :elec_wood, amp: 1.5, rate: 1.0, pan: 0.2
    sample :drum_snare_soft, amp: 0.8, rate: 2.0, finish: 0.1 # 叠层增加质感
  when 2 # "眼" - 轻微的打点
    sample :elec_wood, amp: 0.6, rate: 1.5, pan: -0.2
  end
  
  sleep 0.25
end
```

### 17.2.3 [久石让] 轻盈华尔兹分解 (Ghibli Arp)
吉卜力风格常见的钢琴左手织体，不是“咚次次”，而是流动的分解和弦。

```ruby
# 适合场景：温馨回忆、飞行、田园
live_loop :ghibli_waltz do
  use_bpm 140 # 3/4 拍通常速度较快
  
  # 和弦走向：IV - V - iii - vi (典型的日系王道进行)
  keys = [chord(:f3, :major7), chord(:g3, :dom7), chord(:e3, :minor7), chord(:a3, :minor7)].ring
  current_chord = keys.tick
  
  use_synth :piano
  
  # 节奏型：根音(强) - 5音 - 3音 - 7音 - 3音 - 5音 (6/8 感觉)
  # 这种上下起伏的分解比柱式和弦更灵动
  pattern = [0, 2, 1, 3, 1, 2] 
  
  pattern.each do |idx|
    # velocity 模拟真人触键力度变化：第一下重，后面轻
    vel = idx == 0 ? 0.7 : 0.4 
    play current_chord[idx], amp: vel, release: 2, pan: rrand(-0.1, 0.1)
    sleep 1
  end
end
```

---

## 17.3 和声与氛 (Harmony & Atmosphere)

### 17.3.1 [古风] 瑶琴/古筝轮指 (Tremolo Pluck)
模拟弹拨乐器的“摇指”或“轮指”技法，用于制造持续的高频背景。

```ruby
# 适合场景：仙侠背景、情感高潮的背景层
live_loop :guzheng_tremolo do
  use_synth :pluck
  # coef 控制弦的衰减，越大声音越短促，越小越像古筝
  use_synth_defaults coef: 0.1, amp: 0.5, release: 0.3
  
  # 选用五声音阶
  notes = scale(:d4, :major_pentatonic, num_octaves: 2)
  
  # 快速重复同一个音，模拟轮指
  target_note = notes.choose
  
  with_fx :reverb, room: 0.8 do
    8.times do
      # 细微的音高偏移(detune)和时间偏移，模拟真实演奏的不完美
      play target_note + rrand(-0.1, 0.1), pan: rrand(-0.3, 0.3)
      sleep 0.125 + rrand(-0.01, 0.01) 
    end
  end
  
  # 偶尔换个音
  sleep [0, 0.5, 1].choose
end
```

### 17.3.2 [Zimmer] 铜管号角 "BRAAAM" (The Horn Blast)
那个著名的《盗梦空间》式的低频轰鸣。

```ruby
# 适合场景：反派登场、巨大物体移动、危机
define :braaam do |note|
  use_synth :fm # FM 合成最适合做这种金属质感
  
  # divisor: 0.5 制造低八度的厚度，depth 控制金属感强弱
  with_fx :distortion, distort: 0.4 do
    with_fx :reverb, room: 0.9, mix: 0.6 do
      play note, divisor: 0.501, depth: 1000, 
           attack: 0.5, sustain: 2, release: 3, 
           amp: 1.5, cutoff: 80
           
      # 叠加一层锯齿波增加高频嘶嘶声
      use_synth :saw
      play note, attack: 0.4, release: 3, cutoff: 60, amp: 0.5
    end
  end
end

# 调用示例
# braaam :c1
```

### 17.3.3 [通用] 冰冷无人机长音 (Drone Pad)
无论古风还是电影配乐都通用的铺底。

```ruby
# 适合场景：开头引入、空气感
live_loop :drone_pad do
  use_bpm 60
  use_synth :hollow # 或者是 :dark_ambience
  
  # 极慢的 Attack 和 Release
  with_fx :lpf, cutoff: 70 do # 砍掉高频，只留氛围
    play :c2, attack: 4, sustain: 4, release: 4, amp: 0.8
    play :g2, attack: 5, sustain: 4, release: 5, amp: 0.5
  end
  
  sleep 8
end
```

---

## 17.4 音色设计与人声辅助 (Sound Design & Vocals)

### 17.4.1 模拟竹笛 (Synth Flute)
如果你没有人录真笛子，用这个凑合，或者作为 Demo 占位。

```ruby
define :play_bamboo_flute do |n, d|
  # n = 音符, d = 时值
  
  # 层1：基音（正弦波）
  use_synth :sine
  # 使用 vowel (元音) 滤波器模拟管乐的共振峰
  with_fx :vowel, vowel_sound: 3, voice: 4 do
    play n, attack: 0.2, sustain: d*0.9, release: d*0.2, amp: 0.6
  end
  
  # 层2：气声（白噪）- 笛子的灵魂在于漏气的声音
  use_synth :bnoise
  with_fx :hpf, cutoff: 100 do # 高通滤波，只留高频嘶嘶声
    play n, attack: 0.1, sustain: d*0.8, release: d*0.1, 
         amp: 0.15, pan: rrand(-0.1, 0.1)
  end
  
  sleep d
end
```

### 17.4.2 人声合唱垫 (Choir Pad Synthesis)
用来在副歌部分支撑你录制的真实戏腔/人声。

```ruby
live_loop :synth_choir do
  use_synth :dark_ambience
  
  # 和弦：C Major -> A Minor
  chords = [chord(:c4, :major), chord(:a3, :minor)].ring
  c = chords.tick
  
  # 模拟合唱团：稍微解谐 (Detune) 
  with_fx :chorus, mix: 0.5, depth: 3 do
    with_fx :reverb, room: 0.9 do
      # 把和弦里的每个音都弹出来
      c.each do |n|
        play n, attack: 1, sustain: 2, release: 2, amp: 0.5
      end
    end
  end
  sleep 4
end
```

---

## 17.5 效果器链 (FX Chains)

### 17.5.1 侧链压缩效果 (Fake Sidechain)
让配乐像现代 Pop/EDM 一样有“抽吸感”，给人声或底鼓让位。

```ruby
# 把这个放在 live_loop 内部包裹 play 语句
with_fx :lpf, cutoff: 130 do |l|
  # 控制 cutoff 快速升降，模拟音量被压低
  # 也可以控制 :level
  control l, cutoff: 30 
  control l, cutoff: 130, cutoff_slide: 0.4
  
  # 你的 Synth 代码写在这里...
  play :c4, sustain: 1
end
```

### 17.5.2 磁带老化效果 (Lo-Fi/Vintage)
适合做“回忆杀”或古风 Intro。

```ruby
with_fx :wobble, phase: 10, wave: 1 do # 缓慢的音高抖动
  with_fx :bitcrusher, sample_rate: 10000, bits: 12 do # 降低采样率
    with_fx :hpf, cutoff: 40 do # 切掉超低频
       # 你的旋律...
    end
  end
end
```

---

## 17.6 实用工程模板 (Structure Templates)

### 17.6.1 完整工程起手式
这是第 14 章提到的分轨思路的代码化。

```ruby
# ==========================================
# Sonic Pi 电影/古风配乐工程模板
# ==========================================

use_bpm 90
set_volume! 1.0

# 全局变量：用于控制段落 (Intro, Verse, Chorus)
set :section, :intro 

# 1. 节奏组 (Drums & Percussion)
live_loop :rhythm do
  s = get[:section]
  if s == :intro
    # 静音或只敲边
    sleep 4
  elsif s == :verse
    # 轻打击乐
    sample :elec_wood, rate: 0.8, amp: 0.5
    sleep 1
  elsif s == :chorus
    # 史诗脉冲
    sample :bd_haus, amp: 1.5
    sleep 1
  end
end

# 2. 低音组 (Bass)
live_loop :bass do
  sync :rhythm
  # Bass 代码...
end

# 3. 和声组 (Chords/Pad)
live_loop :harmony do
  sync :rhythm
  # Pad 代码...
end

# 4. 旋律/人声提示 (Lead)
live_loop :melody do
  sync :rhythm
  # 旋律代码...
end
```

---

## 17.7 本章小结

*   **积木思维**：不要试图写出“完美的一行代码”。音乐是**层 (Layer)** 的艺术。古风是（人声 + 笛子 + 稀疏打击）；Zimmer 是（低音脉冲 + 弦乐断奏 + 铜管长音）。
*   **随机性 (Randomization)**：电脑太精准了，所以很假。善用 `rrand(min, max)` 来微调 `pan`（声相）、`amp`（音量）和 `sleep`（时值），这是让 Sonic Pi 听起来像真人的秘诀。
*   **频谱位置**：如果你用了 **17.2.1 的史诗脉冲** (占满低频)，就不要再用 **17.3.3 的 Drone** 的低音区，否则声音会糊成一团。

---

## 17.8 练习题

### 练习 1：风格大挪移
**任务**：复制 **17.2.3 [久石让] 华尔兹分解** 的代码。
1.  将 `use_synth :piano` 改为 `use_synth :pluck`。
2.  将和弦改为 `[:c3, :d3, :e3, :g3, :a3]` (五声音阶簇)。
3.  **结果**：你是否将其从“日系钢琴”变成了一种类似“古筝/竖琴”的古风织体？

<details>
<summary>点击查看提示</summary>
织体（Pattern）和音色（Patch）是独立的。分解和弦的写法（织体）决定了音乐的流动感，而乐器选择（音色）决定了风格文化。两者混搭常有奇效。
</details>

### 练习 2：Zimmer 的压迫感
**任务**：使用 **17.2.1 史诗脉冲**。
1.  当前 `accents` 数组是 3+3+2 节奏。
2.  试着将其改为 4/4 拍的直线加速感：`[0.5, 0.8, 1.0, 1.2]`，并配合 `rate` 的变化。
3.  尝试叠加 **17.3.2 BRAAAM** 在每 16 拍的开头。

<details>
<summary>点击查看答案代码</summary>

```ruby
live_loop :tension_builder do
  # 逐渐增强的直线节奏
  4.times do |i|
    play :c2, release: 0.1, amp: 0.5 + (i * 0.2), cutoff: 70 + (i * 10)
    sleep 0.25
  end
end

live_loop :blast do
  sync :tension_builder
  braaam(:c1) # 调用定义好的函数
  sleep 16
end
```
</details>

### 练习 3：给戏腔留个位
**任务**：结合 **17.3.1 古筝轮指** 和 **17.5.1 侧链/陷波**。
假设你的戏腔人声主要集中在 1000Hz - 3000Hz（共振峰）。请修改古筝的代码，使其在这个频段衰减，不抢人声。

<details>
<summary>点击查看答案思路</summary>
使用 `with_fx :eq, mid: -0.5, mid_shelf: 2000` 包裹古筝代码。或者使用简单的 `lpf` (低通) 切掉高频，把高频完全让人声。
</details>

---

## 17.9 常见陷阱 (Gotchas)

| 现象 | 可能原因 | 解决方案 |
| :--- | :--- | :--- |
| **声音逐渐变卡/爆音** | 效果器嵌套过深 | 检查 `with_fx` 是否写在了 `live_loop` 内部且没有结尾。尽量把 reverb 放在 loop 外面。 |
| **节奏听起来像“瘸了”** | 浮点数误差 | 避免写 `sleep 1.0/3.0` 这种除不尽的数太久。尽量用 `use_bpm` 配合整数倍分数。或者定期用 `sync` 对齐。 |
| **低音糊成一团** | 频率冲突 | Zimmer 脉冲占用了 40-100Hz。如果你又加了一个 heavy kick，必须用 EQ 切掉其中一个的超低频，或者把 Kick 的音高定准。 |
| **古风听起来像儿歌** | 过于量化 | 古风需要“散板”。不要让所有音符都在网格上。在 `sleep` 后加 `+ rrand(0, 0.05)` 制造微小延迟。 |
| **代码太长找不到** | 缺乏模块化 | 不要把所有代码堆在一个文件。善用 `run_file "path/to/drums.rb"` (如果是外部分离) 或使用 `define` 封装常用乐句。 |
