# QQ 官方机器人（qq_official）平台适配技术方案

状态：评审已通过（有条件，意见已全部并入）；追加需求"平台模式切换"已并入 / 待实施
目标版本：v1.7.0
作者背景说明：本方案面向插件维护者与审查者，所有结论均经过 AstrBot 上游源码验证，标注了证据位置。

## 1. 背景与目标

本插件（astrbot_plugin_deer_check v1.6.0）目前主要在 aiocqhttp 平台（NapCat / Lagrange 等 OneBot 协议端）下运行。用户希望插件能在 AstrBot 的 qq_official 平台（QQ 开放平台官方机器人）下工作。

目标：在不破坏 aiocqhttp 现有行为的前提下，让核心功能（打卡、日历、生涯、报告、补签、撤销、清空、帮助）在 qq_official 平台完整可用；排行榜类功能以降级模式可用。

## 2. 可行性结论

可以适配。插件 90% 以上代码平台无关：

- 指令识别使用 `@filter.regex()`，平台无关。
- 回复使用 `event.plain_result()` / `event.image_result()`，平台无关。
- 全部是被动回复，无主动消息，符合官方平台被动回复限制。
- 日历绘图、数据库均为本地逻辑，不受平台影响。

存在 3 个官方平台硬限制（无法通过改代码消除）：

1. 群聊中机器人只能收到 @机器人 的消息，打卡方式变为 `@机器人 🦌`。
2. 官方平台没有群成员列表 API，排行榜无法过滤"本群成员"，也无法获取群昵称。
3. 官方平台用户身份是 openid（每个机器人不同），与 QQ 号不互通，历史数据无法迁移。

## 3. 平台差异事实清单（已验证）

以下事实通过阅读 AstrBot 主分支源码验证（qqofficial 平台适配器）：

| 编号 | 事实 | 证据位置（AstrBot 源码） |
|---|---|---|
| F1 | 群消息解析时，机器人自身的 @ 标记会被替换为空格并 strip，`message_str` 即剩余纯文本。`@机器人 🦌` 解析后 message_str 为 `🦌`，插件现有正则 `^🦌+$` 原样可匹配 | `_parse_from_qqofficial` 与 `_strip_bot_mention_markup`（qqofficial_platform_adapter.py，约 886-902、1008-1018 行） |
| F2 | 群消息 sender 为 `member_openid`（字符串），昵称取 `author.username`，群场景下可能为空 | 同上，约 985-989 行 |
| F3 | 群 id 为 `group_openid`（字母数字混合字符串），不是数字群号 | on_group_at_message_create，约 231-243 行 |
| F4 | 本地图片经 `image_result` 发送时被读取为 base64 并调用官方富媒体接口上传，发送路径可用 | `_parse_to_qqofficial`（约 1101-1116 行）与 `upload_group_and_c2c_image` |
| F5 | 官方平台仅有群信息接口 `/v2/groups/{group_openid}/info`，返回群名与群人数，没有群成员列表 API | qqofficial_group_info.py 全文 |
| F6 | 插件发送时机安全：AstrBot pipeline 对每个 yield 的结果 await 发送完成后才继续迭代，handler 的 finally 删除临时文件不会早于发送完成 | AstrBot pipeline 处理模型（行为性结论，实施时以冒烟测试复核） |
| F7 | 官方平台被动回复限制为同一触发消息 5 分钟内最多约 5 次回复；本插件单条指令最多回复 2 条（文字 + 图片），余量充足 | QQ 开放平台公开文档（平台侧限制，非 AstrBot 代码） |

## 4. 现状代码分析

与适配相关的代码位置：

| 位置 | 现状 | 问题 |
|---|---|---|
| main.py `_guard` 232-238 行 | 白名单比较 `int(group_id) not in self.group_whitelist` | qq_official 群 id 是字符串，`int()` 抛 ValueError，配置白名单后所有指令崩溃 |
| resources/deer_core.py `_build_member_name_map` 24-48 行 | 仅在 `get_platform_name() == "aiocqhttp"` 时调 `get_group_member_list`，失败或非 aiocqhttp 均返回空 dict | 空 dict 使调用方一律走"无法获取群成员信息"报错路径，降级无从谈起 |
| main.py 三处调用：群报约 941 行、排行约 1450 行、扣排行约 1975 行 | `if not name_map:` 则报错返回 | 在 qq_official 下永远报错，功能不可用 |
| main.py `get_sender_name()` 15 处调用（257、369、464、497、533、611、769、864、898、1096、1554、1666、1759、1868、2203 行，已逐一核对） | 直接使用，无空值兜底 | F2 场景下昵称为空，日历图、生涯图、分析图标题出现空名字 |
| main.py `_init_db` 67-105 行 | `created_at` 列迁移已有成熟模式（PRAGMA table_info + ALTER TABLE） | 可复用该模式为昵称持久化加列 |

