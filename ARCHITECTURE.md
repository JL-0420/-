# 余音之下：架构说明与扩展方案

> 本文档基于 2026-10-19 对项目源码的实际通读整理，不是凭空规划。
> **2026-10-19 更新 ①：A 方案（§7）十项已全部执行完毕**，§6 改为修复记录。
> **2026-10-19 更新 ②：卡池扩到 10 张、加入状态效果系统（力量 / 易伤 / 多段 / 抽牌）、10 张像素风卡图齐备（§11）。**
> **2026-10-19 更新 ③：敌人 AI 轮转 + 狂暴（§12）；关卡系统上线，5 层怪物 + 属性倍率，打赢进下一层（§13）。**
> **2026-10-19 更新 ④：玩家 + 5 只怪物像素立绘、四套代码动画（待机 / 前冲 / 受击 / 死亡），战斗界面终于有画面了（§14）。**
> **2026-10-20 更新 ⑤：存档系统上线（进新一层自动存、死亡删档），顺手修掉了玩家死亡后卡死的问题（§15）。**
> **2026-10-21 更新 ⑥：绯奈改成透明底立绘 + 逐帧动画（idle / attack / hit / death，AnimatedSprite2D），战斗里终于像「站在场景里」而不是贴方块（§14.1–§14.3）。**
> **2026-10-21 更新 ⑦：防御统一成一套程序化护罩（玩家 / 怪物共用，零图片资源），「双倍蓄力」补了专属能量特效并加了 `has_fx` 内部标记（§14.5）。**
> **2026-10-21 更新 ⑧：技能牌也有了施法动作——专属技能「举手过头顶蓄力」（cast 帧），通用纯增益「抬手托起金光」（buff 帧 + 程序化金色光环）；护盾牌只出护罩不摆动作（§14.2、§14.5）。**
> **2026-10-22 更新 ⑨：怪物机制改成「固定牌库」——每只怪配一套牌，每回合随机抽 1 张打出（牌库不消耗），牌库同时也是掉落池；奖励从「3 选 1」改为「随机掉牌库内 2 张、两张都拿」，并保底至少 1 张怪物牌（`is_monster_card`）。标题页删除「继续游戏」「背包」两个按钮（存档逻辑保留，见 §15、§18.1）。详见 PROJECT_LOG 第四十一轮。**
> **2026-10-22 更新 ⑩：战斗数值换成「攻击力/防御力 + 百分比卡牌」体系**——卡牌伤害 = 攻击力 × `percent_attack`，
> 护盾 = 防御力 × `percent_defense`，带会心（5% ×2）、免伤公式（先减防御再按 `防御/(300+防御)` 减免，上限 5 倍防御）、
> 力量每点 +10 攻击力；`CardData` 增加 `source`/`quality`/`tags` 三个分类字段；新卡「恶魔城」（**领域卡**，带优先级碰撞）。
> 「回合」= 任意一方执行完一次行动。详见 PROJECT_LOG 第四十三轮。
> **2026-10-22 更新 ⑪：面板化重做 + 成长体系**
> - **背包面板统一**：`BagPanel`（局内局外共用），860×540 居中，右侧竖排按钮「属性 / 物品 / 卡牌」，卡片原尺寸 1/4、同名牌堆叠、牌库 ↔ 背包双向移动（只有奖励页「编辑卡组」可编辑）
> - **金币 + 稀有度 + 出售**：7 级稀有度（普通→本源），原价 15~4000，出售 60%
> - **物品系统**：`ItemData` 资源自动扫描注册、100 格物品栏、可用（小型回血药剂）
> - **经验与阶位**：升级 `100 × 1.04^(min(L,80)-1)`；击杀 `基础 × (1+等级%) × 2^(阶位+1)`；越级惩罚；阶位靠**提交素材**升，低阶打高阶按差值表削伤害（双向）
> - **属性按等级现算**：`CharacterData.eff_*`，成长值每个角色一份、累乘
> - 新增章节：§23 等级 / 经验 / 阶位系统，§24 物品系统与背包面板
> **2026-10-23 更新 ⑫：领域放牌门 + 抗性优先级（§25）**
> - **领域**：后放的领域**低 4 级以上**才放不出来；**低 1/2/3 级**能放但按碰撞表被压制（×0.5/×0.1/×0.01）；
>   **同阶**能放但只发挥 70%，对方领域消失/被替换后自动恢复 100%（`can_deploy_domain` / `DOMAIN_EQUAL_MULT` / `*_domain_seq`）
> - **抗性**：抗性数值当「抗性优先级」用，卡牌效果优先级 = 实际阶位（保底不低于「使用者阶位 - 1」）；
>   差距 ≤0 / 1 / 2 / 3 / >3 → ×1 / ×0.5 / ×0.1 / ×0.01 / 免疫；**易伤**已接入，元素状态待做
> **2026-10-23 更新 ⑬：局外商店 + 调试关卡编辑器 + 怪物逐帧动画（§28、§29、§14.6）**
> - **商店**：只卖「已经获得过」的非专属、非初始卡（`unlocked_cards` 解锁记录）；
>   入口三个：右上「商店」按钮 / 点击路边商人 / 靠近商人按 F
> - **调试面板**：760×520 两个页签——「人物」（等级 / 阶位）+「**关卡编辑器**」
>   （怪物类型 / 等级 / 阶位可改，✕ 删关、+ 添加关卡，数量暂锁 1；`LEVELS` 由 `const` 改 `var`）
> - **怪物逐帧动画**：5 只怪 × idle / attack / hit 各 4 帧 = **60 帧**
>   （`assets/characters/enemies/`，320×320 画布、脚底基线 300）；防御护罩 / 强化光环仍是程序化特效
> - **尺寸**：史莱姆 130 / 骷髅兵 220 / 石像鬼 250 / 狂战士 290 / 首领 330（人物 260 为基准）
> - **手牌**：`y=525`，卡面描述默认在屏幕外，悬停抬起 90px 后完整露出
> - **素材修**：`assets/items/*.png` alpha 太低（不透明仅 2~17%）→ 重映射拉实心，原图备份 `tools/_items_backup/`

> 包含：A 方案（UI 修复，§7）、C 方案（角色 / 局外背包 / 剧情扩展设计，§10）、美术管线（§11）、敌人 AI（§12）、关卡配置（§13）、立绘演出（§14）、存档（§15）
> **开发过程记录见 [`PROJECT_LOG.md`](PROJECT_LOG.md)** —— 2026-10-19 ~ 10-20 的改动时间线、数据统计与踩坑记录。
> 引擎：Godot 4.7.2（`C:\Users\17833\Downloads\Godot_v4.7.2-stable_win64.exe`）
> 项目：`C:\Users\17833\Documents\卡牌`

---

## 1. 项目概览

| 项 | 值 |
|---|---|
| 引擎版本 | Godot 4.7.2 stable |
| 渲染方式 | Mobile（`renderer/rendering_method="mobile"`，Windows 驱动 D3D12） |
| 主场景 | `res://scenes/TitleScreen.tscn`（2026-10-20 起改为标题页；战斗场景是 `Battle.tscn`） |
| 全局自动加载 | `GameState`（`scripts/autoload/game_state.gd`） |
| 窗口拉伸 | `canvas_items` + `expand` |
| 卡池 | 11 张（2 基础 + 8 奖励 + 1 角色专属），全部带像素风卡图 |
| 关卡 | 5 层：史莱姆 → 骷髅兵 → 石像鬼 → 狂战士 → 恶魔领主，带属性倍率（见 §13） |
| 角色 | 绯奈（粉发少女，专属卡「双倍蓄力」，见 §17） |

游戏形态：类《杀戮尖塔》的单敌人卡牌战斗原型。已实现抽牌、出牌、能量、护盾、敌人意图、弃牌重洗、胜利奖励选卡，以及力量 / 易伤状态、多段攻击、卡牌抽牌效果。

---

## 2. 文件职责

```
卡牌/
├── project.godot                     # 项目配置（主场景、autoload、渲染设置）
├── icon.svg                          # 项目图标
├── ARCHITECTURE.md                   # 本文档
├── scenes/
│   ├── TitleScreen.tscn              # 标题页：选角色 + 开始/继续（主场景，§18）
│   ├── Battle.tscn                   # 战斗场景（根节点 Control + battle.gd）
│   └── card.tscn                     # 单张卡牌 UI 模板（牌面图 + 悬停凸起）
├── assets/
│   ├── cards/                        # 11 张像素风卡图（1152×864，rage 裁白边后 641×864）
│   │   ├── strike.jpg / defend.jpg / quick_slash.jpg / twin_strike.jpg / bash.jpg
│   │   ├── cleave.jpg / iron_wall.jpg / meditation.jpg / second_wind.jpg / rage.jpg
│   │   ├── double_strike.jpg         # 角色专属卡「双倍蓄力」
│   │   └── frames/                   # 牌面边框背景（§19.9）
│   │       ├── frame_common.png      # 通用：青蓝石框
│   │       ├── frame_common_border.png   # 通用边框层（中间透明，叠最上层）
│   │       ├── frame_special.png     # 专属：白金粉框
│   │       └── frame_special_border.png  # 专属边框层
│   ├── characters/                   # 立绘：怪物 6 张不透明 JPG；绯奈是透明 PNG + 逐帧帧目录
│   │   ├── player.jpg / slime.jpg / skeleton.jpg / gargoyle.jpg / berserker.jpg / demon_lord.jpg
│   │   ├── feina_battle.png / feina_portrait.png   # 绯奈：战斗立绘 + 主立绘（透明底）
│   │   └── feina/                    # 绯奈逐帧动画：idle / attack / hit / death_00~03 + cast_00~02 + buff_00~03 + walk_00~07，另有 head.png（状态栏头像）。walk 8 帧由 tools/make_walk_v2.py 基于 idle_00 程序生成（逐行整数位移 + 摆臂位移场），服装与待机 100% 一致
│   ├── backgrounds/
│   │   └── battle_bg.jpg             # 战斗/标题页背景（1280×720）
│   ├── ui/                           # UI 用生成图
│   │   ├── status_player_bg.png      # 玩家状态卡背景（粉金主题，480×200）
│   │   └── status_player_border.png  # 玩家状态卡边框层（叠最上层）
├── generated-images/                 # 生图 API 的原始输出目录（可随时删）
├── tools/                            # 美术加工脚本（绿幕抠图 / 切帧 / 拼检查图，见 §14.3）
│   ├── chroma_key.py                 # 绿幕 → 透明 PNG + 统一画布
│   ├── split_frames.py               # sprite sheet → 逐帧 PNG（逐帧抠图 + 身体中心对齐）
│   ├── key_glow.py                   # 黑底发光特效 → 按亮度抠成透明 PNG（§14.5）
│   ├── contact_sheet.py              # 多帧拼检查图
│   └── measure_anchor.py             # 量每帧头/身体锚点，挑对齐方式
└── scripts/
    ├── autoload/
    │   └── game_state.gd             # 全局状态：牌组、血量、卡池、角色、存档
    ├── battle/
    │   └── battle.gd                 # 战斗控制器（核心逻辑，约 860 行）
    ├── ui/
    │   ├── card_ui.gd                # 卡牌表现层（CardUI，含悬停凸起）
    │   ├── gem.gd                    # 蓝色立体棱形（能量球 / 卡面费用共用，§19.4.1）
    │   ├── theme_dungeon.gd          # 统一 UI 主题（§19.2）
    │   └── title_screen.gd           # 标题页逻辑（选角色 / 开始 / 继续，§18）
    ├── fx/
    │   ├── shield_fx.gd              # 防御护罩（纯程序化，玩家/怪物共用，§14.5）
    │   └── charge_fx.gd              # 「双倍蓄力」收尾光环 + 光粒（纯程序化，§14.5）
    └── data/
        ├── card_data.gd              # 卡牌数据模型（Resource）
        ├── enemy_data.gd             # 怪物数据模型（§13）
        ├── level_data.gd             # 关卡数据模型（§13）
        ├── character_data.gd         # 角色数据模型（§17）
        ├── cards/                    # 10 个卡牌资源（详见 §11 卡池表）
        ├── enemies/                  # 5 个怪物资源（slime / skeleton / gargoyle / berserker / demon_lord）
        └── levels/                   # 5 个关卡资源（level_01 ~ level_05，含倍率配置）
```

> 2026-10-19：已清理残留的 `scenes/TestCard.tscn`、`scripts/test_card.gd` 与 `Battle.tscn2776916740.tmp`（原 §6 问题 9）。
> 2026-10-19：旧的《鸣潮》占位图 `暂时使用.png` 已删除，全部换成本项目的像素风卡图。

---

## 3. 架构分层

```
┌──────────────────────────────────────────────────┐
│  控制层：battle.gd（Battle 场景根节点）            │
│  战斗流程 / 回合调度 / 结算 / UI 刷新 / 奖励面板    │
└───────────┬──────────────────────┬───────────────┘
            │ 读写                  │ 实例化 + setup()
┌───────────▼──────────┐  ┌────────▼───────────────┐
│  数据层               │  │  表现层                │
│  CardData（Resource） │  │  card.tscn + CardUI    │
│  GameState（Autoload）│  │  点击 → card_clicked    │
└──────────────────────┘  └────────────────────────┘
```

### 3.1 数据层

**`CardData`（`card_data.gd`，extends Resource）** —— 卡牌的静态数据：
- `id / display_name / description / cost`
- `card_type`：攻击(0) / 技能(1) / 能力(2)
- `target_type`：敌方单体 / 敌方全体 / 自身 / 无目标
- 数值：`damage / block`
- 结构：`hits`（攻击段数，默认 1）、`draw`（打出后抽牌数）
- 状态：`vulnerable`（施加给敌人的易伤层数）、`strength`（自己获得的力量）
- `art`：牌面图（像素风卡图）

每张具体卡牌是一个 `.tres` 资源文件（`scripts/data/cards/*.tres`），用 `preload` 引用。

**`GameState`（`game_state.gd`，Autoload）** —— 跨战斗持久化的运行时状态：
- `deck: Array[CardData]` —— 当前牌组
- `player_max_hp / player_hp` —— 玩家血量
- `STARTING_DECK` —— 初始 4 打击 + 4 防御
- `CARD_POOL` —— 8 张**怪物牌**（战利品牌，全部标记 `is_monster_card = true`）。2026-10-22 起奖励不再统一从这里抽，而是从**本层怪物自己的牌库**掉落，这里只剩兜底用途
- `LEVELS: Array[LevelData]` —— 关卡序列，当前 `level_index` 指向正在打的那关
- 方法：`reset_run()`（含重置关卡进度）、`add_card_to_deck()`、`get_random_card_choices()`、`current_level()`、`advance_level()`、`level_progress_text()`

