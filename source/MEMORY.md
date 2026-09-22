# MEMORY.md - Durable Facts & Decisions

长期有效的技术决定和事实。不写日常琐事（那些进 `memory/YYYY-MM-DD.md`）。

## 渠道

- **微信**：`openclaw-weixin` 2.4.8（腾讯官方插件），扫码登录。**账号 id 每次重新扫码都会变**（当前 `ACCOUNT_ID-im-bot`，历史：`ACCOUNT_ID_OLD` → `ACCOUNT_ID_OLD` → `ACCOUNT_ID_OLD`）。旧凭证备份在 `~/.openclaw/openclaw-weixin/backup-*`。
- **飞书**：自建应用 `FEISHU_APP_ID`（应用名「星野」），事件走 WebSocket 长连接，不需要公网 URL。
- 微信插件的 capabilities 只声明 `chatTypes: ["direct"] + media`——**没有群聊、没有语音通话**，这是平台边界。

## 模型分工

- **文本对话**：DeepSeek `deepseek/deepseek-v4-pro`。
- **图片理解**：智谱 `zai/glm-5v-turbo`（配置在 `tools.media.models`，capabilities `["image"]`）。
- 注意：`glm-5.3-flash` 虽然支持图片输入，但它是强制思考模型，媒体理解链路默认关思考会被智谱以 400 拒绝，所以图片用 `glm-5v-turbo`。

## 语音

- **转录（STT）**：本地 whisper `small` 模型，包装脚本 `workspace/tools/whisper-transcribe.py`。
  - whisper 依赖 ffmpeg；imageio-ffmpeg 的二进制被复制成 `~/.openclaw/bin/ffmpeg.exe` 并加入用户 PATH。
  - 用 `--initial_prompt "以下是普通话的句子。"` 强制输出简体（whisper 默认吐繁体）。
- **合成（TTS）**：Edge TTS，provider `microsoft`，音色 `zh-CN-XiaoxiaoNeural`，rate -5%、pitch -3%。**不需要 API key**。
- `tts.auto = inbound`：老师发语音才用语音回。
- **飞书能发原生语音条**（需 ffmpeg 转 Ogg/Opus）；**微信禁止发任何音频/文件附件**——插件 outbound 没有 voice_item 实现，会把 mp3 当「文件附件」发出（协议里本应是 VOICE 类型，属类型错配），2026-09-20 排查时标为疑似触发微信侧下行静默拦截。硬规则已写进 `IDENTITY.md` / `USER.md`。

## 主动消息

- Heartbeat：`every 3h`，`activeHours 09:00-23:00`，`target: last`（发给最近联系人），自定义 prompt 让她有由头才发。
- **前置条件：机器不能处于待机**。定时器在系统挂起期间不会触发，心跳只能等系统唤醒后补跑一次。这条是"从没收到过主动消息"的**主因**，详见「常开与电源」。
- **已知偶发故障**：机器刚唤醒后的**补跑**那一次，收件人可能被写成 `user:o9cq808...@im.wechat`（多了 `user:` 前缀），微信返回 `sendMessage ret=-3 errmsg=invalid arguments`，cron 记为 `failed` 并进入 30 秒退避。正常会话内/正常时段的主动发送用不带前缀的 `o9cq808...@im.wechat`，是好的。
- **排查入口**：`openclaw cron list` / `openclaw cron get <jobId>` / `openclaw cron runs <jobId>`；心跳任务的 `declarationKey` 是 `heartbeat:main`。注意 cron 配置**已不在** `~/.openclaw/cron/jobs.json`（该路径只是日志里的 storeKey，文件已不存在）。

## 记忆检索

- Embedding 走 **智谱** `embedding-3`（2048 维），配置在 `memory.search`：
  `provider: openai-compatible` + `remote.baseUrl: https://open.bigmodel.cn/api/paas/v4` + 智谱 key。
  - 原来默认走 `openai/text-embedding-3-small`，但本机连不上 OpenAI，导致索引降级成 `fts-only`、向量检索被暂停。
- 索引重建命令：`openclaw memory index --agent main --force`（改 provider 后必须重建，并重启网关）。
- 注意：CLI 的 `openclaw memory search` 对中文只做字面匹配（「猫」「怕生」搜不到），但**对话里 agent 用的向量检索是正常的**（「小动物」能拐到「猫」）。

## 微信下行暂停事件（2026-09-20，已闭环）

**现象**：约 19:00 起，机器人回复在服务端全部返回 `ret=0`，但用户手机一条都收不到。上行（用户→机器人）全程正常，用户收朋友的普通微信消息也正常。

