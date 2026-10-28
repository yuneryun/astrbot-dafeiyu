---
created: 2026-10-28
tags: [AI技术, AstrBot, 插件]
---

# 2026-10 AstrBot 插件清单

关联：[[2026-10-AstrBot大肥鱼配置实录]]

## 已装

| 插件 | 版本 | 作用 |
|---|---|---|
| `astrbot_plugin_private_companion` | 6.7.0 | **核心**：陪伴引擎（日程/日记/情绪/关系/主动/插嘴） |
| `astrbot_plugin_meme_manager` | 5.2.8 | 表情包（dafeiyu 536 张 / official-001 397 / AstrBot娘 208 / ddzs987 / deepseek-fanart） |
| `astrbot_plugin_shared_chat_memory` | 1.2.0 | 跨会话记忆（BM25 + bge-m3 向量） |
| `astrbot_plugin_anysearch` | 0.3 | 实时联网搜索（免费匿名额度） |
| `astrbot_plugin_parser` | 1.6.1 | 万能链接解析（B站/抖音/小红书/微博/快手/YouTube/TikTok/X/网易云…） |
| `astrbot_plugin_qq_tools` | 0.2.1 | 引用回复/撤回/群管理/定时唤醒/头像/网页浏览 |
| `astrbot_plugin_profile` | — | QQ 资料管理（头像/昵称/签名） |
| `astrbot_plugin_qzone` | — | QQ 空间（发说说/点赞/评论） |

## 故意不装（与 private_companion 功能重叠，装了会打架）

| 图里看到的 | 为什么拒装 |
|---|---|
| 情绪管理（弗洛伊德双驱情绪模型） | private_companion 有情绪模拟 + 关系状态机 |
| 记忆日记 | private_companion 每天 23:10 自动写 |
| 主动消息 Proactive Chat | private_companion 有主动引擎（8 次/天） |
| QQ资料管理 | 已装 `astrbot_plugin_profile` |
| QQ空间 | 已装 `astrbot_plugin_qzone` |

## 想要但 NAS 装不了

| 功能 | 原因 |
|---|---|
| 虚拟手机逛 App（看 B 站/操作安卓） | 需要安卓模拟器 + GUI，NAS 无桌面 |
| 截图（`get_screenshot`） | 需要 Windows 图形会话 |
| 听音频（`listen_audio`） | 需要声卡 + loopback 录音 |

这些只有在 Windows 本机跑机器人才能用。NAS 上放弃。

## 依赖备忘

- parser 需要：`tqdm curl_cffi msgspec apilmoji bilibili-api-python yt-dlp`
- qq_tools 需要：`playwright`（只装了 pip 包，Chromium 未装 → 网页浏览功能暂不可用）
- private_companion 需要：`chinese-calendar lunarcalendar`
- 表情语义索引需要：embedding provider（硅基 bge-m3）+ vision provider（Qwen3-VL-8B）