**`EnemyData`（`enemy_data.gd`，extends Resource）** —— 一只怪的属性 + 固定牌库：
- `id / display_name / max_hp`
- **`tier`**（2026-10-22 新增）—— 怪物类型：`0` 普通 / `1` 精英 / `2` 首领，
  决定牌库张数上限（普通 5、精英 7、首领不限），静态方法 `EnemyData.deck_limit_for(tier)`
- **`deck: Array[CardData]`**（2026-10-22 新增）—— 怪物固定牌库：每回合**随机抽 1 张**打出、**牌库不消耗**；
  这套牌同时是本层怪物的**掉落池**。每只怪至少配 1 张 `is_monster_card = true` 的牌
- `attack_damage / heavy_damage / defend_block / intent_pattern` —— **仅当 `deck` 为空时**作为旧机制兜底
- `enrage_threshold / enrage_bonus`：狂暴阈值与加伤
- `art`：立绘

> 记录表规范见 [`截至10.22记录.md`](截至10.22记录.md) 的「卡牌记录表格规范」一节：
> 怪物表的「牌库」列含基础卡，「怪物牌库」工作表是总池不含基础卡，张数上限按 `tier` 走。
> 用 `python tools/update_card_tables.py` 从 `.tres` 同步两份表。

**`LevelData`（`level_data.gd`，extends Resource）** —— 一层怎么上怪：
- `id / display_name`
- `enemy: EnemyData` —— 这层出现的怪物
- `hp_multiplier / damage_multiplier / block_multiplier` —— 三种属性增幅倍率，最终值 = 怪物裸属性 × 倍率（四舍五入）

### 3.2 表现层

**`CardUI`（`card_ui.gd`，extends Control）** —— 一张卡牌的显示与交互：
- 子节点挂在 `Visual` 视觉层下：`Art` / `CostLabel` / `NameLabel` / `DescLabel`
- `setup(data)` 从 CardData 填数据（含 `art` 牌面图，为空则隐藏图但保留区域）
- `_gui_input()` 检测鼠标左键 → `card_clicked.emit(self)`
- `_process()` 处理悬停凸起：`Visual` 上移 90px + 提升 `z_index`，鼠标离开「卡槽 ∪ 凸起后卡牌」才落回（详见 §5.1）

**立绘（`PlayerSprite` / `EnemySprite`，TextureRect）** —— 战斗演出层：
- 玩家立绘固定（`PLAYER_ART` 常量，还没有角色系统）；怪物立绘来自 `EnemyData.art`
- 四套代码动画：待机呼吸浮动、攻击前冲、受击闪白 + 抖动、死亡淡出（详见 §14）

### 3.3 控制层

**`battle.gd`（extends Control）** —— 整个战斗的调度中心：
- 状态：`draw_pile / discard_pile / hand`、玩家/敌人血量护盾、能量、`battle_over`
- 状态效果：`player_strength`（力量）、`enemy_vulnerable`（易伤层数）
- **关卡**：`start_battle()` 从 `GameState.current_level()` 读 `LevelData`，把怪物裸属性 × 倍率算成本关实际数值
- **敌人牌库（2026-10-22）**：`enemy_deck = enemy_data.deck`；`decide_enemy_intent()` 每回合从牌库**随机抽 1 张**作为下一张要打的牌（`enemy_next_card`），`enemy_card_damage() / enemy_card_block()` 负责乘关卡倍率，`_play_enemy_card()` 结算（只处理 `damage/hits/block`）。牌库为空时退回旧的 `intent_pattern` 循环兜底
- 敌人伤害：`enemy_attack()` 统一结算（狂暴加成、玩家护盾抵消、飘字）
- **奖励（2026-10-22）**：`show_reward_panel()` 从本层怪物牌库随机掉 **2 张**（保底 1 张怪物牌），**两张都进局内背包**
- 流程：`start_battle()` → `start_player_turn()` → 玩家出牌 → `end_player_turn()` → `enemy_turn()` → 循环
- 结算：`apply_card_effect()` 按 `card_type` 分派 → `deal_attack()` 处理多段攻击
- 伤害公式：`calc_player_attack_damage()` = (基础 + 力量) × (易伤时 1.5)，最后 `floor`
- 奖励：`show_reward_panel()` / `_on_reward_card_clicked()` / `close_reward_panel()`（奖励期间隐藏手牌）

---

## 4. 核心数据流

> ⚠️ **本节写于 2026-10-19，部分流程已变**：伤害改成百分比制、护盾改成持续 3 回合没回合清零、
> 敌人改成固定牌库出牌 + 能量系统、奖励改成掉 2 张。现行规则以 §23 / §24 为准，本节只看**整体骨架**。


```
启动
 └─ GameState._ready() → reset_run()        # 初始牌组 8 张，血量 80
 └─ Battle._ready()
      ├─ end_turn_button.pressed → _on_end_turn_pressed
      ├─ start_battle()
      │    ├─ draw_pile ← GameState.deck.duplicate().shuffle()
      │    ├─ player_hp ← GameState.player_hp
      │    ├─ enemy_hp = 50
      │    ├─ decide_enemy_intent()
      │    └─ start_player_turn()
      │         ├─ player_block = 0
      │         ├─ energy = max_energy
      │         ├─ draw_cards(5) → add_card_to_hand() → CardUI.setup()
      │         └─ update_all_ui()
      └─ skip_button.pressed → _on_skip_pressed

回合循环
 玩家点卡 → CardUI.card_clicked → _on_card_clicked()
      ├─ 能量检查（不足则能量文字闪红提示）→ 扣能量
      ├─ apply_card_effect()
      │    ├─ TYPE_ATTACK：deal_attack() 按 hits 段数循环结算
      每段伤害 = 攻击力 × percent_attack → 蓄势×2 → 会心 → 易伤 → 防御减免 → 护盾
      ⚠️ 旧版这里是「(基础+力量)×(易伤1.5)」，10-22 已换百分比制（见 §23.1 / §11.9）
      │    │     卡面带易伤字段的，打完再挂易伤；敌死则 battle_over + show_reward_panel()
      │    ├─ TYPE_SKILL：玩家加护盾 / 给敌人挂易伤
      │    └─ TYPE_POWER：给自己加力量（本场永久）
      ├─ 卡牌带 draw 字段则抽牌
      └─ 卡牌进弃牌堆 → queue_free()

结束回合 → end_player_turn()　（玩家护盾同样按 3 回合计时递减）
      ├─ 全部手牌进弃牌堆
      └─ enemy_turn()
           ├─ 敌人护盾计时 -1（护盾持续 3 回合，重复获得刷新；见 §23.6）
           ├─ 有 enemy_next_card → _play_enemy_card()（只结算 damage/hits/block，乘关卡倍率，
           │    伤害再经 enemy_attack 加狂暴加成）
           │  牌库为空 → 退回旧意图（攻击 6 / 防御 +6 护盾 / 重击 10）
           ├─ 玩家死亡？→ battle_over
           ├─ 敌人身上的易伤层数 -1
           ├─ decide_enemy_intent()   # 从怪物牌库随机抽下一张（见 §12）
           └─ start_player_turn()

胜利 → show_reward_panel() → 隐藏手牌 + 从本层怪物牌库随机掉 2 张
      │    保底规则：至少 1 张是 is_monster_card（避免全掉打击/防御）
      │    非最后一层：标题提示「下一层：XXX」
      │    最后一层：标题改为「通关！」，按钮变「收下并重新开始」
      └─ 点「收下」或点任意一张 → 2 张都进 GameState.run_loot → close_reward_panel()
           ├─ 不是最后一层 → GameState.advance_level() → start_battle() 打下一层
           └─ 是最后一层 → GameState.reset_run() → start_battle() 从第 1 层重开
```

---

## 5. 场景节点树

### Battle.tscn（✅ 2026-10-19 修正后坐标）
```
Battle (Control, script=battle.gd)
├── PlayerSprite (TextureRect)   # (100,230)-(360,490)  玩家立绘
├── EnemySprite (TextureRect)    # (790,240)-(1050,500) 怪物立绘
├── EnergyLabel (Label)          # (20,20)-(140,43)
├── EndTurnButton (Button)       # (20,120)-(140,155)  min 120×35
├── Hand (HBoxContainer)         # 底部 y=500，居中
├── PlayerHPLabel (Label)        # (20,50)-(180,73)
├── PlayerBlockLabel (Label)     # (20,80)-(140,103)
├── PlayerStatusLabel (Label)    # (20,160)-(160,183) 力量，无状态时隐藏
├── LevelLabel (Label)           # (900,90)-(1060,113) 关卡进度「第 N / M 层」
├── EnemyHPLabel (Label)         # (900,120)-(1060,143) 显示怪物名 + 血量
├── EnemyBlockLabel (Label)      # (900,150)-(1060,173)
├── EnemyIntentLabel (Label)     # (900,180)-(1018,203)
├── EnemyStatusLabel (Label)     # (900,206)-(1060,229) 易伤，无状态时隐藏
└── RewardPanel (Panel, visible=false)
    ├── TitleLabel
    ├── CardContainer (HBoxContainer)
    └── SkipButton (120×40)
```

### card.tscn（✅ 2026-10-19 二次调整：牌面图区 + 悬停凸起）
```
card (Control, min_size 160×220, script=card_ui.gd)   # 根的矩形是"卡槽"，永不移动
└── Visual (Control 160×220, mouse_filter=IGNORE)     # 视觉层，随悬停整体上抬
    ├── Background (Panel 160×220, mouse_filter=IGNORE)
    ├── Art (TextureRect)        # (8,42)-(152,148) 牌面图，KEEP_ASPECT_COVERED
    ├── CostLabel (Label)        # 左上角 (8,6)-(36,32)
    ├── NameLabel (Label)        # (44,8)-(152,31)
    └── DescLabel (Label)        # (8,154)-(152,214)，自动换行
```

### 5.1 卡面布局与悬停交互（2026-10-19 新增）

**卡面分区**（160×220）：顶部 40px 放费用 + 名称，中部 `Art` 区 106px 放牌面图（`CardData.art`，每张卡用自己的 `assets/cards/<id>.jpg`；无图时留空但**空间照样保留**），底部 60px 放描述。

**悬停凸起**：鼠标进入卡牌 → `Visual` 层在 0.12s 内上移 `HOVER_LIFT = 90px` 并 `z_index = 10` 盖住左右邻牌；移出后平滑落回。

**关键设计（避免抖动）**：凸起后卡牌离开了鼠标位置，若用 `mouse_exited` 直接回落，会「抬起→判定离开→落回→再进入」无限抖动。因此：

1. 悬停判定在 `_process` 里做，用 **「卡槽矩形 ∪ 抬起后的卡牌矩形」** 的并集判断，两个区域任意一个含鼠标就保持凸起；
2. 记录进入时的 `_rest_position` / `_slot_rect`，抬起期间每帧用 `position = _rest_position + (0, _lift)` 维持位置，防止 `HBoxContainer` 重排覆盖；
3. `_lift` 用指数插值平滑，未悬停且已归零时把位置完全交还容器（不再干预）。

**未悬停的牌被窗口下沿裁切属预期**（视口 648 高、卡槽 y=500、卡高 220），只要求露出名称，凸起时完整可见即可。

---

## 6. 问题清单与修复记录（✅ 2026-10-19 全部已修复）

### ✅ 1. 主场景指向错误（已修复）

`project.godot` 原为：
```ini
run/main_scene="uid://d0taq3qno4cno"
```
但 `uid://d0taq3qno4cno` 是 **`card.tscn`** 的 uid，而战斗场景 `Battle.tscn` 的 uid 是 `uid://qjawxpfkd608`。按 F5 会加载一张孤立的卡牌。

**已修复**：改为 `run/main_scene="res://scenes/Battle.tscn"`（直接用路径，避免 uid 漂移）。

### ✅ 2. 玩家护盾标签与能量标签重叠（已修复）

原来 `EnergyLabel` 在 `(20,20)-(81,43)`，`PlayerBlockLabel` 在 `(0,0)-(40,23)` 且带 `custom_maximum_size = Vector2(20,150)`，两者在左上角重叠。

**已修复**：删掉 `custom_maximum_size`，左侧信息栏重排为竖列 —— 能量 `(20,20)` / HP `(20,50)` / 护盾 `(20,80)`。

### ✅ 3. 结束回合按钮尺寸只有 8×8（已修复）

原 `(20,60)-(28,68)`，宽高各 8px，靠文字最小尺寸强行撑开。

**已修复**：`(20,120)-(140,155)` + `custom_minimum_size = Vector2(120, 35)`。

### ✅ 4. 敌人意图标签零尺寸（已修复）

原 `offset_right=900`（= left）、`offset_bottom=180`（= top），尺寸 0×0。

**已修复**：`offset_right=1018, offset_bottom=203`。

### ✅ 5. 卡牌费用标签锚点异常（已修复）

`CostLabel` 原为 `layout_mode=1` + `anchors_preset=15`，但偏移 `(76,106)-(-64,-85)`，数字落在卡牌下半部中间。

**已修复**：改 `layout_mode=0` 固定 `(8,6)-(36,32)`，费用回到左上角。

### ✅ 6. 卡牌名称只留 40px 宽（已修复）

`NameLabel` 原宽 40px，长名字会被裁切。

**已修复**：`(44,8)-(152,31)`，与费用同排、宽度 108px；同时 `DescLabel` 重排为 `(8,44)-(152,200)` 自动换行，占满卡面下半部。

### ✅ 7. 能量不足无视觉反馈（已修复）

`battle.gd` 原来能量不足只 `print("能量不足")`，玩家点击感觉"没反应"。

**已修复**：新增 `flash_energy_insufficient()` —— 能量文字显示"能量不足！"并变红，0.5 秒后由 `_restore_energy_label()` 恢复；用 `Tween` 持有者避免连点时动画叠加。

### ✅ 8. 文本标点中英文混用（已修复）

场景用中文冒号、脚本拼英文冒号，混用。

**已修复**：场景 `text` 与 `battle.gd` 的 `update_all_ui()` / `update_energy_label()` 全部统一为**中文冒号**（`能量：3 / 3`、`玩家 HP：80 / 80`、`护盾：0`、`意图：攻击 6`）。

### ✅ 9. 残留文件（已清理）

- `scenes/Battle.tscn2776916740.tmp` —— 已删
- `scenes/TestCard.tscn` + `scripts/test_card.gd`（含 `.uid`）—— 已删，全项目无引用
- `battle.gd` 里的 `print()` 调试输出暂保留（B 方案阶段换成屏幕战斗日志）

### ✅ 10. 牌池过小（已修复）

原来 `GameState.CARD_POOL` 只有 `strike` 和 `defend` 两张，奖励名义抽 3 张实际最多 2 张。

**已修复**：卡池扩到 10 张（2 基础 + 8 奖励），奖励池塞满 8 张新卡，3 选 1 真实随机。详见 §11。

### ✅ 11. 奖励面板被手牌遮挡（已修复，2026-10-19 第二轮发现）

