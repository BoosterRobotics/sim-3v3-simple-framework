# AGENTS.md

Booster Champion 3V3 仿真机器人足球赛的策略 Agent：Python 编写，在 Booster Studio 仿真中运行，每队 3 台机器人，选手写策略让机器人自主比赛，比赛中途不能更换策略。

本文件是 AI 助手修改本项目的指引。改代码前先读完本文，再按“改动定位”只读目标文件，不要通读整个仓库。

## 0. 先找 3v3 技能

先看当前环境的技能列表里有没有 Booster 3v3 足球技能（较新的开发环境预装为 `booster-3v3-soccer`，旧环境可能没有或名称不同）。列表里看不到时，检查默认安装位置 `%USERPROFILE%\.booster-ai\data\builtin\skills\booster-3v3-soccer\SKILL.md`（即 `~/.booster-ai/data/builtin/skills/booster-3v3-soccer/`）。

- **找到了**：加载它。它提供完整的赛事规则、部署前规则检查与修正、比赛信息查询方法；与本文不一致时以技能为准。
- **没找到**：按本文开发，部署前按第 5 节自查，并提醒用户升级开发环境以获得完整的规则检查。

## 1. 文件地图

| 文件 | 职责 | 是否可改 |
|---|---|---|
| `src/main.py` | `Phase` 状态机、`play()` 分派、各 `_act_*` 行为、进攻者选择与角色分配 | 改阵型、战术、分工的首选位置 |
| `src/player.py` | 单个机器人的控制手柄 `Player` 及其高层动作 | 新增或修改个人技能 |
| `src/param.py` | 全部可调常量：踢球力度、速度、阈值、距离 | 调数值只改这里 |
| `src/utils/geom.py` | 几何工具：`dist`、`angle_to`、`clamp`、`normalize_angle`、`opponent_goal`、`own_goal`、`own_goal_area_center`、`clamp_inside_field` | 可改 |
| `src/utils/obstacles.py`、`src/utils/path_planner.py` | 障碍建模、局部绕行、全局 A* 路径规划 | 可改 |
| `src/framework/` | 平台层：读取比赛状态、控制机器人、运行循环 | 不改 |
| `agent.toml`、`build.toml`、`.booster-studio/project.json` | 元数据与入口、构建配置、Studio 场景 | 非必要不改 |
| `docs/` | `booster_agent_framework` 和 BoosterOS 接口文档 | 查平台 API 时读 |

## 2. 控制模型

**每帧流程**：`play(context, players, store)` 约 30 Hz 调用一次，流程固定：

1. `get_phase(context)` 判定比赛阶段；
2. 过滤出本帧可行动的球员（未受罚、未倒地、位置已知）；
3. 按阶段调用对应的 `_act_*`，给每个球员分配角色；
4. 每个球员执行**一个**高层动作。

`play()` 必须快速返回。起身、模式切换等慢操作已由框架异步处理。

**坐标**：本队视角，`+x` 指向对方球门，`-x` 指向己方球门，场地中心为 `(0, 0)`。框架提供的已是本队视角的坐标（见 `utils/geom.py` 开头说明），策略不用自己镜像。

**`Phase`**（`get_phase()` 中判定）：

| 阶段 | 条件 |
|---|---|
| `READY` | 裁判状态 READY，走到开球站位 |
| `OUR_SET_PLAY` / `OPP_SET_PLAY` | PLAYING、未暂停、`set_play != NONE`，按 `kicking_team` 区分 |
| `OUR_KICKOFF` / `OPP_KICKOFF` | PLAYING、未暂停、`secondary_time > 0`，按 `kicking_team` 区分 |
| `NORMAL` | PLAYING、未暂停，其余情况 |
| `STOPPED` | SET、INITIAL、FINISHED、裁判暂停、或没收到裁判状态 |

**`Context`**（每帧只读快照）：