**实际原因**：**不是拦截、也不是丢弃，而是微信侧把下行消息压进了队列**。约 1.5 小时后（20:34）队列整体补投，用户「一次性涌进来一串」。所以服务端 `ret=0` 是诚实的——它收下了，只是排在队里。

**为什么会暂停：仍未查明。** 微信对 `sendmessage` 只返回 `ret`，没有任何投递状态或原因字段，客户端无从判断。以上是**观察到的行为**，不是根因结论。

**关键判据**：问用户「是一次性涌进来一串吗」。
- 答「是」→ 排队补投（等就行）
- 答「只有最新那条到了」→ 那才是真拦截

**处置：等，不要重新绑定。** 本次为排查产生了 4 个机器人身份（`ACCOUNT_ID_OLD` → `ACCOUNT_ID_OLD` → `ACCOUNT_ID_OLD` → `ACCOUNT_ID`）、清本地 token 重扫 ×3、微信侧主动解绑 ×1、网关多次重启 —— **全部无效**，恢复是它自己到点。频繁重绑在微信看来属异常行为，有延长限制的风险。

**已排除项（均有硬证据，下次不必重查）**：发送令牌、绑定关系、机器人身份、接入点 `baseurl`、网关进程、会话/上下文令牌、下行条数配额（当天 101 inbound : 96 outbound，严格 1:1，无超额）。

**唯一会明确报错的场景**：`errcode -14`（令牌失效）。实测只在「用户在微信里主动解绑机器人」之后出现；插件会因此暂停该账号的所有请求 60 分钟（内存态，重启网关即清）。

**排查工具**（留在 `%TEMP%`）：
- `weixin-probe.mjs`：用保存的令牌直连 iLink（`notifystart` / `getconfig` / `sendmessage`）打印**原始响应**，用来判断服务端到底认不认这条链路。
- `wx-qr.mjs`：把扫码链接渲染成白底大图 HTML，比终端字符二维码好扫。

**次日复核（2026-09-21 18:22–18:24）**：上行 3 : 下行 3，三条往返全部成功。

**补充验证：换网络不是诱因（2026-09-21 19:56–19:58）。** 用户实测三种网络状态各发一条，我这边读日志核对：

| 网络状态 | 上行 / 下行 | 端到端耗时 |
|---|---|---|
| 移动数据（关 Wi-Fi） | 19:56:19 → 19:56:26 | 6.9 s |
| Wi-Fi | 19:56:55 → 19:56:59 | 4.5 s |
| 飞行模式 20 秒后重连 | 19:58:11 → 19:58:14 | 3.0 s |

三条全部成功、**零报错**（无 `ret=-3`、无 `getUpdates` 超时），用户确认三条回复都实际收到。

**结论：Wi-Fi ↔ 流量切换、强制重连都不会打断微信链路** —— 以后再怀疑"网络/异地导致断了"，可以直接排除这一项。

两点读数注意：
- 表的「端到端耗时」**包含模型推理时间**，不等于网络时延，**不能用来比较网络质量**。
- 三次样本量很小，只能证明"这三次没问题"；**物理意义上的异地（换城市）无法在本地模拟**，只能等实际出门时验证。

## 常开与电源（2026-09-21 排查）

**结论：网关"掉线"不是关机，是系统挂起。**

**根因**：这台笔记本是 **S0 现代待机（连接待机）** 机器 —— **屏幕一黑，系统立刻进待机，桌面（Win32）进程被挂起**。而原来的「关闭显示」超时是**插电 5 分钟 / 电池 3 分钟**，所以人一离开，网关几分钟后就冻住了。

**铁证**（网关自己的看门狗日志）：
```
liveness heartbeat delayed: overdue=64802541ms    → 冻了 18.0 小时
liveness heartbeat delayed: overdue=12827949ms    → 冻了 3.56 小时
```
同时系统已连续运行 50+ 小时**从未重启**（`Win32_OperatingSystem.LastBootUpTime`）→ 排除关机/重启的可能。

**排查中走过的弯路（别再走）**：
- ❌ 只改「睡眠超时」**方向不对**：S0 机器上**息屏才是待机触发点**，睡眠超时形同虚设。
- ❌ 一度误判为"机器没开机" —— 实际是机器开着但睡过去了（日志出现空档 ≠ 关机）。

**已做的修改**（当前电源计划，插电+电池都改）：

| 设置 | 之前 | 现在 |
|---|---|---|
| 在此时间后关闭显示 `VIDEOIDLE` | AC 5 分钟 / DC 3 分钟 | **从不** |
| 在此时间后睡眠 `STANDBYIDLE` | AC 从不 / DC 3 分钟 | **从不** |
| 在此时间后休眠 `HIBERNATEIDLE` | 从不 / 从不 | 从不 |

