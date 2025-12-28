# 第 4 章：和弦进行到可听配器——“会和弦”到“像作品”

## 4.1 开篇：从“报菜名”到“建筑学”

你可能已经掌握了基础的五线谱知识，知道 C 大调的顺阶和弦是 C, Dm, Em, F, G, Am, Bdim，也能在 Sonic Pi 里写出这样的代码：

```ruby
# 典型的“程序员式写歌”
# 听起来僵硬、机械、像是在测试声卡
live_loop :chords do
  use_synth :saw
  play chord(:c4, :major)
  sleep 1
  play chord(:g4, :major)
  sleep 1
  play chord(:a4, :minor)
  sleep 1
  play chord(:f4, :major)
  sleep 1
end
```

这段代码的问题在于：**它只是在“报出”和弦的名字（报菜名），而不是在“构建”音乐（建筑学）。**

在真实的声学乐（如钢琴或管弦乐团）中，演奏者绝不会如此生硬地搬运音块。本章的目标是教你如何将抽象的“和弦符号（I-V-vi-IV）”转化为具有风格特征的**织体（Texture）**。

我们将从**声部连接（Voice Leading）**入手解决生硬感，利用**扩展音（Extensions）**建立色彩，最后通过**算法化的演奏模式**来实现古风的留白、久石让的流利与 Hans Zimmer 的压迫感。

---

## 4.2 常见进行的“情绪词典”与频谱色彩

在开始写代码之前，我们需要建立一个“情绪映射”。不同的和弦进行不仅是乐理上的排列，更是特定文化和频率能量的某种暗示。

### 1. 史诗/流行万能公式 (I - V - vi - IV)
*   **代码表示**：`[:c, :g, :a, :f]` (配合大/小调性质)
*   **能量特征**：
    *   **I (稳定)** -> **V (最大张力/导向)** -> **vi (意外的阴郁/转折)** -> **IV (在大调中带有抬升感的半终止)**。
    *   这是 Hans Zimmer 在构建宏叙事时常用的骨架（例如《狮子王》或英雄主题的变体）。
*   **Rule of Thumb**：如果你不知道写什么，先写这个，把重点放在音色设计上。

### 2. 久石让/J-Pop 的“王道进行” (IV - V - iii - vi)
*   **代码表示**：`[:f, :g, :e, :a]`
*   **听感分析**：
    *   这种进行**不从主音（I）开始**，一上来就是不稳定的 IV 级，给人一种“故事已经开始”或“半途切入”的流动感。
    *   **关键点**：**V -> iii** (G -> Em) 的连接非常细腻。如果把 iii (Em) 换成 **III7 (E7, `chord(:e, :7)`)**，引入了 `#G` 音，会强烈导向 `Am`，瞬间增加戏剧性和宿命感（离调）。

### 3. 华语古风/悲情 (vi - IV - I - V) 或 (i - VII - VI - V)
*   **代码表示**：
    *   现代古风（如《凉凉》类）：`[:a, :f, :c, :g]`。
    *   传统/异域小调下行：`[:a, :g, :f, :e]`。
*   **风格密码**：
    *   古风的和声往往不强调“功能性解决”（Dominant -> Tonic），而是强调“色彩的并置”。
    *   **小调下行** (Am - G - F - E) 配合 Phrygian 调式（弗里几亚），是表现苍凉感、武侠危机感的核心。

---

## 4.3 声部连接 (Voice Leading)：消除“搬家感”

这是让你的代码听起来从“业余”变“专业”的性价比最高的手段。

**声学问题**：
当你从 C4 (C-E-G) 换到 F4 (F-A-C) 时，所有音符都向上大跳。在物理声学中，人耳对于大幅度的频率跳变非常敏感，会将其感知为“断裂”。在钢琴演奏中，这也意味着手需要整个抬起来移动（搬家）。

**解决原则**：**懒惰原则（Law of Laziness）**。
1.  **共同音保持**：如果两个和弦有相同的音，那个音不要动。
2.  **就近解决**：其他音走最短的半音/全音路径。