## 5. 改造项明细

### 5.1 P0 白名单崩溃修复（main.py）

改动：

1. `__init__` 中把 `group_whitelist` 归一化为字符串列表：`self.group_whitelist = [str(g) for g in self.config.get("group_whitelist", [])]`。
2. `_guard` 中比较改为 `str(group_id) not in self.group_whitelist`，删除 `int()` 转换。

兼容性：aiocqhttp 下 `get_group_id()` 本就返回字符串，后台配置面板填写的数字会被 `str()` 归一化，两平台行为一致，无回归。

### 5.2 P0 排行榜/群报/扣排行降级改造（deer_core.py + main.py 三处）

改动 1：`_build_member_name_map` 返回值语义改为 `dict | None`：

- aiocqhttp 且 API 成功：返回 dict（可能为空）。
- aiocqhttp 且 API 失败：返回 None（保留现有日志）。
- 非 aiocqhttp 平台：直接返回 None，不再尝试导入 aiocqhttp 内部类。
- 新增 `mode` 参数（`auto` / `official` / `onebot`，默认 `auto`，定义见 5.3）：`official` 直接返回 None；`onebot` 仅在平台为 aiocqhttp 时调用 API；`auto` 按平台检测。分支矩阵中"None（非 aiocqhttp）"一行在 mode=official 时同样成立。
- main.py `_build_member_name_map` 2106-2110 行的透传 wrapper 同步将返回注解与文档字符串更新为 `dict | None`。

改动 2：三处调用方统一按四分支矩阵处理（评审修正：原"非 None 走原路径"与空 dict 降级互相矛盾，选定语义如下）：

| name_map 取值 | 含义 | 行为 |
|---|---|---|
| 非空 dict | aiocqhttp API 成功且返回成员 | 原路径：按群成员过滤，显示群昵称 |
| 空 dict | aiocqhttp API 成功但成员列表为空 | 降级：全库统计展示 |
| None（API 失败） | aiocqhttp 调用异常 | 降级：全库统计展示 |
| None（非 aiocqhttp） | qq_official 等平台无成员列表 API | 降级：全库统计展示 |

降级伪代码（删除原 `if not name_map: 报错返回` 分支）：

```
name_map = await self._build_member_name_map(event, group_id, self.platform_mode)
degraded = name_map is None or not name_map
if degraded:
    name_map = {}  # 不再作为过滤集合使用
```

降级模式下不再把 `group_user_ids = set(name_map.keys())` 作为过滤集合，改为不过滤。全库统计是有意的设计选择（评审确认非隐私缺陷）：现有表结构（main.py 73-78、91-96 行）不记录群与平台，无法只取"本群"数据；官方平台 openid 与 QQ 号不互通，同一用户在不同群是否共用同一 openid 以平台实际规则为准，因此降级统计一律全库展示、不区分群也不做去重假设。此行为必须通过范围标识向用户明示，不得静默混同"本群"统计。附带说明：此前在 aiocqhttp 上获取列表失败时报错拒绝，改造后统一降级展示，属于行为改善，需写入 CHANGELOG。

降级时的显示名解析顺序：

1. 数据库 checkin/klittra_checkin 表 `user_name` 列中该 user_id 最近一次记录的非空昵称；
2. 兜底 `用户{openid最后6位}`。

`_create_ranking_image` 需同步改动（评审修正，原"无需改动"结论错误）：deer_core.py 175-176 行页脚固定为"本群共有 ... 人参与打卡"，降级全库统计时会误标统计范围。为函数新增统计范围参数（如 `scope_label: str`），非降级保持原文案，降级传"全库统计（未过滤本群成员）"；`_truncate_text` 长名截断逻辑不变。群报与扣排行的出图文案同样核对范围标识。

### 5.3 P0 平台模式配置项（追加需求：模式切换 official / onebot）

背景：原方案依赖平台自动检测（get_platform_name）。按用户要求增加显式模式配置，使排行榜类功能的统计口径可预期、可控制。

边界说明：本配置不改变机器人实际运行的平台（平台由 AstrBot 平台配置决定），只门控"群成员名单获取方式"，影响范围为排行、群报、扣排行三个功能的统计口径；打卡、日历、生涯、报告、补签、撤销、清空、帮助不受影响。

`_conf_schema.json` 新增配置项：

```json
"platform_mode": {
    "description": "平台模式",
    "hint": "auto=自动检测（默认）；official=QQ官方机器人模式，排行榜不做本群过滤、直接全库统计并显示库内昵称；onebot=OneBot协议端模式，排行榜按本群成员过滤并显示群昵称。",
    "type": "string",
    "default": "auto",
    "options": ["auto", "official", "onebot"]
}
```

AstrBot 配置 schema 对 string 类型支持 options 枚举，后台渲染为下拉框。

