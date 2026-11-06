# 设计文档

## 1. 目标与非目标

**目标**

- G1 清单：集中记录所有 API key 及其元数据，便于人查看、搜索、维护。
- G2 供给：AI Agent、项目、MCP、Skill 通过统一方式取用 key，项目中不再保存明文。
- G3 迁移：低成本把现有分散在 `.env` 中的 key 导入。
- G4 安全：明文只存在 Keychain；按用途最小授权；可审计。

**非目标**

- LLM 供应商切换（CC-Switch 的领域）。
- 团队共享 / 云同步（v1 只做单机；可依赖 iCloud Keychain 作为后续选项）。
- 跨平台（v1 只做 macOS）。

## 2. 架构

```
┌──────────────────────┐      ┌──────────────────────┐
│  KeyHub.app (SwiftUI) │      │   keyhub CLI (Swift) │ ← Agent / 项目 / MCP 调用
│  菜单栏 + 管理窗口      │      │  list/get/run/env/... │
└──────────┬───────────┘      └──────────┬───────────┘
           │        KeyHubCore (Swift Package)       │
           └───────────────┬─────────────────────────┘
                 ┌─────────┴──────────┐
                 │                    │
        macOS Keychain          元数据存储
        （仅存 key 值）     ~/Library/Application Support/keyhub/
                               keyhub.sqlite（或 JSON）
                               audit.log
```

- **KeyHubCore**：共享的 Swift Package，包含模型、Keychain 封装、元数据存储、引用解析、导入扫描。App 与 CLI 都依赖它。
- **App 与 CLI 共享 Keychain**：使用同一 Keychain access group（需同一 Team ID 签名）；开发期可退化为使用 `kSecClassGenericPassword` + 固定 service 前缀 `keyhub.`。
- CLI 不依赖 App 常驻即可工作。

## 3. 数据模型

```swift
struct KeyEntry {
    let id: UUID
    var slug: String          // 唯一标识，引用用：keyhub://<slug>，如 "zhihu"、"qcc"
    var envName: String       // 默认环境变量名，如 "ZHIHU_API_KEY"
    var platform: String      // 平台名，如 "知乎开放平台"
    var consoleURL: URL?      // 控制台 / 申请地址
    var account: String?      // 注册账号（手机号 / 邮箱）
    var tags: [String]        // 如 ["mcp", "search"]
    var usages: [Usage]       // 在哪用
    var note: String?         // 费用、套餐等备注
    var createdAt: Date
    var expiresAt: Date?
    var lastUsedAt: Date?     // 由 CLI 调用时更新
    var policy: AccessPolicy  // 访问策略
}

struct Usage {               // 记录"给谁用"
    var kind: Kind            // .project / .mcp / .skill / .global
    var name: String          // 如 "CodexLocalAgent"、"qcc-mcp"
    var path: String?         // 项目路径或配置文件路径
}

enum AccessPolicy {
    case always               // 无需确认
    case session(minutes: Int)// Touch ID 解锁一次，N 分钟内免确认（默认）
    case everyTime            // 每次都要 Touch ID
}
```

Keychain 条目：`service = "keyhub.<slug>"`，`account = <slug>`，值为 key。元数据库只存 `slug` 关联，不存值。

一个平台多个 key（如个人 / 测试）用不同 slug：`qcc`、`qcc-test`。

## 4. 引用格式

```
keyhub://<slug>
```

`.env.ref` 文件（可提交到 git）：

```dotenv
# 左边是注入的变量名，右边是引用；也可以写普通值
ZHIHU_API_KEY=keyhub://zhihu
QCC_API_KEY=keyhub://qcc
LOG_LEVEL=info
```

解析规则：

1. `keyhub run` 默认读取当前目录的 `.env.ref`，可用 `--file` 指定；
2. 值以 `keyhub://` 开头的解析为 Keychain 中对应 key；其它原样传递；
3. 引用不存在时报错并退出（非 0），提示 `keyhub add <slug>`。

## 5. 供给方式