C 大三和弦 (C-E-G) 到 F 大三和弦，最近的走法是 (C-F-A)，这是 F 和弦的第二转位。

```text
ASCII 示意图：声部连接 (Voice Leading)

[生硬的写法 / Root position jump]
频域高区:             C5 (跳跃 5度)
                     ^
G4 ------------------|
E4 ------------------+-> A4 (跳跃 4度)
C4 ------------------+-> F4 (跳跃 4度)
(C Major)           (F Major 原位)
* 听感：块状、分离、笨重

[平滑的写法 / Smooth Leading]
A4 (仅上行 2度) <----+
F4 (仅上行 1度) <----+-- G4
C4 (保持不动!)  <======= C4
                    E4
(F inv 2)           (C Major)
* 听感：像胶水一样粘合，内部色彩在流动
```

**Sonic Pi 实现技巧**：
Sonic Pi 的 `chord_invert` 虽好用，但为了更精细的控制，建议直接控制八度。

```ruby
# 自动寻找最近转位的逻辑很难写，但我们可以手动“微调八度”
live_loop :smooth_voice do
  use_synth :tri
  
  # C Major: C4, E4, G4
  play chord(:c4, :major), release: 2
  sleep 2
  
  # F Major (转位): C4, F4, A4
  # 我们不写 chord(:f4, :major)，因为那会是 F4, A4, C5
  # 我们写 chord(:f3, :major, invert: 2) -> 得到 C4, F4, A4
  play chord(:f3, :major, invert: 2), release: 2
  sleep 2
end
```

> **Rule of Thumb**：当你写和弦进行时，试着让所有音符都保持在 `:c4` 到 `:c5` 这个狭窄的“黄金中频区”内，不要让它们跑出去。

---

## 4.4 延伸音与色彩：风格的调味料

三和弦是骨架，延伸音（Extensions）是皮肉。对于你想要的三种风格，有着截然不同的“香料”配方。

### 1. 华语古风的“五声性中立” (sus2, add9, 6)
古风音乐（基于五声调式）非常忌讳强烈的三全音冲突（如 B 到 F）。为了让和弦听起来有“中国味”，我们必须**模糊**大三/小三和弦的界限。

*   **秘密武器**：**sus2 (挂二和弦)**。
*   **原理**：`Csus2` 的组成音是 C-D-G。它没有 E（决定大调性质的音）或 Eb（决定小调性质的音）。这三个音 (1, 2, 5) 都在中国传统的五声音阶内。
*   **听感**：空灵、像水墨画留白、没有攻击性。
*   **代码**：`chord(:c4, :sus2)` 或 `chord(:c4, :add9)`（增加高八度的2音，更明亮）。

### 2. 久石让的“怀旧滤镜” (Major7, add9)
久石让的钢琴曲听起来“丰富、温暖且略带伤感”，秘诀在于他不吝啬使用 **Major7** 和 **9音**。

*   **秘密武器**：**Major7 (大七和弦)**。
*   **原理**：在 F 和弦中加入 E 音 (F-A-C-E)。E 音和 F 音是大七度，和 C 音是大三度，产生极其丰富的泛音列摩擦。
*   **听感**：都市感、回忆、纯真。
*   **代码**：`chord(:f4, :major7)`。

### 3. Hans Zimmer 的“悬停张力” (sus4, minor7)
Zimmer 喜欢用 **sus4** 制造一种“未解决”的紧张感，或者通过极简的小三和弦堆叠。

*   **秘密武器**：**sus4 解决到 major**。
*   **用法**：先播放 `chord(:c4, :sus4)` (C-F-G)，听众会期待 F 解决到 E，但你故意拖延，制造悬疑感。

---

## 4.5 织体设计：代码即乐手

这是本章的核心。有了和弦后，怎么“弹”决定了风格。我们将直接给出种风格的算法模板。

