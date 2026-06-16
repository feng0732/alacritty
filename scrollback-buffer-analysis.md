# Alacritty 滚动缓冲用户滚动入口完整分析（修正版）

> **修正要点**：本文档特别澄清了此前的事实偏差——`ViModeCursor::scroll()` 不滚动视口（只计算光标位置），`vi_motion` 的自动滚动发生在 `ViModeCursor::motion()` 的末尾调用 `scroll_to_point`，搜索流程的滚动恢复仅在 Vi 模式下生效。新增 5.10 节详细对照三类搜索动作（顶层/ViAction/SearchAction）的滚动行为，特别是行内搜索（f/F/t/T）的两步流程（启动不滚动、输入字符后跳转）与语义搜索（*/#）的多次跳转特性。

---

## 一、核心数据结构回顾

### 1.1 视口偏移 `display_offset`

`Grid::display_offset` 是用户滚动的核心状态变量，表示用户向上滚动查看历史的行数：

```rust
// Grid 定义
pub struct Grid<T> {
    raw: Storage<T>,              // 环形缓冲区存储行数据
    display_offset: usize,        // ← 用户滚动视口偏移（0=在底部，正值=查看历史）
    max_scroll_limit: usize,      // scrollback 最大行数
    // ...
}

// Grid::scroll_display 修改 display_offset
pub fn scroll_display(&mut self, scroll: Scroll) {
    self.display_offset = match scroll {
        Scroll::Delta(count) => {
            min(max((self.display_offset as i32) + count, 0) as usize, self.history_size())
        },
        Scroll::PageUp => min(self.display_offset + self.lines, self.history_size()),
        Scroll::PageDown => self.display_offset.saturating_sub(self.lines),
        Scroll::Top => self.history_size(),
        Scroll::Bottom => 0,
    };
}
```

- `display_offset = 0`：视口在底部，显示最新内容
- `display_offset > 0`：用户向上滚动，查看 `display_offset` 行之前的历史
- `display_offset = history_size()`：视口在最顶部，显示最早的历史

### 1.2 `Scroll` 枚举

```rust
pub enum Scroll {
    Delta(i32),      // 滚动指定行数（正=向上，负=向下）
    PageUp,          // 向上翻一页
    PageDown,        // 向下翻一页
    Top,             // 滚动到最顶部
    Bottom,          // 滚动到底部
}
```

### 1.3 三个层级的 "scroll" 方法辨析（关键！）

这是最容易混淆的地方，三个不同的 `scroll` 方法含义完全不同：

| 方法名 | 所在位置 | 作用 | 是否修改 display_offset |
|--------|---------|------|------------------------|
| `Grid::scroll_display(scroll: Scroll)` | `grid/mod.rs:163` | 修改 Grid 的 `display_offset` | ✅ **是** |
| `Term::scroll_display(scroll: Scroll)` | `term/mod.rs:389` | 调用 Grid::scroll_display + 钳位 vi 光标 + 标记损坏 | ✅ **是**（间接） |
| **`ViModeCursor::scroll(term, lines: i32)`** | `vi_mode.rs:190` | **只计算光标新位置，不滚动视口！** 名称具有迷惑性 | ❌ **否** |
| `ActionContext::scroll(scroll: Scroll)` | `event.rs:707` | 完整上下文滚动：记录偏移 + 调用 scroll_display + 同步选区/搜索/重绘 | ✅ **是**（间接） |
| `scroll_terminal(x, y, mult)` | `input/mod.rs:760` | 鼠标滚轮决策逻辑，最终调用 `ctx.scroll` | ✅ **是**（间接） |
| `scroll_to_point(point)` | `term/mod.rs:884` | 视口外时滚动视口使点可见 | ✅ **可能是**（点在视口外时） |

**重点纠偏**：`ViModeCursor::scroll()` 方法命名虽然叫 "scroll"，但它 **完全不滚动视口**，只是辅助计算翻页时光标应该跳到的新位置。真正滚动视口由 `ctx.scroll()` 独立完成。

---

## 二、鼠标滚轮滚动完整调用链

### 2.1 调用链路总览

```
winit WindowEvent::MouseWheel
        ↓
Processor::mouse_wheel_input (input/mod.rs:725)
        ↓
Processor::scroll_terminal (input/mod.rs:760)
        ├─→ process_mouse_bindings → binding.action.execute → ctx.scroll
        ├─→ mouse_report (鼠标模式下)
        ├─→ write_to_pty (Alt Screen 模式下发送上下键序列)
        └─→ ctx.scroll(Scroll::Delta(lines)) [默认路径]
                ↓
ActionContext::scroll (event.rs:707)
                ↓
Term::scroll_display (term/mod.rs:389)
                ↓
Grid::scroll_display (grid/mod.rs:163)
                ↓
修改 display_offset
```

### 2.2 详细调用步骤

**步骤 1：winit 事件入口**

`event.rs:1980` 接收操作系统鼠标滚轮事件：

```rust
WindowEvent::MouseWheel { delta, phase, .. } => {
    self.ctx.window().set_mouse_visible(true);
    self.mouse_wheel_input(delta, phase);  // → input/mod.rs:725
}
```

**步骤 2：`mouse_wheel_input` 处理滚动类型**

`input/mod.rs:725` 根据滚动类型（行/像素）处理：

```rust
pub fn mouse_wheel_input(&mut self, delta: MouseScrollDelta, phase: TouchPhase) {
    let multiplier = self.ctx.config().scrolling.multiplier;  // 默认 3
    match delta {
        // 行滚动（普通鼠标滚轮）
        MouseScrollDelta::LineDelta(columns, lines) => {
            let new_scroll_px_x = columns * self.ctx.size_info().cell_width();
            let new_scroll_px_y = lines * self.ctx.size_info().cell_height();
            self.scroll_terminal(
                new_scroll_px_x as f64,
                new_scroll_px_y as f64,
                multiplier as f64,
            );
        },
        // 像素滚动（精确触控板）
        MouseScrollDelta::PixelDelta(mut lpos) => {
            match phase {
                TouchPhase::Started => {
                    self.ctx.mouse_mut().accumulated_scroll = Default::default();
                },
                TouchPhase::Moved => {
                    // 25度以内视为纯水平滚动
                    if lpos.x.abs() / lpos.x.hypot(lpos.y) > 0.9 {
                        lpos.y = 0.;
                    } else {
                        lpos.x = 0.;
                    }
                    self.scroll_terminal(lpos.x, lpos.y, multiplier as f64);
                },
                _ => (),
            }
        },
    }
}
```

关键点：
- `scrolling.multiplier` 配置项（默认 3）将滚轮事件放大
- 像素滚动有 `accumulated_scroll` 累积机制，累积到整行才触发滚动
- 接近水平的滚动（<25度）过滤掉 Y 分量，反之过滤 X 分量

**步骤 3：`scroll_terminal` 多分支决策**

`input/mod.rs:760` 根据终端模式决定滚动行为：

```rust
fn scroll_terminal(&mut self, new_scroll_x_px: f64, new_scroll_y_px: f64, multiplier: f64) {
    // 累积像素滚动
    self.ctx.mouse_mut().accumulated_scroll.x += new_scroll_x_px * multiplier;
    self.ctx.mouse_mut().accumulated_scroll.y += new_scroll_y_px * multiplier;

    // 计算整行数
    let lines = (self.ctx.mouse().accumulated_scroll.y / height).abs() as usize;
    let is_scroll_up = new_scroll_y_px > 0.;

    // 分支 1：匹配鼠标绑定（如 Shift+WheelUp 绑定特定动作）
    let event = if is_scroll_up { MouseEvent::WheelUp } else { MouseEvent::WheelDown };
    if lines != 0 && self.process_mouse_bindings(event) {
        for _ in 1..lines {
            self.process_mouse_bindings(event);
        }
    }
    // 分支 2：鼠标报告模式（终端应用请求鼠标事件）
    else if self.ctx.mouse_mode() {
        let code = if is_scroll_up { MOUSE_WHEEL_UP } else { MOUSE_WHEEL_DOWN };
        for _ in 0..lines {
            self.mouse_report(code, ElementState::Pressed);
        }
    }
    // 分支 3：Alt Screen + Alternate Scroll（如 vim/tmux 中滚轮转成上下键）
    else if self.ctx.terminal().mode()
        .contains(TermMode::ALT_SCREEN | TermMode::ALTERNATE_SCROLL)
        && !self.ctx.modifiers().state().shift_key()
    {
        let line_cmd = if is_scroll_up { b'A' } else { b'B' };
        let mut content = Vec::with_capacity(3 * lines);
        for _ in 0..lines {
            content.push(0x1b);  // ESC
            content.push(b'O');
            content.push(line_cmd);
        }
        self.ctx.write_to_pty(content);  // 发送 ESC OA / ESC OB（上下键）
    }
    // 分支 4：默认路径 — 直接滚动视口
    else if lines != 0 {
        let lines = if is_scroll_up { lines as i32 } else { -(lines as i32) };
        self.ctx.scroll(Scroll::Delta(lines));  // ← 核心调用
    }
}
```

