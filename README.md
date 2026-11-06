# keyhub

> 面向 AI Agent 的本地 API Key 中心（macOS）。所有 key 只录一次，人能看清楚，Agent 能按需取用，项目里不再散落明文 `.env`。

**状态：设计阶段，尚未开始编码。**

## 解决什么问题

AI Agent 用到的 MCP、Skill、脚本越来越多，每个都要 API key（知乎、企查查、Tavily、高德……）。现状是：

- key 散落在各项目的 `.env`、`~/.zshrc`、各种 MCP 配置 JSON 里，靠 `.gitignore` 防泄漏；
- 同一个 key 复制多份，轮换时容易漏；
- 记不清自己到底注册过哪些平台、哪个 key 给了谁用。

keyhub 要做两件事：

1. **清单** —— 一个地方看清所有 key：平台、注册账号、用途、在哪用、到期时间；
2. **供给** —— Agent / 项目 / MCP 需要 key 时从 keyhub 取，而不是各自持有一份明文。

> 不做的事：不做 LLM 供应商切换（那是 [CC-Switch](https://github.com/farion1231/cc-switch) 这类工具的领域）。

## 设想中的用法

```bash
# 列出所有 key（只有元数据，不含值）—— Agent 先用这个了解有哪些 key
keyhub list

# 项目里用可提交到 git 的引用文件替代 .env
cat .env.ref
# ZHIHU_API_KEY=keyhub://zhihu
# QCC_API_KEY=keyhub://qcc

# 运行时注入，不落盘
keyhub run -- npm start

# 只给某个命令注入指定 key（最小授权）
keyhub run --keys zhihu,qcc -- python agent.py

# 兼容必须读 .env 的老项目
keyhub env > .env

# 扫描已有的 .env 文件并导入
keyhub import ~/Work
```

MCP 配置里不再写明文：

```json
{
  "mcpServers": {
    "qcc": {
      "command": "keyhub",
      "args": ["run", "--keys", "qcc", "--", "npx", "qcc-mcp"]
    }
  }
}
```

## 文档

- [调研报告](docs/research.md) —— 竞品与可行性
- [设计文档](docs/design.md) —— 数据模型、架构、安全
- [CLI 规格](docs/cli.md) —— 命令定义
- [路线图](docs/roadmap.md) —— MVP 与后续计划

## License

[MIT](LICENSE)
