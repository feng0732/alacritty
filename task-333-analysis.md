# Alacritty VT 转义解析代码分析

## 1. 整体架构概览

Alacritty 的 VT 转义解析分为两层，由外部 `vte` crate（v0.15.0）提供：

```
PTY 字节流
   │
   ▼
┌──────────────────────────────────────────────────────────┐
│ vte::Parser（状态机，lib.rs）                              │
│   逐字节驱动状态机，产生 Perform trait 回调                   │
└──────────────────────┬───────────────────────────────────┘
                       │ Perform trait 方法
                       ▼
┌──────────────────────────────────────────────────────────┐
│ vte::ansi::Processor + Performer（ansi.rs）                │
│   Processor 持有 Parser + 同步更新状态                       │
│   Performer 实现 Perform，将原始回调翻译为 Handler 调用       │
└──────────────────────┬───────────────────────────────────┘
                       │ Handler trait 方法
                       ▼
┌──────────────────────────────────────────────────────────┐
│ alacritty_terminal::term::Term（term/mod.rs）               │
│   实现 Handler trait，操作 Grid/Cursor/Colors 等终端状态     │
└──────────────────────────────────────────────────────────┘
```

关键代码位置：
- 状态机核心：`vte` crate `src/lib.rs`（外部依赖，不在本仓库内）
- 高层处理器：`vte` crate `src/ansi.rs`（同上）
- Alacritty 入口：[event_loop.rs](file:///d:/fz/0601/solo-dogfeeding/code/333-alacritty/alacritty_terminal/src/event_loop.rs#L404)
- 终端状态实现：[term/mod.rs](file:///d:/fz/0601/solo-dogfeeding/code/333-alacritty/alacritty_terminal/src/term/mod.rs#L1059)
- 重导出：[lib.rs](file:///d:/fz/0601/solo-dogfeeding/code/333-alacritty/alacritty_terminal/src/lib.rs#L20)

---

## 2. 关键对象详解

### 2.1 `vte::Parser` — 底层状态机

**来源**：`vte` crate `src/lib.rs`

Parser 实现了 Paul Williams 的 ANSI 解析器状态机（[原文](https://vt100.net/emu/dec_ansi_parser)），核心字段：

```rust
pub struct Parser<const OSC_RAW_BUF_SIZE: usize = MAX_OSC_RAW> {
    state: State,                          // 当前状态
    intermediates: [u8; MAX_INTERMEDIATES], // 中间字节缓冲（最多2个）
    intermediate_idx: usize,
    params: Params,                        // CSI/DCS 参数
    param: u16,                            // 当前正在构建的参数
    osc_raw: Vec<u8>,                      // OSC 原始字节缓冲
    osc_params: [(usize, usize); MAX_OSC_PARAMS], // OSC 参数偏移
    osc_num_params: usize,
    ignoring: bool,                        // 超限时忽略标志
    partial_utf8: [u8; 4],                 // 跨 chunk 的 UTF-8 拼接
    partial_utf8_len: usize,
}
```

**状态枚举 `State`**：

| 状态 | 含义 |
|------|------|
| `Ground` | 基础态，普通字符直接输出 |
| `Escape` | 收到 ESC (0x1B) |
| `EscapeIntermediate` | ESC 后收到中间字节 (0x20-0x2F) |
| `CsiEntry` | ESC[ 之后 |
| `CsiParam` | 收到 CSI 参数字节 (0x30-0x39, 0x3B) |
| `CsiIntermediate` | CSI 参数后收到中间字节 |
| `CsiIgnore` | CSI 序列出错，忽略直到终结 |
| `DcsEntry` | ESC P 之后 |
| `DcsParam` | DCS 参数阶段 |
| `DcsIntermediate` | DCS 中间字节阶段 |
| `DcsPassthrough` | DCS 数据流阶段（逐字节 put） |
| `DcsIgnore` | DCS 序列出错，忽略 |
| `OscString` | ESC] 之后，OSC 数据阶段 |
| `SosPmApcString` | SOS/PM/APC 字符串阶段 |

**常量限制**：
- `MAX_INTERMEDIATES = 2`：中间字节最多 2 个
- `MAX_OSC_RAW = 1024`：OSC 原始缓冲（std 特性下是 Vec，无硬上限）
- `MAX_OSC_PARAMS = 16`：OSC 最多 16 个参数

### 2.2 `vte::ansi::Processor` — 高层处理器

**来源**：`vte` crate `src/ansi.rs`

Processor 封装了 Parser，额外管理：
- **同步更新（Synchronized Update）**：BSU/ESU (CSI ?2026h/l) 缓冲机制
- **前驱字符**：用于 REP（重复上一个字符）

```rust
pub struct Processor<T: Timeout = StdSyncHandler> {
    state: ProcessorState<T>,  // 包含 preceding_char 和 sync_state
    parser: crate::Parser,
}
```

### 2.3 `vte::ansi::Performer` — 适配器

**来源**：`vte` crate `src/ansi.rs`

Performer 实现 `vte::Perform` trait，是 Parser 和 Handler 之间的翻译层：

```rust
struct Performer<'a, H: Handler, T: Timeout> {
    state: &'a mut ProcessorState<T>,
    handler: &'a mut H,
    terminated: bool,  // 同步更新时置 true 让 Parser 提前终止
}
```

翻译规则：

| Perform 方法 | Handler 方法 |
|---|---|
| `print(c)` | `handler.input(c)` + 记录 preceding_char |
| `execute(HT)` | `handler.put_tab(1)` |
| `execute(BS)` | `handler.backspace()` |
| `execute(CR)` | `handler.carriage_return()` |
| `execute(LF/VT/FF)` | `handler.linefeed()` |
| `execute(BEL)` | `handler.bell()` |
| `execute(SUB)` | `handler.substitute()` |
| `execute(SI)` | `handler.set_active_charset(G0)` |
| `execute(SO)` | `handler.set_active_charset(G1)` |
| `csi_dispatch(...)` | 根据 action char + intermediates 分发到 30+ 个 Handler 方法 |
| `esc_dispatch(...)` | 根据 byte + intermediates 分发到 Handler 方法 |
| `osc_dispatch(...)` | 解析 OSC 参数后调用 handler.set_title/clipboard_store 等 |
| `hook/put/unhook` | 当前未处理，仅 debug 日志 |
| `terminated()` | 返回 `self.terminated` |

### 2.4 `vte::ansi::Handler` — 终端行为接口

**来源**：`vte` crate `src/ansi.rs`

Handler 定义了 30+ 个方法，涵盖终端所有语义动作。关键方法：

| 方法 | 语义 |
|---|---|
| `input(char)` | 显示字符 |
| `goto(line, col)` | 移动光标 |
| `move_up/down/forward/backward` | 光标移动 |
| `linefeed()` / `carriage_return()` | 换行 / 回车 |
| `clear_screen(ClearMode)` | 清屏 |
| `clear_line(LineClearMode)` | 清行 |
| `set_mode / unset_mode` | 设置/重置 ANSI 模式 |
| `set_private_mode / unset_private_mode` | 设置/重置 DEC 私有模式 |
| `terminal_attribute(Attr)` | SGR 属性（颜色、粗体等） |
| `set_scrolling_region` | 设置滚动区域 |
| `set_title` | 设置窗口标题 |
| `save_cursor_position / restore_cursor_position` | 保存/恢复光标 |
| `configure_charset / set_active_charset` | 字符集配置 |
| `reset_state` | 全量重置 |

### 2.5 `Term` — Alacritty 的 Handler 实现

**来源**：[term/mod.rs#L1059](file:///d:/fz/0601/solo-dogfeeding/code/333-alacritty/alacritty_terminal/src/term/mod.rs#L1059)

`impl<T: EventListener> Handler for Term<T>` 是整个解析链的终点。核心字段：

```rust
pub struct Term<T> {
    grid: Grid<Cell>,              // 主屏幕缓冲
    inactive_grid: Grid<Cell>,     // 备用屏幕缓冲
    active_charset: CharsetIndex,  // 当前字符集
    tabs: TabStops,                // 制表位
    mode: TermMode,                // 终端模式标志位
    scroll_region: Range<Line>,    // 滚动区域
    colors: Colors,                // 颜色状态
    cursor_style: Option<CursorStyle>,
    event_proxy: T,                // 事件发送代理
    title: Option<String>,
    title_stack: Vec<Option<String>>,
    keyboard_mode_stack: Vec<KeyboardModes>,
    damage: TermDamageState,       // 脏区域追踪
    config: Config,
}
```

---

## 3. 完整调用链追踪

以 `ESC[1;2H`（移动光标到第2行第3列）为例：

```
1. PTY 输出字节: [0x1B, 0x5B, 0x31, 0x3B, 0x32, 0x48]
                     ESC    [      1      ;      2      H

2. EventLoop::pty_read()
   └─ state.parser.advance(&mut **terminal, &buf[..unprocessed])
      └─ ansi::Processor::advance(handler, bytes)
         ├─ 检查 sync_state.timeout → 无同步更新
         └─ parser.advance_until_terminated(&mut performer, bytes)
            └─ 逐字节驱动状态机：

   字节 0x1B (ESC):
     state=Ground → anywhere() → state=Escape, reset_params()

   字节 0x5B ([):
     state=Escape → advance_esc() → state=CsiEntry, reset_params()

   字节 0x31 (1):
     state=CsiEntry → advance_csi_entry() → action_paramnext(0x31), state=CsiParam
     (params 缓冲: [1])

   字节 0x3B (;):
     state=CsiParam → advance_csi_param() → action_param()
     (params 缓冲: [1, 默认值])

   字节 0x32 (2):
     state=CsiParam → advance_csi_param() → action_paramnext(0x32)
     (params 缓冲: [1, 2])

   字节 0x48 (H):
     state=CsiParam → advance_csi_param() → action_csi_dispatch()
     → performer.csi_dispatch(params=[1,2], intermediates=[], ignore=false, action='H')
     → handler.goto(line=1-1=0, col=2-1=1)
     → Term::goto(0, 1)
       └─ 更新 grid.cursor.point.line 和 column
       └─ damage_cursor()
```

以 `ESC]0;title\x07`（设置窗口标题）为例：

```
1. 字节: [0x1B, 0x5D, 0x30, 0x3B, 0x74, 0x69, 0x74, 0x6C, 0x65, 0x07]
           ESC    ]      0      ;     t      i      t      l      e    BEL

   0x1B: Ground → Escape
   0x5D: Escape → advance_esc() → osc_raw.clear(), state=OscString
   0x30: OscString → action_osc_put(0x30), osc_raw=[0x30]
   0x3B: OscString → action_osc_put_param(), 记录参数边界
   0x74-0x65: OscString → action_osc_put(byte) 逐个写入 osc_raw
   0x07: OscString → osc_end(performer) → performer.osc_dispatch(params, bell_terminated=true)
     → 解析 params[0]=0 (图标/标题), params[1]="title"
     → handler.set_title(Some("title"))
     → Term::set_title → event_proxy.send_event(Event::Title("title"))
```

---

## 4. CSI 分发表

Performer.csi_dispatch 依据 `(action_char, intermediates)` 组合分发：

### 光标移动类

| Action | Intermediates | Handler 调用 | ANSI 名称 |
|--------|--------------|-------------|-----------|
| `@` | - | `insert_blank(n)` | ICH |
| `A` | - | `move_up(n)` | CUU |
| `B` / `e` | - | `move_down(n)` | CUD |
| `C` / `a` | - | `move_forward(n)` | CUF |
| `D` | - | `move_backward(n)` | CUB |
| `E` | - | `move_down_and_cr(n)` | CNL |
| `F` | - | `move_up_and_cr(n)` | CPL |
| `G` / `` ` `` | - | `goto_col(n-1)` | CHA |
| `H` / `f` | - | `goto(row-1, col-1)` | CUP |
| `d` | - | `goto_line(n-1)` | VPA |

### 编辑类

| Action | Handler 调用 | ANSI 名称 |
|--------|-------------|-----------|
| `J` | `clear_screen(mode)` | ED |
| `K` | `clear_line(mode)` | EL |
| `L` | `insert_blank_lines(n)` | IL |
| `M` | `delete_lines(n)` | DL |
| `P` | `delete_chars(n)` | DCH |
| `X` | `erase_chars(n)` | ECH |
| `S` | `scroll_up(n)` | SU |
| `T` | `scroll_down(n)` | SD |
| `I` | `move_forward_tabs(n)` | CHT |
| `Z` | `move_backward_tabs(n)` | CBT |
| `g` | `clear_tabs(mode)` | TBC |

### 模式类

| Action | Intermediates | Handler 调用 | ANSI 名称 |
|--------|--------------|-------------|-----------|
| `h` | - | `set_mode(Mode)` | SM |
| `h` | `?` | `set_private_mode(PrivateMode)` | DECSET |
| `l` | - | `unset_mode(Mode)` | RM |
| `l` | `?` | `unset_private_mode(PrivateMode)` | DECRST |
| `p` | `$` | `report_mode(Mode)` | DECRQM |
| `p` | `?$` | `report_private_mode(PrivateMode)` | DECRQM |

### 属性/状态类

| Action | Intermediates | Handler 调用 | ANSI 名称 |
|--------|--------------|-------------|-----------|
| `m` | - | `terminal_attribute(Attr)` | SGR |
| `m` | `>` | `set_modify_other_keys(mode)` | 设置 modifyOtherKeys |
| `m` | `?` | `report_modify_other_keys()` | 查询 modifyOtherKeys |
| `q` | ` ` (空格) | `set_cursor_style(style)` | DECSCUSR |
| `r` | - | `set_scrolling_region(top, bottom)` | DECSTBM |
| `s` | - | `save_cursor_position()` | SCOSC |
| `u` | - | `restore_cursor_position()` | SCORC |
| `c` | 可选 | `identify_terminal(intermediate)` | DA |
| `n` | - | `device_status(arg)` | DSR |
| `b` | - | 重复 preceding_char | REP |

### Kitty 键盘协议

| Action | Intermediates | Handler 调用 |
|--------|--------------|-------------|
| `u` | `?` | `report_keyboard_mode()` |
| `u` | `=` | `set_keyboard_mode(mode, behavior)` |
| `u` | `>` | `push_keyboard_mode(mode)` |
| `u` | `<` | `pop_keyboard_modes(n)` |

### 窗口操作 (CSI t)

| 参数 | Handler 调用 |
|------|-------------|
| 14 | `text_area_size_pixels()` |
| 18 | `text_area_size_chars()` |
| 22 | `push_title()` |
| 23 | `pop_title()` |

---

## 5. ESC 分发表

Performer.esc_dispatch 依据 `(byte, intermediates)` 组合分发：

| Byte | Intermediates | Handler 调用 | ANSI 名称 |
|------|--------------|-------------|-----------|
| `B` | `(`/`)`/`*`/`+` | `configure_charset(G0..G3, Ascii)` | 指定 ASCII 字符集 |
| `0` | `(`/`)`/`*`/`+` | `configure_charset(G0..G3, SpecialLineDrawing)` | 指定线条绘制字符集 |
| `D` | - | `linefeed()` | IND |
| `E` | - | `linefeed()` + `carriage_return()` | NEL |
| `H` | - | `set_horizontal_tabstop()` | HTS |
| `M` | - | `reverse_index()` | RI |
| `Z` | - | `identify_terminal(None)` | DECID |
| `c` | - | `reset_state()` | RIS |
| `7` | - | `save_cursor_position()` | DECSC |
| `8` | `#` | `decaln()` | DECALN |
| `8` | - | `restore_cursor_position()` | DECRC |
| `=` | - | `set_keypad_application_mode()` | DECKPAM |
| `>` | - | `unset_keypad_application_mode()` | DECKPNM |
| `\` | - | no-op | ST |

---

## 6. 异常兜底与错误恢复机制

### 6.1 Parser 层：参数溢出 → `ignoring` 标志

当 CSI/DCS 参数超过 `Params` 容量时，Parser 设置 `ignoring = true`：

```rust
fn action_csi_dispatch(&mut self, performer, byte) {
    if self.params.is_full() {
        self.ignoring = true;  // 参数超限，标记忽略
    } else {
        self.params.push(self.param);
    }
    performer.csi_dispatch(self.params(), self.intermediates(), self.ignoring, byte as char);
    // 即使 ignoring=true 也会回调，但 Handler 可据此跳过处理
}
```

### 6.2 Parser 层：中间字节超限 → CsiIgnore / DcsIgnore

当中间字节超过 `MAX_INTERMEDIATES=2` 时，`action_collect` 设置 `ignoring=true`，后续在 CSI 中若再收到参数字节则转入 `CsiIgnore` 状态：

```rust
fn advance_csi_intermediate(performer, byte) {
    match byte {
        0x30..=0x3F => self.state = State::CsiIgnore,  // 参数出现在中间字节后→忽略
        ...
    }
}
```

`CsiIgnore` 态下所有字节被静默吞掉，直到终结字节 (0x40-0x7E) 才回到 Ground。

### 6.3 Parser 层：`anywhere` 处理 — 紧急重置

在任何状态下收到 CAN (0x18) 或 SUB (0x1A)，状态机立即回到 Ground：

```rust
fn anywhere(performer, byte) {
    match byte {
        0x18 | 0x1A => { performer.execute(byte); self.state = State::Ground }
        0x1B => { self.reset_params(); self.state = State::Escape }
        _ => (),  // 其他未知字节静默忽略
    }
}
```

### 6.4 Performer 层：未识别序列的 `unhandled!()` 宏

对于无法识别的 `(action, intermediates)` 组合，csi_dispatch 和 esc_dispatch 调用 `unhandled!()` 宏（通常是 debug 日志），不会崩溃，只是静默丢弃。

### 6.5 Handler 层（Term）：未知模式的优雅降级

```rust
fn set_mode(&mut self, mode: ansi::Mode) {
    let mode = match mode {
        ansi::Mode::Named(mode) => mode,
        ansi::Mode::Unknown(mode) => {
            debug!("Ignoring unknown mode {mode} in set_mode");
            return;  // 未知模式直接返回，不做任何操作
        },
    };
    // ... 处理已知模式
}
```

同样的模式用于 `set_private_mode`、`unset_mode`、`unset_private_mode`、`report_mode`、`report_private_mode`。

### 6.6 Handler 层（Term）：Kitty 键盘协议守卫

所有 Kitty 键盘协议方法入口都有配置守卫：

```rust
fn push_keyboard_mode(&mut self, mode: KeyboardModes) {
    if !self.config.kitty_keyboard {
        return;  // 未启用时静默忽略
    }
    // ...
}
```

### 6.7 Handler 层（Term）：OSC 52 剪贴板权限控制

```rust
fn clipboard_store(&mut self, clipboard: u8, base64: &[u8]) {
    if !matches!(self.config.osc52, Osc52::OnlyCopy | Osc52::CopyPaste) {
        debug!("Denied osc52 store");
        return;  // 权限不足时拒绝
    }
    // ...
}
```

### 6.8 Handler 层（Term）：越界保护

所有光标操作都使用 `cmp::min`/`cmp::max` 或 `saturating_sub` 来钳位：

```rust
fn goto(&mut self, line: i32, col: usize) {
    let (y_offset, max_y) = if self.mode.contains(TermMode::ORIGIN) {
        (self.scroll_region.start, self.scroll_region.end - 1)
    } else {
        (Line(0), self.bottommost_line())
    };
    self.grid.cursor.point.line = cmp::max(cmp::min(line + y_offset, max_y), Line(0));
    self.grid.cursor.point.column = cmp::min(col, self.last_column());
}
```

### 6.9 同步更新的超时兜底

如果同步更新超时（150ms）未收到 ESU，event loop 的 poll 会超时触发：

```rust
if events.is_empty() && self.rx.peek().is_none() {
    state.parser.stop_sync(&mut *self.terminal.lock());
    self.event_proxy.send_event(Event::Wakeup);
}
```

同时如果同步缓冲超过 2MiB（`SYNC_BUFFER_SIZE`），也会强制终止同步更新。

### 6.10 Term 层：reset_state 作为终极兜底

收到 `ESC c`（RIS）时，Term 执行全量重置，回到初始状态：

```rust
fn reset_state(&mut self) {
    // 交换回主屏、重置字符集/光标/Grid/TabStops/标题栈/键盘模式栈
    // 只保留 Vi 模式标志
    self.mode &= TermMode::VI;
    self.mode.insert(TermMode::default());
}
```

---

## 7. EventLoop 中的调用衔接

**来源**：[event_loop.rs#L103-L170](file:///d:/fz/0601/solo-dogfeeding/code/333-alacritty/alacritty_terminal/src/event_loop.rs#L103)

```rust
fn pty_read(&mut self, state: &mut State, buf: &mut [u8], ...) {
    loop {
        // 1. 从 PTY 读取字节到 buf
        match self.pty.reader().read(&mut buf[unprocessed..]) { ... }

        // 2. 尝试获取终端锁（FairMutex）
        let terminal = match &mut terminal {
            Some(t) => t,
            None => match self.terminal.try_lock_unfair() {
                None if unprocessed >= READ_BUFFER_SIZE => self.terminal.lock_unfair(),
                None => continue,  // 暂时拿不到锁就先不解析
                Some(t) => t,
            },
        };

        // 3. 核心：将字节喂给解析器
        state.parser.advance(&mut **terminal, &buf[..unprocessed]);

        // 4. 控制持锁时间，避免阻塞 UI 线程
        if processed >= MAX_LOCKED_READ { break; }
    }

    // 5. 如果有非同步更新的字节被处理，通知 UI 重绘
    if state.parser.sync_bytes_count() < processed && processed > 0 {
        self.event_proxy.send_event(Event::Wakeup);
    }
}
```

State 结构体持有解析器实例：

```rust
pub struct State {
    write_list: VecDeque<Cow<'static, [u8]>>,
    writing: Option<Writing>,
    parser: ansi::Processor,  // ← 这就是整个解析链的入口
}
```

---

## 8. 总结

| 层次 | 对象 | 职责 |
|------|------|------|
| I/O | EventLoop + PTY | 读取字节流、管理终端锁 |
| 解析 | vte::Parser | 状态机，将字节流转为 Perform 回调 |
| 适配 | vte::ansi::Processor + Performer | 同步更新管理、Perform→Handler 翻译 |
| 执行 | Term (impl Handler) | 操作 Grid/Cursor/Colors，产生事件通知 |
| 兜底 | Parser ignoring + CsiIgnore + anywhere + Handler guard | 超限忽略、紧急重置、越界钳位、权限检查 |

整个设计遵循 **解析与执行分离** 的原则：Parser 不理解语义，Performer 做语法→语义的翻译，Handler 实现具体终端行为。这使得每一层都可以独立测试和替换。