**步骤 4：`ActionContext::scroll` 执行滚动**

`event.rs:707` 是真正执行滚动的地方，不仅修改 `display_offset`，还要同步相关状态：

```rust
fn scroll(&mut self, scroll: Scroll) {
    let old_offset = self.terminal.grid().display_offset() as i32;
    let old_vi_cursor = self.terminal.vi_mode_cursor;

    // 调用 Term 层滚动
    self.terminal.scroll_display(scroll);

    let lines_changed = old_offset - self.terminal.grid().display_offset() as i32;

    // 搜索时记录 display_offset 变化
    if self.search_active() {
        self.search_state.display_offset_delta += lines_changed;
    }

    // 同步选区（如果按住鼠标选择或 vi 模式选择）
    let vi_mode = self.terminal.mode().contains(TermMode::VI);
    if vi_mode && self.terminal.selection.as_ref().is_some_and(|s| !s.is_empty()) {
        self.update_selection(self.terminal.vi_mode_cursor.point, Side::Right);
    } else if self.mouse.left_button_state == ElementState::Pressed
        || self.mouse.right_button_state == ElementState::Pressed
    {
        let display_offset = self.terminal.grid().display_offset();
        let point = self.mouse.point(&self.size_info(), display_offset);
        self.update_selection(point, self.mouse.cell_side);
    }

    // Vi 模式下滚动视为"输入"，重置输入超时
    if vi_mode {
        self.on_typing_start();
    }

    // 标记需要重绘
    *self.dirty |= lines_changed != 0
        || (vi_mode && old_vi_cursor != self.terminal.vi_mode_cursor);
}
```

**步骤 5：`Term::scroll_display` 终端层处理**

`term/mod.rs:389` 负责 vi 光标钳位和损坏标记：

```rust
pub fn scroll_display(&mut self, scroll: Scroll) {
    let old_display_offset = self.grid.display_offset();
    self.grid.scroll_display(scroll);
    self.event_proxy.send_event(Event::MouseCursorDirty);

    // Vi 模式光标钳位到视口内
    let viewport_start = -(self.grid.display_offset() as i32);
    let viewport_end = viewport_start + self.bottommost_line().0;
    let vi_cursor_line = &mut self.vi_mode_cursor.point.line.0;
    *vi_cursor_line = cmp::min(viewport_end, cmp::max(viewport_start, *vi_cursor_line));
    self.vi_mode_recompute_selection();

    // 视口变化则标记全部损坏
    if old_display_offset != self.grid().display_offset() {
        self.mark_fully_damaged();
    }
}
```

### 2.3 鼠标绑定滚动路径

`Processor::process_mouse_bindings` (`input/mod.rs:1039`) 匹配配置的鼠标绑定：

```rust
fn process_mouse_bindings(&mut self, event: MouseEvent) -> bool {
    let mode = BindingMode::new(self.ctx.terminal().mode(), self.ctx.search_active());
    let mods = self.ctx.modifiers().state();

    for binding in &mouse_bindings {
        if binding.is_triggered_by(mode, mods, &event) {
            binding.action.execute(&mut self.ctx);  // → Action::execute
            return true;
        }
    }
    false
}
```

如果绑定了 `ScrollPageUp` / `ScrollLineUp` 等动作，会在 `Action::execute` 中调用 `ctx.scroll()`。

---

## 三、键盘滚动完整调用链

### 3.1 调用链路总览

```
winit WindowEvent::KeyboardInput
        ↓
Processor::key_input (input/keyboard.rs:22)
        ↓
Processor::process_key_bindings (input/keyboard.rs:178)
        ↓
Action::execute (input/mod.rs:168)
    ├─ 翻页滚动（ScrollPage*）：先调用 ViModeCursor::scroll() 【只算光标！】再 ctx.scroll
    ├─ 单行滚动（ScrollLine*）：直接 ctx.scroll
    ├─ ScrollToTop/Bottom：先 ctx.scroll，再 vi_motion(FirstOccupied) 【会 scroll_to_point】
    └─ 其他动作（CenterAroundViCursor 等）
            ↓
ActionContext::scroll (event.rs:707)
            ↓
Term::scroll_display (term/mod.rs:389)
            ↓
Grid::scroll_display (grid/mod.rs:163)
            ↓
修改 display_offset
```

### 3.2 详细调用步骤

**步骤 1：键盘事件入口**

`event.rs:1968` 接收按键事件：

```rust
WindowEvent::KeyboardInput { event, is_synthetic: false, .. } => {
    self.key_input(event);  // → input/keyboard.rs:22
}
```

**步骤 2：`key_input` 按键处理**

`input/keyboard.rs:22` 是按键处理入口：

```rust
pub fn key_input(&mut self, key: KeyEvent) {
    // IME 输入跳过
    if self.ctx.display().ime.preedit().is_some() { return; }

    let mode = *self.ctx.terminal().mode();
    let mods = self.ctx.modifiers().state();

    // 释放键处理
    if key.state == ElementState::Released {
        self.key_release(key, mode, mods);
        return;
    }

    // 尝试匹配键绑定
    if self.process_key_bindings(&key) {
        return;  // 绑定匹配成功，跳过普通输入
    }

    // ... 普通字符输入 → 写入 PTY
}
```

**步骤 3：`process_key_bindings` 匹配绑定**

`input/keyboard.rs:178` 匹配配置的键绑定，匹配成功后执行 `action.execute(ctx)`。

### 3.3 Action::execute 中滚动动作的详细分解（重点纠偏）

**动作 1：翻页滚动（ScrollPageUp / ScrollPageDown / ScrollHalfPageUp / ScrollHalfPageDown）**

`input/mod.rs:353-380` 是翻页滚动的完整实现。**请注意两步的分工：**

```rust
Action::ScrollPageUp
| Action::ScrollPageDown
| Action::ScrollHalfPageUp
| Action::ScrollHalfPageDown => {
    // ──── 第一步：计算 vi 光标新位置 ────
    // ViModeCursor::scroll() 只是【计算光标目标位置】，不滚动视口！
    let term = ctx.terminal_mut();
    let (scroll, amount) = match self {
        Action::ScrollPageUp => (Scroll::PageUp, term.screen_lines() as i32),
        Action::ScrollPageDown => (Scroll::PageDown, -(term.screen_lines() as i32)),
        Action::ScrollHalfPageUp => {
            let amount = term.screen_lines() as i32 / 2;
            (Scroll::Delta(amount), amount)
        },
        Action::ScrollHalfPageDown => {
            let amount = -(term.screen_lines() as i32 / 2);
            (Scroll::Delta(amount), amount)
        },
        _ => unreachable!(),
    };

    let old_vi_cursor = term.vi_mode_cursor;
    // ❗ 纠偏：ViModeCursor::scroll() 不滚动视口
    // 它只把光标位置沿 amount 方向偏移（相当于"光标跟页一起滚"的视觉效果）
    term.vi_mode_cursor = term.vi_mode_cursor.scroll(term, amount);
    if old_vi_cursor != term.vi_mode_cursor {
        ctx.mark_dirty();
    }

    // ──── 第二步：真正滚动视口 ────
    ctx.scroll(scroll);  // ✅ 这里才修改 display_offset
},
```

**ViModeCursor::scroll() 的内部实现**（`vi_mode.rs:190`）：

```rust
// ❗ 名称有迷惑性：这个方法不做任何"滚动视口"的操作！
pub fn scroll<T: EventListener>(mut self, term: &Term<T>, lines: i32) -> Self {
    // 把光标位置减去 lines（向上滚 lines 行，光标跟着向上移动 lines 行）
    let line = (self.point.line - lines).grid_clamp(term, Boundary::Grid);

    // 找到目标行的第一个非空单元格
    let column = first_occupied_in_line(term, line).unwrap_or_default().column;

    // 只设置光标位置
    self.point = Point::new(line, column);
    self
}
```

