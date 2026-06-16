# PTY 与子进程跨平台架构深度分析

## 一、跨平台抽象层设计

### 1.1 抽象层级结构

Alacritty 通过 **trait 抽象 + 平台条件编译** 的方式实现 PTY 的跨平台支持，形成三层抽象架构：

```
┌──────────────────────────────────────────────────────────────┐
│                     上层调用者 (EventLoop)                   │
│              通过 EventedPty / EventedReadWrite trait 操作    │
└───────────────────────────┬──────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────┐
│                   tty/mod.rs (抽象层)                        │
│  - Options / Shell 配置结构体 (通用)                          │
│  - EventedReadWrite trait (读写抽象)                          │
│  - EventedPty trait (子进程事件抽象)                          │
│  - ChildEvent 枚举 (子进程事件)                               │
│  - setup_env() 通用环境配置                                   │
│  #[cfg(not(windows))] pub use unix::*                        │
│  #[cfg(windows)]     pub use windows::*                       │
└───────────────────┬──────────────────────┬───────────────────┘
                    │                      │
        ┌───────────▼──────┐      ┌───────▼───────────┐
        │  Unix 实现       │      │  Windows 实现     │
        │  tty/unix.rs     │      │  tty/windows/     │
        │  - Pty 结构体    │      │    - mod.rs       │
        │  - Child (std)   │      │    - conpty.rs    │
        │  - UnixStream    │      │    - child.rs     │
        │    (SIGCHLD)     │      │    - blocking.rs  │
        └──────────────────┘      └───────────────────┘
```

### 1.2 核心 trait 定义

**EventedReadWrite trait** —— 可注册到 poller 的读写抽象：

