# Codex 额度恢复自动续跑

Windows 桌面工具，自动监测 Codex 使用额度，在额度恢复后向因额度耗尽而中断的本地对话提交 `continue`，让对话继续执行。

提供单文件 EXE 和 Python 源码两种运行方式。图形界面支持一键打开、关闭监测，显示剩余额度、重置时间及运行记录。

## 功能

- **一键启停**：点击“打开监测”启动，点击“关闭监测”停止；打开程序时默认关闭，关闭窗口也会停止监测。
- **额度显示**：显示短周期、长周期剩余额度和重置时间，时间按电脑本地时区显示。
- **自动续跑**：仅恢复最后一轮因额度耗尽而失败的本地 Codex 对话，提交内容固定为 `continue`。
- **实际额度校验**：到达预计重置时间后仍会查询真实额度，不会仅凭时间判断额度恢复。
- **发送记录**：保存失败轮次和发送结果，重启后继续使用已有记录，减少重复提交。
- **范围配置**：可指定监测对话、排除部分对话，调整检查间隔和每轮续跑数量。
- **后台脚本**：提供 PowerShell 和 CMD 启停入口，与图形界面共享运行锁及发送记录。
- **只读预演**：检查连接、额度和候选对话，不实际发送消息。

程序通过 Codex 桌面应用自带的 `codex-app-tools` MCP 适配器提交消息。`continue` 会出现在原对话中，由原客户端按对话已有设置执行；无需切换窗口、点击输入框或使用剪贴板。

## 运行环境

| 项目     | 要求                                                         |
| -------- | ------------------------------------------------------------ |
| 操作系统 | Windows x64；当前成品已在 Windows 11 x64 上验证              |
| Codex    | 已安装、登录且保持开启的桌面应用，需具有兼容的本地 `codex-app-tools` 桥接 |
| Node.js  | 可通过 `node` 命令找到的 Node.js，供本地 MCP 适配器运行      |
| Python   | EXE 已内置；源码运行需要 Python 3.10+，图形界面还需可用的 Tkinter |
| 网络     | Codex 需要能正常连接服务并查询账号额度                       |

不需要额外填写 API Key。程序使用当前 Codex 桌面客户端登录账号的额度和本地对话。

## EXE 快速开始

仓库提供 Release 后，可从 **Releases → Assets** 下载 `Codex自动续跑.exe`。GitHub 自动生成的 **Source code (zip)** 是源码包。

### 1. 准备环境

安装并登录 Codex 桌面应用，保持应用开启。安装 Node.js 后，在新的 PowerShell 窗口中确认：

```powershell
node --version
```

### 2. 生成配置

双击 EXE，然后关闭窗口。首次打开时会生成：

```text
%LOCALAPPDATA%\CodexQuotaMonitor\config.json
```

按 **Win+R**，输入 `%LOCALAPPDATA%\CodexQuotaMonitor`，即可打开配置所在文件夹。

### 3. 填写本机对话 ID

在本机 Codex 中打开一个本地对话，明确授权自动续跑，并获取该对话的实际 thread ID。例如：

```text
我授权本机额度监测程序在额度恢复后向额度中断的本地 Codex 对话发送 continue。
请告诉我当前对话的实际 thread ID。
```

编辑生成的 `config.json`，将 `caller_thread_id` 的 `YOUR_CODEX_THREAD_ID` 替换为实际 ID，保留该对话可访问。需要填写对话 ID，不能填写对话标题或仓库名称。

`caller_thread_id` 是调用本地工具时使用的对话上下文；要限定实际续跑范围，还需配置 `thread_ids`。

### 4. 开始监测

重新打开 EXE，点击 **打开监测**。正常连接后，窗口会显示额度和运行记录。默认每 60 秒检查一次，可在启动前修改窗口中的检查间隔。

点击 **关闭监测** 或关闭窗口停止。正在进行的本地请求需要完成后才退出，可能等待约 30 秒。停止监测会阻止后续自动提交；已经开始执行的 Codex 对话仍由 Codex 管理。

电脑和 Codex 必须保持运行。休眠期间无法监测，唤醒后会继续检查。本项目未自动配置开机启动。

## 自动续跑规则

每轮检查的主要流程：

1. 查询当前账号的实际额度。
2. 查找监测范围内的本地 Codex 对话，读取最新一轮状态。
3. 将最后一轮因额度耗尽而失败的对话记录为待续跑。
4. 发送前再次核对额度、账号、最新轮次和停止状态。
5. 条件满足时提交 `continue`，保存发送结果。

若服务返回短周期和长周期两个额度窗口，两个窗口均需可用。只恢复一个窗口、额度值缺失、查询失败或存在无法确认的账号限制时，程序继续等待有效数据。

以下情况不会自动续跑：

- 对话已完成、被手动停止、正在执行或等待审批。
- 最新一轮是普通网络错误、请求频率错误等非额度耗尽错误。
- 只是聊天正文或工具输出提到了额度不足，最新一轮没有对应的顶层失败信息。
- 对话属于云端 ChatGPT/Work 或远程主机。
- 同一失败轮次已经成功发送，或者此前发送结果无法确定。