**动作 2：单行滚动（ScrollLineUp / ScrollLineDown）**

`input/mod.rs:381-382`，最简单，直接滚动视口：

```rust
Action::ScrollLineUp => ctx.scroll(Scroll::Delta(1)),
Action::ScrollLineDown => ctx.scroll(Scroll::Delta(-1)),
```

**动作 3：滚动到顶部（ScrollToTop）**

`input/mod.rs:383-391`：

```rust
Action::ScrollToTop => {
    // 第一步：先滚到底（最顶部历史）
    ctx.scroll(Scroll::Top);                 // ✅ 修改 display_offset = history_size()

    // 第二步：把 vi 光标移到最顶行，再找第一个非空字符
    // vi_motion 末尾会调用 scroll_to_point，但视口已在顶部，通常不会再滚动
    // （如果非 Vi 模式，Term::vi_motion 直接 return）
    let topmost_line = ctx.terminal().topmost_line();
    ctx.terminal_mut().vi_mode_cursor.point.line = topmost_line;
    ctx.terminal_mut().vi_motion(ViMotion::FirstOccupied);
    ctx.mark_dirty();
},
```

**动作 4：滚动到底部（ScrollToBottom）**

`input/mod.rs:392-403`：

```rust
Action::ScrollToBottom => {
    ctx.scroll(Scroll::Bottom);              // ✅ display_offset = 0

    let term = ctx.terminal_mut();
    term.vi_mode_cursor.point.line = term.bottommost_line();

    // 调用两次 FirstOccupied，处理跨行换行的情况
    term.vi_motion(ViMotion::FirstOccupied);  // ← 会调用 scroll_to_point
    term.vi_motion(ViMotion::FirstOccupied);
    ctx.mark_dirty();
},
```

**动作 5：以 vi 光标为中心滚动（CenterAroundViCursor）**

`input/mod.rs`（ViAction 分支）：

```rust
Action::Vi(ViAction::CenterAroundViCursor) => {
    let term = ctx.terminal();
    let display_offset = term.grid().display_offset() as i32;
    // 目标：让 vi 光标位于屏幕中间
    let target = -display_offset + term.screen_lines() as i32 / 2 - 1;
    let line = term.vi_mode_cursor.point.line;
    let scroll_lines = target - line.0;  // 计算需要滚动的行数
    ctx.scroll(Scroll::Delta(scroll_lines));  // ✅ 修改 display_offset
},
```

---

## 四、Vi 模式滚动：自动滚动机制与各动作辨析

### 4.1 `Term::vi_motion` —— 顶层入口

`term/mod.rs:839`：

```rust
pub fn vi_motion(&mut self, motion: ViMotion)
where
    T: EventListener,
{
    if !self.mode.contains(TermMode::VI) {
        return;  // 非 Vi 模式直接退出
    }

    // 移动光标（motion 方法内部会在末尾检查是否需要滚动）
    self.vi_mode_cursor = self.vi_mode_cursor.motion(self, motion);
    self.vi_mode_recompute_selection();
}
```

### 4.2 `ViModeCursor::motion()` —— 自动滚动的真正触发点（关键纠偏）

`vi_mode.rs:74-186` 是所有 Vi 移动动作的实现。**关键机制：先移动光标，再在方法末尾统一调用一次 `scroll_to_point` 检查是否需要滚动视口。**

```rust
pub fn motion<T: EventListener>(mut self, term: &mut Term<T>, motion: ViMotion) -> Self {
    // ──── 第一步：根据动作类型移动光标（只改 point，不改 display_offset）────
    match motion {
        ViMotion::Up => {
            if self.point.line > term.topmost_line() {
                self.point.line -= 1;  // 只改光标位置
            }
        },
        ViMotion::Down => {
            if self.point.line + 1 < term.screen_lines() as i32 {
                self.point.line += 1;  // 只改光标位置
            }
        },
        ViMotion::Left => { /* 复杂的跨行换行逻辑，只改 point */ },
        ViMotion::Right => { /* ... */ },
        ViMotion::High => {
            // 跳转到【当前视口】顶部行（基于当前 display_offset，不改变视口）
            let line = Line(-(term.grid().display_offset() as i32));
            let col = first_occupied_in_line(term, line).unwrap_or_default().column;
            self.point = Point::new(line, col);
            // 因为是视口内跳转，下面 scroll_to_point 不会触发滚动
        },
        ViMotion::Middle => {
            // 跳转到【当前视口】中间行
            let display_offset = term.grid().display_offset() as i32;
            let line = Line(-display_offset + term.screen_lines() as i32 / 2 - 1);
            // ... 设置 self.point
        },
        ViMotion::Low => {
            // 跳转到【当前视口】底部行
            let display_offset = term.grid().display_offset() as i32;
            let line = Line(-display_offset + term.screen_lines() as i32 - 1);
            // ...
        },
        ViMotion::FirstOccupied => { /* 找第一个非空字符 */ },
        ViMotion::SemanticLeft => { /* 语义词跳转，只改 point */ },
        ViMotion::WordRight => { /* 词跳转，只改 point */ },
        ViMotion::Bracket => { self.point = term.bracket_search(self.point).unwrap_or(self.point); },
        ViMotion::ParagraphUp => { /* 段落跳转，只改 point */ },
        ViMotion::ParagraphDown => { /* ... */ },
        // ... 其他动作都是只改 point
    }

    // ──── 第二步：统一检查光标是否在视口外，需要则滚动 ────
    // ✅ 所有 Vi 动作都会走到这一行
    term.scroll_to_point(self.point);

    self
}
```

**结论**：
- 所有 `vi_motion` 动作都 **可能** 触发视口滚动（如果光标最终位置在视口外）
- `High/Middle/Low` 是相对于当前视口定位，因此这些动作 **通常不会** 触发滚动（除非视口在这期间被别的事件改变了）

### 4.3 `scroll_to_point` —— 视口外才滚动

`term/mod.rs:884`：

```rust
pub fn scroll_to_point(&mut self, point: Point)
where
    T: EventListener,
{
    let display_offset = self.grid.display_offset() as i32;
    let screen_lines = self.grid.screen_lines() as i32;

    // 视口范围：[-display_offset, screen_lines - display_offset)
    // 点在视口上方 → 向上滚（减小 display_offset）
    if point.line < -display_offset {
        let lines = point.line + display_offset;
        self.scroll_display(Scroll::Delta(-lines.0));  // ✅ 改 display_offset
    }
    // 点在视口下方 → 向下滚（增大 display_offset）
    else if point.line >= (screen_lines - display_offset) {
        let lines = point.line + display_offset - screen_lines + 1i32;
        self.scroll_display(Scroll::Delta(-lines.0));  // ✅ 改 display_offset
    }
    // 点在视口内：什么都不做
}
```

### 4.4 `vi_goto_point` —— 先滚再设光标

`term/mod.rs:855`：

```rust
pub fn vi_goto_point(&mut self, point: Point)
where
    T: EventListener,
{
    // 先滚动视口让点可见
    self.scroll_to_point(point);               // ✅ 可能改 display_offset

    // 再设置光标
    self.vi_mode_cursor.point = point;

    self.vi_mode_recompute_selection();
}
```

这个方法用于搜索跳转、提示跳转等场景，先保证点可见，再把光标放上去。

### 4.5 Vi 模式各动作滚动行为对照表

| Vi 动作 | 是否可能滚动视口 | 原因 |
|---------|-----------------|------|
| `Up/Down` | ✅ 是 | 光标移出视口时触发 `scroll_to_point` |
| `Left/Right` | ✅ 是 | 跨行换行时光标可能移出视口 |
| `First/Last` | ✅ 是 | 跨 wrap 行搜索可能移出视口 |
| `FirstOccupied` | ✅ 是 | 同上，跨 wrap 搜索可能移出 |
| **`High/Middle/Low`** | ⚠️ 通常不 | 基于当前视口定位，结果点在视口内；极端情况才滚 |
| `SemanticLeft/Right*` | ✅ 是 | 语义词搜索可能跨出视口 |
| `WordLeft/Right*` | ✅ 是 | 词搜索可能跨出视口 |
| `Bracket` | ✅ 是 | 括号配对可能在视口外 |
| `ParagraphUp/Down` | ✅ 是 | 段落搜索可能跨出视口 |
| **翻页滚动**（PageUp/Down 等）| ✅ **必定触发** | 独立调用 `ctx.scroll()`，不是通过 `vi_motion` |
| **ScrollToTop/Bottom** | ✅ **必定触发** | 独立调用 `ctx.scroll(Scroll::Top/Bottom)` |
| `CenterAroundViCursor` | ✅ **必定触发** | 独立计算偏移后调用 `ctx.scroll` |

