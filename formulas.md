# 附录 D　公式速查表与常量来源对照

> 本文是**新增文件**，不改动任何现有代码或配置。
> 所有条目均由撰写者亲自打开对应源文件逐行核对，**行号为核对当时的真实行号**（基准：仓库当前 HEAD 的 `src/`）。
> 凡是没能在代码里找到读取点的数值，一律显式标注「未使用」或「未核实」，不做推断填补。

## 0. 阅读约定与统一算例

### 0.1 记号

| 记号 | 含义 |
| --- | --- |
| `ΣX` | 出战队伍全部角色在属性 `X` 上的取值之和（**含**已解锁科技的永久加成） |
| `team.attack` | 队伍攻击力，由 `Σ筋力` 派生 |
| `team.maxHp` | 队伍共享生命池上限，由 `Σ耐力` 派生 |
| `P(x)` | 边际递减概率曲线 `P_max × (1 - e^(-x/K))` |
| `power` | 技能倍率，配表里的整数百分数（`130` = 130%） |
| `rng.*` | 可复现随机源（mulberry32），见 `src/utils/random.ts` |

### 0.2 全文统一算例队伍（下文所有「算例」列都代入这组数）

```
出战 4 人：Σ筋力 = 30    Σ耐力 = 40    Σ敏捷 = 80
          Σ幸运 = 60    Σ魔力 = 50    Σ灵性 = 80
技能：skill_srf_rose_pierce（蔷薇刺突，power = 130，hits = 1，
      effects = [damage 130]，enhanced = [damage 45]，apCost = 1）
敌方单位：defense = 0.2（构造值，方便演示减伤；配表中接近的是 enemy_skeleton 的 0.12），
          未处于防御姿态
```

由此派生（各步推导见对应章节）：

```
team.attack       = round(30 × 5)                     = 150
team.maxHp        = round(40 × 20)                    = 800
team.expMultiplier= 1 + 50 × 0.02                     = 2.0
P_combo(80)       = 0.6 × (1 - e^-1)     ≈ 0.379270
P_crit(60)        = 0.5 × (1 - e^-0.75)  ≈ 0.263817
P_enhance(80)     = 0.5 × (1 - e^-1)     ≈ 0.316060
```

---

# 第一部分　公式速查表

## 1. 队伍聚合（攻击 / 生命 / 经验倍率 / 速度）

| 项 | 公式 | 输入来源 | 位置 |
| --- | --- | --- | --- |
| 队伍六维 | `stats[X] = Σ(roles.attributes[X].value) + techStatBonuses()[X]` | 角色卡属性 + 已解锁科技缓存 | `src/systems/TeamStats.ts:53-65` |
| 科技属性加成 | `bonus[X] += effectValue × max(1, state.rank)` | `tech.json` 的 `stat_bonus` 效果 | `src/systems/TeamStats.ts:22-35`（rank 乘算在 `:31`） |
| 队伍攻击力 | `team.attack = round(Σ筋力 × strengthToAttack)` | `balance.strengthToAttack = 5` | `src/systems/TeamStats.ts:72` |
| 队伍最大 HP | `team.maxHp = round(Σ耐力 × enduranceToHP)` | `balance.enduranceToHP = 20` | `src/systems/TeamStats.ts:73` |
| 队伍经验倍率 | `expMultiplier = 1 + Σ魔力 × magicToExpRate` | `balance.magicToExpRate = 0.02` | `src/utils/math.ts:39-41`，调用点 `src/systems/TeamStats.ts:75` |
| 战斗内队伍攻击力（含增益） | `attack = round((Σ筋力 + buffs) × strengthToAttack)` | 战斗内 buff 汇总 | `src/systems/CombatSystem.ts:229` |
| 单角色速度 | `speed = baseSpeed + 敏捷` | `balance.baseSpeed = 10` | `src/systems/TeamStats.ts:89` |
| 出战人数 | `members = cards.length` | 队伍数组长度 | `src/systems/TeamStats.ts:74` |

> **速度当前不参与任何计算**：`speed` 只是保留字段，行动顺序已改为「我方阶段 / 敌方阶段」双方阶段制，敌方内部按定义顺序出手，不排序。见 `src/systems/CombatSystem.ts:200-210`、`:113`、`:150`。

**算例**：`team.attack = round(30 × 5) = 150`；`team.maxHp = round(40 × 20) = 800`；`expMultiplier = 1 + 50 × 0.02 = 2.0`。

---

## 2. 三概率曲线（连击 / 暴击 / 强化）

统一曲线：`P(x) = P_max × (1 - e^(-x / K))`，`x ≤ 0` 时直接返回 0；`K ≤ 0` 时直接返回 `P_max`。
实现：`src/utils/math.ts:17-21`。

| 概率 | 公式 | 输入 | 参数默认值 | 位置 |
| --- | --- | --- | --- | --- |
| 连击率 | `P_combo = comboMax × (1 - e^(-Σ敏捷 / comboK))` | `Σ敏捷` | `0.6 / 80` | `src/utils/math.ts:24-26` |
| 暴击率 | `P_crit = critMax × (1 - e^(-Σ幸运 / critK))` | `Σ幸运` | `0.5 / 80` | `src/utils/math.ts:29-31` |
| 强化率 | `P_enhance = enhanceMax × (1 - e^(-Σ灵性 / enhanceK))` | `Σ灵性` | `0.5 / 80` | `src/utils/math.ts:34-36` |

判定上限：三种判定在 roll 时都被 `Math.min(0.95, ...)` 截断（即叠满加成也最多 95%）：
`src/systems/ProbabilitySystem.ts:43`（连击）、`:51`（暴击）、`:59`（强化）。

技能可自带 `combo_bonus` 效果补充连击率，配表值 `8` 表示 `+0.08`：`src/systems/DamageCalculator.ts:50-53`。

**参考曲线（实算，便于填表核对）**

| ΣX | P_combo（max 0.6） | P_crit / P_enhance（max 0.5） |
| --- | --- | --- |
| 10 | 0.070502 | 0.058751 |
| 30 | 0.187626 | 0.156355 |
| 60 | 0.316580 | 0.263817 |
| 80 | 0.379270 | 0.316060 |
| 120 | 0.466122 | 0.388435 |
| 200 | 0.550749 | 0.458958 |
| 300 | 0.585889 | 0.488241 |

**算例**：`P_combo(80) ≈ 0.379270`、`P_crit(60) ≈ 0.263817`、`P_enhance(80) ≈ 0.316060`。

---

## 3. 伤害主干（基础伤害 → 连击段数 → 逐段结算 → 保底）

一条完整的伤害链：`resolveAction`（`src/systems/DamageCalculator.ts:63-193`）→ 逐段 `resolveSingleHit`（`:196-251`）。

### 3.1 基础伤害

```
baseDamage = team.attack × power / 100
```
位置 `src/systems/DamageCalculator.ts:126`。
普通攻击视为 `power = 100`：`src/systems/DamageCalculator.ts:96`。