### A. 风格：华语古风/久石让
**织体关键词**：**分解和弦 (Arpeggios) + 流动**
不要同时按下所有键，而是像流水一样拆开。久石让常用“左手跨八度”的分解。

```ruby
# --- 模板 A：久石让/古风 钢琴分解 ---
live_loop :piano_arpeggio do
  use_synth :piano
  # 模拟钢琴的稍微不完美的调音和氛围
  use_synth_defaults hard: 0.4, vel: 0.6, release: 2, stereo_width: 0.8
  
  # 定义和弦进行：IV - V - iii - vi (王道进行)
  chords = [
    chord(:f3, :major7),
    chord(:g3, :dom7),
    chord(:e3, :m7),
    chord(:a3, :m7)
  ].ring
  
  current_chord = chords.tick
  
  # 算法化演奏：根-5-9-10 (跨越十度的宽广织体)
  # 注意：这里我们手动挑选和弦内的音符索引
  # 0=根音, 1=3音, 2=5音, 3=7音
  
  # 模仿左手：根音 -> 5音 -> 7音 -> 3音(高八度)
  # 这种排列避开了低频的浑浊 3 音
  pattern = [current_chord[0], current_chord[2], current_chord[3], current_chord[1] + 12]
  
  pattern.each do |n|
    play n, amp: rrand(0.5, 0.7) # 力度人性化
    sleep 0.25
  end
end
```

### B. 风格：Hans Zimmer / 史诗配乐
**织体关键词**：**Ostinato (固定音型) + 脉冲 + 滤波器动态**
Zimmer 的风格不在于复杂的旋律，而在于**动态（Dynamics）**和**音色墙**。核心是 16 分音符的短促重复，配合 Low Pass Filter (LPF) 的缓缓打开。

```ruby
# --- 模板 B：Zimmer 式脉冲引擎 ---
live_loop :zimmer_pulse do
  # 使用 FM 合成器制造冷峻、金属感的音色，或者 :saw 制造弦乐感
  use_synth :fm
  use_synth_defaults depth: 1, divisor: 2, release: 0.15, amp: 0.6
  
  # 和弦根音：d - a - b - g
  roots = [:d2, :a1, :b1, :g1].ring
  current_root = roots.tick(:chord)
  
  # 滤波器自动化：模拟 Zimmer 音乐中那种“慢慢逼近”的压迫感
  # range(60, 110, 0.5) 意味着 cutoff 从 60 慢慢升到 110
  cutoff_val = range(60, 110, 0.5).look(:chord)
  
  # 节奏型：16分音符推
  # 4 * 4 = 16 个音符 (一小节)
  16.times do
    # 强弱重音设计：每4个音一个重音 (Ta-ka-ta-ka)
    accent = (look % 4 == 0) ? 0.8 : 0.4
    
    play current_root, cutoff: cutoff_val, amp: accent
    
    # 偶尔叠加一个五度音，增加厚度
    play current_root + 7, cutoff: cutoff_val, amp: accent * 0.5 if one_in(4)
    
    sleep 0.25
    tick
  end
end
```

### C. 风格：戏腔伴奏
**织体关键词**：**留白 (Sparse) + 重点装饰**
如果上方有戏腔人声，配器必须极度克制。不要写密集的音符，只在“句读”处点缀。

*   **策略**：
    1.  **Pad (铺底)**：极低音量的长音，维持调性色彩。
    2.  **Pluck (拨弦)**：只在强拍或人声换气时出现。

---

## 4.6 低音线 (Bassline) 的决定性作用

低音决定了和弦的功能。在配乐中，控制低音比控制和弦更重要。

1.  **Slash Chords (转位低音)**：
    *   如果你上面弹 C 大调和弦，下面弹 E 音（C/E），听起来会比原位 C 更流动。
    *   Sonic Pi 写法：`play :e2` (Bass) + `play chord(:c4, :major)` (High)。