---

## 五、搜索跳转完整流程的滚动分析（重点纠偏）

### 5.1 搜索全流程概览

```
start_search()           —— 保存 origin，不滚动视口
       ↓
search_input(c)         —— 用户输入搜索词
       ↓
update_search()         —— 每输入一个字符调用一次
       ↓
goto_match(limit=1000) —— 限制范围内查找，找到就跳
  ├─ Vi 模式：vi_goto_point()       ← 先滚动再设光标 ✅
  └─ 非 Vi 模式：scroll_to_point()  ← 只滚动视口 ✅
       ↓
advance_search_origin() —— 用户按 Enter/N 键跳转下一条
  ├─ scroll_to_point(new_origin)    ← 先对齐到当前匹配 ✅
  └─ goto_match(None)               ← 无限制查找下一条 ✅
       ↓
cancel_search() / confirm_search() —— 退出搜索
  ├─ Vi 模式：search_reset_state()  ← 恢复视口和光标 ✅
  └─ 非 Vi 模式：创建选区（取消时）/ 直接退出（确认时），不恢复视口 ❌
```

### 5.2 `start_search()` —— **不滚动视口**

`event.rs:946`：

```rust
fn start_search(&mut self, direction: Direction) {
    // ... 初始化历史、方向 ...

    // 只保存 origin 点，不做任何滚动
    if self.terminal.mode().contains(TermMode::VI) {
        self.search_state.origin = self.terminal.vi_mode_cursor.point;  // Vi 模式：从 vi 光标开始
        self.search_state.display_offset_delta = 0;
        // ... 光标在底部时 origin 上移一行，为新内容腾出空间（不滚动，只是记录点）
    } else {
        // 非 Vi 模式：origin 从视口边界开始
        let viewport_top = Line(-(self.terminal.grid().display_offset() as i32)) - 1;
        // ...
    }
    // ❌ 注意：start_search 本身不调用任何 scroll_display / scroll / scroll_to_point
}
```

### 5.3 `update_search()` → `goto_match(limit)` —— **找到匹配就滚动**

`event.rs:1503-1527`：

```rust
fn update_search(&mut self) {
    let regex = match self.search_state.regex() { /* ... */ };

    if regex.is_empty() {
        // 空搜索词：恢复初始状态（Vi 模式下会恢复视口）
        self.search_reset_state();  // ← 仅 Vi 模式恢复视口
        self.search_state.dfas = None;
    } else {
        self.search_state.dfas = RegexSearch::new(regex).ok();
        // ✅ 每输入一个字符都尝试查找匹配，找到就跳转
        self.goto_match(MAX_SEARCH_WHILE_TYPING);  // limit=1000 行
    }

    *self.dirty = true;
}
```

### 5.4 `goto_match()` —— 搜索跳转的滚动实现

`event.rs:1554-1605`，核心跳转逻辑：

```rust
fn goto_match(&mut self, mut limit: Option<usize>) {
    let dfas = match &mut self.search_state.dfas { /* ... */ };

    match self.terminal.search_next(dfas, clamped_origin, direction, Side::Left, limit) {
        Some(regex_match) => {
            let old_offset = self.terminal.grid().display_offset() as i32;

            if self.terminal.mode().contains(TermMode::VI) {
                // ✅ Vi 模式：先滚动视口让匹配可见，再设光标
                self.terminal.vi_goto_point(*regex_match.start());
            } else {
                // ✅ 非 Vi 模式：只滚动视口让匹配可见（没有 vi 光标）
                self.terminal.scroll_to_point(*regex_match.start());
            }

            // 记录偏移变化（用于 search_reset_state 恢复视口）
            let display_offset = self.terminal.grid().display_offset();
            self.search_state.display_offset_delta += old_offset - display_offset as i32;

            self.search_state.focused_match = Some(regex_match);
        },
        None if limit.is_none() => self.search_reset_state(),  // 无限搜索没找到：恢复
        None => { /* 有限搜索没找到，延迟继续 */ },
    }
    *self.dirty = true;
}
```

### 5.5 `advance_search_origin()` —— **跳转前后都涉及滚动**

`event.rs:1132-1149`，用户按 "下一条 / 上一条" 时触发：

```rust
fn advance_search_origin(&mut self, direction: Direction) {
    if let Some(focused_match) = &self.search_state.focused_match {
        let new_origin = match direction {
            Direction::Right => focused_match.end().add(self.terminal, Boundary::None, 1),
            Direction::Left => focused_match.start().sub(self.terminal, Boundary::None, 1),
        };

        // ✅ 先对齐 origin 到当前匹配（如果匹配不在视口内，会先滚回来）
        self.terminal.scroll_to_point(new_origin);

        // 重置 offset 计数器，从当前位置开始记新的变化
        self.search_state.display_offset_delta = 0;
        self.search_state.origin = new_origin;
    }

    // ✅ 再跳到下一条（会再次滚动）
    let search_direction = mem::replace(&mut self.search_state.direction, direction);
    self.goto_match(None);  // 无限制查找
    self.search_state.direction = search_direction;
}
```

### 5.6 `cancel_search()` —— **仅 Vi 模式恢复视口（重点纠偏）**

`event.rs:1047-1063`：

```rust
fn cancel_search(&mut self) {
    if self.terminal.mode().contains(TermMode::VI) {
        // ✅ Vi 模式：恢复搜索前的光标位置和视口
        self.search_reset_state();
    } else if let Some(focused_match) = &self.search_state.focused_match {
        // ❌ 非 Vi 模式：不恢复视口！而是创建选区，停留在匹配位置
        let start = *focused_match.start();
        let end = *focused_match.end();
        self.start_selection(SelectionType::Simple, start, Side::Left);
        self.update_selection(end, Side::Right);
        self.copy_selection(ClipboardType::Selection);
    }

    self.search_state.dfas = None;
    self.exit_search();
}
```

**`search_reset_state()` 的具体实现**（`event.rs:1530-1551`）：

```rust
fn search_reset_state(&mut self) {
    // 取消延迟搜索
    // ...

    self.search_state.focused_match = None;

    // ⚠️ 只在 Vi 模式下恢复视口！非 Vi 模式直接 return
    if !self.terminal.mode().contains(TermMode::VI) {
        return;
    }

    // 恢复光标到搜索前 origin
    self.terminal.vi_mode_cursor.point = self.search_state.origin;
    // ✅ 恢复视口到搜索前位置（display_offset_delta 存了累计偏移）
    self.terminal.scroll_display(Scroll::Delta(self.search_state.display_offset_delta));
    self.search_state.display_offset_delta = 0;

    *self.dirty = true;
}
```

### 5.7 `confirm_search()` —— **非 Vi 模式=取消，Vi 模式=保持**

`event.rs:1030-1044`：

```rust
fn confirm_search(&mut self) {
    // 非 Vi 模式：直接取消（会创建匹配的选区，不恢复视口）
    if !self.terminal.mode().contains(TermMode::VI) {
        self.cancel_search();
        return;
    }

    // Vi 模式：取消之前被中断的延迟搜索，退出搜索（保持当前视口/光标位置）
    if self.scheduler.scheduled(timer_id) {
        self.goto_match(None);
    }
    self.exit_search();
}
```

### 5.8 `start_seeded_search()` —— 带初始文本的搜索

`event.rs:984-1027`：

```rust
fn start_seeded_search(&mut self, direction: Direction, text: String) {
    // ... 先 start_search，再逐字符输入搜索词（触发多次 update_search → goto_match）
    // ... confirm_search 退出搜索

    if !self.terminal.mode().contains(TermMode::VI) {
        return;  // 非 Vi 模式到此结束
    }

    // ✅ Vi 模式：再做一次更精准的跳转，找到 origin 之后的下一个目标方向匹配
    let target = self.search_next(origin, Direction::Right, Side::Right).and_then(|rm| {
        // ... 根据目标方向找到更精确的匹配位置
    });

    if let Some(target) = target {
        // ✅ 最终用 vi_goto_point（滚+设光标）把光标定到目标上
        self.terminal_mut().vi_goto_point(target);
        self.mark_dirty();
    }
}
```

### 5.9 搜索流程滚动对照表

