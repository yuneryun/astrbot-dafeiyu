# 🐋 大肥鱼 · AstrBot QQ 机器人配置实录

> 在 NAS（8G 内存的 x86 小主机）上养一只住在 QQ 里的蓝色小鲸鱼。
> 技术栈：**AstrBot（飞牛商店版 4.29）+ NapCat（Docker）+ DeepSeek（百炼 tokenplan）**。
> 这份实录的价值不在「装起来了」，而在**踩过的六个坑**——每一个都曾让人以为是玄学。

## 架构

| 组件 | 位置 | 说明 |
|---|---|---|
| AstrBot | 商店版，systemd 托管 | WebUI 6185 / OneBot WS 6199 |
| NapCat | Docker 容器 | WebUI 6099，`restart=always` |
| 模型 | `deepseek-v4.1-flash-1017` | 非推理版，比 `v4.1-flash` 快约一倍 |
| 表情包 | meme_manager | 5 个包、1300+ 张，带 FAISS 语义索引 |
| 活人感 | private_companion 6.7.0 | 日程/日记/情绪/关系/主动/插嘴 |

**关键认知**：AstrBot 由 systemd 管，面板显示「未运行」是假的，点启用会端口冲突。只用 `systemctl`。

## 六个坑（按踩坑代价排序）

### 1. 插件配置有四层，改错层等于没改

private_companion 的配置同时存在于四个位置，只有一个是真相源：

| 层 | 例子 | 是否生效 |
|---|---|---|
| 顶层平坦字段 | `config.json["default_proactive_enabled"]` | ✅ 运行时读这里 |
| `legacy_compat_config` | `enable_segmented_proactive_reply` | ✅ 分段/日记开关在这 |
| 子块（如 `message_debounce_config`） | `text_message_debounce_seconds` | ✅ 防抖在这 |
| `basic_config` | WebUI 显示的那份 | ⚠️ 部分被同步到顶层 |
| `_conf_schema.json` 的 `default` | — | ⚠️ **启动时覆盖上面各层** |

**诊断法**：改完 → 重启 → **读回配置文件**。值被打回就说明写错位置。
- 键在 `legacy_compat_config` → 写那里（注意实际 JSON 是扁平 dict，不是 `items` 嵌套）
- 键不在 legacy 但被打回 → **要改 schema 的 `default`**，因为 AstrBot 每次启动按 default 重写顶层

### 2. 人格不生效先怀疑旧会话

模型看到历史里自己说过「我是 Claude」会继续演。备份后清空 `conversations` 表再测。

### 3. 分段回复：试了四条路才落地

| 方案 | 结果 |
|---|---|
| 正则只按句末标点 | 长句不拆 |
| 正则加逗号 | 拆出「那可不，」4 字碎片 |
| 插件 LLM 标记分段 | 模型从不输出 `<<PRIVATE_COMPANION_SPLIT>>` |
| **人格教换行 + 正则认 `\n`** | ✅ 采用 |

```json
"regex": ".*?[。？！~…，,\\.\\!\\?\\n]+|.+$",
"split_words": ["。","？","！","~","…","，",",",".","!","?","\n"],
"interval": "1.0,2.5"
```

**教训**：分段这种事，写在人格里（模型天生会换行）比逼它输出特殊标记靠谱。

### 4. 表情包乱用是概率问题，不是分类问题

三处叠加：`probability=100`（每条必发）+ `max_memes_per_message=2` + 语义索引建不起来。

修法：概率 → 50、一次一张；配 `openai_embedding`（硅基 bge-m3）+ `openai_chat_completion` 视觉模型（Qwen3-VL-8B）；WebUI 里跑完 1300 张图的描述后自动建 FAISS 索引。

**注意**：视觉供应商有内容审查，返回 `image_url_unsafe` 的图永久失败、计数永远对不上。移走（不删）并从 `semantic_metadata.json` 的 `images` 摘掉条目，重启即自动补建。

### 5. 记忆库被指令污染

cron / future_task 的 note（「你现在醒来了…【适合主动开口】…」）被记忆插件当对话存进 `memories`，每轮检索都捞到，机器人照着演，越演越假。

```sql
DELETE FROM memories WHERE content LIKE '%你现在醒来了%' OR content LIKE '%适合主动开口%';
```

### 6. QQ 登录态会失效，密码不是万能的

- 普通掉线：`NAPCAT_QUICK_PASSWORD` 能自动重登
- **风控型掉线**（`你的用户身份已失效`）：密码会触发**短信验证**，等于把扫码换成收短信，照样要人
- 与操作系统无关：NapCat 跑的就是腾讯官方 NTQQ 的 Linux 客户端，换 Windows 只是换壳
- 真正有效：**养号**（几天不折腾）、用官方客户端登录几次、或换有年头的老号

## 插件选型：先查冲突

| 截图里常见的 | 本栈现状 | 处理 |
|---|---|---|
| 情绪管理（弗洛伊德双驱） | private_companion 有情绪模拟 | 不装 |
| 记忆日记 | 已有 `bot_diaries` | 不装 |
| 主动消息 Proactive Chat | 已有主动引擎 | 不装 |
| Anysearch 联网搜索 | 无 | ✅ 装 |
| 万能解析器（B站/抖音/小红书…） | 无 | ✅ 装 |
| qq_tools（引用/撤回/群管理） | 无 | ✅ 装 |

判断标准是「这个能力本栈有没有」，不是「插件是不是热门」。

## 人格：两条路线别混用

- **群友碎片风**：`K0nd1us/qq-agent` 的「小鲸鱼」——1~10 字、允许只回「？」「草」、傲娇爱搭不理
- **黏人陪伴风**：`whyever77/dafeiyu-astrbot` 的 v2.1——1~3 句自然短句、有 5 组口吻示例、撒娇想念吃醋

私聊陪伴选后者。把碎片风硬套到陪伴上，就是「老气/淡淡」的来源。

**人格不会自己进化**（实验开关默认关），但行为会长：关系状态机、情绪模拟、表达学习、每日复盘、陪伴记忆、日记。

## 目录

- `notes/2026-10-AstrBot大肥鱼配置实录.md` — 完整实录
- `notes/2026-10-AstrBot插件清单.md` — 插件与依赖
- `notes/2026-10-QQ风控与掉线应对.md` — 掉线记录与应对

## 致谢

- [AstrBot](https://github.com/AstrBotDevs/AstrBot)
- [NapCat](https://github.com/NapNeko/NapCatQQ)
- [private_companion](https://github.com/menglimi/astrbot_plugin_private_companion)（「我会永远陪着你」）
- [whyever77/dafeiyu-astrbot](https://github.com/whyever77/dafeiyu-astrbot)（人格来源）
- [meme_manager](https://github.com/anka-afk/astrbot_plugin_meme_manager)
- 大肥鱼形象：DeepSeek 蓝鲸的社区二创，非官方