2.  **Pedal Point (持续音/踏板音)**：
    *   **这是 Zimmer 的杀手锏**。
    *   *做法*：上方和弦一直在变（C -> F -> G -> C），但低音一直保持 C (`:c2`) 不动。
    *   *声学效果*：F/C 和 G/C 会产生巨大的**张力（Tension）**，因为低音与上方和弦打架。当最后回到 C/C 时，释放感是巨大的。

---

## 4.7 常见陷阱与错误 (Gotchas)

1.  **低频泥浆 (Muddy Mix)**：
    *   *现象*：声音听起来像“闷在被子里”，或者音箱发出“嗡嗡”声。
    *   *原因*：在 `:c2` 或 `:c3` 的低音区演奏了密集的三和弦（如 `chord(:c2, :major)`）。低频区的临界频带（Critical Bandwidth）很宽，相邻的频率（如 C2 和 E2）会互相干扰产生粗糙的拍频。
    *   *Rule of Thumb*：**低音区只允许存在八度或五度**。把三度音（决定大/小的音）放在 `:c3` 甚 `:c4` 以上。

2.  **默认音量过载 (Clipping)**：
    *   *现象*：爆音，红灯亮。
    *   *原因*：`play chord(...)` 会同时触发 3-4 个合成器实例。Sonic Pi 默认 `amp: 1`，叠加后变成 3 或 4。
    *   *修正*：播放和弦时，务必降低 `amp:`，例如 `amp: 0.4`。

3.  **甚至不像人弹的**：
    *   *现象*：机械感。
    *   *修正*：
        *   **Velocity (力度) 随机**：`amp: rrand(0.5, 0.7)`。
        *   **Timing (时值) 微偏**：不要完全对齐 `sleep`，虽然这在 Sonic Pi 里较难控制，但可以通过 `time_warp` 实现，或者简单地在 ADSR 的 `attack` 上加微小随机值 `attack: rrand(0, 0.05)`。

---

## 4.8 本章小结

1.  **选字典**：
    *   史诗 = I-V-vi-IV
    *   久石让 = IV-V-iii-vi (Major7/add9)
    *   古风 = vi-IV-I-V (sus2/无三度)
2.  **做平滑**：使用 `invert` 或手动八度调整，确保和声在中频区像蛇一样游动，而不是像青蛙一样跳跃。
3.  **定织体**：
    *   钢琴分解 = 叙事/温情。
    *   低音脉冲 + 滤波开合 = 紧张/动作/史诗。
    *   留白拨弦 = 戏腔/人声伴奏。
4.  **理频段**：低音只留根音/五音（骨头），和声色彩放在中高频（皮肉）。

---

## 4.9 练习题

### 基础题 (50%)

**练习 1：和弦整形师**
写一个 `live_loop`，循环播放 `C - G - Am - F`。
*   **现状**：直接写 `play chord(:c4, :major)` 等会导致大跳。
*   **任务**：修改 `chord` 的八度或 `invert` 参数，使得所有音符的运动范围都保持在 `:c4` 到 `:c5` 之间（即最高音不超过 C5，最低音不低于 C4）。
*   **提示**：C (C-E-G) 到 G (G-B-D)，最近的连接是保留 G 音。G 和弦哪个转位是以 G 为最高音或中间音的？

<details>
<summary>点击查看参考答案</summary>

```ruby
live_loop :smooth_chords do
  use_synth :tri
  use_synth_defaults release: 1.5, amp: 0.5
  
  # 这里的写法是为了追求声部平稳连接
  play chord(:c4, :major)             # C4, E4, G4
  sleep 1
  
  # 目标：G Major (G, B, D)。
  # 如果用 invert: 1 (B, D, G)，这样 G4 被保留在顶部，非常平滑
  play chord(:g3, :major, invert: 1)  # B3, D4, G4
  sleep 1
  
  # 目标：Am (A, C, E)。
  # 刚才在 B3, D4, G4。A minor 原位是 A3, C4, E4。
  # invert: 1 是 C4, E4, A4。
  play chord(:a3, :minor, invert: 1)  # C4, E4, A4
  sleep 1
  
  # 目标：F Major (F, A, C)。
  # 刚才在 C4, E4, A4。F invert 2 是 C4, F4, A4。
  # 两个音(C, A)都不用动！最完美的连接。
  play chord(:f4, :major, invert: 2)  # C4, F4, A4
  sleep 1
end
```
</details>

