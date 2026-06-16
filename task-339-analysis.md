# Alacritty PTY 与子进程运转机制分析

## 一、整体架构概览

Alacritty 的 PTY（伪终端）与子进程系统采用**多线程异步 I/O** 架构，主要分为三层：

```
┌──────────────────────────────────────────────────────────┐
│                     渲染层 (主线程)                      │
│  Display / Renderer / DamageTracker                      │
│  (负责获取终端状态并绘制到屏幕)                           │
└───────────────────────┬──────────────────────────────────┘
                        │  Arc<FairMutex<Term>>
┌───────────────────────▼──────────────────────────────────┐
│                     终端状态层                            │
│  Term / Grid / VTE Parser                                │
│  (维护终端缓冲区，解析 ANSI 转义序列)                     │
└───────────────────────┬──────────────────────────────────┘
                        │  EventedPty trait
┌───────────────────────▼──────────────────────────────────┐
│                  PTY I/O 线程 ("PTY reader")             │
│  EventLoop / Poller / PTY                                │
│  (负责与子进程通信，读写 PTY 数据)                        │
└───────────────────────┬──────────────────────────────────┘
                        │  fork/exec
┌───────────────────────▼──────────────────────────────────┐
│                    子进程 (Shell)                        │
│  /bin/bash 或用户指定的程序                               │
└──────────────────────────────────────────────────────────┘
```

---

## 二、入口条件与创建流程

### 2.1 触发入口