**算例**：`baseDamage = 150 × 130 / 100 = 195`。

### 3.2 连击段数（不是「判定一次」，而是几何式累加）

```
comboRate = min(0.95, P_combo + comboBonus + skillComboBonus(skill))
baseHits  = max(1, skill.hits ?? 1)
totalHits = baseHits
while totalHits < 8 且 rng.chance(comboRate):
    totalHits += baseHits          # 每次成功连击追加「一整个技能段数」
    comboTriggered = true
totalHits = min(totalHits, 8)
```
位置 `src/systems/DamageCalculator.ts:129-136`；硬上限 `MAX_HITS_PER_ACTION = 8` 在 `:20`。

> 含义：`hits: 2` 的 2 段技，连击一次即 +2 段（变 4 段），不是 +1 段。

**算例**：`comboRate = min(0.95, 0.379270 + 0 + 0) = 0.379270`，`baseHits = 1`。
期望段数 `= 1/(1-0.379270) ≈ 1.611` 段（受 8 段封顶截断）。
若本次判定**恰好连击 1 次**，则 `totalHits = 2`。下文按 `totalHits = 2` 演示。

### 3.3 单段结算（暴击 → 浮动 → 减伤 → 保底）

```
crit           = rng.chance(min(0.95, P_crit + critBonus))
critMultiplier = crit ? (critDamageBase + P_crit) : 1
variance       = rng.range(damageVarianceMin, damageVarianceMax)
defenseRate    = clamp(target.defense + (targetDefending ? defendReduction : 0), 0, 0.95)
rawDamage      = baseDamage × critMultiplier × variance
damage         = max(minDamage, round(rawDamage × (1 - defenseRate)))
```
位置 `src/systems/DamageCalculator.ts:222-223`（暴击）、`:229`（浮动）、`:230-234`（减伤）、`:236`（原始）、`:237`（保底与取整）。

参数默认值：`critDamageBase = 1.5`、`damageVarianceMin = 0.95`、`damageVarianceMax = 1.05`、`minDamage = 1`、`defendReduction = 0.5`（`src/data/tables/balance.json:13-17`）。

**算例**（`totalHits = 2`，`defense = 0.2`，浮动取中值 1.0）：

| 段 | 暴击 | 计算 | 结果 |
| --- | --- | --- | --- |
| 1 | 否 | `round(195 × 1 × 1.0 × (1-0.2))` | **156** |
| 2 | 是 | `raw = 195 × (1.5 + 0.263817) = 343.94`；`round(343.94 × 0.8)` | **275** |
| 合计 | — | `156 + 275` | **431** |

若两段都未暴击：`156 × 2 = 312`。若两段都暴击：`275 × 2 = 550`。
保底口径：即使 `power` 极低或减伤拉满，单段也至少 `1` 点。

### 3.4 强化（灵性触发）追加效果

```
enhanced = skill.enhanced 非空 ? rng.chance(min(0.95, P_enhance + enhanceBonus)) : false
```
位置 `src/systems/DamageCalculator.ts:226-227`。触发后由 `applyEnhancedEffects` 结算（`:302-349`）：

| 强化效果类型 | 公式 | 位置 |
| --- | --- | --- |
| `damage` | `extra = round(team.attack × value / 100)`，并累加进 `hit.damage` | `:315` |
| `heal` | `extra = round(team.attack × value / 100)` | `:323` |
| `buff` / `debuff` | 推入 `target.buffs`，`value` 取 `±abs(value)` | `:331-334` |
| `combo_bonus` | **仅写日志**，不改变本次结算 | `:341-343` |

**算例**：技能 `enhanced = [damage 45]`，触发 1 次 → `extra = round(150 × 45 / 100) = round(67.5) = 68`。
单段伤害由 156 → 224。

### 3.5 敌方攻击（不享受队伍加成）

敌人没有队伍合并属性，用「自身 attack 组一个 stub 队伍」，概率固定为 `combo 0.08 / crit 0.1 / enhance 0`：
`src/systems/DamageCalculator.ts:352-375`（固定概率在 `:372`）。

---

## 4. 辅助技（heal / buff / debuff / combo_bonus）数值

纯辅助技（`power = 0` 或目标非敌方）走 `applySupportEffects`：`src/systems/DamageCalculator.ts:254-299`。

| 效果 | 公式 | 位置 |
| --- | --- | --- |
| 治疗（技能 `effects[heal]`） | `amount = round(team.attack × value/100 + Σ灵性 × 0.5)` | `:265` |
| 治疗对象（`ally_all`） | 取 `[actor]`——**但 `caster.hp` 是我方共享池，等价于全队回血** | `:266`，目标解析 `src/systems/CombatSystem.ts:477-479` |
| `buff` / `debuff` | `signed = ±abs(value)`，`duration = effect.duration` | `:277-283` |
| `combo_bonus` | 作为一条 `source = "XX技能(连击 +N%)"`、`duration = 3`、`value = 0` 的伪 buff 记录 | `:290-298` |
| 战斗中属性→概率折算 | `crit += Σbuff(luck) × 0.004`；`enhance += Σbuff(spirit) × 0.004`；`combo += Σbuff(agility) × 0.004` | `src/systems/CombatSystem.ts:508-510` |
| 连击加成 buff 解析 | 从 `source` 正则抠 `连击 +(\d+)%`，累加 `N/100` | `src/systems/CombatSystem.ts:501-506` |
| 被动回血「回气」 | 每大回合对**全体我方**固定回 `10` 点 | `src/systems/CombatSystem.ts:554-559`（`heal = 10` 在 `:556`） |

**算例**：`skill_alst_healing_antler`（`heal value = 90`）→
`amount = round(150 × 0.90 + 80 × 0.5) = round(135 + 40) = 175`。

> ⚠️ 已核对的**文案/实现不一致**：该技能描述写「恢复 90% **魔力**值的生命」，但代码用的是 `team.attack`（由筋力派生）而不是魔力。同类技能（鹿角祝福 / 花语祝福 / 茶点侍奉 / 天使赐福 / 银盾庇护 …）全部如此。此处只陈述事实，是否修正由策划裁决。

**算例**：连击 buff 来源 `skill_rn_dual_draw` 的 `combo_bonus = 12` → `comboRate += 12/100 = 0.12`，持续 3 回合。

---

## 5. 升级经验需求与升级成长量

| 项 | 公式 | 参数 | 位置 |
| --- | --- | --- | --- |
| 升到下一级所需经验 | `expForLevel(level) = round(expBase × level^expExponent)` | `expBase = 20`，`expExponent = 1.6` | `src/utils/math.ts:44-46` |
| 加经验（单角色） | `gained = max(0, round(amount × (1 + 魔力 × magicToExpRate)))` | 魔力 = 角色自身 + 科技加成 | `src/systems/CharacterFactory.ts:228-229`、`:219-222` |
| 循环升级 | `while exp >= expForLevel(level) { exp -= need; level++ ; applyLevelUp() }` | — | `src/systems/CharacterFactory.ts:232-237` |
| 战斗经验总量 | `reward.exp = round(Σ(敌人 exp) × team.expMultiplier)` | `def.exp` 不随 `count` 放大 | `src/systems/CombatSystem.ts:616,627` |
| 队伍升级入口 | 每个存活成员**独立**按同一 `exp` 值结算（不是平分） | — | `src/systems/DungeonSystem.ts:385-393`、`src/scenes/BattleScene.ts:1565` |

