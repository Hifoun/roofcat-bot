# QQ 群聊 AI Bot 复刻说明（脱敏版）

## 目标

实现一个基于 QQ 群聊的 AI bot，核心目标不是"问答助手"，而是一个像群友一样自然接话、会看图、会发本地表情包、能记住短期上下文、能参与连续对话的角色 bot。

bot 的最终行为标准：

- 像群友，不像客服/模型
- 不复述题干，不解释思考过程
- 接梗、吐槽、糊弄、短句优先
- 能处理群名片、别名、回复链和连续对话
- 图片先本地识别，再把识图结果作为上下文交给文本模型，而不是直接把识图描述发出来
- 所有服务可一键启动/关闭

---

## 技术栈

### QQ 协议层

使用：

- NapCat
- OneBot v11
- 反向 WebSocket

连接方式：

```text
NapCat → ws://127.0.0.1:8080/onebot/v11/ws → NoneBot
```

### Bot 框架

使用：

- NoneBot2
- nonebot-adapter-onebot
- fastapi
- httpx
- websockets

NoneBot 监听：

```text
HOST=127.0.0.1
PORT=8080
DRIVER=~fastapi+~httpx+~websockets
```

### 文本模型

使用 DeepSeek Chat API。

要求：

- API Key 不硬编码，从环境变量读取
- 模型：`deepseek-chat`
- `temperature=0.7`
- `max_tokens=200`

### 图像模型

本地部署 Qwen3-VL 2B：

```text
Qwen/Qwen3-VL-2B-Instruct
```

服务形式：

```text
FastAPI
127.0.0.1:8090
POST /describe
```

输入图片，返回中文识图描述。

GIF 处理：

- 如果图片是 GIF，抽 4 帧：
  - 首帧
  - 1/3
  - 2/3
  - 末帧
- 每帧缩放到固定大小
- 横向拼接成一张图
- 再送入 QwenVL

---

## 目录结构建议

```text
BotRoot/
├── bot.py
├── .env
├── manage.ps1
├── memes/
├── docs/
│   ├── behavior-model.md
│   ├── architecture.md
│   └── modules.md
└── src/
    └── plugins/
        ├── ai_chat.py
        ├── basic.py
        └── meme.py

QwenVLRoot/
├── server.py
├── manage.ps1
├── models/
│   └── Qwen3-VL-2B-Instruct/
└── tmp/
```

---

## 服务管理

实现一个统一 `manage.ps1`，管理三个进程：

```text
NapCat
NoneBot
QwenVL
```

支持命令：

```powershell
.\manage.ps1 start      # 启动全部
.\manage.ps1 stop       # 关闭全部
.\manage.ps1 restart    # 重启全部
.\manage.ps1 status     # 查看状态

.\manage.ps1 bot        # 只启动 NoneBot
.\manage.ps1 napcat     # 只启动 NapCat
.\manage.ps1 qwen       # 只启动 QwenVL

.\manage.ps1 stopbot
.\manage.ps1 stopnap
.\manage.ps1 stopqwen
```

要求：

- 每个服务独立进程运行
- PID 写入 `.pids/`
- 终端关闭不影响服务
- 之后可用 `stop` 清理所有进程

---

## 插件模块

### basic.py

实现基础命令：

```text
/ping → pong!
/echo <内容> → 原样返回
/help /帮助 /menu → 帮助菜单
```

所有命令 `block=True`，避免穿透到 AI。

---

### meme.py

实现本地表情包模块。

目录：

```text
memes/
```

支持格式：

```text
.jpg
.jpeg
.png
.gif
.webp
```

核心函数：

```python
has_memes() -> bool
pick_meme() -> str | None
```

发送时使用：

```python
MessageSegment("image", {"file": meme_path, "sub_type": "1"})
```

`sub_type=1` 用来尽量显示成 QQ 表情包风格。

---

## AI 核心模块 ai_chat.py

### 全局配置

需要配置（占位符替换）：

```python
BOT_SELF_ID = "<BOT_QQ_ID>"
BOT_ALIASES = ["昵称A", "昵称B", "昵称C"]
TARGET_USER_ALIASES = {
    "别名X": "<USER_QQ_ID>",
}
RANDOM_REPLY_RATE = 0.20
ACTIVE_CONV_TIMEOUT = 120
ACTIVE_CONV_MAX_REPLIES = 3
```

