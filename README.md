# lark-cli Multi-Account Setup

[飞书 CLI](https://github.com/larksuite/cli) 多账号解决方案。适用于同时使用企业版和个人版飞书的用户。

## 问题

lark-cli v1.0.0 不支持多账号切换。`--as` 只切换 user/bot 身份，不能切换不同租户的账号。`config init --new` 会清除之前所有账号的密钥。

## 方案

利用未文档化的 `LARKSUITE_CLI_CONFIG_DIR` 环境变量，为每个账号创建独立的配置目录，通过 wrapper 脚本一键切换。

```bash
lark-work calendar +agenda        # 企业版账号
lark-personal docs +list          # 个人版账号
```

## 使用方法

将 [SETUP_GUIDE.md](./SETUP_GUIDE.md) 的内容交给你的 AI agent（Claude Code / Cursor / Codex 等），agent 会自动完成配置。

或者直接告诉你的 agent：

> 按照这个指南帮我配置飞书 CLI 多账号：https://github.com/Aojianlong/lark-cli-multi-account/blob/main/SETUP_GUIDE.md

## 相关链接

- [lark-cli 官方仓库](https://github.com/larksuite/cli)
- [Feature Request: Multi-account / Profile support (Issue #29)](https://github.com/larksuite/cli/issues/29)