**升级成长量**：

```
delta = max(0, round(levelUpBaseGain × potentialMultiplier[潜力] ) + rng.int(-1, 1))
```
位置 `src/systems/CharacterFactory.ts:246-257`（公式在 `:252`）。`levelUpBaseGain = 2`。

| 潜力 | 倍率 | `round(2 × 倍率)` | `delta` 区间 |
| --- | --- | --- | --- |
| A | 2.0 | 4 | 3 – 5 |
| B | 1.6 | 3 | 2 – 4 |
| C | 1.3 | 3 | 2 – 4 |
| D | 1.1 | 2 | 1 – 3 |
| E | 0.9 | 2 | 1 – 3 |
| F | 0.7 | 1 | 0 – 2 |

**算例**：
- `expForLevel(1) = round(20 × 1^1.6) = 20`；`expForLevel(5) = round(20 × 5^1.6) ≈ 263`；`expForLevel(10) ≈ 796`。
- 队伍打赢 `enemy_slime`（`exp = 14`），`expMultiplier = 2.0` → `reward.exp = 28`；队伍里每个成员各得 **28** 点（不是 28÷4）。
- 某 A 潜力角色升 1 级 → 该属性 `+3 ~ +5`。

> ⚠️ 已核对的**经验倍率被应用两次**：
> 第一处在 `src/systems/CombatSystem.ts:627`（`reward.exp = round(Σ敌人exp × team.expMultiplier)`），
> 第二处在 `src/systems/CharacterFactory.ts:229`（`gained = round(amount × (1 + 该角色魔力 × 0.02))`）。
> 调用链：`BattleScene.ts:1565` / `DungeonSystem.ts:388` 把**已经乘过队伍倍率**的 `reward.exp` 再交给 `gainExp`。
> 后果：`Σ魔力 = 50` 时队伍倍率 2.0，每个成员还会再乘一次自己的倍率（≥ 1），实际入账约为设计值的 2 倍以上。
> 此处只陈述事实，是否视为 bug 由策划裁决。

**实测算例（承接上式）**：`reward.exp = 28`；某成员自身 `魔力 = 12` → `gained = round(28 × (1 + 0.24)) = round(34.72) = 35`。

> ⚠️ 已核对的**未使用字段**：`balance.levelUpTechPointBonus`（默认 `0`）在 `src/` 内**没有任何读取点**，仅存在于 `balance.ts:54`、`balance.json:24` 与生成物 `balance.gen.ts`。当前为死配置。

---

## 6. 招募成本与稀有度权重

| 项 | 公式 | 参数 | 位置 |
| --- | --- | --- | --- |
| 本次招募金币花费 | `round(recruitGoldCost × recruitCostGrowth^recruitCount)` | 平价 **5 000**（D67 取消递增，`recruitCostGrowth` = 1）；改前 `120`、`1.35`（配表 / UI 待同步） | `src/ui/RecruitPanel.ts:112-115` |
| 本次招募科技点花费 | `max(0, recruitTechPointCost)`，**恒定不递增** | `2` | `src/ui/RecruitPanel.ts:118-120` |
| `recruitCount` | 本次营地面板会话内已成功招募的次数（面板成员变量，读档不累计） | — | `src/ui/RecruitPanel.ts:114` |
| 模板权重 | `weight = rarityWeights[rarity] × (rarity ≥ 4 ? 1 + rarityBonus : 1) × (unlockedByDefault ? 0.35 : 1)` | 见 `balance.json:38-41` | `src/systems/CharacterFactory.ts:159-169` |
| 新角色起始等级 | `clamp(rarity, 1, 9)` | — | `src/systems/CharacterFactory.ts:71-73` |
| 初始属性总和 | `initialStatTotal + (rarity - 1) × 3`，每项 ≥ `initialStatMin` | `60` / `1` | `src/systems/CharacterFactory.ts:109-112` |
| 可重复科技价格 | `round(cost × 1.6^rank)` | `1.6` | `src/systems/TechSystem.ts:55-59` |

`rarityWeights = {1:40, 2:30, 3:18, 4:9, 5:3}`（`src/data/tables/balance.json:41`）。

**算例**（`recruitCount` 从 0 起，未研究「招募许可 II」；**平价，D67 取消递增 ⇒ `recruitCostGrowth` = 1**）：

| 第几次 | 金币 | 科技点 |
| --- | --- | --- |
| 1 | `round(5000 × 1^0)` = **5 000** | 2 |
| 2 | `round(5000 × 1^1)` = **5 000** | 2 |
| 3 | `round(5000 × 1^2)` = **5 000** | 2 |
| 4 | `round(5000 × 1^3)` = **5 000** | 2 |
| 11 | `round(5000 × 1^10)` = **5 000** | 2 |

> **改前（D67 前）**：`120 × 1.35^n` ⇒ 120 → 162 → 219 → 295 → 2413（逐次递增）；D67 取消递增，D306 → 3 500、D310 → 5 000 ⇒ 现行为平价 **5 000**。

---

## 7. 敌人 `count` 折算（一群 = 一个合体单位）

配表 `enemies.json` 存「单只」数值 + `count: N`，构造战斗单位时折算：`src/systems/CombatSystem.ts:139-162`。

| 字段 | 折算规则 | 位置 |
| --- | --- | --- |
| `hp` / `maxHp` | `def.hp × count`（**线性**） | `:143` |
| `attack` | `round(def.attack × (1 + (count - 1) × 0.35))`（**递减式**，刻意不线性） | `:144`（`0.35` 是硬编码） |
| `defense` / `speed` / `exp` / `gold` / `techPoints` / `drops` | **一律不变**（奖励不随数量放大） | `:154`，奖励侧 `:614-625` |
| `name` | `count > 1` 时缀 ` ×N` | `:148` |

**算例**：`enemy_slime`（`hp 42`、`attack 8`、`count 3`、`exp 14`、`gold 10`）
→ `hp = 42 × 3 = 126`；`attack = round(8 × (1 + 2 × 0.35)) = round(13.6) = 14`；`exp` 仍 `14`、`gold` 仍 `10`。

单敌对照（`count = 1`）：`attack = round(8 × 1.0) = 8`。
`count = 2`：`attack = round(8 × 1.35) = round(10.8) = 11`。

---

## 8. AP 系统与逃跑成功率

### 8.1 AP 三项与升耗

