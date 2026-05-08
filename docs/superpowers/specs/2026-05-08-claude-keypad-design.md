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

核心价值：**把 Claude Code 的权限确认 UI 从终端移到独立小屏**，按键直接做决定，避免「在主屏幕来回切焦点 + 按数字 + 回车」的低效循环。

### 1.1 目标用户与场景

- 重度使用 Claude Code 的开发者
- 经常处理「需要确认大量工具调用」的任务（如 agent 自动化、多文件修改）
- 希望解放主显示器空间、减少 context-switch

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
| 键盘 | 用户现有 QMK/VIA 可编程键盘，至少 4 个空白键 | 4 键映射为 F20 / F21 / F22 / F23（HID 标准里几乎不用的码位）；语音键支持 Tap / Hold 区分 |
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
│   • F23 (语音) 区分 Tap / Hold:        │
│       Tap   = F23                      │
│       Hold  = Shift+F23                │
└────────────────────────────────────────┘

┌─ PC 后台服务 claude-keypad-svc         ┐
│   • 全局监听 F20-F23 (Win32 keyboard hook) │
│   • 串口连接屏幕 (CDC)                 │
│   • 内嵌 HTTP server (127.0.0.1 only)  │
│       ← Claude Code hook POST prompt   │
│       → 回复 allow / deny / ask        │
│   • Whisper 本地推理 + 云端 STT 客户端 │
│   • Transcript JSONL 解析（取 model / │
│     token / context）                  │
│   • 系统托盘                           │
└────────────────────────────────────────┘

┌─ Claude Code Hook 脚本 (~30 行)        ┐
│   • PreToolUse / Stop / SessionStart   │
│   • POST localhost:48420               │
│   • 阻塞等待回应（带超时）             │
│   • 输出 hookSpecificOutput JSON       │
└────────────────────────────────────────┘

┌─ 配置 GUI (Tauri)                      ┐
│   • STT 引擎选择                       │
│   • 语音键模式 (Push-to-talk / Toggle) │
│   • 危险词高亮规则                     │
│   • 项目白名单                         │
│   • 屏幕显示主题                       │
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
{"type":"toast","level":"info","text":"语音录制中..."}
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
| 键 4 Tap | F23 | （v2 考虑：拒绝 + 录音说明原因） | 录音 → STT → 注入到 Claude Code 输入框 |
| 键 4 Hold | Shift+F23 | 同 Tap | Push-to-talk：按下开始录音，松开停止+发送 |

「无操作」可以在配置里改成「滚动屏幕历史」「打开 GUI」等，v1 默认空。

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
│ Tokens (session)           │
│   ▓▓▓▓▓▓▓▓░░  12,453       │
│                            │
│ Context                    │
│   ▓▓▓▓░░░░░░  34%          │
│                            │
└────────────────────────────┘
```

数据源：`claude-keypad-svc` 解析 `transcript_path` 指向的 JSONL。

> **v1 不显示**：5h 配额、7d 配额、配额重置时间——Claude Code 不暴露此数据，砍出 MVP，进 v2 待研究清单。

**5.1.3 提示态**

录音中、网络错误、服务未连接等用底部 toast 显示，2 秒后回到主状态。

### 5.2 危险词高亮

服务端在推 `prompt` 消息时，把命中的危险词位置编入 `danger` 字段；屏幕负责渲染颜色。规则在配置 GUI 里可编辑，默认表：

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

## 6. 语音输入

### 6.1 模式

| 模式 | 触发 | 行为 |
|--|--|--|
| Push-to-talk | 长按键 4（Hold）| 按下开始录音，松开停止 + 立刻 STT 出结果 |
| Toggle | 短按键 4（Tap）| 第一次按开始录音，第二次按停止 |

模式选哪个由用户配置，但**两个手势同时支持**——配置项决定 Tap 走「toggle 切换」还是「忽略」，Hold 始终是 PTT。

### 6.2 STT 引擎

抽象成 `STTEngine` interface，三个实现：

| 实现 | 默认 | 依赖 |
|--|--|--|
| `WhisperLocal` | ✅ 默认 | faster-whisper Python 子进程，模型 small/medium 自动选 |
| `OpenAICloud` | 可选 | OPENAI_API_KEY，调用 whisper-1 |
| `WindowsSpeech` | 可选 | Win+H 系统语音，via UI Automation |

切换在配置 GUI 里完成；切换 `WhisperLocal` 时自动下载模型（首次约 500MB）。

### 6.3 文本注入

录音转文字后，把结果注入当前焦点的 Claude Code 终端。方案：

1. 把文本写入剪贴板
2. 模拟 Ctrl+V

不直接 SendInput 打字（中文输入法兼容性差）。

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

## 8. 配置 GUI

技术栈：**Tauri**（Rust 后端 + WebView 前端，安装包小、原生集成）。

### 8.1 主要配置项

```
[一般]
  开机自启           [✓]
  显示系统托盘图标    [✓]

