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

## 三、配置覆盖机制详解

### 3.1 覆盖底层：ParsedOptions 如何工作

理解优先级之前，先搞清楚 `ParsedOptions` 的内部实现（[cli.rs:356-425](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/cli.rs#L356-L425)）：

**数据结构**：
```rust
pub struct ParsedOptions {
    config_options: Vec<(String, Value)>,  // 顺序存储的 (原始字符串, TOML 值) 列表
}
```

**核心方法 `override_config()`**（[cli.rs:381-396](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/cli.rs#L381-L396)）：
```rust
pub fn override_config(&mut self, config: &mut UiConfig) {
    let mut i = 0;
    while i < self.config_options.len() {
        let (option, parsed) = &self.config_options[i];
        match config.replace(parsed.clone()) {
            Err(err) => {
                // 无效选项 → 从列表中移除（swap_remove）
                self.config_options.swap_remove(i);
            },
            Ok(_) => i += 1,  // 有效 → 继续下一个
        }
    }
}
```

**两个关键特性**：

1. **顺序决定优先级**：按 Vec 顺序依次调用 `config.replace()`，**后出现的同名配置会覆盖先出现的**（因为后调用的 replace 会覆盖前一次的结果）
2. **失败自动清理**：如果某个 option 无效（字段不存在或类型错误），会从列表中删除，下次重新应用时不会再报错

**追加方式**：
- `extend_from_slice(other)`：把另一个 ParsedOptions 的内容追加到末尾（[cli.rs:421-424](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/cli.rs#L421-L424) 通过 `DerefMut` 实现）
- 追加在末尾的优先级更高

### 3.2 三层配置覆盖模型

Alacritty 的配置是一个**洋葱式叠加**结构，从内到外（优先级从低到高）依次是：

```
低优先级 ──────────────────────────────────────────────── 高优先级

┌─────────────────┐  ┌────────────────────┐  ┌────────────────────┐
│  基础配置       │  │  全局 IPC 覆盖     │  │  窗口级覆盖         │
│  (config file)  │  │  (global_ipc_...)  │  │  (window_config)   │
└────────┬────────┘  └─────────┬──────────┘  └─────────┬──────────┘
         │                     │                        │
         └─────────────────────┴────────────────────────┘
                               │
                               ▼
                      最终生效的 config
```

关键代码：
- 全局覆盖存储：[event.rs:98](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L98) `global_ipc_options: ParsedOptions`
- 窗口级覆盖存储：[window_context.rs:68](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/window_context.rs#L68) `window_config: ParsedOptions`
- 窗口级应用入口：[window_context.rs:265](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/window_context.rs#L265) `self.config = self.window_config.override_config_rc(...)`

### 3.3 冲突优先级：同一项谁覆盖谁？

这是核心问题。当多层覆盖针对**同一个配置项**时，优先级如下（高优先级覆盖低优先级）：

| 优先级 | 覆盖来源 | 适用场景 |
|--------|---------|----------|
| 🏆 最高 | 后发的 IPC Config 消息 | 同一条路径上多次调用 config，后发的赢 |
| 🥈 次高 | 窗口级覆盖（window_config） | 窗口专属的配置 |
| 🥉 中 | 全局 IPC 覆盖（global_ipc_options） | 全局运行时设置 |
| 最低 | 基础配置（配置文件 + 启动 CLI） | 底层默认值 |

下面分场景拆解：

#### 场景 A：多次 IPC Config 消息作用于同一个窗口

每次 `alacritty msg config` 调用 `add_window_config()`，会把新的 options **追加**到 `window_config` 末尾（[window_context.rs:359](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/window_context.rs#L359)）：

```rust
pub fn add_window_config(&mut self, config: Rc<UiConfig>, options: &ParsedOptions) {
    self.window_config.extend_from_slice(options);  // 追加到末尾
    self.update_config(config);
}
```

→ **后发的 config 消息优先级更高**，同名字段会覆盖之前的。

#### 场景 B：全局 IPC Config vs 窗口级 IPC Config

假设有这样的操作序列：
```bash
alacritty msg config cursor.style=Beam          # 全局设置（window_id=None）
alacritty msg config --window-id 3 cursor.style=Underline  # 只改窗口 3
```

对于窗口 3 来说：
1. 第一步全局 config → 窗口 3 的 `window_config` 追加了 `cursor.style=Beam`
2. 第二步窗口级 config → 窗口 3 的 `window_config` 又追加了 `cursor.style=Underline`
3. `window_config` 顺序：[Beam, Underline]
4. 应用时后出现的 Underline 覆盖 Beam → **窗口级赢**

对于窗口 1（没发过窗口级 config）：
- 只有 Beam → 全局设置生效

**结论**：窗口级 IPC Config 的优先级高于全局 IPC Config（因为后追加到 window_config）。

#### 场景 C：CreateWindow 自带 options vs 全局 IPC 覆盖

**仅适用于追加窗口**（初始窗口不应用任何 IPC 覆盖）。

代码（[event.rs:177-180](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L177-L180)）：
```rust
let mut config_overrides = options.config_overrides();     // ① 先放 CreateWindow 自带的
config_overrides.extend_from_slice(&self.global_ipc_options);  // ② 再追加全局的
```

→ global 在后面 → **global_ipc_options 优先级 > CreateWindow 自带 options**

这个设计有点反直觉（通常更具体的应该优先级更高），但代码确实是这样写的。

#### 场景 D：创建窗口后再发全局 Config

```bash
alacritty msg create-window -o cursor.style=Beam     # 窗口创建时自带 Beam
alacritty msg config cursor.style=Underline          # 后续全局设置 Underline
```

创建时：`window_config = [Beam, global(空)]`
后续全局 config：`window_config` 追加 `Underline` → `[Beam, Underline]`
→ Underline 在后面 → **后续全局 config 覆盖创建时自带的 option**

### 3.4 ⚠️ 关键事实：初始窗口 vs 追加窗口的配置组装差异

两条创建路径的配置组装方式**完全不同**：

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
    window_options,        // ← options.option 不会被应用到 config
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

→ **追加窗口 = 基础配置 + CreateWindow 自带 options + global_ipc_options**
→ **追加窗口的 window_config = CreateWindow 自带 options + global_ipc_options**
→ （注：global 在后面，优先级更高）

在 `WindowContext::additional()` 中可以确认（[window_context.rs:163](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/window_context.rs#L163)）：
```rust
window_context.window_config = config_overrides;  // 存入！
```

#### 差异对照表

| 配置组成 | 初始窗口 | 追加窗口 |
|---------|---------|---------|
| 基础配置文件 | ✅ | ✅ |
| 启动时 CLI 覆盖 | ✅（已融合进基础 config） | ✅（已融合进基础 config） |
| global_ipc_options（运行时全局） | ❌ **不继承** | ✅ 继承（优先级高于 CreateWindow options） |
| 本次 CreateWindow 的 -o options | ❌ **不应用** | ✅ 应用（优先级低于 global） |
| window_config 存储内容 | 空 | CreateWindow options + global_ipc_options |

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

### 3.5 重置配置：reset 的行为

`alacritty msg config --reset` 可以清除运行时覆盖。

#### 窗口级 reset（指定 window_id）

调用 `reset_window_config()`（[window_context.rs:343-351](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/window_context.rs#L343-L351)）：

```rust
pub fn reset_window_config(&mut self, config: Rc<UiConfig>) {
    self.message_buffer.remove_target(LOG_TARGET_IPC_CONFIG);  // 清错误提示
    self.window_config.clear();                                // 清空窗口级覆盖
    self.update_config(config);                                // 重新应用（用基础配置 + 空 window_config）
}
```

→ 效果：该窗口回到**基础配置**状态（不含任何 IPC 覆盖）

#### 全局 reset（window_id=None / -1）

代码（[event.rs:312-317](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L312-L317)）：
```rust
if window_id.is_none() {
    if ipc_config.reset {
        self.global_ipc_options.clear();  // 清空全局覆盖
    } else {
        self.global_ipc_options.append(&mut options);
    }
}
```

加上前面的窗口循环，全局 reset 做了两件事：
1. 对所有窗口调用 `reset_window_config()` → 清空每个窗口自己的 window_config
2. 清空 `global_ipc_options` → 后续新建窗口也不会继承全局覆盖

→ 效果：所有窗口回到**基础配置**状态，未来新窗口也从基础配置开始

#### 注意：重置 vs 配置文件重新加载

reset 只清除**运行时 IPC 覆盖**，不会触及：
- 配置文件中的设置
- 启动时 CLI 参数的覆盖（已融合进基础 config）

### 3.6 配置文件重新加载：覆盖项会保留吗？

配置文件被修改时，`ConfigMonitor` 检测到变化，触发 `EventType::ConfigReload`（[event.rs:343-370](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L343-L370)）：

```rust
(EventType::ConfigReload(path), _) => {
    if let Ok(config) = config::reload(&path, &mut self.cli_options) {
        self.config = Rc::new(config);                  // ① 更新全局基础配置
        
        for window_context in self.windows.values_mut() {
            window_context.update_config(self.config.clone());  // ② 每个窗口重新应用
        }
    }
}
```

每个窗口的 `update_config()` 做了什么（[window_context.rs:261-265](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/window_context.rs#L261-L265)）：

```rust
pub fn update_config(&mut self, new_config: Rc<UiConfig>) {
    let old_config = mem::replace(&mut self.config, new_config);  // 替换基础配置
    self.config = self.window_config.override_config_rc(self.config.clone());  // 重新应用窗口级覆盖
    // ... 后续更新显示、终端等
}
```

**结论**：

| 配置层 | ConfigReload 后的行为 | 是否保留 |
|--------|----------------------|---------|
| 基础配置（文件） | 重新加载，内容可能变 | 新内容 |
| 启动时 CLI 覆盖 | `config::reload()` 时重新应用 | ✅ 保留（融合在基础 config 里） |
| global_ipc_options | ConfigReload 不碰它 | ✅ 完整保留 |
| 窗口级 window_config | 保留 + 在新基础上重新应用 | ✅ 完整保留 |

→ **配置文件重新加载不会清除任何 IPC 覆盖**，所有运行时设置都保留，只是换了一层"底子"。

→ 但是！由于 `override_config` 有**自动清理无效项**的特性（[cli.rs:385-391](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/cli.rs#L385-L391)），如果新配置文件导致某个 IPC 覆盖项变得无效（比如字段被删除了），下次应用时会被自动清掉。

#### global_ipc_options 何时重新应用？

细心的读者可能会问：`global_ipc_options` 在 ConfigReload 中没有被重新应用到窗口上？

答案是：**不需要，也不会。**

因为 `global_ipc_options` 的内容**已经存在每个窗口的 window_config 里了**——当初发全局 IPC Config 消息时，既更新了 global_ipc_options，也调用 add_window_config 追加到了每个窗口的 window_config 里。

所以 ConfigReload 只需要重新应用 window_config 就够了，global_ipc_options 只是一个"模板"，用来给未来新建的窗口做初始化。

### 3.7 GetConfig 的返回边界

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

### 配置覆盖底层
- `ParsedOptions` 定义: [cli.rs:356-359](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/cli.rs#L356-L359)
- `override_config()`（顺序应用 + 失败清理）: [cli.rs:381-396](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/cli.rs#L381-L396)
- `override_config_rc()`: [cli.rs:399-410](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/cli.rs#L399-L410)

### 配置覆盖边界
- `global_ipc_options` 字段: [event.rs:98](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L98)
- IPC Config 处理（含全局/窗口级分发）: [event.rs:293-318](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L293-L318)
- 初始窗口创建（无覆盖）: [event.rs:151-167](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L151-L167)
- 追加窗口创建（有 global + 本次覆盖）: [event.rs:170-195](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L170-L195)
- 窗口级覆盖应用入口: [window_context.rs:261-265](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/window_context.rs#L261-L265)
- `add_window_config()`（追加 + 重新应用）: [window_context.rs:355-363](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/window_context.rs#L355-L363)
- `reset_window_config()`（清空 + 重新应用）: [window_context.rs:343-351](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/window_context.rs#L343-L351)
- 追加窗口存入 window_config: [window_context.rs:160-164](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/window_context.rs#L160-L164)

### 配置文件重新加载
- ConfigReload 事件处理: [event.rs:343-370](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L343-L370)
- `update_config()`（替换基础 + 重应用窗口级）: [window_context.rs:261-284](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/window_context.rs#L261-L284)

### Daemon 与错误隔离
- 初始窗口创建（正常启动）: [event.rs:238-247](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L238-L247)
- IPC CreateWindow 处理（含 daemon 首窗判断）: [event.rs:372-391](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L372-L391)
- 窗口关闭与空窗口检查: [event.rs:417-441](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/event.rs#L417-L441)

### 环境变量
- PTY 环境变量设置: [tty/unix.rs:228-241](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty_terminal/src/tty/unix.rs#L228-L241)
- IpcConfig window_id 从 env 读取: [cli.rs:336](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/cli.rs#L336)
- IpcGetConfig window_id 从 env 读取: [cli.rs:351](file:///d:/fz/0601/solo-dogfeeding/code/340-alacritty/alacritty/src/cli.rs#L351)
