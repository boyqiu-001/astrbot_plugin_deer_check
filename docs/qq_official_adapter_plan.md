# QQ 官方机器人（qq_official）平台适配技术方案

状态：待审查 / 未实施
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
| main.py `get_sender_name()` 16 处调用（257、369、464、497、533、611、769、864、898、1096、1554、1666、1759、1868、2203 行） | 直接使用，无空值兜底 | F2 场景下昵称为空，日历图、生涯图、分析图标题出现空名字 |
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

改动 2：三处调用方统一为以下模式：

```
name_map = await self._build_member_name_map(event, group_id)
if name_map is None or not name_map:
    # 降级：无群成员列表，全部数据库记录参与，名字用库内持久化昵称，缺省用"用户+openid后6位"
    name_map = {}
    use_degraded_names = True
```

降级模式下不再把 `group_user_ids = set(name_map.keys())` 作为过滤集合，改为不过滤（等价于全量成员都在群里）。附带说明：此前在 aiocqhttp 上获取列表失败时报错拒绝，改造后统一降级展示，属于行为改善，需要写入 CHANGELOG。

降级时的显示名解析顺序：

1. 数据库 checkin/klittra_checkin 表 `user_name` 列中该 user_id 最近一次记录的非空昵称；
2. 兜底 `用户{openid最后6位}`。

图片绘制函数 `_create_ranking_image` 无需改动（`_truncate_text` 已处理长名，名字字符串来源变化对绘制无影响）。

### 5.3 P1 昵称持久化（main.py `_init_db` + 两处打卡写入点）

设计：在 `checkin` 表与 `klittra_checkin` 表各新增 `user_name TEXT` 列（复用 `created_at` 的迁移模式），打卡写入时同row更新昵称：

- INSERT 语句改为 `ON CONFLICT(user_id, checkin_date) DO UPDATE SET deer_count = deer_count + excluded.deer_count, user_name = excluded.user_name`。
- 同一天多次打卡时昵称取最新一次。
- 旧数据该列为 NULL，仅影响降级排行榜显示（走兜底名），不影响任何现有功能。

选择加列而非独立昵称表的理由：鹿与扣分属两个数据库文件，加列使两库各自自洽、无跨库查询；且改动面最小（2 条 SQL + 2 段迁移代码）。

### 5.4 P1 昵称空值兜底（main.py）

新增私有方法：

```
def _get_display_name(self, event) -> str:
    name = (event.get_sender_name() or "").strip()
    return name or event.get_sender_id()
```

替换全部 16 处 `event.get_sender_name()` 调用为 `self._get_display_name(event)`。纯机械替换，行为仅在昵称为空时变化（空串变 openid）。

### 5.5 P2 文档与版本

1. README 新增"QQ 官方机器人平台说明"章节：@机器人 触发方式、排行榜降级表现、openid 不互通说明、图片接口繁忙时的重试建议（图片发送失败不影响打卡数据，可补发 `🦌日历` 查看）。
2. CHANGELOG 新增 v1.7.0 条目。
3. metadata.yaml version 更新为 v1.7.0。
4. `_conf_schema.json` 无需改动（白名单 hint 文案补充一句"QQ 官方平台填群 openid"）。

## 6. 明确不做的事情

1. 不申请机器人资质、不改动 AstrBot 平台配置（用户侧操作，README 指路 q.qq.com）。
2. 不改变玩法与指令格式；不放宽正则（F1 已验证 strip 后原样匹配）。
3. 不做官方平台主动消息、定时推送等新功能。
4. 不做 QQ 号与 openid 数据迁移（平台机制上不可行）。
5. 不为 aiocqhttp 群成员列表失败新增重试逻辑（降级路径已覆盖）。

## 7. 验收标准

1. 回归（aiocqhttp）：打卡、日历、年历、生涯、报告、补签、撤销、清空、帮助行为与 v1.6.0 一致；排行/群报/扣排行仍显示群昵称（name_map 非 None 时走原路径）。
2. 白名单：qq_official 场景（openid 字符串）配置白名单后不再崩溃，命中与未命中行为正确。
3. 降级：构造非 aiocqhttp 平台事件（或临时 mock `_build_member_name_map` 返回 None），排行/群报/扣排行正常出图，展示名来自库内昵称或兜底名。
4. 迁移：旧版本数据库文件加载后自动新增 `user_name` 列，原数据完好，无启动报错。
5. 昵称兜底：`get_sender_name()` 为空时图片标题与数据库记录使用 user_id。
6. 冒烟：用户在真实 AstrBot + qq_official 环境验证 `@机器人 🦌` 打卡并收到日历图（F6 发送时序在此步复核）。

## 8. 实施顺序与回滚

实施顺序（每步独立可验证，完成后各自 git 提交）：

1. 5.1 白名单修复（最小改动，先消除崩溃）。
2. 5.4 昵称兜底 helper 与 16 处替换。
3. 5.3 昵称持久化（迁移 + 写入点）。
4. 5.2 排行降级改造（依赖 5.3 提供库内昵称）。
5. 5.5 文档与版本。

回滚策略：所有改动集中在 main.py 与 resources/deer_core.py 两个文件，单分支顺序提交，任意一步可 `git revert`；数据库加列向后兼容，回滚旧代码后新列被忽略，无需数据回退。