奖励面板 `(240,160)-(1040,560)`，手牌在 y=500 起，卡牌叠在面板上方，把「跳过」按钮盖住了。

**已修复**：`show_reward_panel()` 里 `hand_container.visible = false`，`close_reward_panel()` 里恢复。战斗结束（`battle_over`）时点手牌本来就被拦截，逻辑无副作用。

### ✅ 12. 「跳过」按钮过小（已修复，2026-10-19 第二轮发现）

原 `(350,340)-(390,371)`，只有 40×31，在 800 宽的奖励面板里几乎点不到。

**已修复**：改为 `(340,340)-(460,380)` + `custom_minimum_size = 120×40`，居中且好点。

---

## 7. A 方案：UI 修复实施清单（✅ 已全部执行完毕 2026-10-19）

| # | 文件 | 改动 | 状态 |
|---|---|---|---|
| A1 | `project.godot` | `run/main_scene` → `res://scenes/Battle.tscn` | ✅ |
| A2 | `scenes/Battle.tscn` | `PlayerBlockLabel` → `(20,80)`，删 `custom_maximum_size` | ✅ |
| A3 | `scenes/Battle.tscn` | `EndTurnButton` → `(20,120)-(140,155)` + min 120×35 | ✅ |
| A4 | `scenes/Battle.tscn` | `EnemyIntentLabel` → `(900,180)-(1018,203)` | ✅ |
| A5 | `scenes/Battle.tscn` | `PlayerHPLabel` → `(20,50)`，避开按钮 | ✅ |
| A6 | `scenes/card.tscn` | `CostLabel` 改 `layout_mode=0` → `(8,6)-(36,32)` | ✅ |
| A7 | `scenes/card.tscn` | `NameLabel` → `(44,8)-(152,31)` | ✅ |
| A8 | `scenes/card.tscn` | `Background` `offset_bottom` 对齐 220 | ✅ |
| A9 | `scripts/battle.gd` | 能量不足闪红提示 + `_restore_energy_label` | ✅ |
| A10 | 项目文件 | 删 `.tmp`、`TestCard` 相关残留 | ✅ |

**验证**：`--headless --quit` 与 `--quit-after 120` 均退出码 0，无脚本解析/运行时错误（仅 `user://` 日志写入在受限沙箱下报警，与项目无关）。布局经静态核验：左侧竖列 `20/50/80/120` 四段不重叠，右侧 `120/150/180` 不重叠，卡面费用/名称/描述三块不重叠。

> 若坐标与实际字体渲染有出入，以编辑器内 F5 预览为准再微调。

---

## 8. 验证方式

**编辑器内**：改完 `Battle.tscn` 后按 F5，检查是否进入战斗场景、标签不重叠、按钮文字完整。

**命令行（我可以直接跑）**：
```powershell
& "C:\Users\17833\Downloads\Godot_v4.7.2-stable_win64_console.exe" --headless --path "C:\Users\17833\Documents\卡牌" --quit
```
能加载无解析错误即算通过。注意：`user://` 日志写入在受限沙箱下会报警，不影响项目本身。

---

## 9. 后续路线（B 方案进度）

> 图例：✅ 已完成 / 🟨 部分完成 / ⬜ 未开始

| # | 内容 | 状态 |
|---|---|---|
| 1 | **战斗系统补完**：多敌人、手牌上限、其余状态（中毒/虚弱/护甲） | 🟨 力量 / 易伤 / 多段 / 抽牌已做；中毒、虚弱、护甲、多敌人、手牌上限未做 |
| 2 | **卡池扩充**：至少 10 张可玩卡牌，奖励有真实随机性 | ✅ 2026-10-19 完成（10 张，见 §11） |
| 3 | **敌人 AI**：意图从纯随机改成"轮转 + 条件判断" | ✅ 2026-10-19 完成（见 §12） |
| 4 | **表现层**：卡牌美术、受击闪白、伤害飘字、死亡动画 | ✅ 2026-10-19 完成（§11.3、§14） |
| 5 | **流程**：战斗→奖励→下一场→Boss，加简单存档 | ✅ 5 关流程 + 自动存档已完成（§13、§15） |
| 6 | **调试面板**：把 `print` 换成屏幕战斗日志 | ✅ 2026-10-20 完成（§16） |

---

## 10. C 方案：角色 / 局外背包 / 剧情扩展设计

### 10.0 设计总原则

1. **默认零影响**：所有新系统都是**可选的运行时参数**，默认关闭或默认值 = 现有行为。不做任何东西，现在的战斗原型照常完整可玩。
2. **数据驱动**：角色、技能、天赋、卡牌数量、剧情内容全部是数据文件（`.tres` 或 `.json`），改内容不用改代码。
3. **不侵入 `battle.gd`**：战斗逻辑只在一个入口读取参数（`RunConfig`），其余原样。
4. **剧情完全独立**：单独的 `scripts/story/` + 独立场景 + 独立数据文件，不 import 任何战斗代码，通过信号/回调与主流程交互。

### 10.1 参数化入口：`RunConfig`

新增 `scripts/game/run_config.gd`（RefCounted 或 Resource），描述“一局游戏”的配置：

```gdscript
class_name RunConfig

var character: CharacterData          # null → 用默认战士
var custom_deck: Array[CardData]      # 空 → 用 STARTING_DECK
var use_collection_deck: bool = false # 背包带卡开关（默认 false）
var talent_points: int = 0            # 天赋点
var story_enabled: bool = false       # 剧情系统开关
var story_id: String = ""             # 入口剧情 id
```

`GameState` 增加一个字段：

```gdscript
var current_run: RunConfig = null     # null = 纯原版流程
```

`battle.gd` 的 `start_battle()` 里**只加一段**：

```gdscript
var deck_source := GameState.deck
if GameState.current_run and not GameState.current_run.custom_deck.is_empty():
    deck_source = GameState.current_run.custom_deck
draw_pile = deck_source.duplicate()
```

`current_run == null` 时行为与现在**完全一致**——这就是“仅添加参数，不参与其他构建”。

### 10.2 角色系统

新增三个数据类：

**`scripts/data/character_data.gd`（extends Resource）**

```gdscript
class_name CharacterData

@export var id: String = ""
@export var display_name: String = ""
@export_multiline var description: String = ""
@export var max_hp: int = 80
@export var portrait: Texture2D
@export var starter_deck: Array[CardData] = []   # 该角色初始牌组
@export var skills: Array[SkillData] = []        # 技能列表
@export var talent_tree: Array[TalentData] = []  # 天赋树节点
```

**`scripts/data/skill_data.gd`（extends Resource）**

```gdscript
class_name SkillData

@export var id: String = ""
@export var display_name: String = ""
@export_multiline var description: String = ""
@export var energy_cost: int = 0      # 0 = 被动
@export var cooldown: int = 0         # 回合冷却
@export_enum("被动", "主动") var kind: int = 0
# 效果用“效果类型 + 参数”描述，避免为每个技能写一个类
@export_enum("加护盾", "加伤害", "加能量", "抽牌", "回血") var effect: int = 0
@export var amount: int = 0
```

**`scripts/data/talent_data.gd`（extends Resource）**

```gdscript
class_name TalentData

@export var id: String = ""
@export var display_name: String = ""
@export_multiline var description: String = ""
@export var max_rank: int = 1        # 可点几级
@export var cost_per_rank: int = 1   # 每级消耗点数
@export var requires: Array[String] = []  # 前置天赋 id
@export var effect: int = 0          # 同 SkillData 的效果枚举
@export var amount: int = 0
```

**角色内容即 `.tres` 文件**：`scripts/data/characters/warrior.tres` 等。加新角色 = 加一个 `CharacterData` 资源 + 初始牌组/技能/天赋资源，不写代码。

**天赋树 UI（独立场景）**：`scenes/talent_tree.tscn` —— 节点画布，点节点加点，满足 `requires` 才可点。
> 天赋树展示形式（纯按钮列表 / 连线图）建议先做**列表式**（简单可靠），以后换图形化。

### 10.3 局外背包

**卡牌收藏数据**：新增 `scripts/data/collection.gd`（extends Resource 或 RefCounted）

```gdscript
class_name CardCollection

# card_id → 持有数量
var owned: Dictionary[String, int] = {}

func add(card_id: String, n: int = 1) -> void
func take(card_id: String, n: int = 1) -> void
func count(card_id: String) -> int
func get_owned_cards() -> Array[CardData]   # 通过 id 查回 CardData
```

> 前提：每张 `.tres` 必须有唯一 `id`（`strike.tres`/`defend.tres` 已有）。需要一个“卡牌注册表”把 `id → CardData` 映射起来，建议加 `GameState.CARD_REGISTRY: Dictionary[String, CardData]`，在 `_ready()` 里注册所有已知卡牌。

**初始拥有**：默认 4 张打击 + 4 张防御（与 `STARTING_DECK` 一致），保证“不碰背包也能玩”。

**背包 UI（独立场景）**：`scenes/collection.tscn`
- 左侧：持有卡牌列表（**显示数量**），按类型分组
- 右侧：本次出战牌组（容量上限，比如 10 张）
- 交互：点击左→加到出战牌组（数量 -1 显示），点击右→拿回
- 确认 → 写入 `RunConfig.custom_deck` + `RunConfig.use_collection_deck = true`

**“带进去”的落地**：进战斗前若 `use_collection_deck == true`，`battle.gd` 用 `custom_deck`；否则用 `STARTING_DECK`。默认 false，不碰背包时零影响。

> **数量不进入局内**（重要约定）：“持有数量”只是局外收藏层的记账字段。带卡时按数量展开成牌组数组（带 3 张打击 = `custom_deck` 里 3 个 `strike` 引用），局内 `draw_pile / discard_pile / hand` 仍然只是 `Array[CardData]`，**不感知、不新增任何数量字段或数量相关逻辑**。数量系统与战斗逻辑是两条平行线，只在“进战斗前生成牌组”这一个点交汇。

### 10.4 剧情系统（完全独立）

```
scripts/story/
├── story_data.gd      # 剧情数据模型（Resource）
├── story_engine.gd    # 剧情播放器（纯逻辑，可在任意场景挂载）
└── stories/           # 剧情内容（数据文件，手改方便）
    └── intro.json     # 推荐 JSON：非程序员也能改
scenes/story.tscn      # 独立剧情场景（UI）
```

**数据格式（JSON，方便外部编辑）**：

```json
{
  "id": "intro",
  "title": "序章",
  "start": "n0",
  "nodes": {
    "n0": {
      "speaker": "旁白",
      "text": "你醒了过来……",
      "next": "n1"
    },
    "n1": {
      "speaker": "神秘人",
      "text": "你也要来打牌吗？",
      "next": "n2"
    },
    "n2": {
      "speaker": "你",
      "text": "……",
      "choices": [
        { "text": "接受挑战", "next": "n3" },
        { "text": "先看看再说", "next": "n4" }
      ]
    },
    "n3": { "text": "战斗开始。", "next": "", "action": "start_battle" },
    "n4": { "text": "你后退了一步。", "next": "" }
  }
}
```

- `nodes` 是节点表，`next` 跳转，`choices` 分支，`action` 触发事件（如进战斗）
- **接入点**：`story_engine.gd` 发出 `story_finished` 信号；主流程可监听，`action` 通过 `RunConfig.story_enabled` 决定是否响应
- **完全独立**：story 模块不 `preload` 任何 `battle.gd` 的东西，只发信号。改剧情 = 改 JSON，零代码风险

**剧情触发时机（预留）**：开局前、战斗胜利后、Boss 前。当前阶段只做“开局前可进入”，其余留接口。

### 10.5 新增文件总览

```
scripts/
├── game/
│   └── run_config.gd               # 一局游戏的配置（新）
├── data/
│   ├── character_data.gd           # 角色（新）
│   ├── skill_data.gd               # 技能（新）
│   ├── talent_data.gd              # 天赋（新）
│   ├── collection.gd               # 卡牌收藏（新）
│   └── characters/warrior.tres     # 角色内容（新）
└── story/
    ├── story_data.gd               # 剧情模型（新）
    ├── story_engine.gd             # 剧情播放器（新）
    └── stories/intro.json          # 剧情内容（新）
scenes/
├── collection.tscn                 # 局外背包（新）
├── talent_tree.tscn                # 天赋树（新）
└── story.tscn                      # 剧情场景（新）
```

**对现有文件的改动（全部是“加”不是“改”）**：

| 文件 | 改动 |
|---|---|
| `game_state.gd` | 加 `current_run`、`collection`、`CARD_REGISTRY` 三个字段 + 注册卡牌 |
| `battle.gd` | `start_battle()` 加 3 行读取 RunConfig 的牌组覆盖 |
| `project.godot` | 新增 autoload（如有）、新增场景注册 |
| 其余 | 不动 |

### 10.6 阶段计划

> **当前状态：C 方案整体尚未开始，只有 P0（A 方案 UI 修复）真正完成了。**
> 下表第三列是"是否侵入战斗逻辑"的设计约束，**不是完成度**。

| 阶段 | 内容 | 是否侵入战斗逻辑 |
|---|---|---|
| P0 | A 方案（§7）UI 修复 | 否 —— ✅ 已完成 |
| P1 | `RunConfig` + 四个数据类 + 卡牌注册表 | 否（纯数据） |
| P2 | 角色资源（warrior）+ 天赋树数据 + 天赋树 UI | 否 |
| P3 | 局外背包（收藏 + 带卡 UI） | 否（只写 `custom_deck`） |
| P4 | 剧情系统（JSON + 播放器 + 场景） | 否（只发信号） |
| P5 | 接入：`battle.gd` 读 `RunConfig`、开局前挂剧情、角色血量 | 是（最小改动） |

**每阶段都可独立验收**，P5 之前现有战斗原型始终完整可玩。

---

## 11. 卡池与美术管线（2026-10-19 新增）

### 11.1 十张卡牌

| id | 名称 | 类型 | 费 | 效果 | 新机制 |
|---|---|---|---|---|---|
| `strike` | 打击 | 攻击 | 1 | 造成 6 伤害 | 基线 |
| `defend` | 防御 | 技能 | 1 | 获得 5 护盾 | 基线 |
| `quick_slash` | 快斩 | 攻击 | 1 | 4 伤害，抽 1 张 | `draw` |
| `twin_strike` | 双刃 | 攻击 | 1 | 4 伤害 ×2 | `hits` |
| `bash` | 重击 | 攻击 | 2 | 8 伤害，2 层易伤 | `vulnerable` |
| `cleave` | 横扫 | 攻击 | 1 | 6 伤害（全体目标） | `target_type` |
| `iron_wall` | 铁壁 | 技能 | 1 | 获得 8 护盾 | — |
| `meditation` | 冥想 | 技能 | 0 | 抽 2 张 | `draw` + 0 费 |
| `second_wind` | 喘息 | 技能 | 1 | 5 护盾，抽 1 张 | 复合效果 |
| `rage` | 狂怒 | 能力 | 1 | 获得 2 力量 | `strength` |