[键盘]
  键 1 动作          Allow ▾
  键 2 动作          Allow + Remember ▾
  键 3 动作          Deny ▾
  键 4 Tap 动作       Toggle Recording ▾
  键 4 Hold 动作      Push-to-talk ▾

[屏幕]
  串口设备           COM3 ▾    [刷新]
  空闲状态显示        ●● ●○ ○○
  危险词规则          [编辑...]

[语音]
  STT 引擎           Whisper Local ▾
  Whisper 模型        small ▾
  OpenAI API Key     [_______]
  发送方式            Ctrl+V (剪贴板)

[Claude Code]
  当前项目白名单
    ✓ /path/to/slot_machine
    ✓ /path/to/another
    [+] 添加项目

[关于]
  版本 / 检查更新 / 卸载
```

### 8.2 配置文件

`%APPDATA%\claude-keypad\config.toml`，便于人工编辑、便于朋友间复制配置。

---

## 9. MVP 范围

### 9.1 v1 必须有

- [x] PreToolUse hook 接管权限确认
- [x] 屏幕渲染 prompt（Bash / Edit / Write 三种工具）
- [x] 4 键映射 + 语义
- [x] Whisper 本地 STT + 文本注入
- [x] graceful 降级（服务/屏幕缺失时回 ask）
- [x] 配置 GUI（最小可用）
- [x] Windows 安装包（MSI）

### 9.2 v1 不做（v2+）

- [ ] 5h / 7d 配额显示（数据源缺失，待研究）
- [ ] 触摸屏交互
- [ ] Mac / Linux 支持
- [ ] 多 Claude Code session 并发显示
- [ ] 屏幕历史滚动 / 操作日志
- [ ] OTA 固件更新（v1 用户手动刷）
- [ ] Read / Glob / Grep 等低风险工具的差异化渲染（v1 统一显示）
- [ ] Cursor / Codex 等其他 AI 工具适配

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
| QMK 键盘多样性 → 朋友的键盘不一定是 F20-F23 | 配置 GUI 提供「学习模式」：用户按一下要绑定的键，软件记录其 HID 码 |
| Whisper 首次下载 500MB | 安装包不打包模型；首次启动后台下载 + 进度条 |
| Claude Code hook schema 变更 | 把 hook IO 解析放在独立模块；版本探测 + 兼容降级 |
| 当 Claude Code 进入 plan mode 等特殊模式 | hook 输入有 `permission_mode` 字段，服务端按需特殊处理 |

---

## 11. 后续步骤

1. 用户审阅本文档
2. 进入 `writing-plans` 技能，把上面的 v1 范围拆成可执行任务清单
3. 实施顺序建议：
   - **Phase 1**：PC 服务 + hook 脚本，仅终端打印（验证 hook 链路通）
   - **Phase 2**：屏幕固件 + 串口协议 + 渲染 prompt（验证硬件链路通）
   - **Phase 3**：键盘全局监听 + 4 键完整语义
   - **Phase 4**：语音输入
   - **Phase 5**：配置 GUI + 安装包

每个 phase 自成一个端到端可演示的里程碑。