所有真实 QQ 号应从配置文件或环境变量读取。

---

## 角色提示词

角色是一个群友型 bot，不是助手型 bot。

核心人设：

```text
你是群名片上的名字，也叫你的其他别名。
别人 @你 或在说「你的名字/昵称」就是 @你。
```

硬性规则：

```text
- 别人说「你名字/你昵称/你前半段/你后半段」默认指当前群名片
- 群名片可能是一整句调侃，要整体理解
- 回复里不要复述对方原文的关键词或句子
- 不要用「先X再Y」「我先X」「那得先X」展示思考过程
- 多个关联信息点要整体接住，不只抓一个词
- 「如何评价X」当群友抛梗，不当正式分析题
- 别人让你做轻度玩笑动作时，直接执行
```

玩笑动作例子：

```text
帮我咬他 → 汪！汪汪
你骂他两句 → 你等着
快救我 → 来了
把某人抓起来 → 已逮捕
你去揍他 → 我先凶一下
```

输出风格：

```text
- 每次 30 字以内
- 少标点
- 不长篇分析
- 不上价值
- 不解释梗来源
- 不评价自己的回复
```

---

## 消息记录模块

维护最近群聊记录：

```python
_msg_records: dict[group_id, deque(maxlen=10)]
```

每条记录结构：

```python
{
    "user_id": int,
    "name": str,
    "text": str,
    "reply_to": {
        "user_id": int,
        "name": str,
        "text": str,
    } | None,
}
```

所有群消息都由低优先级 logger 记录：

```python
on_message(priority=0, block=False)
```

记录内容包括：

- 文本
- @对象
- 图片占位 `[图片]`
- 回复链关系

bot 自己发出的消息也要手动写入记录，因为 OneBot 不一定回流 bot 自己的发送事件。

---

## 上下文构建模块

每次调用 DeepSeek 前构建 `member_context`。

上下文内容顺序：

```text
1. 当前群身份
2. 最近群聊
3. 图片识别结果
4. 当前提到的群成员
5. 连续对话提示
6. 当前用户消息
```

示例：

```text
你的本群身份：
- 群名片：<当前群名片>
- 别人说「你名字」「你昵称」「你前半段」「你后半段」默认指这个群名片
- 群名片是一整句话，含义由全部词组共同构成

最近群聊：
A(...): ...
B(...): ...

当前图片识别结果（只当聊天上下文，不要照抄描述）：
图里是一只模糊的小狗，像是在张嘴叫

当前用户消息：
@你 这图怎么样
```

关键要求：

- 图片识别结果只作为上下文
- 不直接把 QwenVL 原始描述发群
- 最终回复必须由 DeepSeek 按角色生成

---

## 身份模块

需要实现：

### 自己身份识别

```text
BOT_SELF_ID
BOT_ALIASES
SELF_ALIASES
```

所有以下内容都指 bot 自己：

```text
所有配置的别名
当前群名片
```

### 群名片优先

如果别人说：

```text
你名字
你昵称
你前半段
你后半段
```

默认指 bot 当前群名片，而不是固定网名。

### 群成员缓存

通过：

```python
bot.get_group_member_list(group_id=...)
```

建立缓存：

```python
_member_cache[group_id][card_or_nickname_lower] = member_info
```

用于：

- 名字解析
- 群成员提及识别
- 别名归一化

---

## 触发规则

### 强触发

以下情况 100% 进入 AI 或图片处理：

```text
@bot
提到 bot 别名
回复 bot
回复图片并 @bot
连续对话窗口内发言
```

### 随机触发

普通群聊文字：

```text
20% 概率触发
触发后 50% AI / 50% 表情包
实际约 10% AI，10% 表情包
```

### 纯图非 @

```text
群聊非 @ 纯图：10% 发本地表情包
私聊纯图：100% 发本地表情包
```

---

## 图片处理模块

### 1. @bot + 纯图

优先级高，block。