模式语义矩阵：

| platform_mode | 行为 |
|---|---|
| auto（默认） | 按 5.2 四分支自动检测：aiocqhttp 走成员列表，其余降级 |
| official | 强制降级：不调用成员列表 API（deer_core 直接返回 None），全库统计 + 库内昵称/兜底名，页脚"全库统计" |
| onebot | 强制成员列表：仅当平台为 aiocqhttp 时调用 get_group_member_list；平台不符或 API 失败时记 warning（如"platform_mode=onebot 但当前平台为 X，降级为全库统计"）并降级，不报错不崩溃 |

默认值选 auto 的理由：升级后 aiocqhttp 老用户与官方平台新用户均零感知，行为与 5.2 现方案完全一致；official / onebot 供用户显式固定统计口径。

实现落点：

1. `main.py __init__`：`self.platform_mode = str(self.config.get("platform_mode", "auto")).strip().lower()`；非法值（不在三枚举内）记 warning 并回落 auto。
2. `deer_core._build_member_name_map` 增加 `mode: str = "auto"` 参数（默认值保证未传参调用向后兼容），按上表实现三分支；main.py 2106-2110 行 wrapper 同步签名。
3. 三处调用（群报约 941 行、排行约 1450 行、扣排行约 1975 行）传入 `self.platform_mode`。

生效时机：配置在插件加载时读取；后台修改保存并重载插件后生效（AstrBot 保存插件配置会触发插件重载）。

### 5.4 P1 昵称持久化（main.py `_init_db` + 三处写入点）

设计：在 `checkin` 表与 `klittra_checkin` 表各新增 `user_name TEXT` 列，复用 `_init_db` 中 `created_at` 的迁移模式（PRAGMA table_info 判列 + ALTER TABLE ADD COLUMN，幂等，重复初始化安全）。

评审修正：INSERT 的列清单、VALUES 占位符、绑定参数、UPSERT 更新子句四处都必须同步带上 user_name，否则 `excluded.user_name` 恒为 NULL，昵称永远写不进去。三处写入点完整契约：

1. 鹿打卡（main.py 330-346 行）：

```sql
INSERT INTO checkin (user_id, checkin_date, deer_count, created_at, user_name)
VALUES (?, ?, ?, ?, ?)
ON CONFLICT(user_id, checkin_date)
DO UPDATE SET deer_count = deer_count + excluded.deer_count,
              user_name = excluded.user_name;
```

绑定参数末尾追加 user_name。保留现有 created_at 语义：冲突更新不改 created_at（维持"首次打卡时间"含义）。

2. 扣打卡（main.py 约 440-451 行）：同样四处带 user_name，计数字段为 `klittra_count`（`DO UPDATE SET klittra_count = klittra_count + excluded.klittra_count, user_name = excluded.user_name`），不可照抄鹿表的 deer_count。

3. 鹿补签（main.py 约 721-731 行，评审发现的第三条 INSERT/ON CONFLICT，原方案遗漏）：同样契约；补签 handler 已在 611 行取得 user_name，直接复用。仅通过补签建记录的用户由此获得持久化昵称。

撤销（约 804-821 行）只减少已有记录次数，不修改昵称。同一天多次打卡/补签时昵称取最新一次。旧数据该列为 NULL，仅影响降级排行榜显示（走兜底名），不影响任何现有功能。

选择加列而非独立昵称表的理由：鹿与扣分属两个数据库文件，加列使两库各自自洽、无跨库查询；改动面为 3 条 SQL 与 2 段迁移代码。

### 5.5 P1 昵称空值兜底（main.py）

新增私有方法：

```
def _get_display_name(self, event) -> str:
    name = (event.get_sender_name() or "").strip()
    return name or event.get_sender_id()
```

替换全部 15 处 `event.get_sender_name()` 调用为 `self._get_display_name(event)`。纯机械替换，行为仅在昵称为空时变化（空串变 user_id）。

### 5.6 P2 文档与版本

1. README 新增"QQ 官方机器人平台说明"章节：@机器人 触发方式、平台模式选择说明（auto/official/onebot 各自的统计口径）、排行榜降级表现、openid 不互通说明、图片接口繁忙时的重试建议（图片发送失败不影响打卡数据，可补发 `🦌日历` 查看）。
2. CHANGELOG 新增 v1.7.0 条目。
3. metadata.yaml version 更新为 v1.7.0。
4. `_conf_schema.json` 新增 platform_mode 配置项（定义见 5.3）；白名单 hint 文案补充一句"QQ 官方平台填群 openid"。

## 6. 明确不做的事情

1. 不申请机器人资质、不改动 AstrBot 平台配置（用户侧操作，README 指路 q.qq.com）。
2. 不改变玩法与指令格式；不放宽正则（F1 已验证 strip 后原样匹配）。
3. 不做官方平台主动消息、定时推送等新功能。
4. 不做 QQ 号与 openid 数据迁移（平台机制上不可行）。
5. 不为 aiocqhttp 群成员列表失败新增重试逻辑（降级路径已覆盖）。