PTY 的创建始于 [WindowContext::new()](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty/src/window_context.rs#L169-L258)，在程序启动或新建窗口时被调用。

**入口条件**：
1. 已完成 OpenGL 显示环境初始化（`Display::new`）
2. 已创建终端状态对象 `Term`
3. 已配置 PTY 选项（shell 程序、工作目录、环境变量等）

### 2.2 PTY 创建流程（Unix 平台）

核心实现在 [tty::new()](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/unix.rs#L195-L307) 中：

#### 步骤 1: 创建 PTY 主从设备
```rust
let pty = openpty(None, Some(&window_size.to_winsize()))?;
let (master, slave) = (pty.controller, pty.user);
```
- 使用 `rustix_openpty::openpty()` 创建伪终端对
- 同时设置初始窗口大小（`Winsize`）

#### 步骤 2: 配置终端属性
```rust
#[cfg(any(target_os = "linux", target_os = "macos"))]
if let Ok(mut termios) = termios::tcgetattr(&master) {
    termios.input_modes.set(InputModes::IUTF8, true);
    let _ = termios::tcsetattr(&master, OptionalActions::Now, &termios);
}
```
- 设置字符编码为 UTF-8

#### 步骤 3: 获取用户信息
通过 [ShellUser::from_env()](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/unix.rs#L129-L159) 获取：
- 用户名（优先 `$USER`，其次 `getpwuid_r`）
- 主目录（优先 `$HOME`，其次 `passwd` 条目）
- Shell 程序（优先 `$SHELL`，其次 `passwd` 条目）

#### 步骤 4: 构建子进程命令
```rust
let mut builder = if let Some(shell) = config.shell.as_ref() {
    let mut cmd = Command::new(&shell.program);
    cmd.args(shell.args.as_slice());
    cmd
} else {
    default_shell_command(&user.shell, &user.user, &user.home)
};
```

**macOS 特殊处理**：使用 `/usr/bin/login` 命令确保 shell 作为 tty session 运行。

#### 步骤 5: 配置子进程标准流
```rust
builder.stdin(slave.try_clone()?);
builder.stderr(slave.try_clone()?);
builder.stdout(slave);
```
- 子进程的 stdin/stdout/stderr 全部重定向到 PTY 的 slave 端

#### 步骤 6: 配置环境变量
```rust
builder.env("ALACRITTY_WINDOW_ID", &window_id);
builder.env("USER", user.user);
builder.env("HOME", user.home);
builder.env("WINDOWID", window_id);
// 用户自定义环境变量
for (key, value) in &config.env {
    builder.env(key, value);
}
// 移除不需要的启动通知环境变量
builder.env_remove("XDG_ACTIVATION_TOKEN");
builder.env_remove("DESKTOP_STARTUP_ID");
```

#### 步骤 7: 预执行钩子（pre_exec）
这是子进程 fork 后、exec 前在**子进程上下文中**执行的关键代码：

```rust
unsafe {
    builder.pre_exec(move || {
        // 1. 创建新进程组
        let err = libc::setsid();
        if err == -1 {
            return Err(Error::last_os_error());
        }

        // 2. 设置工作目录
        if let Some(working_directory) = working_directory.as_ref() {
            libc::chdir(working_directory.as_ptr());
        }

        // 3. 设置控制终端
        set_controlling_terminal(slave_fd)?;

        // 4. 关闭不需要的文件描述符
        libc::close(slave_fd);
        libc::close(master_fd);

        // 5. 重置信号处理为默认值
        libc::signal(libc::SIGCHLD, libc::SIG_DFL);
        libc::signal(libc::SIGHUP, libc::SIG_DFL);
        // ... 更多信号重置
    });
}
```

#### 步骤 8: 注册 SIGCHLD 信号处理
```rust
let (signals, sig_id) = {
    let (sender, recv) = UnixStream::pair()?;
    let sig_id = signal_pipe::register(sigconsts::SIGCHLD, sender)?;
    recv.set_nonblocking(true)?;
    (recv, sig_id)
};
```
- 使用 Unix socket 管道接收 SIGCHLD 信号，避免在信号处理函数中执行复杂操作

#### 步骤 9: 启动子进程
```rust
match builder.spawn() {
    Ok(child) => {
        unsafe { set_nonblocking(master_fd); }
        Ok(Pty { child, file: File::from(master), signals, sig_id })
    },
    Err(err) => Err(Error::new(err.kind(), format!("Failed to spawn command..."))),
}
```
- 设置 master 端为非阻塞模式
- 返回 `Pty` 结构体，包含子进程句柄、master 文件、信号管道

---

## 三、内部流转机制

### 3.1 事件循环启动

在 [WindowContext::new()](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty/src/window_context.rs#L214-L227) 中创建并启动事件循环：

```rust
let event_loop = PtyEventLoop::new(
    Arc::clone(&terminal),
    event_proxy.clone(),
    pty,
    pty_config.drain_on_exit,
    config.debug.ref_test,
)?;

let loop_tx = event_loop.channel();
let _io_thread = event_loop.spawn();  // 启动 "PTY reader" 线程
```

### 3.2 事件循环核心逻辑

[EventLoop::spawn()](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/event_loop.rs#L205-L324) 是整个数据流转的核心：

```rust
pub fn spawn(mut self) -> JoinHandle<(Self, State)> {
    thread::spawn_named("PTY reader", move || {
        // 1. 注册 PTY 到 poller
        unsafe { self.pty.register(&self.poll, interest, poll_opts) };

        'event_loop: loop {
            // 2. 等待 I/O 事件（阻塞）
            self.poll.wait(&mut events, timeout)?;

            // 3. 处理来自 UI 线程的消息
            if !self.drain_recv_channel(&mut state) {
                break;
            }

            // 4. 分发事件
            for event in events.iter() {
                match event.key {
                    tty::PTY_CHILD_EVENT_TOKEN => {
                        // 处理子进程退出事件
                        if let Some(ChildEvent::Exited(status)) = self.pty.next_child_event() {
                            // 读取剩余输出、标记终端退出、通知主线程
                            break 'event_loop;
                        }
                    },
                    tty::PTY_READ_WRITE_TOKEN => {
                        if event.readable {
                            self.pty_read(&mut state, &mut buf, pipe.as_mut())?;
                        }
                        if event.writable {
                            self.pty_write(&mut state)?;
                        }
                    },
                }
            }
        }
    })
}
```

### 3.3 PTY 读取流程（子进程 → 终端）

[pty_read()](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/event_loop.rs#L104-L171) 是数据流入的关键：

```rust
fn pty_read<X>(&mut self, state: &mut State, buf: &mut [u8], mut writer: Option<&mut X>) -> io::Result<()>
{
    loop {
        // 1. 从 PTY master 读取原始字节
        match self.pty.reader().read(&mut buf[unprocessed..]) {
            Ok(got) => unprocessed += got,
            Err(err) => {
                if err.kind() == ErrorKind::WouldBlock && unprocessed == 0 {
                    break;
                }
            },
        }

        // 2. 获取终端锁（FairMutex）
        let terminal = match &mut terminal {
            None => terminal.insert(match self.terminal.try_lock_unfair() {
                // 缓冲区满时强制阻塞等待锁
                None if unprocessed >= READ_BUFFER_SIZE => self.terminal.lock_unfair(),
                None => continue,  // 锁被占用，继续读更多数据
                Some(terminal) => terminal,
            }),
        };

        // 3. 调用 VTE parser 解析字节流
        state.parser.advance(&mut **terminal, &buf[..unprocessed]);

        // 4. 检查是否需要暂停解析避免阻塞终端太久
        if processed >= MAX_LOCKED_READ {
            break;
        }
    }

    // 5. 通知主线程重绘
    if state.parser.sync_bytes_count() < processed && processed > 0 {
        self.event_proxy.send_event(Event::Wakeup);
    }
}
```

**关键设计点**：
- `READ_BUFFER_SIZE = 0x10_0000` (1MB)：单次最大读取量
- `MAX_LOCKED_READ = u16::MAX` (65535)：持有终端锁期间最大解析量，避免阻塞渲染线程
- 使用 `FairMutex` 保证线程公平性

### 3.4 VTE 解析与终端更新

`state.parser.advance()` 调用 [vte::ansi::Processor](https://github.com/alacritty/vte) 解析 ANSI 转义序列，最终调用 [Term](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/term/mod.rs#L1059-L...) 实现的 `Handler` trait：

```rust
impl<T: EventListener> Handler for Term<T> {
    /// 普通字符输入
    fn input(&mut self, c: char) {
        let width = c.width().unwrap_or(0);
        
        if width == 0 {
            // 处理零宽字符
            self.grid[line][column].push_zerowidth(c);
            return;
        }

        // 处理自动换行
        if self.grid.cursor.input_needs_wrap {
            self.wrapline();
        }

        // 写入字符到网格
        if width == 1 {
            self.write_at_cursor(c);
        } else {
            // 处理宽字符（中文等）
            self.write_at_cursor(c);
            self.grid.cursor.point.column += 1;
            self.write_at_cursor(' ');  // 占位符
        }

        // 移动光标
        if self.grid.cursor.point.column + 1 < columns {
            self.grid.cursor.point.column += 1;
        } else {
            self.grid.cursor.input_needs_wrap = true;
        }
    }

    // ... 其他 Handler 方法：goto, move_up, insert_blank, set_mode 等
}
```

### 3.5 PTY 写入流程（终端 → 子进程）

[pty_write()](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/event_loop.rs#L174-L203) 处理键盘输入等数据发往子进程：

```rust
fn pty_write(&mut self, state: &mut State) -> io::Result<()> {
    'write_many: while let Some(mut current) = state.take_current() {
        'write_one: loop {
            match self.pty.writer().write(current.remaining_bytes()) {
                Ok(0) => {
                    state.set_current(Some(current));
                    break 'write_many;
                },
                Ok(n) => {
                    current.advance(n);
                    if current.finished() {
                        state.goto_next();
                        break 'write_one;
                    }
                },
                Err(err) => {
                    state.set_current(Some(current));
                    match err.kind() {
                        ErrorKind::WouldBlock => break 'write_many,
                        _ => return Err(err),
                    }
                },
            }
        }
    }
    Ok(())
}
```

**写入触发流程**：
1. 用户按键 → 主线程捕获键盘事件
2. 通过 `Notifier::notify()` 发送 `Msg::Input` 到事件循环
3. `EventLoopSender::send()` 发送消息并通过 `poller.notify()` 唤醒 I/O 线程
4. 事件循环处理消息，加入 `write_list` 队列
5. 注册写事件兴趣，poller 通知可写时执行 `pty_write()`

---

## 四、结果输出到渲染层

### 4.1 Damage 追踪机制

终端状态更新时，会通过 [TermDamageState](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/term/mod.rs#L217-L266) 记录"损坏"区域：

```rust
struct TermDamageState {
    full: bool,                    // 是否全部损坏
    lines: Vec<LineDamageBounds>,  // 每行的损坏列范围
    last_cursor: Point,            // 上一帧光标位置
}

// 损坏单行的某列范围
fn damage_line(&mut self, line: usize, left: usize, right: usize) {
    self.lines[line].expand(left, right);
}

// 标记全部损坏
fn mark_fully_damaged(&mut self) {
    self.damage.full = true;
}
```

**损坏标记时机**：
- 光标移动时：损坏旧光标和新光标位置
- 字符写入时：损坏写入位置
- 滚动时：标记全部损坏
- 窗口大小改变时：标记全部损坏

### 4.2 渲染层获取损坏信息

在 [Display::draw()](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty/src/display/mod.rs#L775-L880) 中：

```rust
pub fn draw<T: EventListener>(
    &mut self,
    mut terminal: MutexGuard<'_, Term<T>>,
    scheduler: &mut Scheduler,
    message_buffer: &MessageBuffer,
    config: &UiConfig,
    search_state: &mut SearchState,
) {
    // 1. 收集可渲染内容（迭代器模式）
    let mut content = RenderableContent::new(config, self, &terminal, search_state);
    let mut grid_cells = Vec::new();
    for cell in &mut content {
        grid_cells.push(cell);
    }

    // 2. 获取终端 damage 信息
    match terminal.damage() {
        TermDamage::Full => self.damage_tracker.frame().mark_fully_damaged(),
        TermDamage::Partial(damaged_lines) => {
            for damage in damaged_lines {
                self.damage_tracker.frame().damage_line(damage);
            }
        },
    }
    terminal.reset_damage();  // 重置 damage 状态

    // 3. 尽早释放终端锁
    drop(terminal);

    // 4. 添加 UI 元素的 damage（提示、搜索栏等）
    let requires_full_damage = self.visual_bell.intensity() != 0.
        || self.hint_state.active()
        || search_state.regex().is_some();
    if requires_full_damage {
        self.damage_tracker.frame().mark_fully_damaged();
    }

    // 5. 实际渲染
    self.renderer.clear(background_color, config.window_opacity());
    self.renderer.draw_cells(&size_info, glyph_cache, cells);
    
    // ... 渲染光标、下划线、提示等
}
```

### 4.3 完整数据流图

```
子进程输出字节
    │
    ▼
PTY slave 端 → PTY master 端
    │
    ▼
[PTY reader 线程]
pty_read() 读取字节 → state.parser.advance()
    │
    ▼
VTE Parser 解析 → 调用 Term Handler 方法
    │
    ▼
[Term 状态更新]
写入 Grid → 标记 Damage → 发送 Wakeup 事件
    │
    ▼
[主线程]
接收 Wakeup 事件 → 调用 Display::draw()
    │
    ▼
获取 Damage → 收集 RenderableContent → Renderer 绘制
    │
    ▼
OpenGL 渲染 → 屏幕输出
```

---

## 五、关键技术点总结

### 5.1 线程同步
- 使用 `Arc<FairMutex<Term>>` 共享终端状态
- `FairMutex` 保证锁的公平性，避免 I/O 线程饥饿
- I/O 线程尽量缩短持有锁的时间（`MAX_LOCKED_READ` 限制）

### 5.2 非阻塞 I/O
- PTY master 端设置为 `O_NONBLOCK`
- 使用 `polling` crate 的 `Poller` 进行事件驱动 I/O
- 读写操作都处理 `WouldBlock` 错误

### 5.3 信号处理
- SIGCHLD 通过 Unix socket 管道传递到事件循环
- 避免在信号处理函数中执行非异步安全操作
- PTY drop 时发送 SIGHUP 终止子进程

### 5.4 渲染优化
- Damage 机制只重绘变化的区域
- 提前释放终端锁避免阻塞 I/O
- 同步更新（`sync_bytes_count`）机制减少不必要的重绘

---

## 六、涉及核心文件清单

| 文件 | 作用 |
|------|------|
| [alacritty/src/window_context.rs](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty/src/window_context.rs) | PTY 创建入口，连接各组件 |
| [alacritty_terminal/src/tty/unix.rs](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/unix.rs) | Unix 平台 PTY 创建与子进程管理 |
| [alacritty_terminal/src/tty/mod.rs](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/tty/mod.rs) | PTY trait 定义与跨平台抽象 |
| [alacritty_terminal/src/event_loop.rs](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/event_loop.rs) | PTY I/O 事件循环核心 |
| [alacritty_terminal/src/term/mod.rs](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty_terminal/src/term/mod.rs) | 终端状态与 Handler trait 实现 |
| [alacritty/src/display/mod.rs](file:///d:/fz/0601/solo-dogfeeding/code/339-alacritty/alacritty/src/display/mod.rs) | 渲染层入口与 damage 处理 |