| 搜索步骤 | 是否触发视口滚动 | 说明 |
|---------|-----------------|------|
| `start_search()` | ❌ **否** | 只记录 origin，不做任何滚动 |
| `search_input(c)` → `update_search()` | ✅ 有匹配时 | `goto_match(1000)` 找到匹配就跳 |
| `goto_match(limit)` — Vi 模式 | ✅ **是** | 调用 `vi_goto_point` → `scroll_to_point` + 设光标 |
| `goto_match(limit)` — 非 Vi 模式 | ✅ 匹配在视口外时 | 只调用 `scroll_to_point`，不设光标 |
| `advance_search_origin()`（下一条） | ✅ **是** | 先 `scroll_to_point(new_origin)` 再 `goto_match(None)` |
| 搜索词清空 | ⚠️ 仅 Vi 模式 | 调用 `search_reset_state` 恢复视口 |
| `cancel_search()` — Vi 模式 | ✅ **是** | `search_reset_state` 恢复视口和光标 |
| `cancel_search()` — 非 Vi 模式 | ❌ **否** | 不恢复视口，而是创建选区，停留在匹配处 |
| `confirm_search()` — Vi 模式 | ❌ 通常不 | 只是退出搜索，保持当前状态（无结果时可能滚动） |
| `confirm_search()` — 非 Vi 模式 | ❌ **否** | 等价于 cancel_search → 创建选区 |
| `start_seeded_search()` Vi 模式 | ✅ 可能多次 | update_search 多次跳转 + 最终 vi_goto_point |

### 5.10 Vi 搜索快捷动作与行内搜索滚动入口（新增对照）

Alacritty 提供了三类搜索相关的 Action 枚举，分布在三个层级：

1. **顶层 `Action`**：`SearchForward` / `SearchBackward`（/ ? 键）
2. **`ViAction`**：包含搜索跳转、行内搜索、语义搜索等 12 个动作
3. **`SearchAction`**：搜索模式内的操作，聚焦下一条/上一条、确认取消等

以下是所有搜索相关动作的滚动行为对照：

#### 5.10.1 顶层搜索启动动作

| 动作 | 默认绑定 | 是否滚动视口 | 代码行为 |
|------|---------|-------------|---------|
| `Action::SearchForward` | `/`（Vi 模式） | ⚠️ 启动时不滚动；输入时可能滚动 | 调用 `ctx.start_search(Direction::Right)` → 只设置 origin，不滚动。后续输入字符触发 `update_search()` → `goto_match()` 才可能滚动。 |
| `Action::SearchBackward` | `?`（Vi 模式） | ⚠️ 同上 | 调用 `ctx.start_search(Direction::Left)` → 同上。 |

**`start_search()` 实现**（`event.rs:946`）：

```rust
fn start_search(&mut self, direction: Direction) {
    // ... 初始化历史、方向 ...
    // ✅ 只保存 origin 点，不做任何滚动
    if self.terminal.mode().contains(TermMode::VI) {
        self.search_state.origin = self.terminal.vi_mode_cursor.point;
        self.search_state.display_offset_delta = 0;
    } else {
        let viewport_top = Line(-(self.terminal.grid().display_offset() as i32)) - 1;
        self.search_state.origin = match direction {
            Direction::Right => Point::new(viewport_top, Column(0)),
            Direction::Left => Point::new(viewport_bottom, last_column),
        };
    }
    // ❌ 无任何 scroll 调用
}
```

#### 5.10.2 ViAction 搜索跳转动作（搜索后跳转）

这些动作是在已有搜索结果的情况下，在匹配之间跳转：

| 动作 | 默认绑定 | 是否滚动视口 | 代码行为 |
|------|---------|-------------|---------|
| `ViAction::SearchNext` | `n`（Vi 模式） | ✅ **是** | 调用 `search_next(origin, direction, Side::Left)` → 找到匹配后 `vi_goto_point(*regex_match.start())` → `scroll_to_point`。 |
| `ViAction::SearchPrevious` | `N`（Vi 模式） | ✅ **是** | 同上，方向相反。 |
| `ViAction::SearchStart` | （无默认） | ✅ **是** | 从当前位置向左找，调用 `vi_goto_point(*regex_match.start())`。 |
| `ViAction::SearchEnd` | （无默认） | ✅ **是** | 从当前位置向右找，调用 `vi_goto_point(*regex_match.end())`。 |

**`SearchNext` 实现**（`input/mod.rs:212`）：

```rust
Action::Vi(ViAction::SearchNext) => {
    ctx.on_typing_start();
    let terminal = ctx.terminal();
    let direction = ctx.search_direction();
    let vi_point = terminal.vi_mode_cursor.point;
    let origin = match direction { /* 从光标后一格开始 */ };

    if let Some(regex_match) = ctx.search_next(origin, direction, Side::Left) {
        // ✅ vi_goto_point 先滚动再设光标
        ctx.terminal_mut().vi_goto_point(*regex_match.start());
        ctx.mark_dirty();
    }
},
```

#### 5.10.3 ViAction 行内搜索动作（f/F/t/T 系列）

这些是 Vi 模式的"行内字符搜索"，在当前行（支持跨 wrap）搜索单个字符：

| 动作 | 默认绑定 | 是否滚动视口 | 代码行为 |
|------|---------|-------------|---------|
| `ViAction::InlineSearchForward` | `f`（Vi 模式） | ⚠️ 两步：①启动时不滚动；②输入字符后可能滚动 | 第 1 步：调用 `start_inline_search(Direction::Right, false)` → 只设置 `InlineSearchState`，等待输入字符。第 2 步：字符输入触发 `inline_search_input` → `inline_search_next` → `inline_search` → 找到匹配后 `vi_goto_point(point)` → `scroll_to_point`。 |
| `ViAction::InlineSearchBackward` | `F`（Vi 模式） | ⚠️ 同上 | `start_inline_search(Direction::Left, false)` |
| `ViAction::InlineSearchForwardShort` | `t`（Vi 模式） | ⚠️ 同上 | `start_inline_search(Direction::Right, true)` → 找到匹配后 `stop_short`=true，光标停在匹配前一格。 |
| `ViAction::InlineSearchBackwardShort` | `T`（Vi 模式） | ⚠️ 同上 | `start_inline_search(Direction::Left, true)` |
| `ViAction::InlineSearchNext` | `;`（Vi 模式） | ✅ **是** | `inline_search_next()` → 调用 `inline_search(direction)` → `vi_goto_point`。 |
| `ViAction::InlineSearchPrevious` | `,`（Vi 模式） | ✅ **是** | `inline_search_previous()` → 反向搜索。 |

**启动时仅设置状态（不滚动）**（`event.rs:1445`）：

```rust
fn start_inline_search(&mut self, direction: Direction, stop_short: bool) {
    // ✅ 只设置 InlineSearchState，char_pending = true 等待字符
    self.inline_search_state.stop_short = stop_short;
    self.inline_search_state.direction = direction;
    self.inline_search_state.char_pending = true;
    self.inline_search_state.character = None;
    // ❌ 无滚动
}
```

**字符输入后跳转（可能滚动）**（`event.rs:1465`）：

```rust
fn inline_search_input(&mut self, text: &str) {
    let c = match text.chars().next() { Some(c) => c, None => return };
    self.inline_search_state.char_pending = false;
    self.inline_search_state.character = Some(c);
    // ✅ 立即跳转，内部调用 vi_goto_point
    self.inline_search_next();
}
```

**`inline_search` 核心实现**（`event.rs:1674`）：

```rust
fn inline_search(&mut self, direction: Direction) {
    let c = match self.inline_search_state.character { Some(c) => c, None => return };

    // 在当前行（支持 wrap 跨行）搜索字符
    let vi_point = self.terminal.vi_mode_cursor.point;
    let point = match direction {
        Direction::Right => self.terminal.inline_search_right(vi_point, search_character),
        Direction::Left => self.terminal.inline_search_left(vi_point, search_character),
    };

    if let Ok(mut point) = point {
        // t/T 模式：停在匹配前一格
        if self.inline_search_state.stop_short {
            point = match direction { /* prev/next 单元格 */ };
        }

        // ✅ vi_goto_point → scroll_to_point → 可能滚动视口
        self.terminal.vi_goto_point(point);
        self.mark_dirty();
    }
}
```

**关键点**：行内搜索虽然叫"行内"，但支持跨 wrap 行。如果用户正在查看历史，目标行不在视口内，`vi_goto_point` → `scroll_to_point` 会把视口滚回目标行。