定义于 [tty/mod.rs#L65-L78](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/mod.rs#L65-L78)

```rust
pub trait EventedReadWrite {
    type Reader: io::Read;
    type Writer: io::Write;

    /// 注册到 poller（unsafe：底层源必须比 Poller 存活更久）
    unsafe fn register(&mut self, _: &Arc<Poller>, _: Event, _: PollMode) -> io::Result<()>;
    fn reregister(&mut self, _: &Arc<Poller>, _: Event, _: PollMode) -> io::Result<()>;
    fn deregister(&mut self, _: &Arc<Poller>) -> io::Result<()>;

    fn reader(&mut self) -> &mut Self::Reader;
    fn writer(&mut self) -> &mut Self::Writer;
}
```

**EventedPty trait** —— 子进程事件通知抽象：

定义于 [tty/mod.rs#L92-L97](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/mod.rs#L92-L97)

```rust
pub trait EventedPty: EventedReadWrite {
    /// 尝试获取子进程事件（如退出）
    fn next_child_event(&mut self) -> Option<ChildEvent>;
}
```

### 1.3 Token 定义对比

| Token | Unix 值 | Windows 值 | 用途 |
|-------|---------|------------|------|
| PTY_READ_WRITE_TOKEN | 0 | 2 | PTY 读写事件 |
| PTY_CHILD_EVENT_TOKEN | 1 | 1 | 子进程事件 |

两个平台的 `PTY_CHILD_EVENT_TOKEN` 都是 1，保持了一致性。

---

## 二、创建链路对比分析

### 2.1 统一入口

两个平台都导出相同的 `Pty` 结构体和 `new()` 函数签名：

```rust
// 两个平台都满足：
pub fn new(config: &Options, window_size: WindowSize, window_id: u64) -> Result<Pty>
```

由 [tty/mod.rs#L11-L19](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/mod.rs#L11-L19) 通过条件编译统一导出。

### 2.2 Unix 创建链路详解

**入口**：[tty/unix.rs:new()](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/unix.rs#L195-L307)

```
new()
  │
  ├── 1. openpty() → 生成 master + slave 设备对
  │     (rustix_openpty crate)
  │
  ├── 2. termios 配置 → 设置 IUTF8
  │     (仅 Linux/macOS)
  │
  ├── 3. ShellUser::from_env() → 获取用户信息
  │     (USER/HOME/SHELL，fallback 到 getpwuid_r)
  │
  ├── 4. 构建 Command
  │     ├── 用户指定 shell + args
  │     └── 或 default_shell_command()
  │         ├── Linux/BSD: 直接用 shell 路径
  │         └── macOS: /usr/bin/login -flp user /bin/zsh -fc "exec -a -bash bash"
  │
  ├── 5. 重定向 stdio 到 slave
  │
  ├── 6. 配置环境变量
  │     (ALACRITTY_WINDOW_ID, USER, HOME, WINDOWID, 自定义 env)
  │
  ├── 7. pre_exec 钩子 (子进程上下文)
  │     ├── setsid() → 创建新进程组
  │     ├── chdir() → 设置工作目录
  │     ├── TIOCSCTTY ioctl → 设置控制终端
  │     ├── close(slave_fd, master_fd) → 关闭不需要的 fd
  │     └── 重置信号处理为 SIG_DFL
  │
  ├── 8. 注册 SIGCHLD 信号管道
  │     (signal_hook + UnixStream pair)
  │
  └── 9. builder.spawn() → fork+exec
        └── 设置 master 为非阻塞
```

**Unix Pty 结构体**：
```rust
pub struct Pty {
    child: Child,          // std::process::Child
    file: File,            // master 端文件
    signals: UnixStream,   // SIGCHLD 信号管道接收端
    sig_id: SigId,         // 信号注册 ID（用于注销）
}
```

### 2.3 Windows 创建链路详解

**入口**：[tty/windows/conpty.rs:new()](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/windows/conpty.rs#L110-L244)

```
new()
  │
  ├── 1. ConptyApi::new() → 加载 ConPTY API
  │     ├── 优先加载 conpty.dll (Windows Terminal 提供)
  │     └── fallback 到系统 kernel32.dll 的 CreatePseudoConsole
  │
  ├── 2. 创建两对匿名管道
  │     ├── conout (读取子进程输出)
  │     │     └── conout_pty_handle (传给 conpty)
  │     └── conin (写入子进程输入)
  │           └── conin_pty_handle (传给 conpty)
  │
  ├── 3. CreatePseudoConsole() → 创建伪控制台
  │     (传入 window_size, conin/conout 管道句柄)
  │
  ├── 4. 准备 STARTUPINFOEXW
  │     ├── STARTF_USESTDHANDLES (不继承任何句柄)
  │     ├── InitializeProcThreadAttributeList
  │     └── UpdateProcThreadAttribute: PROC_THREAD_ATTRIBUTE_PSEUDOCONSOLE
  │         (将 HPCON 绑定到子进程)
  │
  ├── 5. 构建命令行 cmdline
  │     (默认 powershell，支持参数转义)
  │
  ├── 6. 准备环境变量
  │     (Unicode 环境块 + 去重，Windows 不区分大小写)
  │
  ├── 7. CreateProcessW() → 创建子进程
  │     (EXTENDED_STARTUPINFO_PRESENT flag)
  │
  ├── 8. 包装为非阻塞读写器
  │     ├── UnblockedReader::new(conout) → 后台线程 + piper
  │     └── UnblockedWriter::new(conin) → 后台线程 + pipe
  │
  └── 9. 创建 ChildExitWatcher
        (RegisterWaitForSingleObject 等待进程退出)
```

**Windows Pty 结构体**：
```rust
pub struct Pty {
    backend: Conpty,          // ConPTY 句柄 (HPCON)
    conout: ReadPipe,         // UnblockedReader<AnonRead>
    conin: WritePipe,         // UnblockedWriter<AnonWrite>
    child_watcher: ChildExitWatcher,  // 子进程退出监视器
}
```

### 2.4 两条链路核心差异对比

| 对比维度 | Unix | Windows |
|---------|------|---------|
| **PTY API** | `openpty()` (POSIX 标准) | `CreatePseudoConsole()` (Win10+ ConPTY) |
| **子进程创建** | `fork()` + `exec()` | `CreateProcessW()` (无 fork 语义) |
| **标准流重定向** | 直接绑定 slave fd | 通过 PROC_THREAD_ATTRIBUTE_PSEUDOCONSOLE 间接绑定 |
| **控制终端设置** | `TIOCSCTTY` ioctl + `setsid()` | 由 ConPTY 内核处理 |
| **进程退出通知** | SIGCHLD 信号 → Unix socket 管道 | `RegisterWaitForSingleObject` → IOCP 事件 |
| **非阻塞 I/O** | `O_NONBLOCK` + `epoll/kqueue` | 后台线程 + `piper` 管道（blocking.rs） |
| **环境变量** | 继承父进程 + 增量设置 | 可选自定义环境块（需去重，不区分大小写） |
| **默认 Shell** | 用户 passwd 中的 shell | `powershell.exe` |
| **工作目录** | `chdir()` (pre_exec 中) | `CreateProcessW` 的 `lpCurrentDirectory` 参数 |
| **进程组** | `setsid()` 创建新 session | ConPTY 自动管理 |
| **Drop 行为** | 发送 SIGHUP + wait | ClosePseudoConsole（会阻塞直到 conout 排空） |

### 2.5 设计模式分析

**共同遵循的设计原则**：

1. **RAII 资源管理**：两个平台的 `Pty` 都实现了 Drop，确保资源正确释放
2. **Evented 抽象**：都实现了 `EventedReadWrite` + `EventedPty`，可无缝接入事件循环
3. **非阻塞 I/O**：都提供非阻塞读写能力，适配 `polling` crate 的事件驱动模型
4. **子进程退出异步通知**：都将子进程退出转化为 poller 可接收的事件

**Windows 平台的特殊复杂度**：
- 多了一层 `blocking.rs` 来将同步的管道包装为"异步"（实际是后台线程 + pipe 缓冲）
- 多了 `child.rs` 封装进程退出等待
- ConPTY 有 deadlock 风险：**必须先 drop backend 再 drop conout**（通过字段顺序保证）

> **重要纠正**：[windows/mod.rs#L28-L30](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/windows/mod.rs#L28-L30) 的注释明确指出：
> "Backend is required to be the first field, to ensure correct drop order. Dropping `conout` before `backend` will cause a deadlock (with Conpty)."
>
> Rust 的 drop 顺序是**按字段声明顺序依次 drop**（先声明的先被 drop），所以：
> ```
> Pty {
>     backend: Backend,      // ← 第1字段，先 drop → ClosePseudoConsole
>     conout: ReadPipe,      // ← 第2字段，后 drop → 关闭管道
>     conin: WritePipe,      // ← 第3字段
>     child_watcher: ...,    // ← 第4字段
> }
> ```
> 原因：`ClosePseudoConsole()` [conpty.rs#L99-L103](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/windows/conpty.rs#L99-L103) 会阻塞等待 conout 管道中的数据被排空。如果先 drop conout，那么 ClosePseudoConsole 将永远等待一个已关闭的管道 → **死锁**。

---

## 三、损坏区域（Damage）标记归属深度分析

### 3.1 Damage 系统的核心数据结构

定义于 [term/mod.rs#L216-L266](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/term/mod.rs#L216-L266)

```rust
struct TermDamageState {
    full: bool,                    // 是否全部损坏
    lines: Vec<LineDamageBounds>,  // 每行的损坏列范围 [left, right]
    last_cursor: Point,            // 上一次 damage() 调用时的光标位置
}
```

### 3.2 Damage 标记的两种模式

#### 模式一：延迟推断式（用于字符输入 input()）

**归属**：不在 `input()` 或 `write_at_cursor()` 中直接标记，而是在 `Term::damage()` 被调用时**通过光标移动轨迹推断**。

**核心代码**在 [Term::damage()](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/term/mod.rs#L458-L486) 方法中：

```rust
pub fn damage(&mut self) -> TermDamage<'_> {
    // ... insert mode 特殊处理 ...
    
    // 1. 交换 last_cursor 和当前光标
    let previous_cursor = mem::replace(&mut self.damage.last_cursor, self.grid.cursor.point);

    if self.damage.full {
        return TermDamage::Full;
    }

    // 2. 关键：如果光标位置变了，说明中间经过的字符都被写入了
    //    所以将旧光标位置标记为损坏
    if self.damage.last_cursor != previous_cursor {
        let point = Point::new(previous_cursor.line.0 as usize, previous_cursor.column);
        self.damage.damage_point(point);  // 标记旧光标位置
    }

    // 3. 总是标记当前光标位置（光标本身需要重绘）
    self.damage_cursor();

    TermDamage::Partial(...)
}
```

**设计意图**（源码注释原文）：
> "Add information about old cursor position and new one if they are not the same, so we cover everything that was produced by `Term::input`."

**原理**：
- 假设字符输入是**从左到右、逐字符**进行的（这是终端的典型行为）
- 上一帧光标位置 + 当前光标位置之间的所有字符都被修改过了
- 所以只需要标记旧光标位置 + 新光标位置，就能覆盖整行输入的 damage

**调用链**：
```
PTY reader 线程
  │
  ├── state.parser.advance(term, bytes)
  │     └── Handler::input(c)  → 仅修改 Grid 和 cursor，不标记 damage
  │           (write_at_cursor 也不标记 damage)
  │
  └── event_proxy.send_event(Event::Wakeup)
        │
        ▼
主线程渲染
  │
  └── Display::draw()
        │
        ├── terminal.damage()  → 在这里通过光标比较推断 damage
        │     └── TermDamageState::last_cursor 被更新
        └── terminal.reset_damage()
```

#### 模式二：立即标记式（用于其他终端操作）

**归属**：在各 Handler 方法内部**直接调用 damage 相关方法**。

**典型示例**：

| 操作 | Handler 方法 | Damage 标记方式 | 代码位置 |
|------|-------------|-----------------|----------|
| 光标移动 | `goto()` | 移动前 `damage_cursor()` + 移动后 `damage_cursor()` | [term/mod.rs#L1167-L1170](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/term/mod.rs#L1167-L1170) |
| 插入空白 | `insert_blank()` | `damage_line()` 整行标记 | [term/mod.rs#L1199](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/term/mod.rs#L1199) |
| 向前移动 | `move_forward()` | `damage_line()` 标记路径 | [term/mod.rs#L1238](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/term/mod.rs#L1238) |
| 回车 | `carriage_return()` | `damage_line()` 标记 | [term/mod.rs#L1416](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/term/mod.rs#L1416) |
| 上滚 | `scroll_up_relative()` | `mark_fully_damaged()` 全量 | [term/mod.rs#L789](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/term/mod.rs#L789) |
| 下滚 | `scroll_down_relative()` | `mark_fully_damaged()` 全量 | [term/mod.rs#L762](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/term/mod.rs#L762) |
| DECALN | `decaln()` | `mark_fully_damaged()` 全量 | [term/mod.rs#L1152](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/term/mod.rs#L1152) |
| 删除字符 | `delete_chars()` | `damage_line()` 整行 | [term/mod.rs#L1551](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/term/mod.rs#L1551) |

### 3.3 Damage 生命周期时序

```
       PTY I/O 线程                            主线程
           │                                    │
           │  parser.advance()                  │
           │  ├── input('a') → 写 Grid          │
           │  ├── input('b') → 写 Grid          │
           │  └── cursor 移动                    │
           │                                    │
           │  send_event(Wakeup) ──────────────►│
           │                                    │
           │                                    │  Display::draw()
           │                                    │  ├── terminal.damage()
           │                                    │  │   ├── 比较 last_cursor
           │                                    │  │   ├── 推断输入路径 damage
           │                                    │  │   └── 更新 last_cursor
           │                                    │  ├── terminal.reset_damage()
           │                                    │  └── 渲染损坏区域
           │                                    │
           │  parser.advance()                  │
           │  ├── goto(x, y)                    │
           │  │   ├── damage_cursor() ← 立即标记│
           │  │   └── 移动光标                   │
           │  │       └── damage_cursor() ← 立即标记│
           │  └── ...                           │
           │                                    │
```

### 3.4 设计权衡分析

**为什么 `input()` 不直接标记 damage？**

1. **性能考虑**：`input()` 是最高频的调用（每个字符一次），避免每次调用都做 damage 计算
2. **批量优化**：一帧内可能输入多个字符，统一在 `damage()` 时计算一次即可
3. **光标移动推断**：终端输入大多是顺序的，通过光标位置推断足够准确

**为什么其他操作需要立即标记？**

1. **非顺序操作**：`goto`、`insert_blank`、`scroll` 等操作的影响范围无法通过光标移动推断
2. **影响范围大**：滚动、整行操作等影响范围明确但较大，直接标记更可靠

**潜在边界问题**：
- 如果有非顺序的字符写入（如通过转义序列设置光标后写），`write_at_cursor` 不标记 damage
- 但实际上，光标移动走的是 `goto()` 等方法，那些方法会立即标记 damage_cursor()
- 所以组合起来是正确的：移动光标时标记旧位置+新位置，然后写入的字符会在 damage() 时被覆盖

---

## 四、完整数据流转图（含跨平台）

### 4.1 读取方向链路（子进程输出 → 终端 → 渲染）

**重要说明：** Unix 和 Windows 的数据流架构模型完全不同，**不能套用 Unix 的 master/slave 模型到 Windows**。

**管道创建核准**（[conpty.rs#L118-L130](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/windows/conpty.rs#L118-L130)）：
```rust
// 管道1：ConPTY 输出 → Alacritty 读取
let (conout, conout_pty_handle) = miow::pipe::anonymous(0)?;
//       ↑读端(我们持有)  ↑写端(传给 HPCON)

// 管道2：Alacritty 写入 → ConPTY 输入
let (conin_pty_handle, conin) = miow::pipe::anonymous(0)?;
//       ↑读端(传给 HPCON)  ↑写端(我们持有)

// CreatePseudoConsole(hInput, hOutput, ...)
(api.create)(size, conin_pty_handle, conout_pty_handle, 0, &mut pty_handle);
```

```
                       ┌─────────────────────────┐
                       │     子进程 (Shell)      │
                       └────────────┬────────────┘
                                    │
         ┌──────────────────────────┐
         │                          │
         ▼                          ▼
┌────────────────────────┐  ┌─────────────────────────────────────────────┐
│     Unix 模型          │  │        Windows ConPTY 模型                  │
│  (内核伪终端对 模式)    │  │  (HPCON 星型枢纽 + 两对独立匿名管道)       │
│                        │  │                                             │
│ 子进程 stdout/stderr   │  │  子进程通过 PROC_THREAD_ATTRIBUTE           │
│     ↓ 直接绑定 slave fd │  │  _PSEUDOCONSOLE 绑定到 HPCON               │
│ PTY slave (内核 tty 层) │  │     ↓ (控制台 I/O 由 ConPTY 内核层处理)     │
│     ↓ 内核行规程处理    │  │                                             │
│ PTY master File        │  │  HPCON (ConPTY 内核)                       │
│ (Alacritty 持有)       │  │     ↓ 写入 hOutput                          │
│     ↓ read()           │  │  conout_pty_handle (管道1写端)              │
│ EventedReadWrite::     │  │     ↓ 内核管道传输                          │
│ reader()               │  │  conout (AnonRead 管道1读端)                │
│                        │  │     ↓ UnblockedReader::new()                │
└──────────┬─────────────┘  │     ├─ 后台线程: loop { 阻塞读 conout }     │
           │                │     │   读到数据 → piper pipe 写端           │
           │                │     └─ Alacritty 端: piper pipe 读端         │
           │                │        (注册到 polling，产生可读事件)        │
           │                │        → EventedReadWrite::reader()         │
           │                └──────────────────┬──────────────────────────┘
           └───────────────────────┬───────────┘
                                   │
                      ┌────────────▼────────────┐
                      │ EventedReadWrite trait  │
                      │    (平台无关抽象)        │
                      └────────────┬────────────┘
                              │
┌─────────────────────────────▼────────────────────────────────────┐
│                PTY reader 线程 (事件循环)                         │
│                                                                  │
│  Poller 等待可读事件                                              │
│      │                                                           │
│      ├── PTY_READ_WRITE_TOKEN → pty_read()                       │
│      │     ├── 读取原始字节到 buf                                 │
│      │     ├── 获取 FairMutex<Term> 锁                           │
│      │     └── state.parser.advance(term, &buf[..n])             │
│      │           └── VTE Parser 解析 → 调用 Handler trait        │
│      │                ├── input(char) → 写 Grid + 移动 cursor    │
│      │                ├── goto/scroll/... → 状态变更 + damage    │
│      │                └── ...                                    │
│      │                                                           │
│      └── PTY_CHILD_EVENT_TOKEN → next_child_event()              │
│           ├── Unix: 读 UnixStream 信号管道 → child.try_wait()       │
│           └── Windows: mpsc::Receiver::try_recv()                    │
│                (ChildExitWatcher 回调通过 IOCP post)                          │
│                                                                  │
│  → 发送 Event::Wakeup 通知主线程重绘                              │
└─────────────────────────────┬────────────────────────────────────┘
                              │
┌─────────────────────────────▼────────────────────────────────────┐
│                       主线程 (渲染)                               │
│                                                                  │
│  Display::draw()                                                  │
│      │                                                           │
│      ├── 1. RenderableContent 收集可渲染单元                      │
│      │                                                           │
│      ├── 2. terminal.damage()  ← 获取损坏信息                    │
│      │     ├── 如果 full → 全量重绘                               │
│      │     └── 如果 partial → 逐行检查 damage bounds             │
│      │           └── 注意：这里会用 last_cursor 推断 input 的 damage│
│      │                                                           │
│      ├── 3. terminal.reset_damage()  ← 重置损坏状态              │
│      │                                                           │
│      ├── 4. drop(terminal)  ← 尽早释放锁                         │
│      │                                                           │
│      ├── 5. DamageTracker 合并 UI 元素的 damage                  │
│      │     (光标、选择、提示、搜索栏等)                           │
│      │                                                           │
│      └── 6. Renderer::draw_cells() → OpenGL 渲染                 │
└──────────────────────────────────────────────────────────────────┘
```

### 4.2 写入方向链路（键盘 → 子进程输入）

```
┌──────────────────────────────────────────────────────────────────┐
│                    主线程 (winit 事件)                           │
│  键盘事件 → 输入处理 → Notifier::notify(bytes)                   │
└─────────────────────────────┬────────────────────────────────────┘
                              │
                    EventLoopSender::send(Msg::Input)
                              │
                              ▼
                    mpsc channel + poller.notify()
                              │
┌─────────────────────────────▼────────────────────────────────────┐
│                PTY reader 线程 (事件循环)                         │
│                                                                  │
│  poller.wait() 被唤醒                                             │
│      │                                                           │
│      ├── drain_recv_channel() → 写入 write_list 队列            │
│      ├── 注册 writable 兴趣                                      │
│      └── pty_write() → EventedReadWrite::writer().write()       │
│                                                                  │
└─────────────────────────────┬────────────────────────────────────┘
                              │
            ┌─────────────────┴─────────────────┐
            │                                   │
┌───────────▼───────────────────┐  ┌────────────▼─────────────────────────────┐
│        Unix 写入路径           │  │           Windows 写入路径                │
│                               │  │                                          │
│ PTY master File.write()      │  │ UnblockedWriter (Alacritty 端)           │
│      ↓ 写入 master fd         │  │      ↓ 写入                             │
│ 内核 PTY 层                   │  │ piper pipe 缓冲                          │
│      ↓ 行规程处理             │  │      ↓ (后台线程阻塞取数据)              │
│ PTY slave                    │  │ AnonWrite (管道2写端 conin)               │
│      ↓                        │  │      ↓ 内核管道传输                      │
│ 子进程 stdin                  │  │ conin_pty_handle (管道2读端，传给 HPCON)  │
│                               │  │      ↓ 作为 HPCON 的 hInput               │
└───────────────────┬───────────┘  │ HPCON (ConPTY 内核)                      │
                    │              │      ↓ 翻译控制台输入                    │
                    └──────┬───────┘      │ 传递到子进程                      │
                           │              └───────────────┬───────────────────┘
                           │                              │
                           ▼                              ▼
                        子进程获得输入数据
```

### 4.3 跨平台退出与释放链路对比

#### 4.3.1 子进程退出通知链路（8 个阶段）

| 阶段 | Unix 退出流程 | Windows 退出流程 |
|------|-------------|-------------------|
| ① 触发源 | 子进程调用 `exit()` / 收到终止信号 | 子进程调用 `ExitProcess()` / 被终止 |
| ② 内核通知 | 内核向父进程发送 `SIGCHLD` 信号 | 内核将子进程 HANDLE 置为 signaled 状态 |
| ③ 通知传递机制 | `signal-hook` 将 1 字节写入 `UnixStream` 配对 pipe | `RegisterWaitForSingleObject` 等待完成 |
| | → `poller` 检测到 pipe 可读事件 | → 系统线程池执行 `child_exit_callback()` |
| ④ 回调/转发动作 | (无额外回调，直接进入下一步) | 回调中：`GetExitCodeProcess()` 获取退出码 |
| | | → `mpsc::Sender` 发送 `ChildEvent::Exited` |
| | | → `poller.post(CompletionPacket)` 投递 IOCP 事件 |
| ⑤ 事件循环感知 | `Poller::wait()` 返回 `PTY_CHILD_EVENT_TOKEN` 可读 | `Poller::wait()` 返回 `PTY_CHILD_EVENT_TOKEN` 事件 |
| ⑥ 取退出状态 | ① 从信号管道读取 1 字节（清空） | `mpsc::Receiver::try_recv()` 取出事件 |
| | ② `child.try_wait()` → `Option<ExitStatus>` | 事件内已包含 `Option<ExitStatus>` |
| ⑦ 循环内收尾 | `drain_on_exit` 为真：`pty_read` 读取剩余数据 | 同左 |
| | `terminal.lock().exit()` → 标记终端退出 | 同左 |
| | `event_proxy.send_event(Event::Wakeup)` | 同左 |
| | `break 'event_loop` → 退出事件循环 | 同左 |
| ⑧ 线程结束清理 | `pty.deregister(poller)` → 注销事件源 | 同左 |
| | 线程函数返回 `(self, state)` | 同左 |

#### 4.3.2 Pty 资源释放顺序（Rust Drop 行为详解）

**Rust Drop 执行规则**：
1. 首先执行**自定义的 `impl Drop for T`** 块（如果存在）
2. 然后按**字段声明顺序**依次 drop 每个字段（先声明的先被释放）
3. 这个顺序是**语言保证**的，与优化级别无关

---

##### Unix Pty 释放顺序详解

**结构体字段声明**（[unix.rs#L102-L107](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/unix.rs#L102-L107)）：
```rust
pub struct Pty {
    child: Child,          // 声明位置 ①
    file: File,            // 声明位置 ②
    signals: UnixStream,   // 声明位置 ③
    sig_id: SigId,         // 声明位置 ④
}
```

**`impl Drop for Pty` 自定义逻辑**（[unix.rs#L309-L321](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/unix.rs#L309-L321)）：
```rust
impl Drop for Pty {
    fn drop(&mut self) {
        // 1. 发送 SIGHUP 终止子进程
        unsafe { libc::kill(self.child.id() as i32, libc::SIGHUP); }

        // 2. 注销 SIGCHLD 信号处理（signal_hook::low_level::unregister）
        unregister_signal(self.sig_id);

        // 3. wait 等待子进程彻底退出，回收 zombie 进程
        let _ = self.child.wait();
    }
}
```

**完整释放时序表**（严格对齐字段声明顺序）：

| 阶段 | 执行顺序 | 对应字段 | 操作内容 | 代码位置 |
|------|---------|---------|---------|---------|
| **Phase A** | 1st | (全部字段仍存活) | 执行 `impl Drop for Pty` 块 | [unix.rs#L309-L321](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/unix.rs#L309-L321) |
| | ① | 读取 `child` | `kill(child.pid, SIGHUP)` → 通知子进程终止 | [unix.rs#L313](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/unix.rs#L313) |
| | ② | 读取 `sig_id` | `unregister_signal(sig_id)` → 从 signal-hook 全局注册表移除 SIGCHLD 回调 | [unix.rs#L317](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/unix.rs#L317) |
| | ③ | 读取 `child` | `child.wait()` → 阻塞等待子进程退出，内核回收进程表项（避免 zombie） | [unix.rs#L319](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/unix.rs#L319) |
| **Phase B** | | | 自定义 Drop 执行完毕，开始按字段声明顺序 drop | |
| | 4th | `child` ① | `Child::drop()` → 关闭子进程句柄（释放 `ChildStdin`/`ChildStdout`/`ChildStderr` 等资源） | std::process |
| | 5th | `file` ② | `File::drop()` → `close(master_fd)` → 关闭 PTY master 文件描述符 → 内核检测到 master 关闭后，向 slave 端的前台进程组发送 SIGHUP，并释放内核 tty 行规程缓冲区 | std::fs |
| | 6th | `signals` ③ | `UnixStream::drop()` → `close(signals_fd)` → 关闭信号通知管道的接收端 fd → signal-hook 的发送端 fd 也会被清理 | std::os::unix::net |
| | 7th | `sig_id` ④ | `SigId::drop()` → **注意：SigId 本身不实现 Drop**，它是一个 usize 值类型，signal 注册已在 Phase A 的 `unregister_signal()` 中手动注销 | signal_hook crate |

> **设计要点**：
> - 信号注销必须在 `child.wait()` 之前：如果先 wait，子进程退出瞬间 SIGCHLD 可能再次触发，此时管道接收端已关闭会导致 panic
> - 自定义 Drop 块里需要 `&mut self` 访问所有字段，所以必须在字段被 drop 之前执行
> - Unix 关闭 master fd 后，内核会自动向 slave 端的前台进程组发送 SIGHUP（这是内核的标准 PTY 行为），所以 kill(SIGHUP) 是给整个进程组发的保险

---

##### Windows Pty 释放顺序详解（**生死攸关的字段顺序**）

**结构体字段声明**（[windows/mod.rs#L27-L34](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/windows/mod.rs#L27-L34)）：
```rust
pub struct Pty {
    // XXX: Backend is required to be the first field, to ensure correct drop order.
    // Dropping `conout` before `backend` will cause a deadlock (with Conpty).
    backend: Backend,          // 声明位置 ① ← 必须第一个！
    conout: ReadPipe,          // 声明位置 ②
    conin: WritePipe,          // 声明位置 ③
    child_watcher: ChildExitWatcher, // 声明位置 ④
}
```

**Pty 没有自定义 `impl Drop for Pty`，完全依赖字段声明顺序**。

**完整释放时序表**（严格对齐字段声明顺序）：

| 阶段 | 执行顺序 | 对应字段 | 操作内容 | 代码位置 |
|------|---------|---------|---------|---------|
| **Phase A** | 1st | `backend` ① | `impl Drop for Conpty` → 调用 Win32 `ClosePseudoConsole(hpc_handle)` | [conpty.rs#L97-L105](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/windows/conpty.rs#L97-L105) |
| | | | **关键点**：该函数会**阻塞**，直到 conout 管道中的所有数据被读取排空为止。这是微软文档规定的行为。 | |
| **Phase B** | 2nd | `conout` ② | `impl Drop for UnblockedReader<AnonRead>` → ① 发送停止信号给后台读取线程 ② join 等待线程退出 ③ 关闭内部 `piper` pipe（释放缓冲） ④ `close(conout_handle)` → 关闭 AnonRead 管道句柄 | [blocking.rs](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/windows/blocking.rs) |
| **Phase C** | 3rd | `conin` ③ | `impl Drop for UnblockedWriter<AnonWrite>` → ① 发送停止信号给后台写入线程 ② join 等待线程退出 ③ 关闭内部 `piper` pipe ④ `close(conin_handle)` → 关闭 AnonWrite 管道句柄 → HPCON 收到管道断开信号，向子进程发送 Ctrl+Z 或退出信号 | 同上 |
| **Phase D** | 4th | `child_watcher` ④ | `impl Drop for ChildExitWatcher` → 调用 Win32 `UnregisterWait(wait_handle)` → 取消 `RegisterWaitForSingleObject` 的等待回调注册，释放等待句柄 | [child.rs#L127-L133](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/windows/child.rs#L127-L133) |

**`child_watcher` 内部字段的 drop 顺序**（[child.rs#L52-L58](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/windows/child.rs#L52-L58)）：
```rust
pub struct ChildExitWatcher {
    wait_handle: AtomicPtr<c_void>,  // 字段①
    event_rx: mpsc::Receiver<...>,   // 字段②
    interest: Arc<Mutex<Option<Interest>>>, // 字段③
    child_handle: AtomicPtr<c_void>, // 字段④
    pid: Option<NonZeroU32>,         // 字段⑤
}
```

在 `impl Drop for ChildExitWatcher` 的自定义逻辑执行完后，按字段顺序继续：
- ① `wait_handle`: AtomicPtr 无特殊 drop
- ② `event_rx`: mpsc::Receiver drop，关闭通道接收端
- ③ `interest`: Arc<Mutex<...>> drop，引用计数 -1，若为 0 则释放 Mutex 和 Interest
- ④ `child_handle`: AtomicPtr 无特殊 drop（子进程句柄由 ConPTY 管理）
- ⑤ `pid`: Option<NonZeroU32> 无特殊 drop

---

> **🔴 Windows 死锁警告（为什么字段顺序不能写反）**：
>
> 如果把 `conout` 放在 `backend` 前面（错误顺序）：
> ```
> 错误: conout ② → backend ①
> ```
> 实际执行：
> 1. conout 先 drop → `close(conout_handle)` → 管道读端立即被关闭
> 2. backend 后 drop → 调用 `ClosePseudoConsole()`
> 3. 但 ClosePseudoConsole **阻塞等待** conout 管道数据被排空
> 4. 然而 conout 管道读端已经关闭，永远不会有"数据读完"的信号
> 5. → **线程永久阻塞，程序无法退出**
>
> 正确顺序 `backend ① → conout ②` 不会死锁，因为：
> 1. backend 先 drop → ClosePseudoConsole 阻塞等待
> 2. 此时 conout 管道句柄**仍然开放**，后台读取线程继续排空数据
> 3. 所有数据读完 → ClosePseudoConsole 返回
> 4. 然后 conout 再 drop，正常关闭管道

---

## 五、关键文件索引

| 文件路径 | 作用 | 核心内容 |
|---------|------|---------|
| [alacritty_terminal/src/tty/mod.rs](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/mod.rs) | 跨平台抽象层 | `EventedReadWrite` trait、`EventedPty` trait、`Options`/`Shell` 配置、`ChildEvent` 枚举 |
| [alacritty_terminal/src/tty/unix.rs](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/unix.rs) | Unix PTY 实现 | `Pty` 结构体、`new()` 创建函数、`pre_exec` 钩子、SIGCHLD 信号管道 |
| [alacritty_terminal/src/tty/windows/mod.rs](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/windows/mod.rs) | Windows PTY 封装 | `Pty` 结构体、Evented 实现、命令行参数转义 |
| [alacritty_terminal/src/tty/windows/conpty.rs](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/windows/conpty.rs) | ConPTY 核心 | `Conpty` 结构体、CreatePseudoConsole、CreateProcessW 流程 |
| [alacritty_terminal/src/tty/windows/child.rs](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/windows/child.rs) | 子进程退出监视 | `ChildExitWatcher`、RegisterWaitForSingleObject、IOCP 事件通知 |
| [alacritty_terminal/src/tty/windows/blocking.rs](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/windows/blocking.rs) | 非阻塞包装 | `UnblockedReader`、`UnblockedWriter`、后台线程 + piper 管道 |
| [alacritty_terminal/src/event_loop.rs](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/event_loop.rs) | I/O 事件循环 | `pty_read()`、`pty_write()`、事件分发、VTE 解析驱动 |
| [alacritty_terminal/src/term/mod.rs](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/term/mod.rs) | 终端状态 | `Term` 结构体、`Handler` trait 实现、`TermDamageState` damage 系统 |
| [alacritty/src/display/mod.rs](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty/src/display/mod.rs) | 渲染层 | `Display::draw()`、damage 收集、渲染调度 |
| [alacritty/src/window_context.rs](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty/src/window_context.rs) | 窗口上下文 | PTY 创建入口、事件循环启动、组件串联 |