| 项 | 公式 / 值 | 位置 |
| --- | --- | --- |
| 初始 AP | `AP_INITIAL = 3` | `src/systems/CombatSystem.ts:42`（用于 `:92`） |
| 每回合基础回复 | `AP_REGEN_BASE = 2` | `:43`（用于 `:94`、`:582`） |
| AP 上限基数 | `AP_CAP_BASE = 8` | `:44`（上限 = `8 + TechSystem.apCapBonus()`，`:93`） |
| 奇数回合额外回复 | `+ TechSystem.apRegenOddBonus()`（「奇袭节拍」= 1） | `:581-584` |
| 回合回复判定 | 读的是**递增之后**的 `round`，即回到第 N 回合按 N 的奇偶性 | `:562`、`:583` |

```
gain = apRegen + (apOddBonus > 0 且 round 为奇数 ? apOddBonus : 0)
ap   = min(apCap, ap + gain)
```
位置 `src/systems/CombatSystem.ts:581-584`。

**技能动态 AP 消耗**（`src/systems/CombatSystem.ts:258-266`）：

```
permanent 技能：cost = (skill.apCost ?? 1) + permStack[skill.id]     # 永续叠加，与 temp 互斥
常规技能    ：cost = (skill.apCost ?? 1) + (usedLastRound.has(id) ? 1 : 0)
```

> 两种升耗**只取其一、不相加**（注释在 `:239-257` 给出了原因：相加会双重计费）。
> 已核对：`skills.json` 里**没有任何技能声明 `escalation`**，所以当前全部走「常规 +1」路径；`permanent` 分支目前是空转。
> 旧字段 `Skill.cooldown` 已废弃，不再被任何逻辑读取（`src/types/skill.ts:41`）。

**算例**：`apCost = 1` 的常规技能，首回合用 → 花 1；同一技能下回合再用 → 花 `1 + 1 = 2`；再下回合仍花 2（`usedLastRound` 只记是/否，不累加）。

**AP 数值口径（默认，无科技）**：

| 回合 | 回合开始时 AP |
| --- | --- |
| 1（开局） | 3 |
| 2 | `min(8, 3 - 消耗 + 2)` |
| … | 每回合 `+2`，封顶 8（研究「战术扩容」后封顶 10） |

### 8.2 逃跑成功率

```
chance = min(0.95, fleeBaseChance + fleeAttempts × fleeChanceStep)
fleeAttempts += 1     # 在判定前自增：每次尝试（含失败）都会推进一档
```
位置 `src/systems/CombatSystem.ts:434-436`。`fleeBaseChance = 0.45`、`fleeChanceStep = 0.15`。

**算例**：

| 第几次尝试 | 成功率 |
| --- | --- |
| 1 | `0.45` |
| 2 | `0.60` |
| 3 | `0.75` |
| 4 | `0.90` |
| 5 | `min(0.95, 1.05)` = **0.95** |

另有**回合超时强制撤退**：`round > balance.maxRounds(60)` 时 `outcome = 'fled'`（`src/systems/CombatSystem.ts:589-592`）。

---

## 9. 迷宫生成：主干步数、房间格换算、元素数量

### 9.1 难度阶梯（唯一真源）

`src/config/mazeDifficulty.ts:44-51`。

| 档 | 尺寸 | `mainPathSteps`（玩家步） | 支路数 | 支路长度（玩家步） |
| --- | --- | --- | --- | --- |
| N1 | 15×15 | 20 | 0 | 0 / 0 |
| N2 | 19×19 | 32 | 2 | 4 – 6 |
| N3 | 23×23 | 48 | 4 | 6 – 10 |
| N4 | 27×27 | 58 | 6 | 8 – 14 |
| N5 | 31×31 | 74 | 8 | 10 – 18 |
| N6 | 35×35 | 90 | 10 | 12 – 24 |

**层 → 档映射**：进入第 `floor` 层就用第 `floor` 档（`src/systems/MazeGenerator.ts:73`，`src/scenes/DungeonScene.ts:129`、`:447`），并在界面上显示 `N{spec.level}`（`src/scenes/DungeonScene.ts:267-269`）。到 `floor ≥ MAZE_MAX_DIFFICULTY = 6` 时出口只剩「返回营地」（`src/scenes/DungeonScene.ts:415`）。

### 9.2 玩家步数 ⇄ 房间格换算

```
roomPathLength = max(2, floor(mainPathSteps / 2) + 1)      # 主干
roomLen        = max(1, round(stepLen / 2))                # 单条支路，stepLen 先 rng.int(min, max)
maxSteps（物理上限） = 2 × (rooms - 1)，rooms = floor((w-1)/2) × floor((h-1)/2)
```
位置：主干 `src/systems/MazeGenerator.ts:98`；支路 `:115-116`；物理上限夹取 `:218-224`。
**换算依据**：房间格之间隔一格墙，走一步房间 = 走两格地图（注释在 `src/config/mazeDifficulty.ts:24-31`、`src/systems/MazeGenerator.ts:97`）。

出口固定在主干末端，因此**最短通关步数 = 主干步数**：`src/systems/MazeGenerator.ts:109-110`；校验在 `:504-514`（`shortestPathSteps` vs `spec.mainPathSteps`）。

**算例**：

| 档 | 主干房间格 | 单条支路房间格 | `rooms` | 物理上限 `maxSteps` |
| --- | --- | --- | --- | --- |
| N1 | `max(2, 10+1) = 11` | 无支路 | 49 | 96 |
| N3 | `max(2, 24+1) = 25` | `round([6,10]/2)` = 3 – 5 | 121 | 240 |
| N6 | `max(2, 45+1) = 46` | `round([12,24]/2)` = 6 – 12 | 289 | 576 |

### 9.3 元素数量

```
enemyCap   = max(2, floor(floorTiles × 0.12))
enemyCount = min(enemyCap, max(1, round(floorTiles × config.enemyDensity)) + extraEnemies)
treasureCount / saveCount / eventCount / trapCount   # 直接读配表
```
位置 `src/systems/MazeGenerator.ts:161-176`；`0.12` 与下限 `2` 在 `:169` 硬编码。

**层间密度递增**（下探每一层额外加成）：`enemyDensity × (1 + (nextFloor - 1) × 0.12)`，`0.12` 硬编码于 `src/scenes/DungeonScene.ts:443`。

元素摆放顺序：宝箱 → 存档点 → 事件 → 陷阱 → 敌人，从洗牌后的地板格依次取空位（`src/systems/MazeGenerator.ts:161-176`，`take/place` 在 `:131-158`）。

**算例（N1，可精确推算）**：N1 尺寸 15×15 且 `branchCount = 0`，
地板格数 = 主干房间格 11 + 连接格 10 = **21**（推导：`mainPath` 有 11 个房间格，
相邻房间之间各补 1 格连接，共 10 格；`MazeGenerator.ts:100-107`）。
于是 `enemyCap = max(2, floor(21 × 0.12)) = max(2, 2) = 2`。
`dungeon_root` 的 `enemyDensity = 0.035` → `max(1, round(21 × 0.035)) = max(1, 1) = 1`
→ `enemyCount = min(2, 1 + 0) = 1`。N2 及以上含支路，地板格数随种子变化，`enemyCount` 需实跑确定。