初始牌组仍是 4 打击 + 4 防御；奖励池只放 8 张新卡。

**演出现状（2026-10-21）**：护盾牌（防御 / 铁壁 / 喘息）出统一护罩、不摆施法动作；冥想、狂怒播通用施法动作 + 金色光环；专属卡「双倍蓄力」有自己的举手动作 + 粉色光晕（§14.5）。攻击牌统一播 `attack` 帧。

### 11.2 状态效果规则

- **力量（strength）**：玩家身上，每点让攻击伤害 +1，**本场战斗永久**，战斗开始时清零。
- **易伤（vulnerable）**：挂在敌人身上，受到的攻击伤害 **×1.5**（`floor`），每个敌人回合结束 **-1** 层。
- **结算顺序**：伤害 = `(基础 + 力量) × 易伤倍率` → 先扣护盾 → 再扣血；多段攻击**逐段**计算（护盾逐段消耗）。
- 攻击牌自带的易伤在**打完之后**才挂上，当回合不享受自己的加成（避免自我连锁）。

数值已用临时脚本实测：力量 +2 后双刃每段 6（共 12）、重击 10 伤 + 2 易伤、随后打击 (6+2)×1.5=12、带 5 护盾时只吃 7 点、冥想抽牌 +2 —— 六项全部符合预期。

### 11.3 像素风卡图管线

1. 用火山方舟（Doubao Seedream）文生图，统一提示词骨架：`Pixel art illustration for a fantasy trading card, 16-bit retro RPG style. … Dark dungeon background with teal and amber accents, dramatic rim lighting. no text, no letters, no borders, …`
2. **尺寸要求**：API 最小 921600 像素（512×512 会被拒），采用 **1152×864**，正好贴合卡图区 144×106 的 1.36:1 比例。
3. 原始输出落在 `generated-images/`，用 Pillow 脚本检测并裁掉模型偶发加上的纯白边，再归位到 `assets/cards/<id>.jpg`。
4. `.tres` 里通过 `art = ExtResource("2_art")` 引用，`card_ui.setup()` 直接铺到 `Art`（`KEEP_ASPECT_COVERED`，填满不拉伸）。
5. 换美术资源只改 `.tres`，代码不动。

> 已知小瑕疵：`rage` 生成时被模型加了左右白边，已裁切（641×864），竖构图在横卡面里会裁掉上下部分，只剩拳头主体，可接受；想更完美可重生成一张横构图。

---

## 12. 敌人 AI 规则（2026-10-19 新增）

### 12.1 招式表

| 招式 | 效果 |
|---|---|
| 攻击 | 6 点伤害 |
| 防御 | 自己 +6 护盾 |
| 重击 | 10 点伤害 |

### 12.2 轮转逻辑

- **基础循环**：`攻击 → 防御 → 重击 →` 循环
- **狂暴**：敌人血量 ≤ 30%（50 血即 ≤15）时切换为 `重击 → 攻击 → 重击 →` 循环，且所有攻击伤害 **+2**
- 每场战斗的循环起点随机（`randi() % 3`），不是每局都从攻击开场
- 意图预告的数值就是实际伤害，狂暴加成已经算进显示里（**2026-10-23 起意图只报类型，不再显示数值**，见 §25.3）

### 12.3 关键实现

- `decide_enemy_intent()`：`pattern[enemy_intent_index % pattern.size()]`，**替换了原来的 `randi() % 2`**
- `enemy_attack(base_damage)`：狂暴加成 → 扣玩家护盾 → 扣血，玩家血量同步回 `GameState`
- `is_enemy_enraged()`：`enemy_hp / enemy_max_hp <= 0.3`
- `_intent_text()` 与结算共用 `enemy_attack_value()`，避免预告和实际打出的数字对不上（2026-10-23 起改为只报类型）
- 敌人状态标签同时显示「易伤 N」和「狂暴」（`EnemyStatusLabel`，无状态时隐藏）

### 12.4 验证结果

临时脚本实测 10 项全过：基础循环 `[0,1,2]`、狂暴阈值（15 狂暴 / 16 不狂暴）、狂暴循环 `[2,0,2]`、狂暴普攻 8、狂暴重击 12、5 护盾吃重击掉 7、意图文案与结算完全一致。

---

## 13. 关卡与怪物配置指南（2026-10-19 新增）

> 全部数据驱动：加怪物 / 加关卡 / 调难度都**不改代码**，只改 `.tres` 和一行列表。

### 13.1 当前关卡表

| 关 | 关卡资源 | 怪物 | HP | 攻击 | 重击 | 防御 | 倍率（血/伤/防） |
|---|---|---|---|---|---|---|---|
| 1 | `level_01` | 史莱姆 | 30 | 5 | 8 | 4 | 1.0 / 1.0 / 1.0 |
| 2 | `level_02` | 骷髅兵 | 45 | 6 | 10 | 6 | 1.0 / 1.0 / 1.0 |
| 3 | `level_03` | 石像鬼 | 60 → 69 | 5 → 6 | 9 → 10 | 10 → 12 | 1.15 / 1.1 / 1.2 |
| 4 | `level_04` | 狂战士 | 55 → 66 | 9 → 10 | 14 → 16 | 3 | 1.2 / 1.15 / 1.0 |
| 5 | `level_05` | 恶魔领主（BOSS） | 80 → 100 | 10 → 12 | 16 → 19 | 8 → 9 | 1.25 / 1.2 / 1.15 |

### 13.2 怪物可调属性

文件：`scripts/data/enemies/*.tres`（`EnemyData`）

| 字段 | 含义 | 示例 |
|---|---|---|
| `display_name` | 战斗界面显示的名字 | `"史莱姆"` |
| `max_hp` | 基础血量 | 30 |
| `attack_damage` | 普通攻击伤害 | 5 |
| `heavy_damage` | 重击伤害 | 8 |
| `defend_block` | 防御时获得的护盾 | 4 |
| `intent_pattern` | 招式循环，`0`=攻击 `1`=防御 `2`=重击 | `[0,1,2]` |
| `enrage_threshold` | 狂暴阈值（血量比例） | 0.3 |
| `enrage_bonus` | 狂暴后攻击加伤 | 2 |

**招式循环怎么配**（这是塑造怪物性格的关键）：

| 写法 | 效果 | 对应怪物 |
|---|---|---|
| `[0,1,2]` | 攻击 → 防御 → 重击 | 史莱姆 |
| `[0,2,1]` | 攻击 → 重击 → 防御 | 骷髅兵 |
| `[1,1,0,2]` | 防 → 防 → 攻 → 重 | 石像鬼（坦克） |
| `[0,2,0]` | 攻 → 重 → 攻 | 狂战士（暴力） |
| `[0,0,1,2]` | 攻 → 攻 → 防 → 重 | 恶魔领主 |

数组长度随意，循环会自动取模。

### 13.3 关卡可调属性

文件：`scripts/data/levels/*.tres`（`LevelData`）

| 字段 | 含义 |
|---|---|
| `display_name` | 关卡名，如「第 3 层 · 石像鬼（强化）」 |
| `enemy` | 指向一个怪物 `.tres` |
| `hp_multiplier` | 血量倍率（在怪物裸 HP 上乘） |
| `damage_multiplier` | 攻击 / 重击倍率 |
| `block_multiplier` | 护盾倍率 |
| **`enemy_level`** | **本层怪物等级**（2026-10-23 新增，默认 1） |
| **`enemy_tier`** | **本层怪物阶位**（2026-10-23 新增，默认 0） |

**最终数值 = 怪物基础值 × (1+属性成长)^(等级−1) × 对应倍率，四舍五入取整。**

> 怪物配置里只写「0 阶 1 级」的基础值，等级/阶位由关卡决定。
> 等级和阶位相互独立，约束：**等级 > 10 × (阶位 + 1)**。
> 5 个测试关卡目前都是 Lv.1 · 0 阶。

例：石像鬼 HP 60 × 1.15 = 69，防御 10 × 1.2 = 12。

> 想复用同一只怪做不同难度：建两个 LevelData 都指向同一个 EnemyData，只改倍率就行。

### 13.4 添加一个新怪物（3 步）

1. 在 `scripts/data/enemies/` 新建 `wolf.tres`，照抄 `slime.tres` 改数值：

```ini
[gd_resource type="Resource" script_class="EnemyData" format=3]

[ext_resource type="Script" path="res://scripts/data/enemy_data.gd" id="1_enemy"]

[resource]
script = ExtResource("1_enemy")
id = "wolf"
display_name = "野狼"
max_hp = 40
attack_damage = 7
heavy_damage = 12
defend_block = 2
intent_pattern = [0, 0, 2]
enrage_threshold = 0.4
enrage_bonus = 3
```

2. 保存（编辑器会自动导入；命令行可跑 `--headless --import`）。
3. **不用改任何代码**，这个怪物就能被关卡引用了。

### 13.5 添加一个新关卡（3 步）

1. 在 `scripts/data/levels/` 新建 `level_06.tres`，指向怪物并设倍率：

```ini
[gd_resource type="Resource" script_class="LevelData" load_steps=3 format=3]

[ext_resource type="Script" path="res://scripts/data/level_data.gd" id="1_level"]
[ext_resource type="Resource" path="res://scripts/data/enemies/wolf.tres" id="2_enemy"]

[resource]
script = ExtResource("1_level")
id = "level_06"
display_name = "第 6 层 · 野狼"
enemy = ExtResource("2_enemy")
hp_multiplier = 1.3
damage_multiplier = 1.2
block_multiplier = 1.0
```

2. 打开 `scripts/autoload/game_state.gd`，在 `LEVELS` 数组末尾加一行：

```gdscript
const LEVELS: Array[LevelData] = [
	preload("res://scripts/data/levels/level_01.tres"),
	...
	preload("res://scripts/data/levels/level_05.tres"),
	preload("res://scripts/data/levels/level_06.tres"),   # ← 新增这一行
]
```

3. 保存即可。界面进度会自动变成「第 X / 6 层」，打完第 6 层自动通关。

### 13.6 常见需求速查

| 想做的事 | 怎么做 |
|---|---|
| 让第 3 关更肉 | 改 `level_03.tres` 的 `hp_multiplier` |
| 加一个超强最终 BOSS | 新建怪物 + 关卡，末尾追加到 `LEVELS` |
| 同一只怪越来越强 | 建 N 个 LevelData 指向同一 EnemyData，倍率递增 |
| 让怪物更爱放重击 | 改 `intent_pattern`，如 `[0,2,0,2]` |
| 让怪物更早狂暴 | 调低 `enrage_threshold`（0.5 = 半血就狂暴） |
| 缩短 / 加长整局 | 增删 `LEVELS` 里的条目 |
| 调整某关的难度曲线 | 只改该关的三种倍率，怪物本身共用 |

### 13.7 验证

临时脚本实测 8 项全过：史莱姆 30/5/8/4、石像鬼倍率后 69/6/10/12、恶魔领主 100/12/19/9、关卡推进 `true → index 1`、最后一关推进 `false`、进度文案「第 3 / 5 层」、狂战士 45% 血狂暴、越界关卡返回 null。

---

## 14. 立绘与战斗演出（2026-10-19 新增；2026-10-21 改为透明立绘 + 逐帧动画）

### 14.1 素材

| 文件 | 角色 | 尺寸 | 底色 |
|---|---|---|---|
| `assets/characters/feina_battle.png` | 绯奈战斗立绘（面朝右、战斗预备姿势） | 1024×1024 | **透明** |
| `assets/characters/feina_portrait.png` | 绯奈主立绘（角色面板 / 标题页） | 1024×1024 | **透明** |
| `assets/characters/feina/idle_00~03.png` | 绯奈待机帧 | 640×640/帧 | **透明** |
| `assets/characters/feina/attack_00~03.png` | 绯奈攻击帧 | 640×640/帧 | **透明** |
| `assets/characters/feina/hit_00~03.png` | 绯奈受击帧 | 640×640/帧 | **透明** |
| `assets/characters/feina/death_00~03.png` | 绯奈倒地帧 | 640×640/帧 | **透明** |
| `assets/characters/feina/cast_00~02.png` | 绯奈专属施法帧（举手聚球，3 帧） | 640×640/帧 | **透明** |
| `assets/characters/feina/buff_00~03.png` | 绯奈通用增益帧（托起金光） | 640×640/帧 | **透明** |
| `assets/characters/player.jpg` | 旧玩家兜底立绘 | 1024×1024 | 不透明 |
| `slime.jpg` / `skeleton.jpg` / `gargoyle.jpg` / `berserker.jpg` / `demon_lord.jpg` | 5 只怪物（暂时保持不透明） | 1024×1024 | 不透明 |

- 绯奈：`CharacterData.battle_anim_dir` 指向帧目录，`battle.gd` 运行时构建 `SpriteFrames`；没配帧目录就退回 `battle_sprite` 单帧待机
- 怪物立绘：`EnemyData.art` 字段，仍是 `TextureRect` + `KEEP_ASPECT_COVERED`，尚未透明化
- 玩家显示为 260×260（640 帧 × 0.40625），怪物 260×260

### 14.2 动画

**玩家（绯奈）：逐帧动画，`AnimatedSprite2D`**

| 动作 | 帧数 | 帧率 | 循环 | 触发 |
|---|---|---|---|---|
| idle | 4 | 6 fps | ✅ | 战斗开始 / 一次性动作播完 |
| attack | 4 | 14 fps | ❌ | 打出攻击牌（`_play_lunge` 里 `play("attack")`） |
| hit | 4 | 12 fps | ❌ | 玩家受击（`_play_hit` 里 `play("hit")`） |
| death | 4 | 8 fps | ❌ | 玩家 HP ≤ 0（`_play_death` 里 `play("death")`） |
| cast | 3 | 10 fps | ❌ | 专属技能牌（`has_fx`）：抬手 → 举过头顶 → 掌心聚球（第 4 帧收势已删，见 §14.5） |
| buff | 4 | 10 fps | ❌ | 通用纯增益牌（抽牌 / 力量）：抬手托起金光 |

一次性动作播完由 `animation_finished` → `_on_player_anim_finished()` 切回 idle；`battle_over` 时不切回（死亡保持倒地帧）。

**程序化位移叠加在帧动画上**（不会和帧内姿势冲突，因为切帧时已经把身体位置归一化）：

| 效果 | 实现 | 时长 |
|---|---|---|
| **待机呼吸** | 循环 Tween，`position.y` 上下浮动 8px | 1.2s / 1.4s 循环 |
| **攻击前冲** | 向对方方向 `position.x` 冲 42px 再回弹 | 0.10s 去 + 0.20s 回 |
| **受击** | `modulate` 拉亮到 (2.2,2.2,2.2) 再回落 + 左右抖动 ±7px | 0.25s / 0.14s |
| **死亡** | `modulate.a` → 0 + `scale` → 基准 × 0.85 | 0.7s |