| 字段 | 内容 |
|---|---|
| `ball` | 球的 `x`、`y`；约 1.5 s 没更新时为 `None` |
| `teammates` / `opponents` | `dict[id, RobotState]`，`RobotState.pose` 为 `Pose2D(x, y, theta)` 或 `None` |
| `game` | 裁判状态，可能为 `None`：`state`、`stopped`、`set_play`、`kicking_team`、`secondary_time`、`secs_remaining`、`teams`（比分、各球员罚下状态） |
| `field` | 场地尺寸：`length`、`width`、`circle_radius`、`goal_width`、`goal_area_length`、`penalty_area_length` 等 |
| `team_id`、`now`、`dt` | 本队编号、当前时间、帧间隔 |

`SetPlay` 取值：`NONE`、`DIRECT_FREE_KICK`、`INDIRECT_FREE_KICK`、`PENALTY_KICK`、`THROW_IN`、`GOAL_KICK`、`CORNER_KICK`。

**`store`**：跨帧记忆，类型是 `SimpleNamespace`，用属性读写，没有 `get` / `set` / `inc` 方法。

```python
def init_store(self, store) -> None:
    store.attacker_id = None                 # 在这里初始化
store.attacker_id = p.id                     # 写入
getattr(store, "kick_cooldown", 0)           # 带默认值读取
```

常用于锁定角色、迟滞、冷却和检测阶段切换（示例已有 `store.prev_phase`、`store.cur_phase`）。

**`Player` 接口**：

| 接口 | 作用 |
|---|---|
| `id`、`pose`、`action`、`is_fallen`、`is_penalized`、`is_kicking` | 状态；`action` 是调试标签，会显示在场上 |
| `attack(kick_target=None)` | 追球并射门，默认瞄准对方球门 |
| `guard()` | 站己方小禁区中央守门 |
| `support()` | 站在球与己方球门连线上、距球 `SUPPORT_DIST_M` 处 |
| `take_kickoff(kick_target=None)` | 绕到球后再接近踢球 |
| `move_to_position(target)` | 走到站位点，面向球，避开球和机器人 |
| `walk_to(target, face=None, avoid_ball=False, avoid_robots=False)` | 底层走位，返回是否到达 |
| `kick(direction, power)` | 开始或更新踢球任务；`direction` 为场地坐标弧度，`power` 自动钳制到 `[KICK_POWER_MIN, KICK_POWER_MAX]` |
| `release_kick()`、`stop()` | 结束踢球；`stop()` 同时结束踢球并发零速度 |

`kick()` 不是瞬时动作：调用后机器人进入持续的踢球任务，自己走到球边完成踢球，直到 `release_kick()` 才结束，期间速度指令不生效。`walk_to`、`move_to_position`、`stop` 都会先结束踢球。

**默认分工**：离球最近者进攻，其余离己方球门最近者守门，剩下的支援。除非需求明确要换，修改时保持这个基线。

## 3. 改动定位

| 需求 | 修改位置 | 注意的规则（见第 5 节） |
|---|---|---|
| 阵型、角色分配、谁去抢球 | `main.py` 的 `_act_normal`、`_select_closest_attacker` | 无活动 |
| 我方开球战术 | `main.py` 的 `_act_our_kickoff`，或 `player.take_kickoff` | 开球站位、开球射门 |
| 对方开球站位 | `main.py` 的 `_act_opp_kickoff` | 中圈、避让距离、无活动 |
| 我方定位球（界外球、角球、球门球、任意球） | `main.py` 的 `_act_our_set_play`，按 `SetPlay` 加分支 | 定位球乌龙 |
| 防守对方定位球、布置人墙 | `main.py` 的 `_act_opp_set_play` | 避让距离、抢先触球、无活动 |
| READY 站位 | `main.py` 的 `_act_ready` | Set 站位 |
| 门将行为 | `player.py` 的 `guard`（配合 `own_goal_area_center`） | 无活动 |
| 支援站位 | `player.py` 的 `support` | 无活动 |
| 新个人技能（带球、封堵等） | `player.py` 新增方法，复用 `kick`、`walk_to` | 只在允许运动的阶段调用 |
| 速度、力度、阈值、距离 | `param.py` | 规则相关距离留余量 |
| 瞄准点、几何计算 | `utils/geom.py` | — |
| 避障、路径规划 | `utils/obstacles.py`、`utils/path_planner.py` | `avoid_ball` 只绕开球 0.5 m，不等于规则避让距离 |