**算例（层间递增）**：`dungeon_moss` 的 `enemyDensity = 0.045`，下探到第 3 层时
`0.045 × (1 + 2 × 0.12) = 0.0558`（`src/scenes/DungeonScene.ts:443`）。

---

## 10. 宝箱 / 事件 / 陷阱：硬编码权重

### 10.1 宝箱内容（`src/systems/MazeGenerator.ts:531-543`）

```
goldBase = 20 + config.floor × 25
roll = rng.float()
roll < 0.45               → { kind: 'gold', amount: rng.int(goldBase, round(goldBase × 2.2)) }
0.45 ≤ roll < 0.90        → { kind: 'item', itemId: 从 5 项池任取, count: rng.int(1, 2) }
roll ≥ 0.90               → { kind: 'tech', amount: rng.int(1, 2) }
```
- 概率分界 `0.45 / 0.90`、金币基数 `20 + floor × 25`、上限倍率 `2.2` 全部硬编码在 `:532-542`。
- 道具池是字面量数组：`['item_potion_s','item_potion_l','item_ore','item_crystal','item_coin_pouch']`（`:538`）。
- ⚠️ 已核对：这里传入的是 `config.floor`（配表字段），而 `dungeons.json` 四张图**都写 `"floor": 1`**，所以 `goldBase` 恒为 `45`，不随实际层数增长。真正随层数变的是 `enemyDensity`（见 §9.3）。

**算例**：四张地牢任意一层开箱，金币分支金额区间 `[45, round(45 × 2.2)] = [45, 99]`。

### 10.2 事件点内容（`src/systems/MazeGenerator.ts:546-558`）

| 事件 | 权重 | 概率 |
| --- | --- | --- |
| `event_spring` 泉水 | 30 | 30% |
| `event_merchant` 商人 | 20 | 20% |
| `event_shrine` 神龛 | 25 | 25% |
| `event_ambush` 埋伏 | 25 | 25% |

权重硬编码为字面量数组（`rng.weighted`，权重和恰好 100）。文案与选项在 `src/systems/DungeonSystem.ts:324-353`。

### 10.3 陷阱内容（`src/systems/MazeGenerator.ts:561-571`）

| 类型 | 权重 | 概率 | 参数 |
| --- | --- | --- | --- |
| `damage` 伤害 | 50 | 50% | `value = rng.int(8, 20)` |
| `slow` 迟缓 | 25 | 25% | `value = 3`（固定 3 回合） |
| `teleport` 传送 | 25 | 25% | `value = 3`（未被使用） |

权重与 `rng.int(8, 20)`、固定 `3` 全部硬编码在 `:562-570`。

### 10.4 陷阱伤害公式（`src/systems/DungeonSystem.ts:286-290`）

```
damage = round(pool.maxHp × value / 100) + value
pool.hp = max(1, pool.hp - damage)      # 陷阱不能致死，至少留 1
```

**算例**：`pool.maxHp = 800`、`value = 15` →
`damage = round(800 × 0.15) + 15 = 120 + 15 = 135`；剩余 HP 至少 1。

> 注意公式里那个 **`+ value` 的平地追加项**：陷阱越「高级」（value 越大）既有百分比成长，也有绝对成长，这是硬编码在表达式结构里的，改它必须动代码。

---

## 11. 迷宫事件的具体数值（`src/scenes/DungeonScene.ts`）

| 事件选项 | 数值 | 位置 |
| --- | --- | --- |
| 泉水·饮下 | `heal = round(pool.maxHp × 0.35)` | `:509` |
| 泉水·灌水袋 | 获得 `item_potion_s × 1` | `:513` |
| 商人·买药水 | 花 `80` 金币，得 `item_potion_l × 1` | `:520-521` |
| 商人·卖材料 | 每份 `max(1, round(price × 0.5))` 金币，素材池 `item_ore / item_crystal / item_core` | `:528-536`（`0.5` 在 `:532`） |
| 神龛·献金 | 花 `50` 金币，得 `2` 科技点 | `:546-547` |
| 神龛·祈祷 | 随机一名队员、随机一项属性 `+3` | `:553-559` |
| 埋伏·迎战 | 直接开战 | `:568-569` |
| 埋伏·逃跑 | `rng.chance(0.5)` 成功则随机传送，否则被迫开战 | `:570-578` |

另有**首次通关奖励**：`gameState.addTechPoints(config.firstClearTechPoints)`，数值来自 `dungeons.json`（**`8 / 9 / 10 / 11`，合计 38**），读点在 `src/scenes/DungeonScene.ts:481`。

> 🔧 **勘误（第 101 轮 · 落地实测）**：本文原写四图首通为 **`3 / 3 / 5 / 10`（合计 21）**，那是 **D23** 按**旧的 79 点科技树**标的价，**已被 D246 取代**。现配表 `dungeons.json` 的 `firstClearTechPoints` 实为 **`8 / 9 / 10 / 11`（合计 38）**，口径为「**首通 38 + 主线任务 14**」（D246 / D255）；验收硬门 #5（D297）用的就是 **38**。

**算例**：`maxHp = 800` 时泉水恢复 `round(800 × 0.35) = 280` 点。

---

## 12. 视野半径与判定

| 项 | 定义 | 位置 |
| --- | --- | --- |
| 视线半径 | `SIGHT_RADIUS = 5`（格，**曼哈顿距离**） | `src/systems/DungeonSystem.ts:24` |
| 可见范围 | `|dx| + |dy| ≤ 5`，且不越界 | `:126-132` |
| 遮挡规则 | `dist > 2` 时做简易视线检查（Bresenham），被墙挡住的格子不点亮 | `:134` |
| 视线算法 | Bresenham 直线插值，`grid[y][x] !== 0`（墙/虚空）即判定被挡 | `:142-172`（`guard < 64` 步上限在 `:157`） |
| 视野状态机 | `2 = 当前可见` → 下一帧降级为 `1 = 已探索`；`0 = 未知` | `:117-123`、`:135` |
| 玩家所在格 | 无条件置 `2`（`|dx|+|dy| = 0` 本身也满足，这里显式兜底） | `:138` |

> 因此实际可见区域是「半径 5 的曼哈顿菱形」∩「半径 2 以内无视遮挡 + 半径 3–5 需通视」。

**算例**：玩家在 `(1,1)` 时，`(4,1)`（`dist = 3`）若与 `(1,1)` 之间有墙则不点亮；`(3,1)`（`dist = 2`）无论如何都点亮。

**已探索比例**（HUD 用）：`explored / total`，分母只统计 `grid !== 0` 的地板格：`src/systems/DungeonSystem.ts:185-196`。

---

## 13. 相机缩放与迷宫像素尺寸