**敌人：仍是原 Tween 方案**（只有前冲 / 受击闪白 / 死亡淡出，没有帧动画）。

触发点：
- 玩家打出攻击牌 → 玩家播 attack 帧 + 前冲，敌人受击
- 敌人回合攻击 → 敌人前冲，玩家播 hit 帧 + 受击
- 敌人 HP ≤ 0 → 敌人死亡淡出（**延迟 0.7s 再弹奖励面板**，否则面板会立刻盖住立绘）
- 玩家 HP ≤ 0 → 玩家播 death 帧（单膝跪地）+ 淡出

### 14.3 透明立绘 + 逐帧动画产线（2026-10-21 新增）

**绿幕出图 → 按颜色抠 → 切帧 → 对齐画布 → 运行时组装**，全流程脚本在 `tools/`。

1. **生成**：`ark_generate_image`，提示词固定「纯绿色背景（#00FF00），无阴影、无地面投影、无文字、无边框」。多帧动画要求「一次性生成一条横向 sprite sheet，每帧等宽等高、间距均匀、人物居中、服装比例完全一致」（2048×512 = 4 帧 × 512）。
2. **抠图**：`tools/chroma_key.py` 用 chroma key（按绿色程度算 alpha），**不用** `remove_background`（AI 抠图对纯绿幕会留绿边）。despill 把绿色压到 `min(g, max(r,b))` —— 第一版只压到 `+3` 时，绯奈剑光的粉色辉光边缘会留一圈淡绿雾。
3. **切帧 + 对齐**：`tools/split_frames.py`
   - 逐帧单独 chroma key
   - 对 alpha 做**形态学开运算**（`MinFilter(9)` → `MaxFilter(9)`）把细长的剑身、光刃、飘发丝削掉，只留身体主体
   - 水平：身体主体外框中心对齐画布中心；垂直：身体主体外框底边对齐固定基线（剑尖插地不会把人物顶起来）
   - **走路这类腿部动作大的组用 `--anchor upper`**（按上半身中心对齐），否则「整体框居中」会把腿的摆动抵消掉
   - 所有帧同一画布 640×640、不缩放，切换时人物大小恒定
4. **接入**：`battle.gd` 的 `_build_frames_from_dir()` 按 `idle_00.png / attack_00.png / …` 命名约定扫描目录 → 构建 `SpriteFrames` → `AnimatedSprite2D`。`CharacterData.battle_anim_dir` 为空或目录不存在则退回静态立绘单帧待机。

**踩过的坑（给下次）：**

- AI 逐帧生成时人物在格子里会**随机漂移**（attack 四帧身体中心在原 sheet 里从 152 漂到 270px）。若按「头部中心」对齐，剑和长发会把锚点带偏，切换动作时身体横跳 70+ px。**必须用「开运算后的身体主体」做锚点。**
- **AI 每条 sheet 之间人物比例也会漂移**：cast 那条被画得比 idle 矮约 13%，只对齐位置会让人物在施法时明显变小。用 `split_frames.py --scale` 做**组间缩放补偿**（cast 1.10x、buff 1.04x），系数用 `tools/measure_scale.py` 量「去细线后的下盘面积 / 主体高」对比得出。attack / hit / death 不需要补偿——那是动作姿势（蹲身、倒地），不是比例问题。
- 归一化身体位置会同时抹掉动作本身的位移（比如挥砍时重心前移）。这是**想要的**：位置全部交给程序化 tween，帧动画只负责姿势，否则两份位移叠加会滑步。
- 生成多帧 sheet 时模型可能把剑伸出格子边界。切帧后若发现剑被截断，重生成而不是硬切。
- `--script` 模式不加载 autoload，`GameState` 会报 `Identifier not found`。要测战斗逻辑，得用「临时场景真实实例化 Battle.tscn + 自动调用方法 + 录帧」的方式。

### 14.4 坑：z_index 在这里不按预期排序

给立绘设 `z_index = -10`、奖励面板设 `z_index = 100` 后，**立绘仍然盖在面板上面**（用像素采样确认过：面板覆盖区里采样到的是立绘颜色，不是面板背景色）。而卡牌悬停时 `z_index = 10` 盖住邻卡是正常生效的。

结论：跨不同父节点的 z_index 排序在这个工程里不可靠。**解决方案是打开奖励面板时直接 `visible = false` 隐藏立绘和手牌**，关掉再恢复 —— 比依赖层级排序稳。

> 以后再遇到"某个东西莫名盖住别的"，先考虑用 `visible` 控制，别浪费时间去调 z_index。

### 14.6 敌人逐帧战斗动画（2026-10-23 新增）

- **素材**：`assets/characters/enemies/<enemy_id>/{idle,attack,hit}_00~03.png`
  - 5 只怪 × 3 套 × 4 帧 = **60 帧**，统一画布 **320×320**、脚底基线 **300**
  - 产线按 `godot-art-frames` 技能：绿幕 sprite sheet → 切帧 → chroma key → 统一画布
  - **绿色主体（史莱姆）不能用绿幕**，改用纯白底；`tools/enemy_sheet_cut.py` 支持
    `green / white / magenta` 三种 key
- **接入**：`Battle.tscn` 新增 `EnemyAnim`（`AnimatedSprite2D`，z_index -10）
  - `_setup_enemy_view()`：有 `idle_00.png` 就用帧动画，否则退回旧的 JPG 立绘
  - `idle` 循环；敌人攻击播 `attack`（+ 前冲位移）；受击播 `hit`（+ 闪白/抖动）；
    播完 `animation_finished` 回 `idle`
  - 敌人防御：沿用程序化护罩（`spawn_shield_fx`）；**强化**（展开领域）：`spawn_buff_fx`
    金色光环 —— 这两类**不画帧**，和角色一样用代码搓
- **尺寸**（主人定，逐只指定，人物显示 260 为基准）：
  - 史莱姆 **130**（人物一半高）｜骷髅兵 **220**（比人物低）｜石像鬼 **250**
  - 狂战士 **290**（比人物高一点）｜恶魔领主 **330**（高一个头）
  - 脚底统一落在 `ENEMY_FOOT_Y = 486`；查表 `ENEMY_HEIGHT_BY_ID`，没配的默认 260

**防御护罩：一套程序化特效，玩家和怪物共用**

- `scripts/fx/shield_fx.gd`（`ShieldFX`，`Node2D` + `_draw`）：青色六边形能量罩 + 两圈内环 + 顶点白色光点，整体缓慢旋转
- 入场 0.62 → 1.0 缩放（`TRANS_BACK` 回弹）+ 淡入，保持 0.28s，淡出 0.34s 后 `queue_free`
- 触发点：
  - 玩家打出带护盾的技能牌（`apply_card_effect` 里 `data.block > 0`）
  - 怪物防御（`enemy_turn` 的 `Intent.DEFEND`）
- 半径：玩家 128、怪物 118（按立绘略缩）
- **零图片资源**：形状全在 `_draw` 里画，两边共用同一个类，省掉两套素材

**施法动作（cast / buff，2026-10-21 追加）**

- 专属技能牌（`has_fx = true`）→ 播 3 帧 `cast`：抬手 → 举过头顶 → 掌心聚球；动作结束后由 `ChargeFX` 补光环 / 光粒收尾
- 通用纯增益牌（`_is_general_buff()`：不产护盾且带抽牌 / 力量，目前是冥想、狂怒）→ 播 `buff` 帧：抬手托起金光，同时播放程序化 `BuffFX`
- 护盾牌（防御 / 铁壁 / 喘息）**不播施法动作**，只出护罩 —— 三类牌的演出互不重叠
- 播放逻辑在 `battle.gd` 的 `apply_card_effect()` 开头（先摆姿势，再结算数值）

**「双倍蓄力」专属演出（2026-10-21 改版）**

- 施法动作：`cast` 三帧（抬手 → 举过头顶 → 掌心聚球），10fps。**第 4 帧「收势」已删**——AI 把能量球画到了脸侧、手臂姿势也歪，删掉后动作停在最帅的聚球瞬间直接回 idle
- 收尾特效：`scripts/fx/charge_fx.gd`（`ChargeFX`，纯程序化 `Node2D`）。光效自己**延迟 0.30s** 等 3 帧动作播完，再出「粉色光环扩散 + 14 颗光粒从上落下」，总时长 1.0s
- 早期版本用一张 `charge_burst.png` 大爆发图（加色混合、266px 直径），和帧里自带的能量球叠在一起太大太糊 → 已弃用并删除资源；`tools/key_glow.py` 保留，以后生成发光特效还能用

**通用增益特效（BuffFX，纯程序化）**

- `scripts/fx/buff_fx.gd`：中心柔光 + 两圈向外扩散的金色光环 + 16 个上升闪烁光点，0.9s 后自毁
- 颜色 `Color(1.0, 0.84, 0.42)`（金），和护罩的青蓝、蓄力的粉明确区分
- 第一版光环 alpha 0.6、光点半径 2–4.5，实拍在暗背景上几乎看不见 → 提到 alpha 0.85、半径 3–6、并加了中心柔光

**「已做特效」的标记方式**

- `CardData.has_fx: bool` —— 「是否已有专属演出特效」，**内部标记，卡面不显示**
- 目前只有 `double_strike.tres`（`has_fx = true`）
- 以后给别的卡加特效：`.tres` 里置 `has_fx = true`，再在 `battle.gd` 里接对应的 `spawn_xxx_fx()`（目前只接了 `spawn_charge_fx`，别的卡想播不同特效得先加分支）

---

## 15. 存档与死亡流程（2026-10-20 新增）

### 15.1 存档内容

文件：`user://save.json`（Windows 大致在 `%APPDATA%\Godot\app_userdata\卡牌\save.json`）

```json
{
	"version": 1,
	"player_max_hp": 80,
	"player_hp": 63,
	"level_index": 2,
	"deck": ["strike", "strike", "defend", "bash"]
}
```

- **牌组只存 `id` 字符串**，读档时通过 `GameState.card_registry`（id → CardData）查回资源对象
- 注册表在 `_ready()` 里由 `STARTING_DECK` + `CARD_POOL` 自动构建
- 存档格式异常、或牌全丢失时，回退到初始牌组，不会裸奔

### 15.2 存读时机

| 时机 | 动作 |
|---|---|
| 游戏启动（`GameState._ready`） | **不自动读档**，由标题页「继续游戏」按钮决定（§18.1） |
| 打赢一层、进入下一层 | `save_game()` 写一次 |
| 玩家血量归零 | `delete_save()` 删档，不留死档 |
| 全部通关重置 | `delete_save()` 删档，下次从头开始 |

### 15.3 死亡流程（以前会卡死）

以前玩家一死只是 `battle_over = true`，然后**游戏就卡住了** —— 点牌没反应、结束回合也没反应，只能强退。

现在：玩家立绘淡出 → 屏幕中央显示「你倒下了……」→ 2 秒后自动 `reset_run()` 从第 1 层重开。

### 15.4 注意：沙箱里测不了 user://

DSH 的工作区沙箱**禁止写入 `%APPDATA%`**，headless 测试时 `user://` 返回错误码 12（`ERR_FILE_CANT_WRITE`）。

为此存档路径做成了变量而非常量：

```gdscript
var save_path: String = "user://save.json"   # 正常运行用这个
# 测试时可临时改成 "res://.test_save.json"（项目内可写）
```

测试时指向 `res://` 后 8 项测试全过（写入 / 读档 / 数值恢复 / 牌组恢复 / 死档拒存 / 清理）。**真实运行游戏时 `user://` 可正常写入**。

---

## 16. 屏幕战斗日志（2026-10-20 新增）

### 16.1 为什么做

调数值时来回翻控制台太难受 —— 打印出来的是 `第 1 段：造成 6 点伤害`，但看不到当时的血量/能量/护盾状态。把日志搬到屏幕中间，一边打一边看。

### 16.2 实现

- 界面节点：`BattleLog`（Panel，半透明 StyleBoxFlat）+ `LogText`（RichTextLabel，BBCode）
- 位置 `(372,292)-(768,476)`，正好卡在左右两个立绘之间那片空白
- 最多保留 **9 行**，超出自动滚掉最旧的
- **控制台和屏幕双写**：所有 `print` 改成了 `combat_log()`，内部仍然 `print`，所以控制台输出没丢

```gdscript
func combat_log(text: String, color: Color = Color(0.86, 0.88, 0.92)):
	print(text)
	_log_lines.append("[color=#%s]%s[/color]" % [color.to_html(false), text])
	while _log_lines.size() > MAX_LOG_LINES:
		_log_lines.pop_front()
	if log_text != null:
		log_text.text = "\n".join(_log_lines)
```

### 16.3 记录的内容

进入关卡（层数 / HP / 攻击 / 重击）、出牌、每段伤害、获得护盾、获得力量、敌人挂易伤、敌人攻击、能量不足、战斗胜负、加入牌组、跳过奖励、通关重置。

### 16.4 坑

原来的 `print("a ", x, " b")` 是多参数调用，批量替换成 `combat_log()` 后直接报 `Too many arguments`（`combat_log` 只收 1 个字符串 + 1 个颜色）。9 处多参数调用全部改写成格式化字符串（`"获得 %d 点护盾" % data.block`）才通过。

> 以后再加日志：`combat_log("文字 %d" % 数值)`，别用逗号拼参数。

---

## 17. 角色系统：绯奈（2026-10-20 新增）

### 17.1 角色一览

| 项 | 值 |
|---|---|
| 名字 | **绯奈** |
| 称号 | 被除名的「回响」剑士 |
| 形象 | 粉发少女，白色金边战裙，粉色光刃细剑 |
| 血量 | 80 |
| 初始牌组 | 4 打击 + 4 防御 + **1 双倍蓄力**（9 张） |

### 17.2 背景故事

> 绯奈曾是王都魔法学院里成绩最差的学生。
>
> 别人一次就能点亮的灯，她要试两次；别人一剑斩断的木桩，她要斩两刀。
>
> 直到某天，导师在她的术式里发现了一个错误——她把「重复咏唱」的符文画反了。这个错误让她的每一次挥剑都带着回响：只要她愿意，同一次斩击就能在两秒之后「再发生一次」。
>
> 学院把这判定为危险术式，将她除名。
>
> 绯奈倒不太在意。
>
> 「反正一刀砍不死的，补一刀就是了。」
>
> 她说这话的时候，发梢还在冒着粉色的余烬。

> 设定和机制是对应的：她的「回响」能力 = 卡牌效果「双倍蓄力」，一次斩击打两遍。

### 17.3 专属技能卡「双倍蓄力」

