# 路线图

## v0.1 MVP（CLI 优先）

先做 CLI，验证"清单 + 供给"闭环，再补界面。

- [ ] KeyHubCore：模型、Keychain 读写、JSON 元数据存储
- [ ] `add` / `list` / `show` / `set` / `rm`
- [ ] `get` / `run`（含 `.env.ref` 解析）/ `env`
- [ ] `import --dry-run`：扫描 `.env` 并按值去重展示
- [ ] `agent-skill`：输出 Agent 使用说明
- [ ] 在自己的 2～3 个项目和 MCP（知乎、企查查）上实际替换验证

## v0.2 App

- [ ] SwiftUI 菜单栏：搜索、复制（自动清剪贴板）
- [ ] 管理窗口：列表、详情、编辑、标签筛选
- [ ] 扫描导入向导
- [ ] Touch ID + `session` 策略

## v0.3 集成与安全

- [ ] `import --rewrite`：改写 `.env` 与 MCP 配置（带备份）
- [ ] `run` 输出脱敏
- [ ] 审计日志与 App 查看
- [ ] 到期提醒

## 以后

- [ ] `keyhub mcp`：MCP server 模式（`list_keys`、`request_key`）
- [ ] 额度查询插件（DeepSeek、Kimi、硅基流动、OpenRouter…）
- [ ] Homebrew 分发
- [ ] iCloud 同步元数据