| 方式 | 场景 | 安全性 |
|---|---|---|
| `keyhub run -- <cmd>` | 项目启动、MCP 启动 | 高：只注入子进程，不落盘 |
| `keyhub get <slug>` | Agent / 脚本临时取值 | 中：值进入调用方 stdout |
| `keyhub env > .env` | 兼容必须读 `.env` 的项目 | 低：落盘明文，需自行 gitignore |
| 全局 `~/.keyhub/env.sh` | 少数到处使用的 key | 低：所有 shell 进程可见，默认关闭 |

### 5.1 Agent 集成

- 提供一份 Skill / `CLAUDE.md` 片段（`integrations/agent-skill.md`）：告诉 Agent 需要 key 时先 `keyhub list` 查看，再优先用 `keyhub run --keys ... -- <cmd>` 注入，避免把值打印到对话中。
- `keyhub list --json` 输出元数据（slug、envName、platform、tags、usages），**永不包含值**。
- 后续可选：MCP server 模式（`keyhub mcp`），暴露 `list_keys`、`request_key`（缺 key 时弹 App 让人补录），默认不暴露取值工具。

### 5.2 MCP 配置改写

导入时识别 `~/.claude.json`、`~/.cursor/mcp.json`、Claude Desktop 配置等文件中 `env` 字段里的明文 key，提供"一键改写"为：

```json
{ "command": "keyhub", "args": ["run", "--keys", "qcc", "--", "<原 command>", "<原 args>..."] }
```

改写前自动备份原文件。

## 6. 导入扫描

`keyhub import <dir>` / App 中"扫描导入"：

1. 递归扫描 `.env`、`.env.*`（排除 `.env.example`、`.env.ref`）、MCP 配置文件，跳过 `node_modules`、`.git` 等；
2. 识别疑似 key 的变量：名称匹配 `*_KEY|*_TOKEN|*_SECRET|*_API_KEY`，或值匹配已知前缀（`sk-`、`ghp_`、`AKIA` 等）；
3. **按值去重**，同一个值出现在多个项目时合并为一条，并把各位置记录为 `usages`；
4. 交互确认 slug / 平台后写入 Keychain；
5. 可选：把原 `.env` 改写为 `.env.ref`（备份原文件）。

## 7. 安全

- 明文只存在 Keychain；元数据库、日志、导出物都不含值（`env` 命令除外，需显式调用）。
- `AccessPolicy` 默认 `session(15)`：Touch ID（LocalAuthentication）解锁后 15 分钟内 CLI 免确认，平衡 Agent 连续调用与安全。
- `run` 对子进程 stdout/stderr 做脱敏（把出现的 key 值替换为 `***keyhub:<slug>***`），可 `--no-mask` 关闭。
- 审计日志 `audit.log`：时间、slug、调用命令、父进程名、工作目录。App 中可查看。
- 已知局限：能执行命令的 Agent 理论上总能拿到注入的值，keyhub 的目标是缩小暴露面与留痕，而非绝对隔离。

## 8. App 界面（v1）

- **菜单栏**：搜索框 + 最近使用；点击复制值（复制后 30 秒自动清空剪贴板）。
- **管理窗口**：列表（平台、slug、用途、到期、最后使用）+ 详情编辑；按标签/用途筛选。
- **添加**：粘贴 key → 按前缀猜平台 → 填 slug/envName（自动生成）→ 保存。
- **扫描导入**向导。
- **到期提醒**（后续）。

## 9. 技术选型

| 部分 | 选择 | 理由 |
|---|---|---|
| App | SwiftUI（macOS 14+） | 原生菜单栏、Touch ID、Keychain |
| CLI | Swift + swift-argument-parser | 与 App 共享 Core 代码 |
| 元数据 | SQLite（GRDB）或 JSON | 数据量小，JSON 足够起步 |
| 分发 | Homebrew cask（App 内含 CLI，安装时软链） | — |

## 10. 待定问题

- [ ] Keychain access group 需要开发者签名；未签名开发期的共享方案。
- [ ] 元数据是否随 iCloud 同步。
- [ ] 额度查询插件接口定义（参考 HeyQuota Provider 设计）。