| 字段 | 值 |
|---|---|
| id | `double_strike` |
| 名称 | 双倍蓄力 |
| 类型 | 技能 / 1 费 |
| 效果 | **下一次攻击牌的伤害翻倍** |
| 卡面 | 粉色剑光 + 残影二重斩 |
| 演出 | ✅ **已做动作 + 特效**（`has_fx = true`：`cast` 举手过头顶动作 + 头顶粉色能量光晕，见 §14.5） |

实现方式：
- `CardData.double_next_attack: bool` 标记这张卡是否施加蓄势
- `battle.gd` 的 `player_double_next` 状态，打出后置 `true`
- `deal_attack()` 开头读取并**立即消耗**，本次攻击的**所有段数**都 ×2
- 结算顺序：`(基础 + 力量) × 易伤 × 蓄势`，然后才扣护盾

实测：打击 6 → 12；双刃 4×2 段 → 8×2 段 = 16；力量 2 + 蓄势时双刃 → (4+2)×2×2 = 24。

### 17.4 立绘与逐帧动画

| 用途 | 文件 | 说明 |
|---|---|---|
| 战斗立绘 | `assets/characters/feina_battle.png` | 1024×1024 **透明底**，四分之三侧身面朝右、战斗预备姿势 |
| 主立绘 | `assets/characters/feina_portrait.png` | 1024×1024 **透明底**，正面全身站姿，用于「角色」面板与标题页 |
| 逐帧动画 | `assets/characters/feina/{idle,attack,hit,death}_00~03.png` | 640×640/帧、透明底；身体中心水平对齐、脚底同基线（产线见 §14.3） |

### 17.5 查看角色面板

- 左上角「角色」按钮 → 弹出面板 `(140,50)-(880,540)`
- 左侧 272×272 主立绘（透明底，透出面板背景），右侧名字 + 称号 + 完整背景故事
- 打开面板时**收起手牌**（否则手牌会压住关闭按钮 —— z_index 又不可靠），关闭后恢复

### 17.6 数据驱动

- 角色定义：`scripts/data/characters/feina.tres`（`CharacterData`）
- 字段：`id / display_name / title / story / max_hp / battle_sprite / portrait / battle_anim_dir / starter_deck_ids`
- `battle_anim_dir` 指向逐帧帧目录；留空则退回 `battle_sprite` 做单帧待机
- `starter_deck_ids` 存卡牌 id 字符串，`GameState.build_starter_deck()` 通过注册表查回
- **添加新角色**：新建 `.tres` + 生成两张透明立绘（可选：再用 `tools/` 做一套帧动画），把 `.tres` 追加到 `GameState.CHARACTERS` 即可，不用改逻辑

---

## 18. 标题页与选角色（2026-10-20 新增）

### 18.1 流程

```
启动 → TitleScreen（标题页）
        └─ 「开始游戏」→ 清旧档 + 重置 → 进 Map.tscn（初始地图）→ 走到地下城按 F 进 Battle
```

- 主场景：`res://scenes/TitleScreen.tscn`
- `GameState._ready()` **不自动读档** —— 否则玩家还没选人就被塞进旧存档
- **2026-10-22：标题页的「继续游戏」「背包」两个按钮已删除。**
  - 读档入口没了，但**存档逻辑完整保留**（`GameState.save_game / load_game / has_save / delete_save`，
    `title_screen.gd::_on_continue` 也留着并加了「不要误删」注释）。
    每进一层仍然写档，只是暂时没有 UI 入口；以后想恢复，加回按钮 + connect 即可。
  - 「背包」：标题页不再提供全局收藏查看入口，收藏本身照常累积与带出。

### 18.2 标题页布局

| 区域 | 内容 |
|---|---|
| 顶部 | 大标题「余音之下」+ 副标题 BENEATH THE ECHO（2026-10-23 由「地牢回响 / DUNGEON ECHO」改名） |
| 中左 | 选中角色的主立绘 280×280 |
| 中右 | 名字 / 称号 / 生命与牌组信息 / 故事第一段 |
| 左右 | 切换角色的箭头（只有 1 个角色时自动隐藏） |
| 底部 | 「开始游戏」（2026-10-22 起只剩这一个按钮） |

背景复用战斗背景图（`battle_bg.jpg`），加 `modulate` 压暗。

### 18.3 选角色（为多角色预留）

- `GameState.CHARACTERS: Array[CharacterData]` —— 所有可选角色
- `GameState.selected_index` —— 标题页选中的下标
- 选人后写入 `GameState.current_character`，战斗开始用它的立绘、血量、初始牌组
- **加新角色只做两件事**：把 `.tres` 追加到 `CHARACTERS`、生成两张立绘。标题页的箭头会自动出现

### 18.4 存档相关的一个坑

「开始游戏」原本靠 `delete_save()` 清旧档，但在受限环境里**文件可能删不掉**（`DirAccess.remove_absolute` 返回 1 = 未授权）。

改成**不依赖删除、直接覆盖写**：

```gdscript
GameState.reset_run()
GameState.save_game()   # 覆盖成第 1 层的新档
```

死亡重开、通关重置也走同样的「先重置再覆盖写」，避免残留死档。

### 18.5 术语改名

「格挡」→「**护盾**」，全局 42 处（UI 文本、战斗日志、文档）。代码里的英文变量名 `player_block / enemy_block` 保持不变，只改玩家看到的字。

---

## 19. UI 重排（2026-10-21 新增）

> 来源：`UI_OPTIMIZATION_PLAN.md`（C 档，已执行）

### 19.1 设计稿与缩放

- 基础分辨率固定 **1152×648**（`project.godot` 的 `window/size/viewport_*`），`stretch/aspect = keep`
- 所有 UI 按 1152×648 写死像素坐标，窗口缩放时整体等比缩放（留黑边），不再错位
- 原来的 `expand` 会让根节点随窗口变大，固定 offset 全部漂移

### 19.2 统一主题

- `scripts/ui/theme_dungeon.gd`（`DungeonTheme.build()`）返回一个 `Theme`；`battle.gd` / `title_screen.gd` 的 `_ready()` 里 `theme = DungeonTheme.build()`
- 覆盖 Label / Button（normal / hover / pressed / disabled / focus）/ Panel / RichTextLabel / ProgressBar
- 颜色分层：主文字 `#dee5ef`、次要 `#94a1b3`、粉 `#f0a8cc`、金 `#f2ca73`、警告红 `#f06b6b`、护盾青 `#8cdbff`
- 场景里已有的 `theme_override_*` 优先级更高，只用在需要强调的地方（卡面费用金、敌人意图琥珀等）

### 19.3 战斗界面布局（1152×648）

| 区域 | 节点 | 位置 / 说明 |
|---|---|---|
| 玩家状态卡 | `PlayerCard`（`Control`） | `(16,16)-(256,116)`：生成图背景 `status_player_bg.png` + 边框层 `status_player_border.png`（最上层）+ 头像（立绘裁头 `head.png`）+ 名字 + HP 条 + 护盾；粉为主、白金点缀 |
| 敌人状态卡 | `EnemyCard` | `(876,16)-(1136,126)`：层数 + 名字 + HP 条 + 护盾 + 意图 |
| 战斗日志 | `BattleLog` | `(372,10)-(790,106)`：顶部中间半透明，字号 12 |
| 能量球 | `EnergyPips` | `(16,124)-(200,148)`：圆形 Panel，亮 = 可用，数量跟随 `max_energy` |
| 能量数值 | `EnergyLabel` | `(204,120)-(256,148)` 右对齐 |
| 牌堆计数 | `DeckLabel` | `(16,152)-(256,172)`：抽牌 / 弃牌 |
| 角色按钮 | `CharacterButton` | `(16,178)-(116,212)` |
| 撤退按钮 | `retreat_button`（代码创建） | `(124,178)-(224,212)`，和角色按钮并排、样式照抄（2026-10-23 新增） |
| 玩家状态 | `PlayerStatusLabel` | `(16,220)-(256,244)`：◆ 力量 / ◇ 蓄势 |
| 敌人状态 | `EnemyStatusLabel` | `(876,134)-(1136,158)`：◆ 易伤 / ◆ 狂暴 |
| 结束回合 | `EndTurnButton` | `(1000,580)-(1136,636)`，136×56 |
| 手牌 | `Hand` | `y=525`（2026-10-23 下移 77px），卡 160×200：**默认只露出上半截（艺术图 + 名称），卡面描述在屏幕边界下面**；悬停抬起 90px 后整张卡完整露出 |

- 中间区域完全留给立绘和特效（伤害飘字位置未变）
- 手牌 5 张横跨 `160–992`，结束回合从 `1000` 起，不重叠；能量球 / 角色按钮都在左上竖列，也不碰手牌

**撤退（2026-10-23）**

- 点「撤退」→ 半透明遮罩 + 中央确认面板（380×190，屏幕正中）：「确定要撤退吗？」+ 确认撤退 / 继续战斗
- 确认后：`settle_run_loot()`（本局获得的卡**照常带进全局收藏**）→ `delete_save()` → 回 `Map.tscn`
- 结算路径和失败完全一致，只是不显示「你倒下了」

### 19.4 卡面（160×200）

**布局原则：图在上半，文本占一半。** 卡图放上半（84px），图下方一行是「费用棱形 + 名字」，再下面放描述（不要求全文可见）。

| 区域 | 节点 | 规格 |
|---|---|---|
| 背景 | `Background` | `TextureRect` 拉伸铺满，两张牌面边框图按 `CardData.frame_style` 切换（§19.9） |
| 卡图 | `Art` | `(12,12)-(148,88)`，`KEEP_ASPECT_COVERED`，缩进到边框内侧 |
| 费用 | `CostGem` + `CostLabel` | `(8,96)-(30,118)`，蓝色立体棱形（`GemIcon`），数字白字嵌在棱形中心 |
| 名称 | `NameLabel` | `(34,94)-(152,120)`，15 号，与费用同一行 |
| 描述 | `DescLabel` | `(8,122)-(152,194)`，12 号，自动换行 |

- 文本区（名字行 + 描述）从 y=94 到 194，共 100px = 卡面一半
- 费用不再是左上角文字，而是和名字一起排在卡图下方

### 19.4.1 蓝色立体棱形（GemIcon）

`scripts/ui/gem.gd`（`GemIcon`，Control + `_draw`）—— 能量球和卡面费用**共用同一个控件**：

- 画法：菱形四角 + 上半亮面 + 下半暗面 + 左上玻璃高光 + 深色棱边
- `set_lit(bool)`：点亮（蓝） / 熄灭（灰暗），能量球按「当前能量」切换
- 卡面里用 22×22，能量球用 20×24

### 19.5 奖励面板（2026-10-22 改为掉落 2 张）

- `(160,150)-(992,570)`，金边 2px + 12px 圆角 + 不透明底（0.97）
- 标题两行居中：「战斗胜利！」+「获得 2 张卡（下一层：XXX）」，22 号金色
- **2 张**掉落卡 `HBoxContainer alignment=1` 居中（从本层怪物牌库随机抽，保底 1 张怪物牌）
- 底部按钮 140×44 居中：「收下」（最后一层是「收下并重新开始」）
- 点任意一张卡也等于全部收下；两张都进 `GameState.run_loot`

### 19.6 面板动画

- `_pop_in(panel)`：0.18s 淡入 + 0.94→1.0 缩放，`pivot_offset` 设为中心
- 面板显隐仍用 `visible` 控制（z_index 不可靠的老坑没变，见 §14.4）

### 19.7 踩坑：PowerShell 改 `.tres` 会加 BOM

用 `Set-Content -Encoding UTF8` 改 `.tres` 后，Godot 报 `Parse Error: Expected '['` —— 文件头多了 `EF BB BF`。改数据文件请用 Python 读写 bytes（或 `-Encoding utf8NoBOM`），改完检查首字节。

### 19.8 踩坑：铺满屏幕的容器会吃掉鼠标

UI 重排后 `Hand`（HBoxContainer）被拉成全屏宽 `(0,448)-(1152,648)`，`mouse_filter` 默认 **STOP** → 整个底部区域被手牌容器拦截，`EndTurnButton`（1000,580）正好在它矩形里，**点不动**。

修法：`Hand` 的 `mouse_filter` 设为 **IGNORE（2）** —— 容器不接收鼠标，卡片自己（`CardUI` 根 Control 仍是 STOP）照常可点，空白处穿透到下面的控件。

规则：**任何铺满一大片区域的容器 / 装饰性 Panel，都要显式设 `mouse_filter = IGNORE`**，只有真正要接点击的控件才留 STOP。`PlayerCard` / `EnemyCard` / `EnergyPips` 这些虽然也是默认 STOP，但不在交互控件上方，暂不动；`CharacterPanel` / `RewardPanel` 需要挡住背后的点击，保持 STOP 是对的。

> 验证注意：`--headless` 下 GUI 鼠标是 dummy 的（`gui_get_hovered_control()` 返回 null），测按钮命中要用 `--write-movie` 带渲染跑。

### 19.9 牌面背景（2026-10-21 新增）

两套 4:5 的牌面边框底图，320×400（由 1024×1280 缩放，2× 超采样）：

| 文件 | 用途 | 风格 |
|---|---|---|
| `assets/cards/frames/frame_common.png` | 所有普通卡 | 青蓝石框 + 深蓝衬底 + 四角青蓝雕花 |
| `assets/cards/frames/frame_special.png` | 角色专属卡 | 白金粉框 + 四角粉色宝石 + 深紫衬底 |

- `CardData.frame_style`：0 = 通用，1 = 专属；`double_strike.tres` 设 1
- `card_ui.setup()` 按 `frame_style` 切 `background.texture`（`FRAME_COMMON` / `FRAME_SPECIAL` 预加载）
- `Background` 节点从 Panel 改成 `TextureRect`（`expand_mode=1` + `stretch_mode=0` 拉满）
- 卡图缩进到 `(12,12)-(148,88)`，避免压住边框

**边框层（叠在最上层）**

卡图 / 文字会盖住背景图的边框，所以额外抠了一层「只有边框」的透明 PNG：

- `tools/make_frame_border.py`：中心矩形挖空，四角只保留亮雕花（暗衬底也挖掉，避免在卡图角上压出黑块）
- `frame_common_border.png` / `frame_special_border.png`：320×400，透明率约 73%
- 卡面里新增 `FrameBorder`（`TextureRect`，在 `Visual` 最上层），`card_ui` 按 `frame_style` 与背景图一起切换
- 描述区缩到 `y=122~180`，给底部边框让位
- 角色状态卡也用同一套做法：`assets/ui/status_player_bg.png`（粉金主题，480×200）+ `status_player_border.png`（边框层，最上层），见 §19.3

### 19.10 手牌排布与行走过场（2026-10-21 新增）

**手牌：手动排布，不再用 HBoxContainer**