#### 5.10.4 ViAction 语义搜索动作（* / #）

搜索光标下的语义化词（或选中的文本）：

| 动作 | 默认绑定 | 是否滚动视口 | 代码行为 |
|------|---------|-------------|---------|
| `ViAction::SemanticSearchForward` | `*`（Vi 模式） | ✅ **是** | 调用 `semantic_word` 获取光标下的词 → `ctx.start_seeded_search(Direction::Right, seed_text)` → 逐字符输入搜索词（触发多次 `update_search` → `goto_match`）→ 退出搜索 → Vi 模式下额外 `vi_goto_point` 精确定位。 |
| `ViAction::SemanticSearchBackward` | `#`（Vi 模式） | ✅ **是** | 同上，方向 `Direction::Left`。 |

**实现**（`input/mod.rs:285`）：

```rust
Action::Vi(ViAction::SemanticSearchForward | ViAction::SemanticSearchBackward) => {
    // 获取搜索种子：选中词优先，否则光标下的语义词
    let seed_text = match ctx.terminal().selection_to_string() {
        Some(selection) if !selection.is_empty() => selection,
        _ => ctx.semantic_word(ctx.terminal().vi_mode_cursor.point),
    };

    if !seed_text.is_empty() {
        let direction = match self { /* Right / Left */ };
        // ✅ start_seeded_search 会触发多次滚动
        ctx.start_seeded_search(direction, seed_text);
    }
},
```

**`start_seeded_search` 完整流程**（`event.rs:984`）：

```rust
fn start_seeded_search(&mut self, direction: Direction, text: String) {
    // 1. 先 start_search（不滚动）
    self.start_search(direction);

    // 2. 逐字符输入搜索词 → 每次触发 update_search → goto_match → 可能滚动
    for c in text.chars() {
        if let '$' | '('..='+' | '?' | '['..='^' | '{'..='}' = c {
            self.search_input('\\');  // 转义正则特殊字符
        }
        self.search_input(c);  // ← 每次输入都触发 update_search + goto_match
    }

    // 3. confirm_search 退出搜索
    self.confirm_search();

    // 4. Vi 模式：额外做一次更精准的跳转
    if !self.terminal.mode().contains(TermMode::VI) { return; }
    let target = self.search_next(origin, ...).and_then(|rm| { /* 更精确定位 */ });
    if let Some(target) = target {
        // ✅ 再次调用 vi_goto_point
        self.terminal_mut().vi_goto_point(target);
        self.mark_dirty();
    }
}
```

#### 5.10.5 SearchAction 搜索模式内操作

这些是在搜索输入框激活时（`BindingMode::SEARCH`）的操作：

| 动作 | 默认绑定 | 是否滚动视口 | 代码行为 |
|------|---------|-------------|---------|
| `SearchAction::SearchFocusNext` | `Enter`（搜索模式，非 Vi） | ✅ **是** | `ctx.advance_search_origin(ctx.search_direction())` → 先 `scroll_to_point(new_origin)` 对齐，再 `goto_match(None)` 无限制查找。 |
| `SearchAction::SearchFocusPrevious` | `Shift-Enter`（搜索模式，非 Vi） | ✅ **是** | 同上，方向相反。 |
| `SearchAction::SearchConfirm` | `Enter`（搜索模式，Vi） | ⚠️ 仅无结果时可能 | Vi 模式：取消延迟搜索 → `goto_match(None)` 无结果则 `search_reset_state`（恢复视口）；非 Vi 模式：直接 `cancel_search()` → 创建选区，不滚动。 |
| `SearchAction::SearchCancel` | `Esc`（搜索模式） | ⚠️ 仅 Vi 模式 | Vi 模式：`search_reset_state` 恢复视口和光标；非 Vi 模式：创建选区，停留在匹配处，不滚动。 |
| `SearchAction::SearchClear` | （无默认） | ⚠️ 取决于后续 | 调用 `cancel_search()` → `start_search(direction)`。是否滚动取决于取消时的操作。 |
| `SearchAction::SearchDeleteWord` | `Ctrl-W`（搜索模式） | ⚠️ 取决于匹配 | 只删除搜索词的最后一个词 → 调用 `update_search()` → `goto_match()`，找到匹配则滚动。 |
| `SearchAction::SearchHistoryPrevious` | `Up`（搜索模式） | ⚠️ 取决于匹配 | 切换到上一个历史搜索词 → `update_search()` → `goto_match()`。 |
| `SearchAction::SearchHistoryNext` | `Down`（搜索模式） | ⚠️ 取决于匹配 | 切换到下一个历史搜索词 → 同上。 |

**`advance_search_origin` 实现**（`event.rs:1132`）：

```rust
fn advance_search_origin(&mut self, direction: Direction) {
    if let Some(focused_match) = &self.search_state.focused_match {
        let new_origin = match direction { /* 从当前匹配后开始 */ };
        // ✅ 先对齐到当前匹配（如果不在视口内，先滚回来）
        self.terminal.scroll_to_point(new_origin);
        self.search_state.display_offset_delta = 0;
        self.search_state.origin = new_origin;
    }
    // ✅ 再跳到下一条（无限制查找，可能滚动很远）
    let search_direction = mem::replace(&mut self.search_state.direction, direction);
    self.goto_match(None);
    self.search_state.direction = search_direction;
}
```

#### 5.10.6 行内搜索与普通搜索的滚动入口对比

| 维度 | 行内搜索（f/F/t/T） | 普通搜索（/ ?） | 语义搜索（* #） |
|------|---------------------|-----------------|----------------|
| 目标 | 单个字符 | 任意正则 | 光标下的语义词 |
| 范围 | 当前行（支持跨 wrap） | 整个缓冲区（含 scrollback） | 整个缓冲区（含 scrollback） |
| 启动动作 | `start_inline_search` → 只设状态 | `start_search` → 只设 origin | `start_seeded_search` → 触发多次跳转 |
| 字符输入 | 仅等待 1 个字符，输入后立即跳转 | 等待任意长度字符串，每输入一个字符跳转一次 | 自动填入种子词，无需用户输入 |
| 跳转方法 | `inline_search_right/left` → `vi_goto_point` | `goto_match(limit)` → `vi_goto_point` / `scroll_to_point` | 多次 `goto_match` + 最终 `vi_goto_point` |
| 退出方式 | 输入字符后自动结束（或按 Esc） | Enter 确认 / Esc 取消 | 自动填入后自动 confirm 退出 |

#### 5.10.7 所有搜索相关动作滚动行为汇总表

| 层级 | 动作 | 是否触发视口滚动 | 滚动触发点 |
|------|------|-----------------|-----------|
| **顶层 Action** | `SearchForward` | ⚠️ 启动不滚；输入时可能滚 | `update_search` → `goto_match` |
| | `SearchBackward` | ⚠️ 同上 | 同上 |
| **ViAction** | `SearchNext` | ✅ **是** | `vi_goto_point` → `scroll_to_point` |
| | `SearchPrevious` | ✅ **是** | 同上 |
| | `SearchStart` | ✅ **是** | 同上 |
| | `SearchEnd` | ✅ **是** | 同上 |
| | `InlineSearchForward` | ⚠️ 启动不滚；输入后可能滚 | `inline_search` → `vi_goto_point` |
| | `InlineSearchBackward` | ⚠️ 同上 | 同上 |
| | `InlineSearchForwardShort` | ⚠️ 同上 | 同上 |
| | `InlineSearchBackwardShort` | ⚠️ 同上 | 同上 |
| | `InlineSearchNext` | ✅ **是** | 同上 |
| | `InlineSearchPrevious` | ✅ **是** | 同上 |
| | `SemanticSearchForward` | ✅ **是** | `start_seeded_search` → 多次 `goto_match` |
| | `SemanticSearchBackward` | ✅ **是** | 同上 |
| **SearchAction** | `SearchFocusNext` | ✅ **是** | `scroll_to_point` + `goto_match(None)` |
| | `SearchFocusPrevious` | ✅ **是** | 同上 |
| | `SearchConfirm` (Vi) | ⚠️ 仅无结果时 | `goto_match(None)` 无结果 → `search_reset_state` |
| | `SearchConfirm` (非 Vi) | ❌ **否** | `cancel_search` → 创建选区，不滚动 |
| | `SearchCancel` (Vi) | ✅ **是** | `search_reset_state` → 恢复视口 |
| | `SearchCancel` (非 Vi) | ❌ **否** | 创建选区，不滚动 |
| | `SearchClear` | ⚠️ 取决于取消 | `cancel_search` → `start_search` |
| | `SearchDeleteWord` | ⚠️ 取决于匹配 | `update_search` → `goto_match` |
| | `SearchHistoryPrevious` | ⚠️ 取决于匹配 | `update_search` → `goto_match` |
| | `SearchHistoryNext` | ⚠️ 取决于匹配 | `update_search` → `goto_match` |