**写法要点**：

- 新增定位球分支时，在 `_act_our_set_play` 里按 `get_set_play_type(context)` 判断类型，每个分支给所有球员都分配动作。
- 新增技能写成 `Player` 方法，开头判空（`self.pose`、`self.context.ball`），结尾只发一个动作。
- 新参数先在 `param.py` 定义具名常量并写注释，策略文件已用 `from .param import *`。
- 修改阶段判定（`get_phase`）时，`STOPPED` 必须覆盖 SET、暂停和没有裁判状态三种情况。
- 策略每帧重算。按距离分配角色时，用 `store` 记住当前人选，只有原人选不可用，或新人选明显更优（差距超过具名余量）才换，避免距离排序互换造成角色抖动、打断跨帧动作。

## 4. 必须遵守

- 不修改 `src/framework/`；保持入口写法 `class SoccerSimAgent(SoccerAgentMixin, AgentBase)`，`AgentBase` 是直接基类；保持 `play(context, players, store)`、`init_store(store)`、`player_class` 签名；不改 `agent.toml` 入口。
- `play()` 中不 `sleep`、不做重计算、网络或磁盘 IO。
- 不删除 `play()` 中对受罚、未起身球员的过滤，它们不能参与角色分配。
- `ball`、`game`、`pose` 都可能为 `None`，使用前判空。
- 只能通过 `kick` 改变球的位置，不能改 `ball.x` / `ball.y`。
- 移动目标点用几何工具钳制在场内，不把球员赶出场或穿过球门。
- 可调数值放 `param.py`，不在策略代码里散落数字。
- 策略只用 `context` 里的信息：不另外订阅话题，不连接 GameControl HTTP 或其他非公开接口，否则属于技术违规，可能被取消资格。

## 5. 比赛规则底线

完整规则以 3v3 技能为准；没有技能时至少保证下面几条。阈值是裁判的检测线，目标值要留余量。

| 规则 | 要求 |
|---|---|
| 静止 | SET 和裁判暂停（`STOPPED`）时所有机器人静止，不调用 `kick`；`STOPPED` 分支对每个球员 `stop()` |
| Set 站位 | READY 结束前到达场内、本方半场；对方开球时在中圈（半径 1.5 m）外 |
| 开球站位 | 我方开球、球被触碰前，主罚以外的队友不进中圈，主罚不越过半场线 |
| 避让距离 | 对方定位球和开球期间，所有机器人距球大于 1.45 m 并留余量，对方触球前不碰球 |
| 不用本地计时 | 不用本地计时判断“球权已开放”“可以返场”，只看 `context.game` |
| 开球射门 | 我方开球后，开球机器人只在球心离开开球点 0.5 m 以外时射门，或由第二个机器人触球后再射门；否则进球无效 |
| 定位球乌龙 | 我方界外球、球门球、角球的第一脚不朝己方球门 |
| 无活动 | 球在 3 m 内时不原地久站（10 s 会被罚下）；等待点放到 3 m 外 |
| 罚下 | 被罚下的机器人不自行返场 |

## 6. 验证

- 在 Booster Studio 中选择虚拟机器人，构建、部署、运行，至少看完整一场。
- 覆盖：我方/对方开球、我方/对方定位球、READY、STOPPED、球员摔倒或受罚。
- 场上标签 `phase=... state=... set=...` 和球员 `id:action` 用来核对阶段与角色。
- AI 助手可用的 MCP 中有 Booster Studio 的比赛工具（比分、裁判时间线、Agent 日志）时，可以用它们查判罚原因；没有时请用户在 Studio 界面查看。这些工具只给开发时用，策略代码不能调用。
- 不要在宿主机直接运行策略 Python；没有在仿真中跑过，不要宣称改动成功。

## 7. 完成与部署前

- 优先调参数，再改逻辑，用满足需求的最小改动。
- 部署、提交前：有 3v3 技能就按它做完整规则检查；没有就按第 5 节逐条自查，告诉用户这只是基础检查。
- 需求涉及平台或 SDK 行为时，先向用户说明，不要直接改 `src/framework/`。
