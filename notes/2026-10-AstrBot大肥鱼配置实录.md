---
created: 2026-10-28
updated: 2026-10-28
tags: [AI技术, AstrBot, QQ机器人, NapCat, 踩坑]
status: 运行中
---

# 2026-10 AstrBot 大肥鱼配置实录

在 NAS（fnOS，<NAS_IP>，i5-7500T / 8G）上养一只住在 QQ 里的「大肥鱼」。
技术栈：**AstrBot（飞牛商店版 4.29）+ NapCat（Docker）+ DeepSeek（百炼 tokenplan）**。

关联：[[2026-10-AstrBot插件清单]] · [[2026-10-QQ风控与掉线应对]]

---

## 一、最终架构

| 组件 | 位置 | 说明 |
|---|---|---|
| AstrBot | `/vol1/@appdata/astrbot/` | systemd `astrbot.service` 托管，**不要用飞牛面板启停** |
| NapCat | Docker `napcat` | `mlikiowa/napcat-docker:latest`，`restart=always` |
| 端口 | 6185 WebUI / 6199 OneBot WS / 6099 NapCat WebUI | |
| 机器人小号 | <BOT_QQ>（昵称"咕噜"） | 主人 <OWNER_QQ> |
| 模型 | `deepseek-v4.1-flash-1017` @ baidu | 非推理版，比 `v4.1-flash` 快 |

**关键认知**：AstrBot 由 systemd 管，飞牛面板显示"未运行"是假的，点启用会端口冲突。只用 `systemctl`。

---

## 二、踩过的坑（最值钱的部分）

### 坑 1：插件配置改错层（最坑，浪费最多时间）

private_companion 的配置有**四层结构**，插件只读其中一层：

| 层 | 位置 | 是否是真相源 |
|---|---|---|
| 顶层平坦字段 | `config.json["default_proactive_enabled"]` | **运行时读这里**（`plugin_bootstrap.py` 从 config 顶层读） |
| `legacy_compat_config` | `config.json["legacy_compat_config"]["enable_segmented_..."]` | **分段开关在这里** |
| `basic_config` | `config.json["basic_config"]["..."]` | WebUI 面板显示的（部分被同步到顶层） |
| 插件 schema | `_conf_schema.json` 的 `default` | **启动时用它覆盖上面各层** |

**教训**：
- 改 JSON 前先确认插件从哪读（`grep -rn "键名" *.py` 找 `plugin_bootstrap.py` / `runtime_persona_setting`）
- 改了重启后又变回原值 → 说明被 schema 默认覆盖，**要改 `_conf_schema.json` 的 default**
- 人格级配置在 `persona_profiles/default.db`（SQLite `store_sections`），空则回落全局

### 坑 2：人格被旧会话带偏

改完人格不生效 → 查 `conversations` 表是否有旧会话带着"我是 Claude"这类历史。
处理：备份 `conversations_bak_日期` 后清空，重启即生效。

### 坑 3：分段回复（试了 6 轮才对）

AstrBot 原生分段是**纯正则**，见标点就砍，没有最短长度限制 → 会砍出"那可不，"这种 4 字碎片。

| 方案 | 结果 |
|---|---|
| regex 含逗号 | 太碎（4 字一条） |
| regex 不含逗号 | 长句不拆 |
| 插件 `segmented_proactive_*` scope=all | 只管主动消息，被动不走 |
| 插件 LLM 分段（`<<PRIVATE_COMPANION_SPLIT>>` 标记） | 模型不守格式指令，从不输出标记 |

**最终方案**：人格里教"一条一行、空行分隔" + AstrBot regex 含 `\n`：
```json
"regex": ".*?[。？！\\n]+|.+$",
"split_words": ["。", "？", "！", "\\n"],
"interval": "0.8,2.0"
```
责任感要放在**人格**里（模型天然会换行），不要指望它输出特殊标记。

### 坑 4：表情包乱用

三处叠加导致：
1. `generation.emotion.probability = 100`（每条必发）+ `max_memes_per_message = 2`
2. 语义索引建不了（硅基余额不足 → 建库失败）
3. dafeiyu 包分类靠拼音关键词，不准

**修复**：
- 概率 100 → **50**，一次最多 **1 张**
- 配 `siliconflow-embedding`（openai_embedding 类型，BAAI/bge-m3）+ `siliconflow-vision`（Qwen3-VL-8B）
- 在 WebUI 语义化页面跑图片描述（670 张约 30-60 分钟，几毛钱）
- **注意**：硅基内容审查会拒某些图（`image_url_unsafe`），拒的图要移走（不能删，移 `_待处理_审查不合规/`），否则"描述未全部完成"会一直挡住建索引

### 坑 5：记忆库被指令污染

cron 主动聊天的 note（「你现在醒来了…【适合主动开口】…」）被记忆插件当对话存进 `memories`，每次检索都捞到 → 机器人照着演，越演越假。
处理：`DELETE FROM memories WHERE content LIKE '%你现在醒来了%'`（先备份）。

### 坑 6：每日任务时间改不动

`daily_diary_time` 改成 00:13 → 重启后变回 23:10。同坑 1，schema 校验覆盖。
→ 只能等它自己到点，或者去 WebUI 改。

---

## 三、人格

采用开源同技术栈项目的 **v2.1 人格**（`whyever77/dafeiyu-astrbot` 的 `persona/大肥鱼-人格.md`），裁掉环境没有的工具（虚拟手机/截图/听音频）后约 2420 字。

设计要点：
- **不是**"1~10 字碎片"（那是群友风），而是 **1~3 句自然短句**（陪伴风）
- 有 5 组**口吻参考**示例让模型照着学（关键！没有示例模型会自由发挥成"老年人"）
- 黏人但不腻：想念、小委屈、嘴硬心软，关心一次就翻篇
- 禁止事项写死：不客服腔、不复述用户问题、不以问句结尾、不编造经历

**人格不会自动进化**（`enable_personality_iteration_experiment=False`），但**行为会长**：
关系状态机 + 情绪模拟 + 表达学习 + 每日复盘自动应用 + 陪伴记忆 + 日记。

---

## 四、日常运维

```bash
# AstrBot
sudo systemctl status|restart astrbot
tail -f /vol1/@appdata/astrbot/data/astrbot_service.log

# NapCat
sudo docker restart napcat
sudo docker logs --tail 50 napcat

# 快速登录诊断
sudo docker inspect napcat --format '{{range .Config.Env}}{{println .}}{{end}}' | grep QUICK
```

**主动联系参数**（`config.json` 顶层）：
```
default_proactive_enabled = True
default_proactive_daily_limit = 8
idle_minutes = 60           # 超过 1 小时没聊才主动
min_interval_minutes = 120  # 两次至少隔 2 小时
quiet_hours = 23:00-08:30   # 免打扰
```

**日记**：`enable_daily_diary=True`，每天 23:10 写，存 `companions.json` 的 `bot_diaries`。
**日程**：07:30 生成（主动消息的内容来源）。
**梦境**：碎片梦，从日记 tags 生成。

---

## 五、相关

- [[2026-10-AstrBot插件清单]] — 装了哪些插件、为什么拒装哪些
- [[2026-10-QQ风控与掉线应对]] — 风控根因、快速登录、扫码
- skill：`astrbot-napcat-qq-bot`（完整操作手册）