- `Hand` 从 `HBoxContainer` 改成普通 `Control`，`battle.gd` 用 `_layout_hand()` 手动摆位
- 原因：容器自动重排 + 卡片悬停位移（`_process` 里 `position = _rest_position + lift`）会互相覆盖——出牌后剩余卡片重排，但悬停中的卡还记着旧槽位，就会被「拽回」旧位置 → **出牌后偶发错位**
- `_layout_hand()` 在增删手牌后立即调用，同时更新每张卡的 `_rest_position` / `_slot_rect`
- 手牌固定居中：`total = count * 160 + (count-1) * 8`，起始 x = `(1152 - total) / 2`

**行走过场：打完一层走向下一层**

- `walk` 动画 4 帧、10fps、循环，标准侧面朝右
  - **不是 AI 画的**：以 `idle_00.png` 为底，取右脚块复制成亮（×1.35）/ 暗两版，四帧为「开 / 合 / 开 / 合」
  - 绘制顺序固定先暗后亮（亮 = 离屏幕近，永远盖住暗脚）；开时两脚分开、合时大部分重叠只露边缘，明暗交替表示换脚
  - 反方向不另做帧，走的时候 `flip_h` 镜像
- 过场时 `player_sprite.play("walk")` + 位置 tween，**帧序列在动**而不是整张图平移
- `close_reward_panel()` 在 `start_battle()` 之前 `await _play_walk_transition()`：角色播 walk、1.3s 内从 `x=230` 走到 `x=1300` 并淡出，然后复位
- 过场期间敌人立绘保持隐藏（`enemy_sprite.visible = false`），下一关 `start_battle` 会重新设置
- `_build_frames_from_dir` 里循环动画判定改为 `idle` 或 `walk`

---

## 20. 背包系统（2026-10-22 新增）

### 20.1 规则（2026-10-23 改为「局外卡组」制）

- **全局收藏**（`GameState.card_collection`，存 `user://save_slot_N.json`）：拥有的卡
- **局外卡组**（`GameState.deck_ids`）：出战牌组，可重复；**每局开打按它铺牌，收藏不消耗**
- **初始卡**：新建存档时给 4 打击 + 4 防御 → **进收藏 + 进卡组**，之后不再免费补
- **专属卡**：只进卡组、不进收藏 → 天然不可出售
- **出售**：只有怪物掉落卡能卖（`can_sell()`）；右键收藏卡 → 选数量出售
- **局内背包**：战斗中「背包」按钮 → 本局牌组 ↔ 局内战利品，可增可删
  - 点本局卡：从牌组移出 → 进局内背包（局末带出到收藏）
  - 点背包卡：带入本局（加入牌组和抽牌堆）

### 20.2 数据层

`game_state.gd` 新增：

| 成员 | 说明 |
|---|---|
| `collection_path` | 按槽位：`user://save_slot_N.json` |
| `card_collection: Dictionary` | id → 数量 |
| `deck_ids: Array[String]` | 局外卡组（出战牌组，可重复） |
| `run_extra_cards: Array[CardData]` | 旧流程遗留，恒为空 |
| `load_collection()` / `save_collection()` | 读写收藏 |
| `add_to_collection(id, n)` / `take_from_collection(id)` | 增减 |
| `collection_count(id)` / `collection_total()` | 查询 |
| `collection_cards()` | 展开成卡牌数组（每份一个） |
| `all_known_cards()` | 图鉴用：所有卡去重 |

`build_starter_deck()` 在角色初始牌组后追加 `run_extra_cards`。

### 20.3 UI

| 面板 | 脚本 | 入口 | 用途 |
|---|---|---|---|
| `CollectionPanel` | `collection_panel.gd` | ~~标题页「背包」/ 开局选卡~~ | **2026-10-23 起已无入口**（开局选卡取消、地图背包改用 `BagPanel`），文件保留备用 |
| `BagPanel` | `bag_panel.gd` | 战斗内「背包」按钮 | 两栏：本局牌组（点击移出）/ 收藏（点击带入） |

两个面板都是代码构建（`Panel` + `ScrollContainer` + `GridContainer` + 复用 `card.tscn`）。

### 20.4 注意

- 背包按钮与「角色」按钮并排放在状态卡下方：`(16,178)` / `(124,178)`
- 局内移出卡牌要同步三处：`GameState.deck`、`draw_pile`、`discard_pile`，并把手牌里那张 `queue_free`
- 收藏文件独立于 `save.json`，不会随「开始游戏」清档被删

---

## 21. 初始地图（2026-10-22 新增）

### 21.1 流程

```
标题页「开始游戏」→ Map.tscn（初始地图）→ 走到入口按 F / 点右上角「开始」→ Battle.tscn
    → 一直向右走到地下城入口，按 F / 点右上角「开始」→ Battle.tscn（第 1 层）
```

- 「继续游戏」仍直接进 Battle（跳过地图）
- 地图只做展示和入口，不改游戏状态（`reset_run()` 已在标题页完成）

### 21.2 场景结构

```
Map (Node2D, script=map.gd)
├── Background (Sprite2D, 3500×648，草地 + 右侧地下城入口)
├── Player (AnimatedSprite2D, (200,436), scale 0.45)
├── Camera2D (跟随玩家 x，limits 0..3500 × 0..648)
└── UI (CanvasLayer)
    └── Root (Control) ← 右上角「开始」按钮 + 底部操作提示
```

### 21.3 交互

- `A / D` 或 `← / →` 控制左右移动（360 px/s）；移动时播 `walk`，停下播 `idle`，向左时 `flip_h`
- **相机以人物为中心**横向跟随（`position_smoothing` 6），靠近地图两侧时由 `Camera2D` 的 `limit` 自动停住
- 地图长 3500px（约 3 屏宽），角色从左侧起点一路向右
- 走到 `x >= 2580`（地下城门前台阶）→ 底部提示变成金色「已到地下城入口 · 按 F 进入」，**按 F** 进 Battle；角色最远只能走到 `x = 2900`（门中间偏左，不会穿过门）；右上角「开始」按钮也可随时进入（`_entered` 保证只触发一次）
- 角色动画复用 `assets/characters/feina/` 的 idle / walk 帧（`_build_frames` 按目录扫描）

### 21.4 素材

- `assets/backgrounds/map_initial.png`（3500×648，草地长廊 + 最右侧地下城入口；生成 2700×500 后放大）
- 生成后把画面**整体下移 40px**，顶部 40px 用原图顶部做高斯模糊填充（让草地位置更低、角色站得更实），角色落点同步下移到 `y=476`
- **人物胯部以下换成横向道路**：生成一张土路 / 碎石道纹理（2048×512），取中段缩放到 3500×158 贴到 `y=490`，与草地交界用 20px 渐变融合

---

## 22. 局外 Hub / 局内整合面板 / 图鉴（2026-10-22 新增）

### 22.1 流程

```
标题页「开始游戏」→ 地图（局外 Hub）
    → 走到入口按 F / 点右上角「开始」→（收藏里有卡就选 1 张带走）→ 第 1 层
失败 / 通关 → 回到地图（局外），本局获得的卡自动带出到全局收藏
```

- 「继续游戏」仍直接进战斗（跳过地图）

### 22.2 全局收藏与局内掉落

| 阶段 | 行为 |
|---|---|
| 打怪奖励 | 选中的卡进 `GameState.run_loot`（局内背包），不直接进收藏 |
| 失败 / 通关 | `GameState.settle_run_loot()` 把 `run_loot` 全部转入收藏 |
| 出征 | ~~收藏非空时选 1 张带入（**消耗 1 张**）→ `run_extra_cards` → 加进初始牌组~~ → **2026-10-23 取消，直接开打** |
| 不消耗 | 基础卡（打击 / 防御）和专属卡（双倍蓄力）由角色配置生成，不占收藏 |

局内获得的卡**无论有没有加入卡组**，失败/通关都会带出（`run_loot` 不清空，加入卡组时只是复制进 `deck` / `draw_pile`）。

### 22.3 地图（局外 Hub）

右上角五个按钮：**开始 / 背包 / 图鉴 / 角色 / 调试**

| 按钮 | 打开 | 说明 |
|---|---|---|
| 开始 | 直接进战斗（2026-10-23 取消出征选卡） | — |
| 背包 | `BagPanel`（收藏模式） | 收藏卡 + 持有数量，默认属性页 |
| 图鉴 | `CompendiumPanel` | 基础 / 怪物 / 专属 三个分类，点卡面放大、再点关闭 |
| 角色 | 代码构建的简介面板 | 立绘（`portrait_small` 320×320）+ 称号 + 故事 |
| 调试 | `map.gd::_open_debug()` 面板 | **改人物等级（1~110）/ 阶位（0~6）**，应用后重算血上限、按满血、写收藏档（2026-10-23） |

- 「调试」按钮在 `Map.tscn` 里是 `DebugButton` 节点：`(988,220)-(1128,260)`，和上面四个一列
- 只有当等级 ≤ 10×(阶位+1) 时底部提示会带括号警告，不阻止应用

- 面板统一通过 `_add_overlay()` 挂到独立的 `CanvasLayer(layer = 10)`，避免被 Hub 按钮盖住
- 面板 x 从 60 起（右缘 972），避开 988 起的按钮列

### 22.4 局内整合面板（RunPanel）

战斗里「角色」按钮 → `RunPanel`（原「背包」按钮已移除）：

- 左：头像 + 名字 + 生命 / **攻击力 / 防御力 / 天赋（2026-10-23 已显示天赋名与当前阶效果）**
- 右上：卡组（本局牌组）
- 右下：局内背包（本局获得的卡，点击加入卡组；失败/通关时带出）
- 不再显示角色简介（简介只在局外地图看）

### 22.5 卡牌悬停

- `CardUI.hover_scale_mode = true`（背包 / 图鉴 / RunPanel）：悬停只做 **1.05 倍轻微放大**，不抬起、无其他效果
- 手牌仍是原来的抬起 90px 方便看全
- 图鉴放大：`CardUI.set_zoomed(true)` → scale 1.9、z_index 200，再点关闭

---

*文档维护：改动代码后同步更新本文档的节点坐标、问题清单与扩展设计。*

## 23. 等级 / 经验 / 阶位系统（2026-10-22 新增）

### 23.1 角色升级

```gdscript
// GameState
const EXP_BASE := 100
const EXP_GROWTH := 1.04
const EXP_FIX_LEVEL := 80     // 80 级后需求固定
const MAX_LEVEL := 110

func exp_to_next(level): return round(100 * 1.04^(min(level,80)-1))
```

| 等级 | 升级需要 |
|---|---|
| 1 → 2 | 100 |
| 20 → 21 | 211 |
| 40 → 41 | 462 |
| 60 → 61 | 1,012 |
| 80+ | 2,216（固定） |

经验跨局保留，存在 `user://collection.json` 的 `level` / `exp`。

### 23.2 击杀经验

```gdscript
// GameState.monster_exp_info(enemy) → {high, value, penalty}
eff_level = min(enemy.level, 80)
tier      = max(0, eff_level / 10 - 1)        // 阶位由等级推导
E         = 基础 × (1 + eff_level%) × 2^(tier+1)
基础：普通 20 / 精英 50 / 首领 200
```

**越级惩罚**（`EXP_PENALTY_GAP = 10`, `EXP_PENALTY_RATE = 0.1`）：
```gdscript
gap = player_level - enemy.level
if gap > 10: value *= 0.1^(gap - 10)
```

### 23.3 阶位（素材提升）

- 阶位**不随等级自动涨**，`add_exp()` 里已去掉自动 +1
- 升 N → N+1 阶要提交 **N 阶的恶魔之核**（首领素材）+ 2 种普通素材
- 入口：`GameState.tier_need_for(next_tier)` / `can_advance_tier()` / `advance_tier()`
- UI：属性页经验条下方

```gdscript
// 掉落（battle.gd _grant_kill_drops）
// 只有恶魔之核分阶位（几阶怪掉几阶），普通素材不分
mon_tier = max(0, enemy.level / 10 - 1)
GameState.add_item(id, n, mon_tier if id == "demon_core" else 0)
```

### 23.4 阶位压制（双向）

```gdscript
const TIER_GAP_MULT := [0.5, 0.1, 0.01]     // 差距 1/2/3
func tier_damage_mult(attacker_tier, defender_tier):
    gap = defender_tier - attacker_tier
    if gap <= 0: return 1.0                  // 高阶打低阶不加成
    return TIER_GAP_MULT[gap-1] if gap <= 3 else 0.0
```

- 玩家 → 怪物：**所有伤害计算之后**（攻击力×百分比 → 蓄势 → 会心 → 易伤 → 防御减免 → 穿透），
  再乘阶位倍率，最后护盾吸收、扣血（2026-10-23 主人明确：阶位压制是最后一步）
- 怪物 → 玩家：狂暴加成之后 / 护盾吸收之前（同样是加成算完再乘）
- **意图预告同步**：`enemy_attack_value()` 也乘同一个倍率

### 23.5 属性按等级现算

`CharacterData` 存**基础值**，实际值靠方法现算（成长值**累乘**）：

```gdscript
func eff_attack(level): return round(attack * (1+attack_growth)^(level-1))
// 同理 eff_max_hp / eff_defense / eff_crit_rate / eff_crit_mult
// eff_penetration（向下取整）/ eff_ignore_defense
```

- 成长值**每个角色一份**：`tools/apply_new_stats.gd` 的 `CHAR_GROWTH_BY_NAME`
- 加新角色：表格填成长值 → `CHAR_GROWTH_BY_NAME` 加一条
- 战斗里所有读属性的地方都走 `eff_*`（攻击力 / 防御力 / 会心 / 穿透 / 无视防御）
- **满血规则**：升级时若原本满血则补满；入场也一样

---

## 24. 物品系统与背包面板（2026-10-22 新增）

### 24.1 ItemData

```gdscript
@export var id, display_name
@export_multiline var description
@export_enum("药剂","消耗品","材料","特殊") var category
@export var art: Texture2D
@export var heal_percent: float      // 药剂：回复最大生命百分比
@export var heal_flat: int           // 药剂：固定回复
@export var max_stack: int
```

- 资源放 `scripts/data/items/*.tres`，启动时 `GameState.load_item_catalog()` 扫目录注册
- **加新物品 = 往这个目录丢一个 .tres，不用改 UI 代码**

### 24.2 库存

```gdscript
GameState.item_inventory   // key → 数量
// 普通物品 key = id
// 首领素材按阶位分开：demon_core@2
GameState.add_item(id, n, tier)      // tier>0 时按阶位存
GameState.item_count(id)             // 不分阶位求和
GameState.item_count_tier(id, tier)  // 取指定阶位
GameState.remove_item(id, n, tier)   // tier=-1 从低阶开始扣
```

### 24.3 背包面板 BagPanel

`scripts/ui/bag_panel.gd`，`extends Control`（自己铺满屏幕 + 全屏遮罩）。

