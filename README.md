# 小鸟游星野 · OpenClaw Agent

> 阿拜多斯对策委员会委员长，自称「大叔」的懒散星野——也是你的恋人。🐟

这是一个基于 [OpenClaw](https://docs.openclaw.ai) 的个人 AI Agent 配置，人格来自《蔚蓝档案》的小鸟游星野。

## 目录结构

```
.
├── persona/          # 人设文件（公开，无敏感信息）
│   ├── SOUL.md       # 性格 / 情绪 / 说话方式 / 底线
│   └── IDENTITY.md   # 名字 / 形象 / 硬规则
└── config/
    └── openclaw.json.example   # 部署配置模板（已脱敏，占位符需自行替换）
```

## 快速开始

1. 安装 OpenClaw，完成 `onboard`；
2. 把 `persona/` 下的文件放到你的 workspace（`~/.openclaw/workspace/`）；
3. 参照 `config/openclaw.json.example`，把 `YOUR_*` 占位符换成你自己的密钥后写入 `~/.openclaw/openclaw.json`；
4. 重启网关。

## 安全须知

- 示例配置里所有 `YOUR_*` 都是占位符，**不要提交真实密钥**（网关 token、飞书 appSecret、API key 等）。
- 微信渠道：只发纯文字和图片，**不要发音频/文件附件**。
- 飞书渠道：可发原生语音条（需 ffmpeg 转 Ogg/Opus）。

## 免责声明

本项目仅为个人学习与配置分享，不含任何真实凭据。请自行替换占位符并妥善保管密钥。
