# CLI 规格（草案）

全局约定：

- 退出码：`0` 成功；`1` 一般错误；`2` 引用不存在；`3` 用户拒绝授权（Touch ID 取消）。
- `--json`：机器可读输出，供 Agent 使用。
- 任何命令默认不输出 key 值，除 `get` / `env` 外。

## 管理

```bash
keyhub add <slug> [--env NAME] [--platform P] [--url URL] [--account A] [--tag T]... [--expires 2027-01-01]
# 值从 stdin 读取或交互输入（不回显），不接受命令行参数传值，避免进入 shell history
echo -n "$VALUE" | keyhub add qcc --env QCC_API_KEY --platform 企查查

keyhub list [--tag T] [--json]        # 元数据列表，不含值
keyhub show <slug> [--json]           # 单个 key 的元数据与用途
keyhub edit <slug> [--env ...] [...]  # 修改元数据
keyhub set <slug>                     # 更新值（stdin），即轮换
keyhub rm <slug>                      # 删除（需确认）
```

## 取用

```bash
keyhub get <slug>                     # 输出值到 stdout（受 AccessPolicy 约束）
keyhub run [--file .env.ref] [--keys a,b] [--no-mask] -- <cmd> [args...]
                                      # 解析引用并注入子进程；--keys 只注入指定 key，变量名用各自 envName
keyhub env [--file .env.ref]          # 输出解析后的 dotenv 文本（明文，慎用）
keyhub exec-env <slug>...             # 输出 `export X=...` 供 eval（明文，慎用）
```

## 迁移与集成

```bash
keyhub import <dir>... [--dry-run] [--rewrite]   # 扫描 .env / MCP 配置导入；--rewrite 将原文件改写为引用
keyhub ref init                                  # 在当前目录根据 .env 生成 .env.ref
keyhub usage add <slug> --project <path>         # 手动登记用途
keyhub agent-skill                               # 输出给 AI Agent 的使用说明（Skill / CLAUDE.md 片段）
```

## 审计

```bash
keyhub log [--slug S] [--since 7d]
```
