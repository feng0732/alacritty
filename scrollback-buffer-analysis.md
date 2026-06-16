# Alacritty 滚动缓冲用户滚动入口完整分析

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
        ├─→ 键盘绑定动作匹配
        │    ├─→ Action::ScrollPageUp / ScrollPageDown
        │    ├─→ Action::ScrollHalfPageUp / ScrollHalfPageDown
        │    ├─→ Action::ScrollLineUp / ScrollLineDown
        │    ├─→ Action::ScrollToTop / ScrollToBottom
        │    └─→ Action::Vi(...) 中的滚动动作
        └─→ ctx.scroll(Scroll::*)
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

    // 提示模式、内联搜索等特殊路径...

    // 步骤 3：尝试匹配键绑定
    if self.process_key_bindings(&key) {
        return;  // 绑定匹配成功，跳过普通输入
    }

    // 搜索输入、Vi 模式等处理...

    // 普通字符输入 → 写入 PTY
}
```

**步骤 3：`process_key_bindings` 匹配绑定**

`input/keyboard.rs:178` 匹配配置的键绑定：

```rust
fn process_key_bindings(&mut self, key: &KeyEvent) -> bool {
    let mode = BindingMode::new(self.ctx.terminal().mode(), self.ctx.search_active());
    let mods = self.ctx.modifiers().state();

    let mut binding_action = |binding: &KeyBinding| {
        // 构造匹配用的 key...
        if binding.is_triggered_by(mode, mods, &key) {
            Some(binding.action.clone())
        } else { None }
    };

    // 遍历所有按键绑定
    for i in 0..self.ctx.config().key_bindings().len() {
        let binding = &self.ctx.config().key_bindings()[i];
        if let Some(action) = binding_action(binding) {
            action.execute(&mut self.ctx);  // ← 执行动作
        }
    }
    // ...
}
```

**步骤 4：`Action::execute` 执行滚动动作**

`input/mod.rs:168` 的 `Execute` trait 实现中，滚动相关动作：

```rust
impl<T: EventListener> Execute<T> for Action {
    fn execute<A: ActionContext<T>>(&self, ctx: &mut A) {
        match self {
            // 整页滚动
            Action::ScrollPageUp | Action::ScrollPageDown
            | Action::ScrollHalfPageUp | Action::ScrollHalfPageDown => {
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

                // 同步 vi 光标
                let old_vi_cursor = term.vi_mode_cursor;
                term.vi_mode_cursor = term.vi_mode_cursor.scroll(term, amount);
                if old_vi_cursor != term.vi_mode_cursor {
                    ctx.mark_dirty();
                }

                ctx.scroll(scroll);  // ← 调用滚动
            },

            // 单行滚动
            Action::ScrollLineUp => ctx.scroll(Scroll::Delta(1)),
            Action::ScrollLineDown => ctx.scroll(Scroll::Delta(-1)),

            // 滚动到顶部
            Action::ScrollToTop => {
                ctx.scroll(Scroll::Top);
                let topmost_line = ctx.terminal().topmost_line();
                ctx.terminal_mut().vi_mode_cursor.point.line = topmost_line;
                ctx.terminal_mut().vi_motion(ViMotion::FirstOccupied);
                ctx.mark_dirty();
            },

            // 滚动到底部
            Action::ScrollToBottom => {
                ctx.scroll(Scroll::Bottom);
                let term = ctx.terminal_mut();
                term.vi_mode_cursor.point.line = term.bottommost_line();
                term.vi_motion(ViMotion::FirstOccupied);
                ctx.mark_dirty();
            },

            // Vi 模式下的 Centering 动作
            Action::Vi(ViAction::CenterAroundViCursor) => {
                let term = ctx.terminal();
                let display_offset = term.grid().display_offset() as i32;
                let target = -display_offset + term.screen_lines() as i32 / 2 - 1;
                let line = term.vi_mode_cursor.point.line;
                let scroll_lines = target - line.0;
                ctx.scroll(Scroll::Delta(scroll_lines));
            },

            // ... 其他动作
        }
    }
}
```

**步骤 5-7：同鼠标滚轮路径**

`ctx.scroll(Scroll::*)` → `ActionContext::scroll` → `Term::scroll_display` → `Grid::scroll_display`

### 3.3 Vi 模式下的滚动

在 Vi 模式下，移动 vi 光标也可能触发滚动（光标移动到视口外时自动滚动）：

`term/mod.rs:893` 的 `vi_motion` 中：

```rust
pub fn vi_motion(&mut self, motion: ViMotion) {
    // ...
    // 如果光标超出视口，自动滚动
    if self.vi_mode_cursor.point.line < Line(-(self.grid.display_offset() as i32)) {
        let scroll = self.grid.display_offset() as i32 + self.vi_mode_cursor.point.line.0;
        self.scroll_display(Scroll::Delta(-scroll));
    } else if self.vi_mode_cursor.point.line
        > Line(-(self.grid.display_offset() as i32) + self.bottommost_line().0)
    {
        let scroll = self.grid.display_offset() as i32 + self.vi_mode_cursor.point.line.0
            - self.bottommost_line().0;
        self.scroll_display(Scroll::Delta(-scroll));
    }
}
```

---

## 四、触摸滚动完整调用链

### 4.1 调用链路总览

```
winit WindowEvent::Touch
        ↓