默认每轮最多续跑 1 个对话。后续新轮次再次因额度耗尽而失败时，可作为新的失败轮次等待恢复。

## 配置说明

### 配置文件位置

| 运行方式                             | 配置位置                                             |
| ------------------------------------ | ---------------------------------------------------- |
| EXE / `desktop_app.py` 图形界面      | `%LOCALAPPDATA%\CodexQuotaMonitor\config.json`       |
| `monitor.py` / CMD / PowerShell 脚本 | 源码目录中的 `config.json`，也可通过 `--config` 指定 |

两种方式默认共享 `%LOCALAPPDATA%\CodexQuotaMonitor\runtime` 下的运行锁和发送状态。修改配置后需停止并重新启动；图形界面修改间隔时会保存到其用户配置。

### 配置示例

```json
{
  "caller_thread_id": "YOUR_CODEX_THREAD_ID",
  "poll_seconds": 60,
  "limit_id": "codex",
  "thread_ids": [],
  "exclude_thread_ids": [],
  "max_resumes_per_cycle": 1,
  "retry_seconds": 300,
  "max_send_attempts": 3,
  "node_path": "",
  "server_path": "",
  "pipe_path": ""
}
```

| 配置项                  | 默认值                 | 说明                                                         |
| ----------------------- | ---------------------- | ------------------------------------------------------------ |
| `caller_thread_id`      | `YOUR_CODEX_THREAD_ID` | 必须填写本机实际、可访问的调用方对话 ID                      |
| `poll_seconds`          | `60`                   | 检查间隔，单位为秒，最小 15                                  |
| `limit_id`              | `codex`                | 额度桶 ID；独立模型额度需填写服务返回的对应 ID，并限定适用对话 |
| `thread_ids`            | `[]`                   | 空数组自动扫描最近 50 个非置顶对话及全部置顶对话中的本地 Codex 对话；非空则仅检查指定 ID，也支持列表之外的旧对话 |
| `exclude_thread_ids`    | `[]`                   | 不续跑的对话 ID，优先于包含范围                              |
| `max_resumes_per_cycle` | `1`                    | 每轮最多成功续跑的对话数量                                   |
| `retry_seconds`         | `300`                  | 发送明确被拒绝后再次尝试的间隔，单位为秒                     |
| `max_send_attempts`     | `3`                    | 同一失败轮次的发送尝试上限；等待额度不计入次数               |
| `node_path`             | `""`                   | Node.js 可执行文件路径；空字符串表示自动发现                 |
| `server_path`           | `""`                   | 本地 `codex-app-tools` 的 `server.mjs` 路径；空字符串表示自动发现 |
| `pipe_path`             | `""`                   | Codex 本地管道路径；空字符串表示自动发现，应用重启后管道可能变化 |

只监测指定对话时，将 `thread_ids` 改为：

```json
"thread_ids": ["第一个实际对话ID", "第二个实际对话ID"]
```

JSON 不支持注释。请保留双引号、逗号及方括号格式，配置项名称必须与模板一致。

## 从源码运行

在源码目录中打开 PowerShell。首次配置时复制模板并编辑：

```powershell
Copy-Item -LiteralPath .\config.example.json -Destination .\config.json
notepad .\config.json
```

已有配置时直接编辑 `config.json`，保留原有设置。填写实际 `caller_thread_id` 后，先执行一次只读预演：

```powershell
python .\monitor.py --dry-run --once
```

预演会查询额度和对话，记录可能的续跑对象，但不发送 `continue`、不更新续跑状态；仍会写入运行日志。

### 前台监测

```powershell
python .\monitor.py
```

按 **Ctrl+C** 停止。

### 图形界面

```powershell
python .\desktop_app.py
```

图形界面使用用户目录配置，首次运行需按 EXE 的配置步骤填写。修改项目中的 `config.json` 不会同步修改图形界面配置。

### 后台监测