**练习 2：一键古风化**
将一个普通的 C 大调和弦进行 `[:c, :a, :f, :g]` (I-vi-IV-V)，通过代码将其改为古风听感。
*   **任务**：不要使用 `:major` 或 `:minor`，全部替换为 `:sus2`。使用 `:pluck` 合成器。
*   **观察**：试着听一下，为什么在大调和小调位置都使用 sus2 却不违和？（答案：因为 sus2 模糊了调性界限）。

<details>
<summary>点击查看参考答案</summary>

```ruby
live_loop :gufeng_vibe do
  use_synth :pluck
  use_synth_defaults amp: 0.8, coef: 0.5 # coef 控制拨弦的闷亮
  
  # 根音环
  roots = [:c3, :a2, :f2, :g2].ring
  
  # 统一使用 sus2，消除三度音的冲突
  # 这种伴奏非常适合叠加五声音阶的旋律
  play chord(roots.tick, :sus2), release: 3
  
  sleep 1
end
```
</details>

---

### 挑战题 (50%)

**练习 3：Hisaishi Arp (久石让式琶音模式)**
编写一个函数 `define :play_arpeggio do |root, type| ... end`。
*   **场景**：模拟《Summer》或《One Summer's Day》那种跳跃的左手。
*   **要求**：不仅是简单的上行。尝试模式：**根音 -> 5音 -> 8音(高八度根音) -> 5音**。
*   **应用**：放入 live_loop 中演奏 IV-V-iii-vi。

<details>
<summary>点击查看参考答案</summary>

```ruby
define :play_arpeggio do |root, type|
  # 获取和弦音：0=根, 1=3音, 2=5音, 3=7音(如果有)
  # 注意：这里们假设传入的是三和弦，手动算八度
  
  # 模式：根 -> 5 -> 8(根+12) -> 5
  # 这种 1-5-8 模式是钢琴伴奏中最稳固的结构
  c = chord(root, type)
  root_note = c[0]
  fifth_note = c[2]
  
  # 手动构建旋律线
  melody = [root_note, fifth_note, root_note + 12, fifth_note]
  
  melody.each do |n|
    play n, release: 0.5, amp: rrand(0.4, 0.6), cutoff: rrand(80, 100)
    sleep 0.25
  end
end

live_loop :hisaishi_style do
  use_synth :piano
  # 典型的久石让走向
  play_arpeggio(:f3, :major7)
  play_arpeggio(:g3, :dom7)
  play_arpeggio(:e3, :m7)
  play_arpeggio(:a3, :m7)
end
```
</details>

**练习 4：Zimmer 的时间膨胀 (思维拓展)**
Hans Zimmer 的配乐经常在一个和弦上停留很久（例如 4-8 小节），但音乐并不无聊，因为内部一直在动。
*   **任务**：
    *   建立两个 `live_loop`。
    *   **Loop A (指挥官)**：每 `sleep 8` 换一个和弦根音，使用 `set :root, ...` 广播。
    *   **Loop B (士兵)**：执行 16 分音符的快速脉冲。使用 `get[:root]` 读取当前根音。
    *   **关键点**：在 Loop B 中加入 `cutoff` 的随机化或周期性变化，让这 8 秒钟内的音色不断变亮再变暗。

<details>
<summary>点击查看提示 (Hint)</summary>
<br>
利用 `vt` (virtual time) 或者 `tick` 来控制 cutoff 的曲线。例如 `cutoff: range(70, 110, 0.2).look` 可以让滤波器慢慢打开。
</details>