Processor::touch (input/mod.rs:840)
        ├─→ TouchPhase::Started  → on_touch_start  (判定是 Tap/Scroll/Select/Zoom)
        ├─→ TouchPhase::Moved    → on_touch_motion
        │       ├─→ 判定手势类型（Tap→Scroll/Select）
        │       ├─→ TouchPurpose::Scroll  → scroll_terminal(0, delta_y, 1.0)
        │       ├─→ TouchPurpose::Select  → mouse_moved
        │       └─→ TouchPurpose::Zoom    → change_font_size
        └─→ TouchPhase::Ended    → on_touch_end
                ↓
scroll_terminal → ctx.scroll → Term::scroll_display → Grid::scroll_display
```

### 4.2 详细调用步骤

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
                // ... 模拟鼠标按下
            } else if delta_y.abs() > MAX_TAP_DISTANCE { // 纵向移动 > 20px
                *touch_purpose = TouchPurpose::Scroll(*start);  // → 滚动模式
                self.on_touch_motion(touch);            // 立即应用当前移动
            }
        },
        // 滚动模式：直接传递给 scroll_terminal
        TouchPurpose::Scroll(last_touch) => {
            let delta_y = touch.location.y - last_touch.location.y;
            *touch_purpose = TouchPurpose::Scroll(touch);
            // multiplier = 1.0，触控屏跟随手指移动
            self.scroll_terminal(0., delta_y, 1.0);
        },
        TouchPurpose::Select(_) => self.mouse_moved(touch.location),
        TouchPurpose::Zoom(zoom) => {
            let font_delta = zoom.font_delta(touch);
            self.ctx.change_font_size(font_delta);
        },
        // ...
    }
}
```

**关键点**：触摸滚动的 multiplier 固定为 `1.0`，实现精确的手指跟随滚动。

**步骤 5+：同鼠标滚轮路径**

`scroll_terminal` 的四个分支逻辑与鼠标滚轮完全相同。

---

## 五、其他滚动入口

### 5.1 输入时自动回到底部

`event.rs:1359` 当用户开始输入时，如果正在查看历史，自动滚回到底部：

```rust
fn on_terminal_input_start(&mut self) {
    self.on_typing_start();
    self.clear_selection();

    // 如果不在底部，自动滚回
    if self.terminal().grid().display_offset() != 0 {
        self.scroll(Scroll::Bottom);
    }
}
```

调用时机：普通字符输入、粘贴、写入 PTY 前。

