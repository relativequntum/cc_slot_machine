# Claude Keypad 设计文档

**日期**：2026-05-08
**状态**：Draft, 待用户批准
**作者**：与 Claude (Opus 4.7) 协作
**产品代号**：`claude-keypad`（最终名称待定）

---

## 1. 产品概述

一个面向 Claude Code 用户的**硬件外设套装**，包含：

- 1 个可编程键盘（用户已有，4 个空白键）
- 1 块 320×480 彩色 LCD 副屏
- 1 套 PC 后台软件（含 Claude Code hook 集成）

核心价值（两条同等重要）：

1. **效率层**：把 Claude Code 的权限确认 UI 从终端移到独立小屏，按键直接做决定，避免「在主屏幕来回切焦点 + 按数字 + 回车」的低效循环。
2. **体验层**：把数字 + Enter 这样需要视觉对焦的输入，换成有清脆物理反馈的「啪、啪、啪」按键。每次确认都是一个小小的多巴胺触发——把使用 Claude Code 这件事变得**爽**。这一点不是附加价值，而是产品差异化的核心。

### 1.1 目标用户与场景

- **任何 Claude Code 用户**——不限重度/轻度，只要会被权限弹窗打断的人都是受众
- 经常处理「需要确认大量工具调用」的任务（如 agent 自动化、多文件修改）尤其受益
- 希望解放主显示器空间、减少 context-switch、并且想让交互过程更有趣

### 1.2 产品定位

**小范围分享给朋友**——不是商业化产品，但要求：

- 一键安装的 PC 软件包
- 朋友拿到键盘 + 屏幕 + 装个软件就能用
- 跨机器迁移配置

不要求：跨平台（v1 仅 Windows）、商业级 UI 打磨、订阅服务、远程协作。

---

## 2. 硬件清单

