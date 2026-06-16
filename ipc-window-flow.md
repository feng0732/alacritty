# Alacritty IPC 多窗口：归属与配置覆盖边界深度分析

## 一、核心问题：新窗口挂到哪个进程？

### 1.1 多实例与 Socket 选择策略

Alacritty 支持单进程多窗口，也支持多进程多实例。IPC 客户端通过 **Socket 三级查找策略** 决定将消息发给哪个进程（即新窗口挂到哪个进程）：

**查找优先级**（[polling/ipc.rs:170-216](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/polling/ipc.rs#L170-L216)）：

```
┌───────────────────────────────────────────────────────────┐
│  优先级 1: CLI --socket 参数                              │
│  （用户显式指定，最明确）                                  │
└───────────────────────┬───────────────────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────────────────┐
│  优先级 2: 环境变量 ALACRITTY_SOCKET                      │
│  （父进程传递，子进程天然继承父进程的 socket）              │
└───────────────────────┬───────────────────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────────────────┐
│  优先级 3: 扫描 socket_dir() 目录                         │
│  遍历所有 Alacritty-*.sock 文件，找到第一个能 connect 的   │
└───────────────────────────────────────────────────────────┘
```

**目录扫描细节**：
- Socket 文件命名格式：`{socket_prefix}-{pid}.sock`
- `socket_prefix()` 平台差异：
  - **Linux**: `Alacritty-{WAYLAND_DISPLAY or DISPLAY}`（替换 `/` 为 `-`）
  - **macOS**: `Alacritty`（无显示服务器隔离）
- 举例：`Alacritty-wayland-0-12345.sock`

**设计意图**：
- 引入 display server 信息 → 多显示器环境下各实例互不干扰
- 带 PID → 一个显示服务器下可运行多个 Alacritty 实例
- 孤儿清理：`ConnectionRefused` 时自动 `fs::remove_file()` 清理崩溃遗留

### 1.2 归属判定：新窗口属于谁？

**结论：谁的 Socket 收到 CreateWindow 消息，新窗口就属于谁的进程。**

场景举例：

| 场景 | 操作方式 | 新窗口归属 |
|------|----------|------------|
| **场景 A** | 用户在窗口 A 的 shell 里执行 `alacritty msg create-window` | 窗口 A 所在的进程（通过 `ALACRITTY_SOCKET` 环境变量） |
| **场景 B** | 用户执行 `alacritty msg --socket /path/to/b.sock create-window` | 指定的 B 进程 |
| **场景 C** | 用户直接执行 `alacritty msg create-window`，且 ALACRITTY_SOCKET 未设置 | 目录扫描找到的**第一个**存活实例 |
| **场景 D** | 无任何运行实例，执行 `alacritty msg create-window` | 报错 "no socket found"，不会自动启动新进程 |

### 1.3 环境变量传递链

**`ALACRITTY_SOCKET` 的设置与传递**（[polling/ipc.rs:42](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/polling/ipc.rs#L42)）：

```
Alacritty 主进程
    │
    ├─ IpcListener::new() 时设置 env::set_var(ALACRITTY_SOCKET, path)
    │  （设置到主进程自身的环境）
    │
    └─ 创建 PTY → fork shell 子进程
           │
           └─ shell 子进程继承 ALACRITTY_SOCKET
                  │
                  └─ 用户在 shell 里执行 alacritty msg
                         │
                         └─ 直接从环境变量读取 socket 路径
```

代码证据：`ALACRITTY_SOCKET` 通过进程环境天然继承，而 `ALACRITTY_WINDOW_ID` 是显式设置的（[tty/unix.rs:230](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty_terminal/src/tty/unix.rs#L230)）。

---

## 二、窗口标识（window_id）的语义与默认值

### 2.1 window_id 的三层含义

IPC 消息中的 `window_id` 字段（`Option<i128>`）决定了配置作用的目标窗口。它有三种状态：

| 状态 | 示例 | 含义 | 代码中的表现 |
|------|------|------|-------------|
| **None（未设置）** | 用户不传 --window-id，且 ALACRITTY_WINDOW_ID 环境变量也没设 | 全部窗口 + 全局 | `window_id = None` |
| **-1（特殊值）** | `--window-id -1` | 全部窗口 + 全局（等同于 None） | 经 `u64::try_from(-1)` 转换失败 → 变成 None |
| **正整数** | `--window-id 3` | 指定单个窗口 | 转 u64 成功 → `Some(WindowId(3))` |

**关键实现**（[polling/ipc.rs:77-78](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/polling/ipc.rs#L77-L78)）：

```rust
let window_id = ipc_config.window_id
    .and_then(|id| u64::try_from(id).ok())  // 负数转换失败 → None
    .map(WindowId::from);
```

**设计巧思**：-1 没有单独的分支判断，而是利用"i128 转 u64 失败"这个机制，自然地落到 None 的语义上。

### 2.2 默认值从哪来？

`window_id` 的默认值由 clap 的 `env = "ALACRITTY_WINDOW_ID"` 自动读取（[cli.rs:336](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/cli.rs#L336)）：

- **在窗口内执行**：shell 继承了 `ALACRITTY_WINDOW_ID` 环境变量（PTY fork 时设置）→ 默认就是当前窗口 ID
- **在窗口外执行**：环境变量未设置 → `window_id = None` → 作用于全部窗口

所以：
- ✅ **在窗口里执行 `alacritty msg config` → 默认只改当前窗口**
- ✅ **在窗口外执行 `alacritty msg config` → 默认改所有窗口 + 更新全局**

### 2.3 三种消息的 window_id 处理

| 消息类型 | window_id 来源 | 事件中的 window_id | 备注 |
|----------|---------------|-------------------|------|
| **CreateWindow** | 无此字段 | 永远是 `None` | 创建新窗口不需要指定目标窗口 |
| **Config** | `--window-id` / `ALACRITTY_WINDOW_ID` / -1 | 正整数 → Some(id)；None/-1 → None | 决定配置作用范围 |
| **GetConfig** | `--window-id` / `ALACRITTY_WINDOW_ID` / -1 | 同上 | 决定查询哪个窗口的配置 |

---

## 三、配置覆盖边界：影响谁？

### 3.1 三层配置覆盖模型

Alacritty 的配置是一个**洋葱式叠加**结构，从内到外依次生效：

```
┌──────────────────────────────────────────────────┐
│  第 1 层：基础配置 (Rc<UiConfig>)                │
│  来自配置文件 + 启动时 CLI 参数（一次性解析）      │
└───────────────────┬──────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────────────┐
│  第 2 层：全局 IPC 覆盖 (global_ipc_options)     │
│  Processor 持有，window_id=None 的 Config 消息设置│
│  影响：所有后续新建的追加窗口 + 可回刷已有窗口     │
└───────────────────┬──────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────────────┐
│  第 3 层：窗口级覆盖 (window_config)             │
│  每个 WindowContext 独立持有                     │
│  影响：仅当前窗口                                │
└──────────────────────────────────────────────────┘
```

关键代码：
- 全局覆盖：[event.rs:98](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L98) `global_ipc_options: ParsedOptions`
- 窗口级覆盖：[window_context.rs:68](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/window_context.rs#L68) `window_config: ParsedOptions`
- 窗口级应用：[window_context.rs:265](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/window_context.rs#L265) `self.config = self.window_config.override_config_rc(self.config.clone())`

### 3.2 IPC Config 消息的作用边界

**`alacritty msg config` 有两种作用范围**，由 `--window-id` 决定：

| window_id | 作用范围 | 对既有窗口 | 对后续新窗口 | 代码行 |
|-----------|----------|-----------|-------------|--------|
| **具体正整数** | 单个指定窗口 | ✅ 立即应用（add_window_config） | ❌ 不影响 | [event.rs:299-308](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L299-L308) |
| **None / -1**（全局） | 全部窗口 + 全局存储 | ✅ 所有窗口立即应用 | ✅ 存入 global_ipc_options，追加窗口继承 | [event.rs:311-318](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L311-L318) |

**执行流程图**（[event.rs:293-318](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L293-L318)）：

```
收到 IpcConfig 事件
    │
    ├─ 步骤 1: 解析 options 为 ParsedOptions
    │
    ├─ 步骤 2: 遍历匹配的窗口（window_id 过滤）
    │     │
    │     └─ 每个窗口: reset ? reset_window_config() : add_window_config()
    │
    └─ 步骤 3: 如果 window_id.is_none()（全局）
           │
           ├─ reset ? global_ipc_options.clear()
           └─ 否则  ? global_ipc_options.append()
```

### 3.3 ⚠️ 关键事实：初始窗口 vs 追加窗口的配置组装差异

这是之前理解有误的核心点。**两条创建路径的配置组装方式完全不同**：

#### 路径 A：初始窗口（create_initial_window）

**调用时机**：
- 正常启动时的第一个窗口（`new_events(Init)` 触发）
- daemon 模式下的第一个 IPC CreateWindow（`gl_config.is_none()` 时触发）

**配置组装**（[event.rs:151-167](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L151-L167)）：

```rust
let window_context = WindowContext::initial(
    event_loop,
    self.proxy.clone(),
    self.config.clone(),   // ← 只有基础配置
    window_options,        // ← 但 options.option 不会被应用到 config
)?;
```

→ **初始窗口 = 基础配置**（不含 global_ipc_options，也不含本次 CreateWindow 的 option 覆盖）
→ **初始窗口的 window_config = 空**

在 `WindowContext::new()` 中可以确认（[window_context.rs:249](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/window_context.rs#L249)）：
```rust
window_config: Default::default(),  // 空！
```

#### 路径 B：追加窗口（create_window）

**调用时机**：
- `gl_config.is_some()` 时的所有 CreateWindow 消息
- 正常启动后的第二个及以后的窗口

**配置组装**（[event.rs:177-191](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L177-L191)）：

```rust
let mut config_overrides = options.config_overrides();     // ① 本次 IPC 带的 -o/--option
config_overrides.extend_from_slice(&self.global_ipc_options);  // ② + 全局 IPC 覆盖
let mut config = self.config.clone();
config = config_overrides.override_config_rc(config);     // ③ 叠加到基础配置

let window_context = WindowContext::additional(
    gl_config, event_loop, self.proxy.clone(),
    config,               // ← 叠加好的完整 config
    options,
    config_overrides,     // ← 覆盖项存入 window_config
)?;
```

→ **追加窗口 = 基础配置 + global_ipc_options + 本次 IPC options**
→ **追加窗口的 window_config = global_ipc_options + 本次 IPC options**

在 `WindowContext::additional()` 中可以确认（[window_context.rs:163](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/window_context.rs#L163)）：
```rust
window_context.window_config = config_overrides;  // 存入！
```

#### 差异对照表

| 配置组成 | 初始窗口 | 追加窗口 |
|---------|---------|---------|
| 基础配置文件 | ✅ | ✅ |
| 启动时 CLI 覆盖 | ✅（已融合进基础 config） | ✅（已融合进基础 config） |
| global_ipc_options（运行时全局） | ❌ **不继承** | ✅ 继承 |
| 本次 CreateWindow 的 -o options | ❌ **不应用** | ✅ 应用 |
| window_config 存储内容 | 空 | global + 本次 options |

#### 实际影响场景

**场景 1：正常启动，再开第二窗口**
- 窗口 1（初始）：基础配置 + 启动时 CLI 覆盖
- 窗口 2（追加）：基础配置 + 启动时 CLI 覆盖 + global_ipc_options（如有） + 本次 options
- 差异：如果期间没有通过 IPC 设置全局配置，两者配置一样；如果有，窗口 2 会多出来自 global_ipc_options 的覆盖

**场景 2：daemon 模式，先发 config 再发 create-window**
```bash
alacritty --daemon
alacritty msg config cursor.style=Beam    # 全局设置
alacritty msg create-window               # 第一个窗口（初始窗口路径）
```
- ❌ **首窗不会有 cursor.style=Beam**！因为走 initial 路径，不应用 global_ipc_options
- 这是一个设计上的不对称性

**场景 3：daemon 模式，发两次 create-window**
```bash
alacritty --daemon
alacritty msg config cursor.style=Beam
alacritty msg create-window     # 窗口 1（初始）→ 无 Beam
alacritty msg create-window     # 窗口 2（追加）→ 有 Beam
```
- 两个窗口的配置不一样！这可能超出用户预期

### 3.4 GetConfig 的返回边界

`alacritty msg get-config` 返回内容取决于 `--window-id`（[event.rs:322-327](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L322-L327)）：

| window_id | 返回内容 | 备注 |
|-----------|----------|------|
| 具体正整数 | 该窗口的完整 config | 基础配置 + 窗口级覆盖 一起序列化 |
| -1 / None / 找不到匹配窗口 | 基础配置 + global_ipc_options | 不含任何窗口级覆盖 |

代码逻辑：
```rust
let config = match self.windows.iter().find(|(id, _)| window_id == Some(*id)) {
    Some((_, window_context)) => window_context.config(),  // 窗口配置（含窗口级覆盖）
    None => &self.global_ipc_options.override_config_rc(self.config.clone()),  // 全局 + global
};
```

---

## 四、Daemon 模式与错误隔离

### 4.1 Daemon 模式首窗特殊路径

**正常模式**：启动 → `new_events(Init)` 触发 `create_initial_window` → 第一个窗口出现

**Daemon 模式**（`--daemon`）：启动 → 不创建任何窗口 → EventLoop 跑起来等 IPC 消息 → 第一个 `CreateWindow` 到来时走 `create_initial_window`

代码证据（[event.rs:382-390](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L382-L390)）：

```rust
if self.gl_config.is_none() {
    // Handle initial window creation in daemon mode.
    if let Err(err) = self.create_initial_window(event_loop, options) {
        self.initial_window_error = Some(err);
        event_loop.exit();
    }
} else if let Err(err) = self.create_window(event_loop, options) {
    error!("Could not open window: {err:?}");
}
```

**判断依据**：`self.gl_config.is_none()` —— GL 配置是否已初始化。

**daemon 首窗的特殊性汇总**：
1. 走 `create_initial_window` 路径
2. **不应用** `global_ipc_options`
3. **不应用** CreateWindow 消息中的 `-o/--option` 覆盖
4. **失败会导致整个进程退出**（见下节）

### 4.2 错误隔离策略：首窗致命，追窗容错

| 场景 | 失败处理 | 代码位置 |
|------|----------|----------|
| **普通模式初始窗口失败** | 保存错误 → event_loop.exit() → run() 返回 Err | [event.rs:239-243](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L239-L243) |
| **daemon 模式首窗（IPC 触发）失败** | 保存错误 → event_loop.exit() → 进程退出 | [event.rs:384-387](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L384-L387) |
| **追加窗口失败** | 仅 error! 日志，不退出，其他窗口继续运行 | [event.rs:388-390](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L388-L390) |

**为什么首窗失败必须退出？**

初始窗口的创建伴随着 **GL 平台初始化**（`WindowContext::initial()` 中完成）：
- 创建 `GlDisplay`（OpenGL 显示连接）
- 选择 `GlConfig`（像素格式）
- 创建第一个 `GlContext`

如果 GL 初始化失败，后续所有窗口都无法创建（因为它们都依赖同一个 `gl_config`）。与其挂一个空壳 daemon，不如直接退出告知用户。

**为什么追加窗口失败不退出？**

- GL 平台已经初始化好了，单个窗口失败可能是特定参数问题（如无效的 config override）
- 已有其他窗口在正常运行，为了一个窗口的失败杀掉整个进程代价太高
- 错误信息通过日志输出，用户可以感知并重试

### 4.3 窗口关闭与进程生命周期

**窗口关闭触发点**：`TerminalEvent::Exit`（[event.rs:417-441](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L417-L441)）

```
终端进程退出
    │
    ▼
TerminalEvent::Exit
    │
    ├─ 如果 window.hold == true → 保留窗口（不关闭）
    │
    ├─ 从 windows HashMap 中移除 WindowContext
    │   └─ Drop 时发送 Msg::Shutdown 给 PTY I/O 线程
    │
    ├─ scheduler.unschedule_window() 取消该窗口所有定时器
    │
    └─ 如果 windows.is_empty() 且 !daemon
           │
           └─ event_loop.exit()  →  整个进程退出
```

**daemon 模式的含义**：即使所有窗口都关了，进程也不退出，继续监听 IPC，可以随时创建新窗口。

---

## 五、环境变量全览

| 环境变量 | 设置方 | 设置时机 | 作用 |
|----------|--------|----------|------|
| **ALACRITTY_SOCKET** | 主进程自身 | `IpcListener::new()` | 标记自己的 socket 路径，子进程继承后可直接找到父进程 |
| **ALACRITTY_WINDOW_ID** | PTY fork 时 | 创建 shell 子进程时 | 标记当前 shell 属于哪个窗口，`alacritty msg config` 默认作用于该窗口 |
| **WINDOWID** | PTY fork 时 | 创建 shell 子进程时 | X11 兼容变量，部分老应用依赖此变量获取窗口 ID |
| **XDG_ACTIVATION_TOKEN** | （被移除） | PTY fork 前 | 防止子进程继承启动通知 token（Linux） |
| **DESKTOP_STARTUP_ID** | （被移除） | PTY fork 前 | 同上 |

---

## 六、关键代码路径索引

### Socket 选择与多实例
- `find_socket()`: [polling/ipc.rs:170-216](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/polling/ipc.rs#L170-L216)
- `socket_prefix()`: [polling/ipc.rs:223-232](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/polling/ipc.rs#L223-L232)
- `socket_dir()`: [polling/ipc.rs:154-167](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/polling/ipc.rs#L154-L167)
- window_id 转换（i128 → Option<WindowId>）: [polling/ipc.rs:77-85](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/polling/ipc.rs#L77-L85)

### 配置覆盖
- `global_ipc_options` 字段: [event.rs:98](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L98)
- IPC Config 处理: [event.rs:293-318](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L293-L318)
- 初始窗口创建（无 global 叠加）: [event.rs:151-167](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L151-L167)
- 追加窗口创建（有 global 叠加）: [event.rs:170-195](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L170-L195)
- 窗口级覆盖应用（update_config 中）: [window_context.rs:261-265](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/window_context.rs#L261-L265)
- `add_window_config()`: [window_context.rs:355-363](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/window_context.rs#L355-L363)
- `reset_window_config()`: [window_context.rs:343-351](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/window_context.rs#L343-L351)
- 追加窗口存入 window_config: [window_context.rs:160-164](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/window_context.rs#L160-L164)

### Daemon 与错误隔离
- 初始窗口创建（正常启动）: [event.rs:238-247](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L238-L247)
- IPC CreateWindow 处理（含 daemon 首窗判断）: [event.rs:372-391](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L372-L391)
- 窗口关闭与空窗口检查: [event.rs:417-441](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L417-L441)

### 环境变量
- PTY 环境变量设置: [tty/unix.rs:228-241](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty_terminal/src/tty/unix.rs#L228-L241)
- IpcConfig window_id 从 env 读取: [cli.rs:336](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/cli.rs#L336)
- IpcGetConfig window_id 从 env 读取: [cli.rs:351](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/cli.rs#L351)