### 5.2 选择时自动滚动

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

    // 构造 Scroll 事件，定时触发
    let event = Event::new(EventType::Scroll(Scroll::Delta(delta / step)), Some(window_id));
    scheduler.schedule(event, SELECTION_SCROLLING_INTERVAL, true, timer_id);
}
```

`SELECTION_SCROLLING_INTERVAL = 15ms`，每 15ms 滚动一次，速度与距离边缘的距离成正比。

### 5.3 搜索结果跳转滚动

`event.rs:1167` 搜索匹配时跳转到结果位置：

```rust
self.terminal.scroll_to_point(new_origin);  // term/mod.rs: 滚动使点可见
```

搜索取消时恢复原视口：

```rust
// event.rs:1547
self.terminal.scroll_display(Scroll::Delta(self.search_state.display_offset_delta));
```

### 5.4 WindowContext 搜索启动时的微调滚动

`window_context.rs:554` 搜索启动时根据光标位置微调视口：

```rust
if display_offset == 0 && cursor_at_bottom && !origin_at_bottom {
    terminal.scroll_display(Scroll::Delta(1));
} else if display_offset != 0 && origin_at_bottom {
    terminal.scroll_display(Scroll::Delta(-1));
}
```

---

## 六、`scroll` 方法在不同上下文的实现

### 6.1 `ActionContext` trait 中的默认实现

`input/mod.rs:96` 提供空实现，由具体上下文覆盖：

```rust
pub trait ActionContext<T: EventListener> {
    fn scroll(&mut self, _scroll: Scroll) {}  // 默认空实现
    // ...
}
```

### 6.2 `ActionContext` for `ActionContext`（事件处理上下文）

`event.rs:707` — 实际用于鼠标/键盘滚动的完整实现：

```rust
impl<'a, N: Notify + 'a, T: EventListener> input::ActionContext<T> for ActionContext<'a, N, T> {
    fn scroll(&mut self, scroll: Scroll) {
        // 完整实现：记录旧偏移、调用 scroll_display、同步选区、搜索状态、标记重绘
        // 详见 2.2 步骤 4
    }
}
```

### 6.3 `ActionContext` for `ActionContext`（测试 Mock）

`input/mod.rs:1218` — 测试用简化实现：

```rust
impl<T: EventListener> super::ActionContext<T> for ActionContext<'_, T> {
    fn scroll(&mut self, scroll: Scroll) {
        self.terminal.scroll_display(scroll);  // 只改 display_offset
    }
}
```

### 6.4 `Processor::scroll` — 鼠标滚轮路径

`input/mod.rs:725-828` — 处理 `MouseScrollDelta`，多分支决策后调用 `ctx.scroll`。

### 6.5 `Term::scroll_display` — 终端层

`term/mod.rs:389` — 负责 vi 光标钳位、损坏标记。

### 6.6 `Grid::scroll_display` — 网格层

`grid/mod.rs:163` — 只修改 `display_offset`，纯逻辑操作。

---

## 七、完整调用关系图

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
    │       │                         匹配动作类型
    │       │                    ┌─────┼─────┬────────┬────────┬────────┐
    │       │                    ▼     ▼     ▼        ▼        ▼        ▼
    │       │                Scroll  Scroll  Scroll  Scroll  ScrollTo  ViCenter
    │       │                PageUp  PageDn  LineUp  LineDn  Top/Bottom
    │       │                    │     │     │        │        │        │
    │       └────────────────────┴─────┴─────┴────────┴────────┴────────┘
    │                                       │
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

## 八、关键设计要点

### 8.1 滚动优先级设计

`scroll_terminal` 的四个分支按优先级顺序判断：

1. **鼠标绑定**（最高优先级）：用户可通过配置覆盖默认行为
2. **鼠标报告模式**：终端应用接管鼠标（如 vim 的 mouse=a）
3. **Alt Screen + Alternate Scroll**：滚轮转成上下键发送给应用
4. **默认滚动**（最低优先级）：直接修改视口偏移

### 8.2 累积滚动机制

像素滚动（触控板）使用 `accumulated_scroll` 累积像素值，达到整行高度才触发滚动，避免抖动：

```rust
self.ctx.mouse_mut().accumulated_scroll.y += new_scroll_y_px * multiplier;
let lines = (self.ctx.mouse().accumulated_scroll.y / height).abs() as usize;
// ... 滚动后取模保留余数
self.ctx.mouse_mut().accumulated_scroll.y %= height;
```

### 8.3 视口锚定

当用户查看历史时（`display_offset != 0`），新的终端输出不会把视口拉回底部，而是"推高" `display_offset`：

```rust
// Grid::scroll_up 中
if self.display_offset != 0 {
    self.display_offset = min(self.display_offset + positions, self.max_scroll_limit);
}
```

### 8.4 触摸手势识别

单指触摸超过 20px 阈值时才判定为滚动/选择，避免点击误触发：

```rust
const MAX_TAP_DISTANCE: f64 = 20.;
if delta_y.abs() > MAX_TAP_DISTANCE {
    *touch_purpose = TouchPurpose::Scroll(*start);
}
```

### 8.5 边缘选择自动滚动

选择文本拖到窗口边缘时，滚动速度与距离边缘的距离成正比，每 15ms 触发一次：

```rust
let delta = if mouse_y < end_top {
    end_top - mouse_y + step    // 越靠上滚越快
} else if mouse_y >= start_bottom {
    start_bottom - mouse_y - step  // 越靠下滚越快
} else { ... };
```

---

## 九、代码参考（相对路径）

| 功能模块 | 文件路径 | 关键行 |
|---------|---------|--------|
| Grid::scroll_display | alacritty_terminal/src/grid/mod.rs | 163-173 |
| Grid::display_offset | alacritty_terminal/src/grid/mod.rs | 134 |
| Term::scroll_display | alacritty_terminal/src/term/mod.rs | 389-408 |
| Term::scroll_to_point | alacritty_terminal/src/term/mod.rs | - |
| MouseScrollDelta 处理 | alacritty/src/input/mod.rs | 725-828 |
| scroll_terminal | alacritty/src/input/mod.rs | 760-828 |
| process_mouse_bindings | alacritty/src/input/mod.rs | 1039-1068 |
| key_input | alacritty/src/input/keyboard.rs | 22-103 |
| process_key_bindings | alacritty/src/input/keyboard.rs | 178-249 |
| Action::execute (滚动) | alacritty/src/input/mod.rs | 353-401 |
| ActionContext::scroll | alacritty/src/event.rs | 707-741 |
| on_terminal_input_start | alacritty/src/event.rs | 1359-1366 |
| update_selection_scrolling | alacritty/src/input/mod.rs | 1116-1148 |
| touch/on_touch_motion | alacritty/src/input/mod.rs | 840-924 |
| TouchPurpose enum | alacritty/src/event.rs | 1711 |
| 搜索滚动 | alacritty/src/event.rs | 1160-1172, 1530-1551 |
| 窗口上下文滚动 | alacritty/src/window_context.rs | 554-559 |
| Scrolling 配置 | alacritty/src/config/scrolling.rs | 1-53 |