**代价**：**屏幕会一直亮着**。在 S0 机器上「屏幕可熄灭」与「进程不被挂起」是互斥的。

**仍会挂起的情况**：手动「开始」→ 睡眠（`UIBUTTON_ACTION=睡眠`）；合盖（该动作未通过 powercfg 暴露，查不到也改不了）。S3 不可用（被 Device Guard 禁用），所以无法改成"屏幕可灭、系统不挂起"。

**回滚**：`powercfg /change monitor-timeout-ac 5` + `powercfg /change monitor-timeout-dc 3`。

**验证方法**：看日志里是否还出现 `liveness heartbeat delayed`；以及 `openclaw cron runs <heartbeat jobId>` 是否按时（每 3h）产生记录。

## 已知限制（踩过的坑）

- **微信下行可能整段暂停并排队补投**（见上一节）。症状是「服务端 `ret=0` 但一条都收不到」时，**先等 1–2 小时，不要重绑**。
- 本机网络：`api.openai.com`、`generativelanguage.googleapis.com`、`huggingface.co` **不通**；`raw.githubusercontent.com` 不通；GitHub release 下载极慢。
- 飞书用 open_id 主动单聊会被拒（错误码 **230101**），改用 chat_id（`oc_...`）可以发。
- 微信主动发消息时没有 contextToken（日志有 warning），能发出但可靠性不如会话内回复。
- 网关是 Windows 计划任务，**仅在「用户登录时」触发**（`LogonType=Interactive`）——**没有「开机时」触发器、也没开自动登录** → 重启后必须手动登录一次才会起来；注销会停（锁屏不影响）。`openclaw gateway install` 也不提供这两个能力。
- **这台笔记本是 S0 现代待机机器，息屏即进待机、桌面进程被挂起** —— 这是网关掉线的真正原因，已于 2026-09-21 把「关闭显示」和「睡眠」都设为「从不」。详见「常开与电源」。

## 运维脚本（源码在 `E:\ai`）

四个自己写的运维脚本，**源码统一放 `E:\ai`**（**不在** `~/.openclaw/tools` —— 那个目录是临时建的，已删除）；日志仍写在 `~/.openclaw/logs/`。这样代码在 E 盘、状态留在 `.openclaw`，换盘或重装都不丢历史记录。

| 文件 | 用途 | 常用命令 |
|---|---|---|
| `keep-awake.ps1` | 持有 SYSTEM 唤醒请求，防系统进 S0 待机把网关挂起；内置「本进程是否被挂起过」检测 | `powershell -File E:\ai\keep-awake.ps1 [-KeepDisplay] [-RunSeconds N]` |
| `weixin-pulse.ps1` | 投递探针：定时发**带时间戳**的微信消息，链路恢复/中断时可对照时间 | `... weixin-pulse.ps1 -IntervalMinutes 30 -MaxRuns 8`；`-DryRun` 只看不发 |
| `healthcheck.ps1` | 一键体检：网关/渠道 · 电源挂起风险 · 计划任务与自动登录 · 当日挂起检测 · 心跳任务 · 记忆索引 · 日志错误分类 + 汇总 | `... healthcheck.ps1 [-SkipSlow]` |
| `ilink-client.mjs` | 直连 iLink 协议（不经插件、不跑 agent），打**原始响应**；子命令 `whoami` / `send` / `listen` / `notify` / `raw` | `node E:\ai\ilink-client.mjs whoami` |

**写这类脚本踩过的三个坑（本机只有 Windows PowerShell 5.1，没有 pwsh）**：
1. **`.ps1` 必须存成 UTF-8 带 BOM**，否则 5.1 会把中文按 GBK 读成乱码。转换写法：`[System.IO.File]::WriteAllText($f, $t, (New-Object System.Text.UTF8Encoding($true)))`
2. `0x80000000` 会被 PowerShell 解析成 Int32 的 `-2147483648`，再转 `[uint32]` 会抛 "too large or too small" —— 字面量要写十进制 `2147483648`，或加 `L` 后缀。
3. 另外：`if` 不能直接当函数参数（`f(if (...) {...})` 在 5.1 不合法），要包一层 `$(...)`。

## 相关文件

- 人格：`SOUL.md`（性格/情绪/内心）、`IDENTITY.md`（名字形象）
- 用户模型：`USER.md`
- 梦境日记：`DREAMS.md`，原始语料在 `memory/.dreams/session-corpus/`