| 常量 / 公式 | 值 | 位置 |
| --- | --- | --- |
| 单格世界像素 | `TILE = 34` | `src/scenes/DungeonScene.ts:48` |
| 屏幕上迷宫目标占比 | `ZOOM_TARGET = 1.15`（视口短边的 1.15 倍） | `src/scenes/DungeonScene.ts:62` |
| 缩放下限 | `ZOOM_MIN = 1`（不缩小） | `src/scenes/DungeonScene.ts:64` |
| 缩放上限 | `ZOOM_MAX = 2.2` | `src/scenes/DungeonScene.ts:65` |

```
mazeSpan  = max(spec.width, spec.height) × TILE
shortSide = min(GAME_WIDTH, GAME_HEIGHT)
raw       = (shortSide × ZOOM_TARGET) / mazeSpan
zoom      = round(clamp(raw, ZOOM_MIN, ZOOM_MAX) × 100) / 100
```
位置 `src/scenes/DungeonScene.ts:752-759`（公式在 `:754-758`）。取两位小数是为了避免非整数倍率下的半像素接缝（注释在 `:757`）。
相机跟随**不做边界夹取**，队伍恒居中，迷宫外用背景色纯黑填充（`:810-819`）。

**算例**（逻辑分辨率默认 1920×1080，`shortSide = 1080`，`shortSide × 1.15 = 1242`）：

| 档 | `mazeSpan` | `raw = 1242 / mazeSpan` | 最终 `zoom` |
| --- | --- | --- | --- |
| N1 | `15 × 34 = 510` | 2.4353 → 触顶 | **2.2** |
| N3 | `23 × 34 = 782` | 1.5882 | **1.59** |
| N6 | `35 × 34 = 1190` | 1.0437 | **1.04** |

---

# 第二部分　常量来源对照表

按「**改它要去哪改**」分三类。列 `当前值` 一律取当前配表 / 源码实际值。

## A 类　配表可改（改 JSON，跑 `scripts/gen-tables.mjs` 即可，不动代码）

| 数值 | 当前值 | 改它的位置 |
| --- | --- | --- |
| 筋力→攻击系数 | 5 | `src/data/tables/balance.json:2`（回退默认 `src/config/balance.ts:93`） |
| 耐力→HP 系数 | 20 | `balance.json:3` / `balance.ts:94` |
| 魔力→经验倍率 | 0.02 | `balance.json:4` / `balance.ts:95` |
| 连击率上限 / K | 0.6 / 80 | `balance.json:6-7` / `balance.ts:97-98` |
| 暴击率上限 / K | 0.5 / 80 | `balance.json:8-9` / `balance.ts:99-100` |
| 强化率上限 / K | 0.5 / 80 | `balance.json:10-11` / `balance.ts:101-102` |
| 暴击伤害基数 | 1.5 | `balance.json:13` / `balance.ts:104` |
| 伤害浮动区间 | 0.95 – 1.05 | `balance.json:14-15` / `balance.ts:105-106` |
| 单段最低伤害 | 1 | `balance.json:16` / `balance.ts:107` |
| 防御减伤率 | 0.5 | `balance.json:17` / `balance.ts:108` |
| 升级基础成长值 | 2 | `balance.json:19` / `balance.ts:110` |
| 初始属性总上限 | 60 | `balance.json:20` / `balance.ts:111` |
| 初始单项最低值 | 1 | `balance.json:21` / `balance.ts:112` |
| 升级经验系数 / 指数 | 20 / 1.6 | `balance.json:22-23` / `balance.ts:113-114` |
| 升级科技点奖励 | 0 | `balance.json:24` / `balance.ts:115`　**（无读取点，见 §5 注）** |
| 队伍上限 / 初始栏位 | 6 / 4 | `balance.json:26-27` / `balance.ts:117-118` |
| 基础速度 | 10 | `balance.json:29` / `balance.ts:120` |
| 逃跑基础率 / 每回合步进 | 0.45 / 0.15 | `balance.json:30-31` / `balance.ts:121-122` |
| 战斗回合上限 | 60 | `balance.json:32` / `balance.ts:123` |
| 招募基础金币 / 递增系数 / 科技点 | **5 000 / 1（平价，D67 取消递增）/ 2**（改前 120 / 1.35 / 2；配表待同步） | `balance.json:34-36` / `balance.ts:125-127` |
| 潜力倍率表 A–F | 2.0/1.6/1.3/1.1/0.9/0.7 | `balance.json:38` / `balance.ts:129` |
| A / B 潜力最多项数 | 2 / 3 | `balance.json:39-40` / `balance.ts:130-131` |
| 稀有度权重 1–5 | 40/30/18/9/3 | `balance.json:41` / `balance.ts:132` |
| 技能倍率 / AP 消耗 / 段数 / 强化效果 | 每技能各自 | `src/data/tables/skills.json`（`power` / `apCost` / `hits` / `effects` / `enhanced`） |
| 敌人 HP / 攻击 / 减伤 / 经验 / 金币 / 科技点 / 群数 / 掉落 | 每敌人各自 | `src/data/tables/enemies.json` |
| 地牢元素数 / 密度 / 首批科技点 / 敌人池权重 | 每图各自 | `src/data/tables/dungeons.json` |
| 科技价格 / 效果值（AP 上限 +2、奇数回能 +1、稀有度加成 1.0） | 每科技各自 | `src/data/tables/tech.json` |
| 道具价格 / 堆叠上限 | 每道具各自 | `src/data/tables/items.json` |
| 角色稀有度 / 技能槽 | 每角色各自 | `src/data/tables/characters.json` |

> 注意：`balance.json` 允许**部分覆盖**，缺失字段回落到 `balance.ts` 的 `BALANCE` 常量（`src/config/balance.ts:139-147`，装配在 `src/data/loaders/DataLoader.ts:48`）。所以「改 balance.json 不生效」的现象通常意味着该键名拼错、回落到了默认值。

## B 类　TS 模块常量（改常量即可，不需要动逻辑，但必须重新构建）

| 数值 | 当前值 | 改它的位置 |
| --- | --- | --- |
| 难度阶梯（尺寸 / 主干步数 / 支路数 / 支路长度） | N1–N6，见 §9.1 | `src/config/mazeDifficulty.ts:44-51` |
| 单段命中次数上限 | 8 | `src/systems/DamageCalculator.ts:20` |
| 视线半径 | 5 | `src/systems/DungeonSystem.ts:24` |
| 单格世界像素 | 34 | `src/scenes/DungeonScene.ts:48` |
| 缩放目标 / 上下限 | 1.15 / 1 / 2.2 | `src/scenes/DungeonScene.ts:62-65` |
| 方向键留白 | 28 | `src/scenes/DungeonScene.ts:45` |
| 逻辑分辨率（可变 `let`） | 1920×1080 | `src/config/constants.ts:23-24`（改走 `setLogicalSize()`，`:27-30`） |
| 顶部信息条高度 | 64 | `src/config/constants.ts:95` |
| `balance.ts` 的 `BALANCE` 默认值 | 同 A 类 | `src/config/balance.ts:92-133`（**A 类配表的回退基线**） |
| 迷宫自检阈值（可达格 < 20 报警） | 20 | `src/systems/MazeGenerator.ts:495-497` |
| 网格尺寸下限 / 奇数化 | ≥11 且为奇数 | `src/systems/MazeGenerator.ts:195-199` |
| 配表容量自检（N1 容量 = w×h/2） | `floor(15×15/2) = 112` | `src/data/loaders/DataLoader.ts:235-239` |