---

## 六、触摸滚动完整调用链

### 6.1 调用链路总览

```
winit WindowEvent::Touch
        ↓
Processor::touch (input/mod.rs:840)
        ├─→ TouchPhase::Started  → on_touch_start  (判定是 Tap/Scroll/Select/Zoom)
        ├─→ TouchPhase::Moved    → on_touch_motion
        │       ├─→ 判定手势类型（横向→Select / 纵向→Scroll，阈值 20px）
        │       ├─→ TouchPurpose::Scroll  → scroll_terminal(0, delta_y, 1.0)
        │       ├─→ TouchPurpose::Select  → mouse_moved
        │       └─→ TouchPurpose::Zoom    → change_font_size
        └─→ TouchPhase::Ended    → on_touch_end
                ↓
scroll_terminal → ctx.scroll → Term::scroll_display → Grid::scroll_display
```

### 6.2 详细调用步骤

**步骤 1：触摸事件入口**

`event.rs:1984` 接收触摸事件：

```rust
WindowEvent::Touch(touch) => self.touch(touch),  // → input/mod.rs:840
```

**步骤 2：`touch` 分发触摸阶段**

`input/mod.rs:840`：

```rust
pub fn touch(&mut self, touch: TouchEvent) {
    match touch.phase {
        TouchPhase::Started => self.on_touch_start(touch),
        TouchPhase::Moved => self.on_touch_motion(touch),
        TouchPhase::Ended | TouchPhase::Cancelled => self.on_touch_end(touch),
    }
}
```

**步骤 3：`on_touch_start` 初始化触摸状态**

`input/mod.rs:849` 根据触摸点数判定手势意图：

```rust
pub fn on_touch_start(&mut self, touch: TouchEvent) {
    let touch_purpose = self.ctx.touch_purpose();
    *touch_purpose = match mem::take(touch_purpose) {
        TouchPurpose::None => TouchPurpose::Tap(touch),        // 第一指：可能是点击
        TouchPurpose::Tap(start) => TouchPurpose::Zoom(TouchZoom::new((start, touch))),  // 第二指：缩放
        // ...
    };
}
```

**步骤 4：`on_touch_motion` 判定手势类型**

`input/mod.rs:882` 是触摸滚动的核心：

```rust
pub fn on_touch_motion(&mut self, touch: TouchEvent) {
    let touch_purpose = self.ctx.touch_purpose();
    match touch_purpose {
        // 从 Tap 状态判断是横向选择还是纵向滚动
        TouchPurpose::Tap(start) => {
            let delta_x = touch.location.x - start.location.x;
            let delta_y = touch.location.y - start.location.y;
            if delta_x.abs() > MAX_TAP_DISTANCE {       // 横向移动 > 20px
                *touch_purpose = TouchPurpose::Select(*start);  // → 文本选择
            } else if delta_y.abs() > MAX_TAP_DISTANCE { // 纵向移动 > 20px
                *touch_purpose = TouchPurpose::Scroll(*start);  // → 滚动模式
                self.on_touch_motion(touch);            // 立即应用当前移动
            }
        },
        // 滚动模式：直接传递给 scroll_terminal
        TouchPurpose::Scroll(last_touch) => {
            let delta_y = touch.location.y - last_touch.location.y;
            *touch_purpose = TouchPurpose::Scroll(touch);
            // multiplier = 1.0，触控屏跟随手指移动（不做滚轮式放大）
            self.scroll_terminal(0., delta_y, 1.0);
        },
        TouchPurpose::Select(_) => self.mouse_moved(touch.location),
        TouchPurpose::Zoom(zoom) => { /* 缩放 */ },
    }
}
```

**关键点**：触摸滚动的 multiplier 固定为 `1.0`，实现精确的手指跟随滚动。

---

## 七、其他滚动入口

### 7.1 输入时自动回到底部

`event.rs:1359` 当用户开始输入时，如果正在查看历史，自动滚回到底部：

```rust
fn on_terminal_input_start(&mut self) {
    self.on_typing_start();
    self.clear_selection();

    // ✅ 如果不在底部，自动滚回
    if self.terminal().grid().display_offset() != 0 {
        self.scroll(Scroll::Bottom);
    }
}
```

调用时机：普通字符输入、粘贴、写入 PTY 前。

### 7.2 选择时自动滚动

`input/mod.rs:1116` 鼠标选择拖到窗口边缘时自动滚动：

```rust
fn update_selection_scrolling(&mut self, mouse_y: i32) {
    // 计算距边缘距离
    let delta = if mouse_y < end_top {
        end_top - mouse_y + step
    } else if mouse_y >= start_bottom {
        start_bottom - mouse_y - step
    } else {
        scheduler.unschedule(TimerId::new(Topic::SelectionScrolling, window_id));
        return;
    };

    // ✅ 构造 Scroll 事件，定时触发
    let event = Event::new(EventType::Scroll(Scroll::Delta(delta / step)), Some(window_id));
    scheduler.schedule(event, SELECTION_SCROLLING_INTERVAL, true, timer_id);
}
```

`SELECTION_SCROLLING_INTERVAL = 15ms`，每 15ms 滚动一次，速度与距离边缘的距离成正比。

### 7.3 提示键盘跳转

`event.rs:1271`：

```rust
HintAction::Text => {
    // ✅ 跳到提示起点
    self.terminal.vi_goto_point(*hint_bounds.start());
    // ...
}
```

### 7.4 WindowContext 搜索启动时的微调

`window_context.rs:554` 搜索启动时根据光标位置微调视口（避免搜索 origin 被新输出推离视口）：

```rust
if display_offset == 0 && cursor_at_bottom && !origin_at_bottom {
    terminal.scroll_display(Scroll::Delta(1));   // ✅ 向上滚 1 行
} else if display_offset != 0 && origin_at_bottom {
    terminal.scroll_display(Scroll::Delta(-1));  // ✅ 向下滚 1 行
}
```

---

## 八、完整调用关系图

```
                                 ┌───────────────────────────┐
                                 │   winit 操作系统事件        │
                                 └─────────┬─────────────────┘
                                           │
            ┌──────────────────────────────┼──────────────────────────────┐
            │                              │                              │
            ▼                              ▼                              ▼
  WindowEvent::MouseWheel      WindowEvent::KeyboardInput      WindowEvent::Touch
            │                              │                              │
            ▼                              ▼                              ▼
mouse_wheel_input (input/mod.rs)   key_input (keyboard.rs)        touch (input/mod.rs)
            │                              │                              │
            ▼                              ▼                              ▼
scroll_terminal (input/mod.rs)    process_key_bindings            on_touch_motion
    4 个分支决策                          │                              │
    ├─ process_mouse_bindings ───→ Action::execute ←───────────────────┘
    │       │                    (input/mod.rs:168)
    │       │             ┌────────┬──────┴──────┬────────┬────────┬──────────┐
    │       │             ▼        ▼             ▼        ▼        ▼          ▼
    │       │        Scroll*   Scroll*     ScrollLine*  Center   Scroll*     Vi 动作
    │       │        PageUp    PageDown    Up/Down   AroundVi  ToTop/Btm  (Up/Down/etc)
    │       │             │        │         │        Cursor     │          │
    │       │             │        │         │           │       │          │
    │       │             │  ViModeCursor::scroll()  │     ctx.scroll  Term::vi_motion
    │       │             │  ❗ 只算光标！          │       │         │
    │       │             │  不滚动视口             │       │    ViModeCursor::motion()
    │       │             └────────┴─────────┴───────────┘       │         │
    │       │                         │                          │      移动光标
    │       │                         │                          │         │
    │       └─────────────────────────┴──────────────────────────┘    scroll_to_point
    │                                       │                         (视口外才滚)
    ├─ mouse_report (鼠标模式)              │
    ├─ write_to_pty (Alt Screen)            │
    └────────────────────────────────────────┘
                        │
                        ▼
            ActionContext::scroll (event.rs:707)
                ├─ 记录 old_offset
                ├─ 调用 Term::scroll_display
                ├─ 同步 search_state.display_offset_delta
                ├─ 同步 selection / vi 光标
                ├─ on_typing_start (Vi 模式)
                └─ 标记 dirty 重绘
                        │
                        ▼
            Term::scroll_display (term/mod.rs:389)
                ├─ 调用 Grid::scroll_display
                ├─ 发送 MouseCursorDirty 事件
                ├─ 钳位 vi_mode_cursor 到视口内
                └─ mark_fully_damaged (offset 变化时)
                        │
                        ▼
            Grid::scroll_display (grid/mod.rs:163)
                └─ 修改 display_offset
                   (clamp 到 [0, history_size()])
                        │
                        ▼
            ┌────────────────────────────────────┐
            │  渲染时 display_iter() 根据 offset  │
            │  从 scrollback 区域读取行进行显示    │
            └────────────────────────────────────┘
```