| 部件 | 选型 | 说明 |
|--|--|--|
| 键盘 | 用户现有 QMK/VIA 可编程键盘，至少 4 个空白键 | 4 键映射为 F20 / F21 / F22 / F23（HID 标准里几乎不用的码位）。v1 各键均为简单映射；v2 时键 4 拆 Tap / Hold |
| LCD 副屏 | **WT32-SC01 Plus** (ESP32-S3 + 3.5" 320×480 电容触摸 LCD + USB-C) | 价格约 150 元；自带外壳；社区例程多。备选：CrowPanel ESP32 3.5" / 淘宝 ESP32-S3 + ILI9488 模块 |
| 连线 | USB-C 数据线（屏幕） + USB（键盘） | 用户自备；可选 USB hub 整合 |

> 触摸屏在 v1 不使用——所有交互通过物理按键。触摸能力为 v2 预留。

---

## 3. 系统架构

### 3.1 组件分工

```
┌─ 屏幕固件 (ESP32-S3, C++)              ┐
│   • LVGL UI 渲染                       │
│   • USB CDC 串口接 PC                  │
│   • 协议: PC→屏 推 JSON; 屏→PC 推 ack  │
└────────────────────────────────────────┘

┌─ 键盘固件 (QMK, 用户已有键盘)          ┐
│   • 4 空白键 → F20 / F21 / F22 / F23   │
│   • v1: F23 单纯映射，不区分 Tap/Hold  │
│   • v2: F23 拆 Tap/Hold (语音用)       │
└────────────────────────────────────────┘

┌─ PC 后台服务 claude-keypad-svc         ┐
│   • 全局监听 F20-F23 (Win32 keyboard hook) │
│   • 串口连接屏幕 (CDC)                 │
│   • 内嵌 HTTP server (127.0.0.1 only)  │
│       ← Claude Code hook POST prompt   │
│       → 回复 allow / deny / ask        │
│   • Transcript JSONL 解析（取 model / │
│     token / context）                  │
│   • TOML 配置文件监听 + 热重载         │
│   • 系统托盘                           │
└────────────────────────────────────────┘

┌─ Claude Code Hook 脚本 (~30 行)        ┐
│   • PreToolUse / Stop / SessionStart   │
│   • POST localhost:48420               │
│   • 阻塞等待回应（带超时）             │
│   • 输出 hookSpecificOutput JSON       │
└────────────────────────────────────────┘

┌─ 配置：TOML 文件 + 系统托盘菜单        ┐
│   • 无独立 GUI 程序                    │
│   • 托盘「Edit config」 → 默认编辑器   │
│   • 见第 8 节                          │
└────────────────────────────────────────┘
```

### 3.2 一次「权限确认」的完整时序

```
Claude Code → PreToolUse hook 脚本 (PowerShell/Python) [hook 配置 timeout=320]
              ↓ HTTP POST localhost:48420/permission
              {
                "session_id": "...", "tool_name": "Bash",
                "tool_input": {"command": "rm -rf node_modules"},
                "cwd": "/path/to/project"
              }
                                     ↓
                          claude-keypad-svc
                                     ↓
                          ① 渲染 prompt 到屏幕（串口）
                          ② 等待按键事件（最多 300s）
                                     ↓
                          按键 / 超时
                                     ↓
              HTTP 200 ← {"decision": "allow|deny|ask",
                          "remember": false|true}
                                     ↓
hook 脚本输出 stdout:
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow|deny|ask"
  }
}

(若 remember=true 且 decision=allow)
  → 服务端额外把规则追加到 <当前 cwd>/.claude/settings.local.json
    （从 hook 输入的 cwd 字段取项目根，per-project 而非全局）
    "permissions.allow": ["Bash(rm -rf node_modules:*)"]
```

### 3.3 IPC 协议

#### 3.3.1 Hook ↔ Service：HTTP/JSON

监听 `127.0.0.1:48420`，仅本机可访问。

| Endpoint | Method | 用途 |
|--|--|--|
| `/permission` | POST | PreToolUse 推送，返回决策 |
| `/session-start` | POST | 通知一个 session 开始（用于状态屏） |
| `/session-stop` | POST | 通知 session 结束 |
| `/health` | GET | hook 启动时探活，超时即 graceful 降级 |

#### 3.3.2 Service ↔ Screen：USB CDC 串口 + 行分隔 JSON

每条消息一行 JSON，`\n` 结束：

**PC → 屏**

```json
{"type":"prompt","tool":"Bash","args":"rm -rf ...","danger":["rm -rf"],"cwd":"slot_machine"}
{"type":"status","model":"claude-opus-4-7","tokens":12453,"context_pct":34}
{"type":"idle"}
{"type":"toast","level":"info","text":"配置已重载"}
```

**屏 → PC**

```json
{"type":"ack","msg_id":"..."}
{"type":"button","key":"F20"}        // 物理按键事件（备份通道，主路径还是走 PC 全局监听）
```

> 按键的主路径是 PC 全局键盘监听；屏幕侧的 button 事件是冗余备份，用于「键盘没插」的特殊场景调试。

---

## 4. 4 个键的语义表

| 物理键 | HID 码 | 在「等待权限」时 | 在「空闲」时 |
|--|--|--|--|
| 键 1 | F20 | `allow`（仅本次） | 无操作 |
| 键 2 | F21 | `allow` + `remember=true` 写入 `.claude/settings.local.json` | 无操作 |
| 键 3 | F22 | `deny` | 无操作 |
| 键 4 | F23 | **v1：无操作**（保留物理按键，但不响应；预留给 v2 语音） | **v1：无操作** |

> v1 不绑定键 4，避免「按了但没反应」的歧义；屏幕上提示文案也只显示 1/2/3。
> v2 计划：键 4 接入语音录入（详见第 6 节，整节移入 v2 范围）。

「无操作」可以在配置里改成「滚动屏幕历史」「打开配置」等，v1 默认空。

---

## 5. 屏幕显示

### 5.1 三种状态

**5.1.1 等待权限（核心 UI）**

```
┌────────────────────────────┐
│ slot_machine        14:23  │
│ ─────────────────────────  │
│ Bash                       │
│                            │
│ rm -rf node_modules &&     │  ← 危险词「rm -rf」红色高亮
│ npm install                │
│                            │
│ ─────────────────────────  │
│ ▣1 Allow  ▣2 All  ▣3 Deny  │
└────────────────────────────┘
```

不同工具的渲染：

- **Bash**：高亮命令；标红危险词（`rm`, `sudo`, `curl|sh`, `dd`, `mkfs`, `> /dev/sd*` 等，规则可配）
- **Edit / Write**：文件路径 + diff 前 ~10 行
- **Read / Glob / Grep**：低风险；显示路径/pattern，按键含义不变

**5.1.2 空闲状态屏（v1 实际可拿到的字段）**

```
┌────────────────────────────┐
│ ●  claude-opus-4-7   14:23 │
│ ─────────────────────────  │
│ slot_machine               │  ← 当前活跃项目
│                            │
│ Session tokens             │
│   12,453                   │  ← 单纯计数，无进度条
│                            │
│ Context                    │
│   ▓▓▓▓░░░░░░  34%          │  ← 占当前模型上下文窗口的比例
│                            │
└────────────────────────────┘
```

字段说明：

- **Session tokens**：本次 session 累计消耗的 token 数（一个不断增长的计数器，没有显式上限——所以**不画进度条**，纯数字展示）。
- **Context**：当前 conversation 已用 token / 当前模型的 context window 上限。这个**有上限**（如 Opus 4.7 是 1M tokens），所以是百分比 + 进度条。当它接近 80% 时屏幕会变成警示色，提醒该 `/compact` 了。

数据源：`claude-keypad-svc` 解析 `transcript_path` 指向的 JSONL，累加 input/output tokens；context 上限根据 transcript 里的 model 字段查一张内置映射表。

> **v1 不显示**：5h 配额、7d 配额、配额重置时间——Claude Code 不暴露此数据，砍出 MVP，进 v2 待研究清单。

**5.1.3 提示态**

网络错误、服务未连接、配置重载等用底部 toast 显示，2 秒后回到主状态。

### 5.2 危险词高亮

服务端在推 `prompt` 消息时，把命中的危险词位置编入 `danger` 字段；屏幕负责渲染颜色。规则在 `config.toml` 的 `[danger_words]` 节编辑，默认表：

```
rm -rf  /  rm -fr
sudo
curl ... | sh / bash
wget ... | sh
dd if=
mkfs
> /dev/sd
chmod 777
:(){...}
```

---

## 6. 语音输入（**整节挪到 v2，v1 不实现**）

> v1 决策：STT（本地 Whisper / 云端 API / 系统语音）和文本注入都属于「重组件」，会显著拉大安装包、跨机一致性测试面、出错路径。v1 先把权限按键和屏幕的核心闭环做扎实，语音延后。
>
> v2 启动时回到本节基线，再做选型与实现。

以下为 v2 设计草案，留作参考：

<details>
<summary>v2 草案展开</summary>

### 6.1 模式（v2）

| 模式 | 触发 | 行为 |
|--|--|--|
| Push-to-talk | 长按键 4（Hold）| 按下开始录音，松开停止 + 立刻 STT 出结果 |
| Toggle | 短按键 4（Tap）| 第一次按开始录音，第二次按停止 |

QMK 层 Tap/Hold 区分：Tap = F23，Hold = Shift+F23（或 QMK custom keycode）。

### 6.2 STT 引擎（v2）

抽象成 `STTEngine` interface，至少两个实现：

| 实现 | 默认 | 依赖 |
|--|--|--|
| `WhisperLocal` | ✅ 默认 | faster-whisper Python 子进程 |
| `OpenAICloud` | 可选 | API key，调用 whisper-1 |

### 6.3 文本注入（v2）

剪贴板 + Ctrl+V，不直接 SendInput 打字（中文输入法兼容性差）。

</details>

---

## 7. Claude Code Hook 集成

### 7.1 安装时写入 `~/.claude/settings.json`

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": ".*",
        "hooks": [
          {
            "type": "command",
            "command": "claude-keypad-hook",
            "timeout": 320
          }
        ]
      }
    ],
    "Stop": [
      { "matcher": ".*", "hooks": [{ "type":"command", "command":"claude-keypad-hook --event stop", "timeout": 5 }] }
    ],
    "SessionStart": [
      { "matcher": ".*", "hooks": [{ "type":"command", "command":"claude-keypad-hook --event session-start", "timeout": 5 }] }
    ]
  }
}
```

`claude-keypad-hook` 是一个安装到 PATH 的小可执行文件（~50 行 Rust 或 Go）。

### 7.2 Hook 脚本逻辑

```
1. 从 stdin 读 JSON
2. GET http://127.0.0.1:48420/health (50ms timeout)
   失败 → 输出 {"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"ask"}}
   exit 0   （graceful 降级到终端原生确认）
3. POST http://127.0.0.1:48420/permission
   含 stdin 中的全部字段
   响应超时 = 310s

   嵌套超时设计（外层包内层，留出处理时间）：
     - 服务端等待按键          : 300s
     - hook 脚本的 HTTP 超时   : 310s
     - Claude Code hook 总超时 : 320s（见 7.1 配置）
4. 解析响应中的 decision，输出 hookSpecificOutput JSON
5. exit 0
```

任何异常路径**都返回 `permissionDecision: "ask"`**，绝不返回阻塞错误（exit 2 会让 stderr 弹给用户）。

### 7.3 超时与降级

| 情况 | 行为 |
|--|--|
| 服务未运行（health 失败） | hook 立刻返回 `ask` |
| 服务运行但屏幕未连接 | 服务返回 `ask`（带 toast 通知用户） |
| 用户 300s 内没按键 | 服务返回 `ask`，屏幕回到空闲状态；下次仍正常推送 |
| 服务崩溃 / hook 进程被杀 | Claude Code hook timeout=320s 自动 fallback |
| 输出 JSON 解析失败 | Claude Code 标准降级到 ask |

---

## 8. 配置：纯 TOML 文件 + 系统托盘（v1 最轻方案）

v1 不做独立 GUI 程序——配置就是一个 TOML 文件，托盘菜单点一下「编辑配置」就用系统默认编辑器（记事本 / VSCode / 别的）打开。保存后服务监听文件变化、热重载。

理由：

- 安装包不带 UI 框架，~3-5MB Rust 单 exe 即可
- 朋友间复制配置 = 复制一个文本文件，简单粗暴
- 改配置不需要再开一个程序窗口
- 有人想自动化（脚本 / Ansible 部署给团队）也直接改文件

代价：

- 第一次配置门槛比纯 GUI 高一点（要看注释/文档）
- 错的 TOML 语法会让服务报错——所以加载失败时托盘弹气泡 + 回退到上一次有效配置

### 8.1 配置文件位置

`%APPDATA%\claude-keypad\config.toml`

### 8.2 默认 TOML 内容（含中文注释）

```toml
# claude-keypad 配置文件
# 改完保存即生效，不需要重启服务。

[server]
# 本地 HTTP 端口，给 Claude Code hook 用
port = 48420

# 服务等待按键的超时（秒），超时后回退到终端原生 ask
permission_timeout_secs = 300

[screen]
# 串口设备名（Windows 上形如 COM3）。设为 "auto" 让服务自动发现
serial_port = "auto"

# 接近 context 上限的告警阈值（百分比）
context_warning_pct = 80

[keys]
# 物理键到动作的映射。可选动作：
#   "allow"            本次允许
#   "allow_remember"   允许 + 写入 .claude/settings.local.json
#   "deny"             本次拒绝
#   "open_config"      打开本配置文件
#   "noop"             什么都不做
key1 = "allow"
key2 = "allow_remember"
key3 = "deny"
key4 = "noop"   # v2 会替换为语音

[danger_words]
# 在权限弹屏上要标红的命令片段（大小写不敏感、可包含空格）
patterns = [
  "rm -rf",
  "rm -fr",
  "sudo",
  "curl ",      # 后续可加更精细规则
  "dd if=",
  "mkfs",
  "> /dev/sd",
  "chmod 777",
]

[projects]
# 项目白名单：仅当 hook 输入的 cwd 命中以下前缀，才走副屏路径
# 留空数组 = 所有项目都走副屏
allow_prefixes = []
```

### 8.3 系统托盘菜单

| 菜单项 | 行为 |
|--|--|
| `claude-keypad - running` | 静态状态行，灰色 |
| `Edit config…` | `start "" "%APPDATA%\claude-keypad\config.toml"` |
| `Reveal config in Explorer` | 打开 APPDATA 目录 |
| `Reload config` | 强制重载（一般不用，服务自己监听文件变化） |
| `Open log…` | 打开最新日志文件 |
| `--` | |
| `Quit` | 退出服务（hook 后续会 graceful 降级到 ask） |

### 8.4 v2 升级路径

如果 v1 上线后发现纯 TOML 对朋友确实太硬核，再加一层：服务自己起 `localhost:<port>/config` 一个 HTML 表单页，托盘菜单 `Edit config…` 改成在浏览器打开它。**这一步不需要换技术栈、不引入新依赖**——服务里嵌个静态 HTML 就行。

---

## 9. MVP 范围

### 9.1 v1 必须有

- [x] PreToolUse hook 接管权限确认
- [x] 屏幕渲染 prompt（Bash / Edit / Write 三种工具）
- [x] 3 键完整语义（Allow / Allow All / Deny）+ 第 4 键留空（不响应）
- [x] 空闲状态屏：模型、Session tokens、Context %
- [x] graceful 降级（服务/屏幕缺失时回 ask）
- [x] TOML 配置 + 系统托盘
- [x] Windows 安装包

### 9.2 v1 不做（v2+）

- [ ] **语音输入**（键 4 + STT + 文本注入）——完整功能延后
- [ ] 5h / 7d 配额显示（数据源缺失，待研究）
- [ ] 触摸屏交互
- [ ] Mac / Linux 支持
- [ ] 多 Claude Code session 并发显示
- [ ] 屏幕历史滚动 / 操作日志
- [ ] OTA 固件更新（v1 用户手动刷）
- [ ] Read / Glob / Grep 等低风险工具的差异化渲染（v1 统一显示）
- [ ] Cursor / Codex 等其他 AI 工具适配
- [ ] HTML 配置表单（v1 仅 TOML）

### 9.3 验收标准

- 安装包跑过：拿一台干净 Win11 机器，按 README 装 → Claude Code 中下次 `Bash` 调用 prompt 出现在屏幕、按键 1 后命令立刻执行
- graceful 降级：拔掉屏幕 USB 后再让 Claude Code 调工具 → 终端弹原生 ask UI（不卡死）
- 状态屏：有 Claude Code session 在跑时，屏幕实时显示 model + token 数 + context %

---

## 10. 风险与开放问题

| 风险 | 缓解 |
|--|--|
| Windows 杀软误报全局键盘监听 | 软件签名（用户 v1 自签名 + 文档说明）；申请 SmartScreen 信誉 |
| WT32-SC01 Plus 国内供货周期 | 备选 CrowPanel / 淘宝模块；spec 把屏幕协议写抽象 |
| QMK 键盘多样性 → 朋友的键盘不一定是 F20-F23 | TOML 里 `[keys]` 节支持把 4 个动作绑到任意 HID 码；托盘菜单提供「Detect last pressed key」帮用户填 |
| Claude Code hook schema 变更 | 把 hook IO 解析放在独立模块；版本探测 + 兼容降级 |
| 当 Claude Code 进入 plan mode 等特殊模式 | hook 输入有 `permission_mode` 字段，服务端按需特殊处理 |

---

## 11. 后续步骤

1. 用户审阅本文档
2. 进入 `writing-plans` 技能，把上面的 v1 范围拆成可执行任务清单
3. 实施顺序建议（v1）：
   - **Phase 1**：PC 服务骨架 + hook 脚本，仅终端打印工具调用信息（验证 hook 链路通、graceful 降级正确）
   - **Phase 2**：屏幕固件 + 串口协议 + 渲染 prompt（验证硬件链路通）
   - **Phase 3**：键盘全局监听 + 3 键完整语义（Allow / Allow All / Deny），端到端打通
   - **Phase 4**：空闲状态屏（Transcript JSONL 解析 → 模型 / Session tokens / Context %）
   - **Phase 5**：TOML 配置 + 系统托盘 + Windows 安装包

每个 phase 自成一个端到端可演示的里程碑。v2 启动时回到第 6 节做语音设计。