```text
60%：
    下载图片
    QwenVL 识图
    把识图结果写入 DeepSeek 上下文
    DeepSeek 生成角色口吻回复
    引用原图发送

40%：
    引用原图
    发本地表情包
```

### 2. 回复图片并 @bot

同样：

```text
60% 自己识图 + DeepSeek
40% 发本地表情包
```

区别：

- 引用的是被回复的原图片消息
- 不是引用当前 @bot 消息

### 3. 文字 + 图

如果消息里有文字和图：

```text
不直接走 60/40
自动识别图片
把图片描述加入 member_context
正常调用 DeepSeek
```

### 4. 回复链有图

如果当前消息文本触发 AI，且回复链里有图：

```text
自动识别被回复图片
把图片描述加入 member_context
正常调用 DeepSeek
```

### 5. GIF

QwenVL 服务端处理：

```text
抽 4 帧 → 拼图 → 识别
```

---

## 连续对话窗口

维护：

```python
_active_convs[group_id] = {
    "participants": set[int],
    "last_active": timestamp,
    "reply_count": int,
}
```

开启条件：

```text
bot 回复某人
bot @某人
bot 主动拉某人进话题
```

生效规则：

```text
2 分钟内
参与者继续说话
即使没 @bot、没提 bot 名字
也视为可能在继续和 bot 对话
```

限制：

```text
最多自动接 3 轮
防止刷屏
```

上下文里标注：

```text
这是连续对话中的被激活回复，对方没有喊你名字但仍在对话中
```

---

## 空消息处理

如果用户 `@bot` 但没有文字：

### 回复链有图

```text
走图片处理 60/40
```

### 回复链有文字

```text
提取被回复文字
构建上下文
调用 DeepSeek
```

### 无回复链

```text
构建最近群聊上下文
用类似「@bot但什么都没说」作为 user_text
让 DeepSeek 判断怎么接
```

失败时 fallback：

```text
发本地表情包
```

---

## 表情包场景

最终保留的表情包发送场景：

```text
1. 私聊纯图：100%
2. 群聊非 @ 纯图：10%
3. 普通群聊随机触发后：约 10%
4. @bot 空文本 DeepSeek 失败 fallback
```

不再使用表情包的场景：

```text
@bot + 纯图
@bot + 文字+图
回复图片并 @bot
```

这些场景走 QwenVL + DeepSeek 或发本地表情包。

---

## QwenVL 服务

接口：

```http
GET /health
POST /describe
```

`/describe` 输入：

```multipart
file=<image>
prompt=<text>
```

输出：

```json
{
  "text": "图片描述"
}
```

默认 prompt：

```text
用中文简短描述这张图，重点说图里有什么和可能的梗。
```

服务实现要求：

- 模型启动时加载到 GPU
- 图片临时保存后读取
- GIF 抽帧拼图
- 识别完成后删除临时文件
- 服务只监听 `127.0.0.1`

---

## 事件优先级

建议：

```python
_logger    priority=0 block=False
_img_meme  priority=3 block=True
ai_chat    priority=5 block=True
```

含义：

```text
logger 永远先记录
纯图强处理优先于 AI
AI 最后处理普通文本/上下文
```

---

## 安全与隐私要求

文档和代码交付时不得暴露：

```text
真实 QQ 号
DeepSeek API Key
真实群号
真实昵称和群名片
真实本地用户名
私有图片路径
```

所有敏感内容用占位符：

```text
<BOT_QQ_ID>
<USER_QQ_ID>
<DEEPSEEK_API_KEY>
<LOCAL_PROJECT_PATH>
```

API Key 应该：

```text
从环境变量读取
不要硬编码
不要写入文档
不要打印日志
```

---

## 复刻验收标准

另一个 agent 复刻完成后，应满足：

### 文本行为

```text
@bot 能自然短句回复
不会像客服
不会复述题干
不会长篇分析
能接玩笑动作请求
能理解当前群名片
```

### 图片行为

```text
@bot 发图：
60% 自己看图并用角色口吻回复
40% 发表情包

文字+图：
图会进入上下文
最终回复不是原始识图描述
```

### 上下文行为

```text
回复链文字能被理解
回复链图片能被识别
连续对话不需要反复 @bot
2 分钟内可自动接话
```