```
Control (1152×648, 全屏)
├── 遮罩 Control (点它 → 关闭)
└── Panel _frame (860×540, 居中 (146,54))
    ├── 背景 TextureRect (anchors 铺满)
    ├── 标题 / 金币 Label
    ├── _content Control (内容区，三个板块共用)
    ├── 竖排按钮 ×3  (属性/物品/卡牌，贴右边缘)
    └── ✕ Button (右上角)
```

**三个板块**

| 板块 | 内容 |
|---|---|
| 属性 | 立绘 → Lv./阶位 → 经验条 → 升阶素材；右侧详细属性 |
| 物品 | 左放大图 + 效果 + 使用按钮；右 100 格（上下滚动） |
| 卡牌 | 局内左牌库右背包；**局外左卡组右收藏**（2026-10-23）—— 同一套栏样式（标题 + 金线 + 每排 5 张）；右键收藏卡出售 |

> 打开背包**默认停在「属性」页**（2026-10-23）。局外面板必须居中——`BagPanel._ready()` 自己定位，
> 调用方**别再设 `position`**，否则整块面板会偏移（地图页曾设 (60,40) 导致偏右下）。

**卡牌渲染**

- `CARD_SCALE = 0.375`（约 60×82），`holder` 固定尺寸 + 卡片 `base_scale`
- **必须先 `add_child` 再 `setup()`**（`@onready` 要进树才有效）
- **`pivot_offset` 必须设 `Vector2.ZERO`**（否则缩放围绕中心，卡片偏移 (50,68)）
- 堆叠：只有背包侧按 `id` 分组，右上角 `×N`；牌库平铺

### 24.4 稀有度与价格

```gdscript
CardData.RARITY_PRICE := [15, 40, 100, 250, 600, 1500, 4000]
CardData.SELL_RATE := 0.6
CardData.RARITY_FRAMES := { 2: [史诗背景, 史诗边框] }   // 没做的回退通用边框
```

**类别（quality）和稀有度（rarity）是两个维度**，别混。

---

## 25. 领域与抗性优先级（2026-10-23 新增）

### 25.1 领域：放牌门 + 同阶 70%

```
带 tag「领域」且 domain_priority > 0 的牌 → 在场上建立领域
优先级 = card_priority(卡牌, 使用者阶位)
```

**放牌门 `can_deploy_domain()`**

| 后放的优先级 vs 对方在场领域 | 结果 |
|---|---|
| 更高 | ✅ 放得出来，完整领域（对方按差距表被压制） |
| **同阶** | ✅ 放得出来，但**只发挥 70%**（先放的保持 ×1） |
| **低 1 / 2 / 3 级** | ✅ 放得出来，但按碰撞表被压制：×0.5 / ×0.1 / ×0.01 |
| **低 4 级以上** | ❌ 放不出来（能量不扣，日志提示几级） |

- 同阶 70% 只在**双方领域都在场且同阶**时成立；**对方领域消失/被替换后自动恢复 100%**
- 先后用 `player_domain_seq` / `enemy_domain_seq`（全局递增 `_domain_seq`）判断，谁序号大谁是后放
- 敌方抽到领域牌走同一套判定，放不出来 = 该回合挥空（意图只显示类型，不再提示会放空）

**压制倍率 `domain_multiplier_by_tier()`**（放得出来之后才用到）

| 差距 | 1 | 2 | 3 | >3 |
|---|---|---|---|---|
| 倍率 | ×0.5 | ×0.1 | ×0.01 | 0 |

### 25.2 抗性优先级

```gdscript
// battle.gd
const RESIST_GAP_MULT: Array[float] = [0.5, 0.1, 0.01]

func resist_multiplier(resist_value: int, effect_priority: int) -> float:
    gap = resist_value - effect_priority        // 抗性 - 效果优先级
    if gap <= 0: return 1.0
    return RESIST_GAP_MULT[gap-1] if gap <= 3 else 0.0

func card_effect_priority(card, user_tier) -> int:
    return card_priority(card, user_tier)        // 实际阶位（含保底）
```

- **效果优先级 = 卡牌实际阶位**：`max(tier_fixed, max(0, 使用者阶位 - 1))`；
  `tier_bonus >= 0` 时是 `max(0, 使用者阶位 - 1) + tier_bonus`
- **保底**：效果优先级不会小于「使用者阶位 - 1」，防止高阶角色用低阶卡被抗性吃光；
  **不进卡面显示**，卡面只显示计算后的阶位
- 已接状态：**易伤**（`apply_vulnerable()` → `resistance_gate()`）
- 未做：灼烧 / 寒冷 / 中毒 / 虚弱 的伤害与状态本身（接口已就绪）
- 抗性读数：`enemy_resist(kind)` / `player_resist(kind)`，kind = burn/cold/poison/weak/vulnerable
- 验证：`godot --headless --path . --script res://tools/verify_resist.gd`

> 领域与抗性**共用同一张差距表**（×1 / ×0.5 / ×0.1 / ×0.01 / 免疫），但方向不同：
> 领域是「低的一方被压制」，抗性是「抗性高于效果优先级就削效果」。

### 25.3 敌人意图只报类型（2026-10-23）

`_intent_text()` 不再预告具体牌名、伤害或护盾，只给三类之一：

| 判定（看下一张要打的牌） | 界面显示 |
|---|---|
| `percent_attack > 0` 或 `damage > 0` | `意图：攻击` |
| 没有伤害，但 `percent_defense > 0` 或 `block > 0` | `意图：防御` |
| 其余（抽牌 / 力量 / 领域 / 纯 debuff） | `意图：强化` |

- 代码：`battle.gd` 的 `_intent_text()` + `enemy_card_kind_text()`
- 牌库为空的旧招式兜底同样只报「攻击 / 防御」
- `enemy_attack_value()` / `enemy_card_damage()` 仍保留（结算与 AI 用），只是不再显示给玩家

---

## 26. 天赋系统（2026-10-22 实现，2026-10-23 接入 UI）

### 26.1 数据

```gdscript
# scripts/data/talent_data.gd
@export var id, display_name, owner_name
@export_multiline var description
@export var tier_texts: Array[String]   # 0=基础, 1~6=一~六阶
@export var proc_chance: Array[float]   # 各档触发率
@export var bonus_ratio: Array[float]   # 各档追加比例
@export var tier6_card_chance: float    # 六阶额外：抽牌回能量的概率
```

- 现有天赋：`scripts/data/talents/echo.tres`（**回响**，绯奈）
- 7 档数值见表格「天赋」工作表（触发率 `[10,10,20,30,30,40,50]%`、追加 `[5,10,10,10,15,15,30]%`）
- 角色绑定：`CharacterData.talent`

### 26.2 战斗触发（`battle.gd::_try_echo_talent()`）

- 每次造成伤害后调用，档位取 `CharacterData.tier_priority`（0~6，自动夹到合法范围）
- 掷 `proc_chance[tier]`，命中追加 `round(本次伤害 × bonus_ratio[tier])` 的**固定伤害**
  （固定伤害直接扣血，**无视任何减免与增伤** —— 名词解释表的定义）
- **六阶**（tier ≥ 6）触发时再有 `tier6_card_chance` 概率：抽 1 张牌 + 回 1 点能量
- 表现：紫色飘字（错开主伤害数字）+ 战斗日志「天赋「回响」追加 N 点固定伤害」

### 26.3 UI

| 位置 | 显示 |
|---|---|
| 属性页（`BagPanel` 属性板块） | 「天赋 · 回响（N 阶）」+ 当前档效果，放在**带边框的可上下滚动文本框**里（12px，自动换行） |
| 局内 `RunPanel` 左侧 | 同上（立绘下方，340×104 滚动框） |

### 26.4 存档（2026-10-23 修）

- `collection.json` 新增 `tier` 字段保存阶位
- 读档**不再按等级推导阶位**（旧代码 `tier = (level-1)/10` 会把升阶结果冲掉）
- 旧档没有 `tier` 字段 → 保持 0 阶

> 阶位与等级是两套独立系统：等级靠打怪经验涨，阶位靠提交素材升。

---

## 27. 存档槽位与局外卡组（2026-10-23 新增）

### 27.1 三个存档位

- 文件：`user://save_slot_1.json` ~ `save_slot_3.json`，**完全独立**
- 每槽保存：`cards`（收藏）、`gold`、`items`、`level`、`exp`、`exp_high`、`tier`、**`deck`（卡组 id 列表）**
- 标题页删掉「开始游戏」，改成 3 个槽位按钮（代码创建，200×112，居中排开）
  - 空槽：`存档 N / — 空 — / 点击新建`
  - 有档：`存档 N / Lv.X 金币 Y / 卡组 A 收藏 B`
- `GameState.open_slot(n)` 有档就 `load_collection()`，没档就 `create_slot(n)`
- `delete_slot(n)` 已实现（暂无 UI 入口）

### 27.2 初始卡发放（只发一次，且**只进卡组**）

- **初始通用卡是独立卡牌**：`strike_initial.tres` / `defend_initial.tres`
  - id 带 `_initial` 后缀，显示名是「打击（初始）」「防御（初始）」
  - 和掉落获得的普通「打击 / 防御」**完全是两张卡**，不会混在一起
- `create_slot()` 按角色 `starter_deck_ids` 发放：
  4×打击（初始） + 4×防御（初始） + 专属 = 卡组 9 张，**仓库 0 张**
- 这 9 张就是「保留的底牌」：不用消耗仓库也能一直带着、也不可出售
- **之后不再免费补**；卡组就是玩家在局外组的牌
- 旧档（没有 `deck` 字段）走 `_legacy_slot` 兼容分支，按角色配置免费铺

### 27.3 出售规则

| 卡 | 能不能卖 |
|---|---|
| 初始底牌「打击（初始）」「防御（初始）」 | ❌ |
| 角色专属卡（双倍蓄力） | ❌ |
| 普通通用卡（掉落的打击 / 防御） | ✅ |
| 怪物掉落卡（快斩 / 重击 / 恶魔城…） | ✅ |

- `can_sell()`：排除 `INITIAL_CARDS` 和 `source = 角色专属`，其余都能卖
- 交互：局外背包**右键**仓库卡 → 出售面板（数量 −/+、单价、确认/取消）→ `sell_cards(id, n)`

### 27.3.1 入组消耗规则（2026-10-23 定稿）

**所有卡一视同仁**：

| 操作 | 结果 |
|---|---|
| 收藏卡 → 加入卡组 | **消耗收藏 1 张**（右栏数量立刻 -1） |
| 卡组卡 → 移出 | **退回收藏 1 张**（专属卡也一样，所以不会消失） |

- 初始 9 张底牌只在卡组里、不占收藏，所以「保留 4 张打击 + 4 张防御」不需要消耗
- 打怪掉落的卡进收藏；想带进下一局就点一下，扣 1 张
- 旧档（收藏里已有打击/防御）照常按「消耗式」运作

> 右栏必须实时刷新：`BagPanel._refresh()` 在局外模式下每次都重新
> `GameState.deck_cards()` / `collection_cards()` 取数据，所以加卡后左右两栏一起变。

### 27.4 局外背包卡牌页

- 左「卡组」（`deck_ids` 展开）右「收藏」（`card_collection` 展开），中间金线
- 左键：卡组卡 → `remove_from_deck()`；收藏卡 → `add_to_deck()`（按 §27.3.1 的消耗规则）
- 每次 `_refresh()` 重新从 `GameState` 取两边数据
- 卡组为空时地图「开始」按钮会拦住并提示「卡组是空的」

---

## 28. 局外商店（2026-10-23 新增）

### 28.1 规则

- **只卖「已经获得过」的卡**：存档里有 `unlocked_cards` 记录，卡只要进过仓库/卡组就永久解锁，
  卖掉、消耗光也不会消失
- **排除**：初始底牌（`INITIAL_CARDS`）、角色专属卡（`source = 角色专属`）
- **价格** = `CardData.base_price()`（原价）；卖出是原价的 60%
- 买到的卡直接进仓库，再自己去背包放进卡组

### 28.2 数据

```gdscript
var unlocked_cards: Dictionary = {}     // id → true，存进存档
GameState.unlock_card(id)               // 获得卡时调用（add_to_collection / add_to_deck）
GameState.shop_stock()                  // → [{card, price}]，按价格升序
GameState.buy_card(id)                  // 扣金币 + 进仓库，失败返回 false
```

- 旧档没有 `unlocked` 字段 → 读档时从仓库 + 卡组里的卡**推断**解锁

### 28.3 UI

- 地图右上按钮列第六个「商店」（`ShopButton`，`(988,268)-(1128,308)`）
- **路边商人**：白毛猫耳美少女、背后背大筐的药水商人（`assets/characters/merchant.png`，457×976 透明底）
  - `Merchant`（`TextureButton`，`(700,245)-(820,505)`），头顶标签「商人 · 点击交易」
  - **三个入口都能开商店**：右上「商店」按钮 / 点击商人 / **靠近（±130px）按 F**
  - 靠近时底部提示变成「路边商人 · 按 F 交易（也可以直接点她）」
  - `_open_shop()` 带防重入（F 是每帧检测，面板开着时不重复创建）
- `ShopPanel`（`scripts/ui/shop_panel.gd`）：860×540 居中面板 + 全屏遮罩
  - 标题「商店」、右上金币、✕ 关闭、说明一行
  - 5 列卡片网格（缩放 0.375，卡片下方写价格），金币不够时价格显示暗红
  - 左键买一张；金币不足 / 买到都走底部 toast 提示
  - 点面板外也能关闭

---

---

## 29. 局外调试面板（2026-10-23）

地图右上「调试」打开，760×520 面板、**两个页签**：

### 29.1 人物

- 等级（1~110）/ 阶位（0~6）输入框 + 「应用」
- 应用后：重算血上限 → 按满血 → 写收藏档 → 底部 toast 提示（等级 ≤ 10×(阶位+1) 时带警告）

### 29.2 关卡编辑器

- 列出 `GameState.LEVELS` 每一层，每行：**层号 / 怪物下拉 / 等级 / 阶位 / 数量 / 删除**
- 怪物下拉：`GameState.enemy_catalog`（`_ready` 里扫描 `scripts/data/enemies/*.tres`，按 display_name 索引）
- 等级 / 阶位：`LineEdit`，输入即写回 `LevelData.enemy_level` / `enemy_tier`
- 数量：暂时**锁定为 1**（显示「×1（锁定）」）
- 「✕」删除该层（至少保留 1 层）；底部「+ 添加关卡」用 `GameState.make_debug_level()` 新建
- 改动**只在本次运行生效**，不写回 `.tres`；下一轮进地下城就按新关卡表打
- ⚠️ 重建 UI 必须用 `_switch_debug_tab.call_deferred(1)`：在 OptionButton / Button 的信号回调里
  直接 free 自己会崩

> `GameState.LEVELS` 因此从 `const` 改成了 `var`（运行时增删关卡的唯一办法）。

---