## C 类　硬编码在逻辑里（改它必须动代码）

| 数值 | 当前值 | 位置 | 说明 |
| --- | --- | --- | --- |
| AP 初始值 | 3 | `src/systems/CombatSystem.ts:42` | 局部 `const`，非配置 |
| AP 每回合回复 | 2 | `src/systems/CombatSystem.ts:43` | 同上 |
| AP 上限基数 | 8 | `src/systems/CombatSystem.ts:44` | 科技加成叠在它之上 |
| 敌人 `count` 攻击放大系数 | 0.35 | `src/systems/CombatSystem.ts:144` | `attack × (1 + (count-1) × 0.35)` |
| 战斗内属性→概率折算 | 0.004 / 点 | `src/systems/CombatSystem.ts:508-510` | 敏捷/幸运/灵性三处 |
| 「回气」固定回血量 | 10 | `src/systems/CombatSystem.ts:556` | 每大回合，全体我方 |
| 敌人 AI：自我治疗血线 | < 0.4 | `src/systems/CombatSystem.ts:640` | `hp / maxHp < 0.4` |
| 敌人 AI：使用攻击技能概率 | 0.65 | `src/systems/CombatSystem.ts:646` | 否则普攻 |
| 我方自动战斗 AI：治疗血线 | < 0.45 | `src/systems/CombatSystem.ts:661` | |
| 逃跑成功率上限 | 0.95 | `src/systems/CombatSystem.ts:434` | |
| 三概率判定上限 | 0.95 | `src/systems/ProbabilitySystem.ts:43,51,59`；连击率另在 `src/systems/DamageCalculator.ts:129` 再夹一次 | 叠满也封顶 95% |
| 防御减伤率上限（clamp 上限） | 0.95 | `src/systems/DamageCalculator.ts:230-233` | |
| 敌方固定概率组 | combo 0.08 / crit 0.1 / enhance 0 | `src/systems/DamageCalculator.ts:372` | 敌方不吃队伍曲线 |
| 连击 buff 默认持续回合 | 3 | `src/systems/DamageCalculator.ts:293` | `combo_bonus` 伪 buff |
| 宝箱概率分界 | 0.45 / 0.90 | `src/systems/MazeGenerator.ts:534,537` | 金币/道具/科技点 |
| 宝箱金币基数公式 | `20 + floor × 25` | `src/systems/MazeGenerator.ts:532` | `floor` 取 `config.floor`（恒 1，见 §10.1） |
| 宝箱金币上限倍率 | 2.2 | `src/systems/MazeGenerator.ts:535` | |
| 宝箱道具池（5 项字面量） | potion_s / potion_l / ore / crystal / coin_pouch | `src/systems/MazeGenerator.ts:538` | |
| 宝箱道具数量 | `int(1, 2)` | `src/systems/MazeGenerator.ts:540` | |
| 宝箱科技点数量 | `int(1, 2)` | `src/systems/MazeGenerator.ts:542` | |
| 事件权重 | 30 / 20 / 25 / 25 | `src/systems/MazeGenerator.ts:550-553` | 泉水/商人/神龛/埋伏 |
| 陷阱权重 | 50 / 25 / 25 | `src/systems/MazeGenerator.ts:564-566` | 伤害/迟缓/传送 |
| 陷阱伤害 `value` 区间 | `int(8, 20)` | `src/systems/MazeGenerator.ts:570` | |
| 陷阱 slow/teleport 的 value | 3 | `src/systems/MazeGenerator.ts:570` | 传送分支未读取该值 |
| 敌人数硬上限系数 | 0.12 | `src/systems/MazeGenerator.ts:169` | `enemyCap = max(2, floor(floorTiles × 0.12))` |
| 敌人上限的下限 / 实际数量下限 | 2 / 1 | `src/systems/MazeGenerator.ts:169`、`:172` | 上限至少 2；最终 `enemyCount` 至少 1 |
| 层间敌人密度递增 | 每层 +12% | `src/scenes/DungeonScene.ts:443` | `×(1 + (n-1) × 0.12)` |
| 陷阱伤害公式结构 | `round(maxHp×v/100) + v` | `src/systems/DungeonSystem.ts:288` | 百分比 + 绝对双段成长 |
| 陷阱伤害默认 `value` | 10 | `src/systems/DungeonSystem.ts:286` | payload 缺失时 |
| 陷阱不可致死保底 | 至少留 1 HP | `src/systems/DungeonSystem.ts:289` | |
| 宝箱默认金币 / 科技点 / 道具 | 50 / 1 / `item_potion_s × 1` | `src/systems/DungeonSystem.ts:263`、`:268`、`:273` | payload 缺失时 |
| 默认敌人 id | `enemy_slime` | `src/systems/DungeonSystem.ts:254` | |
| 视野遮挡豁免半径 | 2（`dist > 2` 才查视线） | `src/systems/DungeonSystem.ts:134` | |
| 视线检查步数护栏 | 64 | `src/systems/DungeonSystem.ts:157` | |
| 泉水恢复比例 | 0.35 | `src/scenes/DungeonScene.ts:509` | `maxHp × 0.35` |
| 商人药水售价 | 80 金币 | `src/scenes/DungeonScene.ts:520` | |
| 素材回收折扣 | 0.5 | `src/scenes/DungeonScene.ts:532` | `price × 0.5`，至少 1 |
| 神龛献金 / 回报 | 50 金币 → 2 科技点 | `src/scenes/DungeonScene.ts:546-547` | |
| 神龛祈祷加成 | +3 单项属性 | `src/scenes/DungeonScene.ts:557` | |
| 埋伏逃跑成功概率 | 0.5 | `src/scenes/DungeonScene.ts:570` | |
| 遇敌数量恒为 1 | 硬截断 | `src/scenes/DungeonScene.ts:598-604` | 多敌入参只取第一个 |
| 稀有度每级 +3 初始属性 | 3 | `src/systems/CharacterFactory.ts:111` | `initialStatTotal + (rarity-1)×3` |
| 起始等级上限 / 下限 | 9 / 1 | `src/systems/CharacterFactory.ts:72` | `clamp(rarity, 1, 9)` |
| A 潜力稀有度修正 | `(rarity+1)/6`，下限 0.2 | `src/systems/CharacterFactory.ts:84` | |
| B 潜力抽取基数 | `round(3 × rarityFactor)` | `src/systems/CharacterFactory.ts:86` | |
| 低潜力池字面量 | `['C','C','D','D','E','F']` | `src/systems/CharacterFactory.ts:100` | |
| 初始解锁角色权重折减 | 0.35 | `src/systems/CharacterFactory.ts:165` | `unlockedByDefault` |
| 技能槽固定数量 | 2 | `src/systems/CharacterFactory.ts:140`、`DataLoader.ts:216-218` | 配表校验也硬性要求 = 2 |
| 可重复科技价格基数 | 1.6 | `src/systems/TechSystem.ts:58` | `cost × 1.6^rank` |
| 可重复科技默认层数上限 | 5 | `src/systems/TechSystem.ts:65` | `tech.maxRank ?? 5` |
| 战斗日志上限条数 | 200 | `src/systems/CombatSystem.ts:687` | |
| 迷宫中传送目标 | 均匀随机地板格 | `src/systems/DungeonSystem.ts:223-230` | `Math.random()`，非种子随机源 |

