# 调研报告

调研日期：2026-11-06

## 1. 结论

可行。已有工具要么是通用密钥管理（重、偏团队），要么只管 LLM 供应商切换 / 余额，**没有一个面向个人开发者、覆盖"所有平台 key 的清单 + 按用途分发给 AI Agent"的轻量 macOS 工具**。

## 2. 竞品

### 2.1 通用密钥管理

| 工具 | 做法 | 不足 |
|---|---|---|
| 1Password CLI（`op run`） | `.env` 写 `op://` 引用，运行时注入子进程，stdout 自动脱敏 | 付费订阅；每次解密弹 Touch ID，Agent 连续调用体验差；没有"用途/在哪用"的清单视角 |
| Infisical / Doppler / Vault | 团队级 secret 管理 | 偏服务端和团队，个人使用过重 |
| direnv / dotenvx | 按目录加载 / 加密 `.env` | 仍是一项目一份，不解决分散问题 |

### 2.2 面向 AI Agent 的 key 工具

| 工具 | 做法 | 不足 |
|---|---|---|
| [MCP Manager](https://github.com/kilabyte/mcp-manager) | Keychain 存储，`launchctl setenv` 注入 GUI 应用 | 只管 MCP |
| [akm](https://docs.rs/api-key-manager) | Keychain CLI，附 Claude Code/Codex skill | 只有 CLI，无清单管理界面 |
| NoxKey | Keychain + MCP + 每次 Touch ID | 闭源；直接把明文返回给 Agent |
| [Locksmith](https://github.com/lorem-dev/locksmith) / Keiko | MCP 启动时注入、输出脱敏 | 部署偏重 |

### 2.3 LLM 供应商切换 / 余额监控（国内，知乎调研）

| 工具 | 做法 | 与本项目关系 |
|---|---|---|
| CC-Switch | 管理 Claude Code/Codex/Gemini CLI 的供应商 key 并一键切换，内置余额查询 | 只管 LLM 供应商；**不做重叠功能** |
| ClaudeKit | 内置终端，把变量写进 `.zshrc` | 同上，且明文落盘 |
| HeyQuota | Tauri 2 + Rust，10 个渠道余额适配器 | 额度查询插件可参考其 Provider 设计 |
| 积分泡泡 | Windows 桌面余额悬浮窗，key 存凭据管理器 | 仅 Windows、仅余额 |

## 3. 关键技术事实

- macOS Keychain 可通过 `security` 命令或 Security.framework 访问；已有 skill（如 last30days）约定从 Keychain 读 key，说明统一命名后有机会被其他工具复用。
- 注入环境变量无法完全防止 Agent 读到 key（能执行命令就能 `env`），只能**缩小暴露面**：按用途最小注入、输出脱敏、调用审计。
- 余额查询：DeepSeek（`/user/balance`）等少数平台有接口；OpenRouter 需 Management Key；OpenAI/Anthropic 需 Admin Key；知乎、企查查这类平台大概率没有 → 额度查询只作为可选插件。

## 4. 差异化定位

| 维度 | 现有工具 | keyhub |
|---|---|---|
| 范围 | LLM 供应商 或 单一 MCP | 所有平台 key |
| 视角 | 存储 / 切换 | **清单**：注册了什么、谁在用、何时到期 |
| 分发 | 全局变量或改配置 | 引用文件 + `run` 按需注入 |
| 迁移 | 手动 | 扫描已有 `.env` 一键导入 |

## 参考

- 1Password：[Securing MCP servers](https://1password.com/blog/securing-mcp-servers-with-1password-stop-credential-exposure-in-your-agent)
- [DeepSeek 查询余额](https://api-docs.deepseek.com/zh-cn/api/get-user-balance.md)
- 知乎：[CC-Switch](https://www.zhihu.com/question/2079933712536547501/answer/2079935249090343469) · [HeyQuota](https://zhuanlan.zhihu.com/p/2083232233058464442) · [积分泡泡](https://zhuanlan.zhihu.com/p/2099337543128987234) · [密钥托管与最小权限](https://zhuanlan.zhihu.com/p/2090152808620226558)
