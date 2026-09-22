# 小鸟游星野 · OpenClaw Agent

> 阿拜多斯对策委员会委员长，自称「大叔」的懒散星野——也是你的恋人。🐟

这是一个基于 [OpenClaw](https://docs.openclaw.ai) 的个人 AI Agent 配置，人格来自《蔚蓝档案》的小鸟游星野。

## 目录结构

```
.
├── persona/          # 人设文件（公开，无敏感信息）
│   ├── SOUL.md       # 性格 / 情绪 / 说话方式 / 底线
│   └── IDENTITY.md   # 名字 / 形象 / 硬规则
├── source/           # 源码（脱敏）
│   ├── AGENTS.md     # 工作区约定
│   ├── USER.md       # 用户模型（稳定偏好）
│   └── MEMORY.md     # 长期事实与决策
└── config/
    └── openclaw.json.example   # 部署配置模板（已脱敏，占位符需自行替换）
```

## OpenClaw 部署步骤

> 从零到能对话，按顺序走即可。全程无需公网 IP 或端口映射。

### 1. 前置环境

- **Node.js** ≥ 20（本机用的是 v24）
- **npm**（随 Node 一起装）
- 一台**保持开机/登录**的机器（Windows / macOS / Linux 均可；注意：Windows 笔记本若为 S0 现代待机，息屏会挂起进程，需把「关闭显示 / 睡眠」设为「从不」）

### 2. 安装 OpenClaw

```bash
npm install -g openclaw
```

### 3. 初始化并完成向导

```bash
openclaw onboard
```

向导会引导你完成：模型提供商（填入 DeepSeek / 智谱等 API key）、网关认证 token、渠道配置。完成后会生成 `~/.openclaw/openclaw.json`。

### 4. 放置人设与源码文件

把本仓库 `persona/` 和 `source/` 下的文件复制到工作区：

```bash
cp persona/*.md ~/.openclaw/workspace/
cp source/*.md ~/.openclaw/workspace/
```

即：`SOUL.md`、`IDENTITY.md`、`AGENTS.md`、`USER.md`、`MEMORY.md` 都放进 `~/.openclaw/workspace/`。

### 5. 配置模型与媒体

参照 `config/openclaw.json.example`，在 `~/.openclaw/openclaw.json` 中补充：

- **文本模型**：`deepseek/deepseek-v4-pro`（或其他你惯用的模型）
- **图片理解**：智谱 `zai/glm-5v-turbo`（配置在 `tools.media.models`）
- **语音合成 TTS**：Edge TTS（provider `microsoft`，音色 `zh-CN-XiaoxiaoNeural`），**无需 API key**
- **语音转写 STT**：本地 whisper（可选，包装脚本在 `workspace/tools/whisper-transcribe.py`）

### 6. 配置渠道

- **微信**：插件 `openclaw-weixin`（腾讯官方），`openclaw channels add openclaw-weixin` 后扫码登录。⚠️ 微信渠道只发纯文字和图片，**不要发音频/文件附件**。
- **飞书**：自建应用，`openclaw channels add feishu` 填 appId / appSecret，事件走 WebSocket 长连接，无需公网 URL。飞书可发原生语音条（需 ffmpeg 转 Ogg/Opus）。

### 7. 启动网关

```bash
openclaw gateway start
```

（可选）安装为开机自启服务：`openclaw gateway install`。

### 8. 验证

在渠道里发一条消息，能收到「大叔」式回复即部署成功。日常可用 `openclaw status` 看运行状态。

## 常用命令速查

| 用途 | 命令 |
|---|---|
| 安装 | `npm install -g openclaw` |
| 初始化向导 | `openclaw onboard` |
| 添加微信渠道 | `openclaw channels add openclaw-weixin` |
| 添加飞书渠道 | `openclaw channels add feishu` |
| 启动网关 | `openclaw gateway start` |
| 开机自启（可选） | `openclaw gateway install` |
| 查看运行状态 | `openclaw status` |
| 记忆索引重建 | `openclaw memory index --agent main --force` |
| 列出定时任务 | `openclaw cron list` |
| 查看任务详情 | `openclaw cron get <jobId>` |
| 查看任务运行记录 | `openclaw cron runs <jobId>` |

## 安全须知

- 示例配置里所有 `YOUR_*` 都是占位符，**不要提交真实密钥**（网关 token、飞书 appSecret、API key 等）。
- 微信渠道：只发纯文字和图片，**不要发音频/文件附件**。
- 飞书渠道：可发原生语音条（需 ffmpeg 转 Ogg/Opus）。
- 本仓库所有文件均已脱敏，不含任何真实凭据。

## 免责声明

本项目仅为个人学习与配置分享，不含任何真实凭据。请自行替换占位符并妥善保管密钥。