---

## 九、关键设计要点与纠偏总结

### 9.1 滚动优先级设计

`scroll_terminal` 的四个分支按优先级顺序判断：

1. **鼠标绑定**（最高优先级）：用户可通过配置覆盖默认行为
2. **鼠标报告模式**：终端应用接管鼠标（如 vim 的 mouse=a）
3. **Alt Screen + Alternate Scroll**：滚轮转成上下键发送给应用
4. **默认滚动**（最低优先级）：直接修改视口偏移

### 9.2 累积滚动机制

像素滚动（触控板）使用 `accumulated_scroll` 累积像素值，达到整行高度才触发滚动，避免抖动：

```rust
self.ctx.mouse_mut().accumulated_scroll.y += new_scroll_y_px * multiplier;
let lines = (self.ctx.mouse().accumulated_scroll.y / height).abs() as usize;
// ... 滚动后取模保留余数
self.ctx.mouse_mut().accumulated_scroll.y %= height;
```

### 9.3 视口锚定

当用户查看历史时（`display_offset != 0`），新的终端输出不会把视口拉回底部，而是"推高" `display_offset`：

```rust
// Grid::scroll_up 中
if self.display_offset != 0 {
    self.display_offset = min(self.display_offset + positions, self.max_scroll_limit);
}
```

### 9.4 纠偏：`ViModeCursor::scroll()` 不滚动视口

这是最容易产生误解的命名。`ViModeCursor::scroll()` 只是翻页滚动时计算光标新位置的辅助方法。翻页滚动的视口滚动由独立的 `ctx.scroll(scroll)` 调用完成。二者配合的视觉效果是"页和光标一起滚"，但代码层面是两个独立的步骤。

### 9.5 纠偏：非 Vi 模式下取消搜索不恢复视口

`search_reset_state()` 只在 Vi 模式下恢复视口和光标：

```rust
fn search_reset_state(&mut self) {
    // ...
    if !self.terminal.mode().contains(TermMode::VI) {
        return;  // ⚠️ 非 Vi 模式直接退出，不恢复视口
    }
    // 仅 Vi 模式执行恢复
}
```

非 Vi 模式下 `cancel_search()` 会为匹配创建选区，视口停留在匹配位置。这是刻意的设计区分。

### 9.6 纠偏：`vi_motion` 的自动滚动在方法末尾统一触发

不是每个 Vi 动作单独判断滚动，而是所有动作在 `ViModeCursor::motion()` 末尾统一调用一次 `term.scroll_to_point(self.point)`：

```rust
// vi_mode.rs:183
term.scroll_to_point(self.point);  // 所有 Vi 动作都经过这里
```

因此 `High/Middle/Low` 虽然是基于当前视口计算点，但依然会执行 scroll_to_point 检查（只是结果通常在视口内，不触发实际滚动）。

### 9.7 触摸手势识别

单指触摸超过 20px 阈值时才判定为滚动/选择，避免点击误触发：

```rust
const MAX_TAP_DISTANCE: f64 = 20.;
if delta_y.abs() > MAX_TAP_DISTANCE {
    *touch_purpose = TouchPurpose::Scroll(*start);
}
```

### 9.8 边缘选择自动滚动

选择文本拖到窗口边缘时，滚动速度与距离边缘的距离成正比，每 15ms 触发一次。

---

## 十、代码参考（相对路径）

| 功能模块 | 文件路径 | 关键行 |
|---------|---------|--------|
| Grid::scroll_display | `alacritty_terminal/src/grid/mod.rs` | 163-173 |
| Term::scroll_display | `alacritty_terminal/src/term/mod.rs` | 389-408 |
| Term::vi_motion | `alacritty_terminal/src/term/mod.rs` | 839-851 |
| Term::vi_goto_point | `alacritty_terminal/src/term/mod.rs` | 855-866 |
| **Term::scroll_to_point** | `alacritty_terminal/src/term/mod.rs` | 884-898 |
| **ViModeCursor::motion** (末尾调用 scroll_to_point) | `alacritty_terminal/src/vi_mode.rs` | 74-186 (关键行 183) |
| **ViModeCursor::scroll** (只算光标不滚视口) | `alacritty_terminal/src/vi_mode.rs` | 190-201 |
| MouseScrollDelta 处理 (mouse_wheel_input) | `alacritty/src/input/mod.rs` | 725-759 |
| scroll_terminal 四分支决策 | `alacritty/src/input/mod.rs` | 760-828 |
| 触摸滚动 (on_touch_motion) | `alacritty/src/input/mod.rs` | 882-924 |
| key_input 入口 | `alacritty/src/input/keyboard.rs` | 22-103 |
| process_key_bindings | `alacritty/src/input/keyboard.rs` | 178-249 |
| **Action::execute** (翻页/单行/顶部/底部滚动实现) | `alacritty/src/input/mod.rs` | 346-403 (纠偏重点行 374 vs 379) |
| CenterAroundViCursor | `alacritty/src/input/mod.rs` | (ViAction 分支) |
| **ActionContext::scroll** (完整滚动上下文) | `alacritty/src/event.rs` | 707-741 |
| on_terminal_input_start (输入自动回底) | `alacritty/src/event.rs` | 1359-1366 |
| update_selection_scrolling (边缘自动滚动) | `alacritty/src/input/mod.rs` | 1116-1148 |
| **start_search (不滚动)** | `alacritty/src/event.rs` | 946-981 |
| **update_search → goto_match** | `alacritty/src/event.rs` | 1503-1605 |
| **goto_match (搜索跳转)** | `alacritty/src/event.rs` | 1554-1605 |
| advance_search_origin (下一条/上一条) | `alacritty/src/event.rs` | 1132-1149 |
| **search_reset_state (仅 Vi 模式恢复视口)** | `alacritty/src/event.rs` | 1530-1551 |
| cancel_search (Vi vs 非 Vi 差异) | `alacritty/src/event.rs` | 1047-1063 |
| confirm_search | `alacritty/src/event.rs` | 1030-1044 |
| start_seeded_search | `alacritty/src/event.rs` | 984-1027 |
| 窗口上下文搜索微调 | `alacritty/src/window_context.rs` | 554-559 |
| Scrolling 配置 | `alacritty/src/config/scrolling.rs` | 1-53 |
| **ViAction 枚举定义**（搜索相关） | `alacritty/src/config/bindings.rs` | 298-335 |
| **SearchAction 枚举定义** | `alacritty/src/config/bindings.rs` | 340-357 |
| **InlineSearchState 结构体** | `alacritty/src/event.rs` | 643-661 |
| **Action::execute 搜索相关实现**（SearchNext/Previous/Start/End） | `alacritty/src/input/mod.rs` | 212-261 |
| **行内搜索启动**（start_inline_search） | `alacritty/src/event.rs` | 1445-1450 |
| **行内搜索字符输入**（inline_search_input） | `alacritty/src/event.rs` | 1465-1478 |
| **行内搜索跳转**（inline_search_next/previous） | `alacritty/src/event.rs` | 1453-1462 |
| **行内搜索核心实现**（inline_search） | `alacritty/src/event.rs` | 1674-1706 |
| **终端层 inline_search_right/left** | `alacritty_terminal/src/term/search.rs` | 565-590 |
| **语义搜索**（Action::execute 中） | `alacritty/src/input/mod.rs` | 285-299 |
| **SearchAction 实现**（SearchFocus* 等） | `alacritty/src/input/mod.rs` | 303-319 |
| **ViAction 行内搜索入口** | `alacritty/src/input/mod.rs` | 271-284 |