---

# 第三部分　改数值的推荐路径

目标：把上述 **C 类硬编码项**也纳入配表，让策划不用碰代码。最小改动方案如下（**本方案不落地，仅列方案与涉及文件**）。

## 方案 1　战斗侧：给 `balance.json` 补键（改动量最小）

在 `BalanceConfig` 里新增字段，值全部走已有的「balance.json 部分覆盖 → 回退默认值」机制：

| 新增键 | 接管哪条硬编码 | 调用点文件 |
| --- | --- | --- |
| `apInitial` / `apRegenBase` / `apCapBase` | `CombatSystem.ts:42-44` | `src/systems/CombatSystem.ts:92-94,581-584` |
| `enemyCountAttackFactor` | `CombatSystem.ts:144` 的 `0.35` | `src/systems/CombatSystem.ts:144` |
| `buffStatToChance` | `CombatSystem.ts:508-510` 的 `0.004` | `src/systems/CombatSystem.ts:508-510` |
| `regenHeal` | `CombatSystem.ts:556` 的 `10` | `src/systems/CombatSystem.ts:556` |
| `enemyHealHpRatio` / `enemySkillChance` / `allyAutoHealRatio` | `CombatSystem.ts:640,646,661` | `src/systems/CombatSystem.ts:634-676` |
| `chanceCap` | 各处的 `0.95` 上限 | `ProbabilitySystem.ts:43,51,59`、`DamageCalculator.ts:129,230` |
| `enemyBaseProbs` | `DamageCalculator.ts:372` 的 `{0.08, 0.1, 0}` | `src/systems/DamageCalculator.ts:372` |
| `maxHitsPerAction` | `DamageCalculator.ts:20` 的 `8` | 同上（需从 `export const` 改为读 balance） |

涉及文件：`src/config/balance.ts`（加字段与默认值）、`src/data/tables/balance.json`（加键）、`src/systems/CombatSystem.ts`、`src/systems/DamageCalculator.ts`、`src/systems/ProbabilitySystem.ts`。
生成物 `src/data/tables/balance.gen.ts` 由 `scripts/gen-tables.mjs` 重跑覆盖。

## 方案 2　迷宫侧：新建 `mazeTables.json`（新配表，语义更清晰）

C 类里迷宫/事件/陷阱的数值成组出现且彼此无关，塞进 `balance.json` 会让「平衡表」语义被污染。推荐单独建 `src/data/tables/mazeTables.json`：

```
{
  "treasure": { "goldThreshold": 0.45, "itemThreshold": 0.90, "goldBase": 20, "goldPerFloor": 25,
                "goldMaxFactor": 2.2, "itemPool": [...], "itemCountMax": 2, "techCountMax": 2 },
  "event":    { "weights": { "event_spring": 30, "event_merchant": 20,
                             "event_shrine": 25, "event_ambush": 25 } },
  "trap":     { "weights": { "damage": 50, "slow": 25, "teleport": 25 },
                "damageValueMin": 8, "damageValueMax": 20, "slowTurns": 3 },
  "element":  { "enemyCapRatio": 0.12, "enemyCapMin": 2, "floorDensityStep": 0.12 },
  "sight":    { "radius": 5, "occlusionFreeRadius": 2 },
  "eventNumeric": { "springHealRatio": 0.35, "potionPrice": 80, "sellFactor": 0.5,
                    "shrineGold": 50, "shrineTechPoints": 2, "prayerStatGain": 3,
                    "ambushFleeChance": 0.5 }
}
```

涉及文件：新增 `src/data/tables/mazeTables.json` 与 `src/types/`（或复用现有类型文件）里的接口、`scripts/gen-tables.mjs` 与 `src/data/tables/index.gen.ts`（生成链路）、`src/data/loaders/DataLoader.ts`（`collectTables()` / `validate()` / getter）、消费点 `src/systems/MazeGenerator.ts`、`src/systems/DungeonSystem.ts`、`src/scenes/DungeonScene.ts`。

## 方案 3　相机与渲染常量

`TILE` / `ZOOM_TARGET` / `ZOOM_MIN` / `ZOOM_MAX`（`src/scenes/DungeonScene.ts:48,62-65`）建议直接提到 `src/config/graphics.ts`（该文件已存在且专责图形设置），仍作模块常量而非配表——它们是渲染口径而非玩法数值，配表化收益低。
`GAME_WIDTH` / `GAME_HEIGHT` 已是可变绑定，无需改动。

## 落地顺序建议

1. **先做方案 1**：全在既有 `balance.json` 通道上，改动面最小，且 `balance.json` 已经验证过「部分覆盖 + 回退」链路。
2. **再做方案 2**：需要新增一条配表生成链路，改动面涉及 `gen-tables.mjs`，建议单独一次提交并验证 `DataLoader.validate()` 通过。
3. **方案 3 最后做**：纯搬迁，零行为变更，回归风险最低。

三项都不涉及存档结构（`save.ts`），因此**不影响已有存档兼容性**。

---

# 附：核对说明

- **核对方式**：逐文件通读 + 全仓 grep 交叉验证（重点验证「某数值是否真有读取点」）。
- **已确认的未使用/存疑项**（不做推测，均直陈事实）：
  1. `balance.levelUpTechPointBonus`（`0`）：`src/` 内无读取点，仅存在于定义与生成物。
  2. `Skill.cooldown`：`src/types/skill.ts:41` 标注 `@deprecated`，无逻辑读取。
  3. `skills.json` 中**无任何技能**声明 `escalation`，故 `permanent` 分支（`CombatSystem.ts:260-263,398-400`）当前不生效。
  4. 敌人 `speed` 与角色 `speed` 均不参与任何计算，仅保留字段。
  5. 宝箱金币基数用的 `config.floor` 恒为 1（四张地牢配表均写 `"floor": 1`），层数增长实际未生效。
  6. 治疗类技能文案写「魔力」，实现用 `team.attack`（筋力派生）。
  7. 经验倍率被应用两次（队伍一次 + 成员一次），见 §5 注。
- **未核实项**：`src/scenes/BattleScene.ts` 的 UI 层数值（血条动画时长、按钮布局像素等）未纳入本表——它们不影响玩法公式。其余即战力数值已全覆盖。