双击 `启动监测.cmd` 启动，双击 `停止监测.cmd` 停止；也可运行：

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\Start-Monitor.ps1 -Background
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\Stop-Monitor.ps1
```

默认共享运行目录的图形界面与脚本不能同时启动监测。停止脚本会请求共享运行目录中的监测实例停止。

### 命令行参数

| 参数               | 作用                                             |
| ------------------ | ------------------------------------------------ |
| `--config PATH`    | 指定脚本配置文件                                 |
| `--state-dir PATH` | 指定运行状态目录，默认使用用户目录中的 `runtime` |
| `--dry-run`        | 只读预演，不发送消息、不修改续跑记录             |
| `--once`           | 检查一轮后退出                                   |

示例：

```powershell
python .\monitor.py --config .\config.json --dry-run --once
```

## 日志和发送状态

默认运行目录：

```text
%LOCALAPPDATA%\CodexQuotaMonitor\runtime
```

`monitor.log` 保存连接、额度、重置时间及续跑记录；`state.json` 保存失败轮次和发送状态；运行锁防止默认实例重复运行。查看最近日志：

```powershell
Get-Content -LiteralPath "$env:LOCALAPPDATA\CodexQuotaMonitor\runtime\monitor.log" -Tail 30
```

发送成功的轮次标记为 `sent`，重启不会对同一失败轮次再次提交。若发送超时，或进程在提交过程中退出，状态会标记为 `unknown`；重启时遗留的 `sending` 也会转为 `unknown`。

遇到结果不明的记录时：

1. 停止监测。
2. 查看原对话，确认是否已收到 `continue` 或已继续执行。
3. 只有确认消息未送达且仍需续跑时，才将 `state.json` 中对应记录的 `status` 改为 `pending`，再重新启动。

保留发送记录有助于防止重复提交，不要为了排错直接清空整个状态文件。明确被拒绝的发送按配置重试，达到尝试上限后需人工检查。

## 常见问题

| 现象                         | 处理方法                                                     |
| ---------------------------- | ------------------------------------------------------------ |
| 找不到 Node.js               | 在新 PowerShell 中运行 `node --version`，检查 PATH，必要时填写 `node_path` |
| 无法连接本地工具服务         | 确认 Codex 已开启并登录、调用方 ID 正确且对话可访问；查看日志，确认桥接版本兼容 |
| EXE 开启后没有自动监测       | 默认关闭，需点击“打开监测”                                   |
| 预计重置时间已到但没有续跑   | 查看最新实际额度，另一个额度窗口或账号限制可能仍不可用       |
| 额度可用，但某个对话没有继续 | 检查最新一轮是否确为额度失败，以及监测范围、发送状态、重试次数和每轮数量限制 |
| 旧对话未被发现               | 在 `thread_ids` 中填写实际 ID，或将其置顶                    |
| 提示已有监测实例             | 先停止已有图形界面或后台脚本，再重新启动                     |
| 修改配置后未生效             | 确认修改的是当前入口使用的配置文件，然后停止并重新启动       |
| 关闭时暂时没有退出           | 等待当前本地请求完成；停止监测不会终止已经运行的 Codex 对话  |
| 更换账号后没有继续           | 程序会核对账号，旧账号记录不会直接用于新账号；检查配置和原对话是否仍可访问 |
| 提示发送结果不明             | 查看原对话，并按上文处理对应 `unknown` 记录                  |

## 构建 EXE

在 Windows 源码目录中执行：

```powershell
python -m pip install --target .\.build-tools pyinstaller
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\build_exe.ps1
```

构建结果为 `dist\CodexQuotaMonitor.exe`，可重命名为 `Codex自动续跑.exe`。这是单文件窗口程序，内置 Python；目标电脑仍需 Node.js 和兼容的 Codex 桌面应用。

公开构建应保持源码中的默认 `caller_thread_id` 为占位符。实际对话 ID 只写入本机配置，不要将配置、运行日志或发送状态打包到公开版本中。

## 验证

```powershell
python -m unittest discover -s . -p 'test_*.py' -v
```

当前测试覆盖额度判断、失败轮次识别、发送去重、崩溃恢复、账号切换、手动继续后的状态变化、停止、预演及图形界面控制等逻辑。现有 26 项测试已通过，实际桌面连接已通过只读查询验证；验证未向业务对话提交续跑消息。

## 项目文件

```text
.
├── desktop_app.py        # 图形界面与监测控制
├── monitor.py            # 额度查询、对话筛选与自动续跑
├── config.example.json   # 配置模板
├── Start-Monitor.ps1     # 脚本启动入口
├── Stop-Monitor.ps1      # 脚本停止入口
├── 启动监测.cmd
├── 停止监测.cmd
├── build_exe.ps1         # Windows EXE 构建脚本
├── version_info.txt      # EXE 版本信息
├── test_monitor.py
├── test_desktop.py
├── DEPLOYMENT.md         # GitHub 上传与部署步骤
└── README.md
```

## 使用范围与限制

- 本项目依赖桌面应用自带的本地 MCP 桥接，该桥接并非承诺稳定的外部接口。应用升级可能改变工具、协议或管道命名，需要以实际日志确认兼容性。
- 每个监测实例只按配置的 `limit_id` 判断额度；使用独立模型额度桶时，应匹配对应额度与对话范围。
- 发送前会重新读取最新状态，但工具没有“轮次未变化时才提交”的原子操作，核对与发送之间仍存在极短的竞争窗口。
- 自动提交 `continue` 后，原对话可能继续执行已有任务。可用 `thread_ids` 指定需要自动续跑的对话。
- 日志和状态文件可能包含对话 ID、标题及账号标识。公开上传时使用配置模板，并保留 `.gitignore` 对 `config.json`、运行目录和日志的忽略规则。
- GitHub 用于保存源码和分发程序，实际额度监测运行在本地 Windows 电脑上。