## 7. 验收标准

1. 分支矩阵（排行/群报/扣排行）：name_map 为非空 dict、空 dict、None 共四态分别验证；仅非空 dict 走原路径显示群昵称，其余全部降级出图。
2. 回归（aiocqhttp，platform_mode=auto）：打卡、日历、年历、生涯、报告、补签、撤销、清空、帮助行为与 v1.6.0 一致；排行/群报/扣排行仍显示群昵称。
3. 白名单：aiocqhttp 旧数字配置命中/未命中；qq_official openid 字符串命中/未命中；空白名单全放行。以上场景均不得崩溃。
4. 模式矩阵：auto 与第 1 条四分支一致；official 在 aiocqhttp 平台上不触发成员列表 API 调用（mock 断言）且直接全库统计；onebot 在 qq_official 平台上记 warning 并降级不崩溃；onebot 在 aiocqhttp 上走成员列表原路径；非法配置值回落 auto 并记 warning；后台修改配置并重载插件后新模式生效。
5. 降级范围标识：降级出图页脚显示"全库统计（未过滤本群成员）"；非降级保持"本群共有 ... 人参与打卡"原文案。
6. 排行不变量：降级模式下排序正确、每月计入上限过滤生效、年度榜不受月上限影响、TopN 按 ranking_display_count 截取、参与总人数展示正确；`_truncate_text` 长名截断正常。
7. 昵称持久化：鹿/扣两库分别覆盖新建库、旧库迁移、重复初始化三种场景；同日二次打卡昵称更新且次数累加；仅通过补签建记录的用户在降级排行中显示真实昵称而非兜底名。
8. 昵称兜底：`get_sender_name()` 为空时图片标题与数据库记录使用 user_id。
9. 开关回归：enable_female_calendar 开/关两态下扣日历指令可用性不变。
10. 冒烟：用户在真实 AstrBot + qq_official 环境验证 `@机器人 🦌` 打卡并收到日历图（F6 发送时序在此步复核）。

## 8. 实施顺序与回滚

实施顺序（每步独立可验证，完成后各自 git 提交）：

1. 5.1 白名单修复（最小改动，先消除崩溃）。
2. 5.3 平台模式配置项（schema、读取校验、传参管道，为降级改造提供模式开关）。
3. 5.5 昵称兜底 helper 与 15 处替换。
4. 5.4 昵称持久化（两库迁移 + 鹿打卡/扣打卡/鹿补签三处写入契约）。
5. 5.2 排行降级改造（mode 三分支门控、四分支语义、范围标识绘图参数、wrapper 注解同步；依赖 5.4 库内昵称与 5.3 模式参数）。
6. 5.6 文档与版本。

回滚策略：所有改动集中在 main.py 与 resources/deer_core.py 两个文件，单分支顺序提交，任意一步可 `git revert`；数据库加列向后兼容，回滚旧代码后新列被忽略，无需数据回退。

## 9. 评审记录

2026-11-10 由 reviewer（PlanReviewer）静态审查，结论：有条件通过（overall_correctness: incorrect，confidence 0.97）。全部意见已并入本方案：

1. P1 已采纳：5.3 补全 INSERT 列清单/占位符/绑定参数契约，扣表改用 klittra_count 专用 SQL。
2. P2 已采纳：补签写入点（main.py 721-731 行）纳入昵称持久化契约与验收。
3. P2 已采纳：`_create_ranking_image` 新增统计范围参数，降级出图显式标注"全库统计"；同步核对群报与扣排行文案。
4. 已修正：get_sender_name 调用数 16 处更正为 15 处（行号经核对全部准确）。
5. 已修正：统一 name_map 四分支语义（非空 dict 原路径；空 dict 与 None 降级），消除 5.2 与验收条款的矛盾；验收改为分支矩阵覆盖。
6. 已采纳：wrapper（main.py 2106-2110 行）返回注解同步；全库降级明确为有意设计并在文中声明。

本次审查为静态审查，未运行构建与测试；实施后按第 7 节逐条验收。

追加记录 2026-11-10：用户追加需求"方案增加模式切换选项（选择官方或是 onebot）"，已并入为 5.3 平台模式配置项（platform_mode：auto / official / onebot，默认 auto）；5.2 的 _build_member_name_map 增加 mode 参数门控；验收新增第 4 条模式矩阵；实施顺序插入对应步骤（新第 2 步）。本版起 5.x 编号重排：新 5.3 = 平台模式配置项，原 5.3 / 5.4 / 5.5 顺延为 5.4 / 5.5 / 5.6；上方评审记录中的"5.3"按当时编号理解，对应本版 5.4。
