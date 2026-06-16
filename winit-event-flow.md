# Winit 窗口与事件链调用顺序详解

本文档沿着实际代码追踪 winit 事件链的每一步，重点讲清：
1. **显示更新与实际绘制的层级关系**（哪些改状态，哪些改 GL，哪些真的画）
2. **等待回调如何设置唤醒时机**（about_to_wait → ControlFlow::WaitUntil）
3. **Frame 事件如何回到窗口重绘**（调度器 → user_event → request_redraw → RedrawRequested）

---

## 一、整体调用时序图

```
(1) 用户敲键盘 / PTY 有数据
        │
        ▼
winit 回调: window_event / user_event
        │
        ▼
(2) Processor 全局层
    ├─ window_event()  ──→ 去重过滤 ──→ WindowContext::handle_event()
    │                                     ├─ 非 Redraw/AboutToWait → 入 event_queue 返回
    │                                     └─ RedrawRequested        ──→ 排空队列 + 处理 + 返回
    │                                                                  │
    │                                                          (3) draw() 实际绘制
    └─ user_event()
        ├─ Wakeup ──→ 标 dirty + request_redraw(如有 frame)
        └─ Frame  ──→ has_frame=true + dirty? request_redraw()
                              │
                              ▼
                   winit 内部排队 RedrawRequested
                              │
                              ▼
                   下一轮 window_event() 收到 RedrawRequested
```

---

## 二、等待回调设置唤醒时机（Scheduler + about_to_wait）

### 2.1 触发链

事件队列即将耗尽时，winit 调用：

**[event.rs#L466-L490](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/event.rs#L466-L490)**
```rust
fn about_to_wait(&mut self, event_loop: &ActiveEventLoop) {
    // 先让所有窗口处理 AboutToWait（排空各自的 event_queue）
    for window_context in self.windows.values_mut() {
        window_context.handle_event(..., WinitEvent::AboutToWait);
    }

    // 然后更新调度器，设置下一次唤醒时间
    let control_flow = match self.scheduler.update() {
        Some(instant) => ControlFlow::WaitUntil(instant),
        None => ControlFlow::Wait,
    };
    event_loop.set_control_flow(control_flow);
}
```

### 2.2 Scheduler::update() 内部机制

**[scheduler.rs#L58-L73](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/scheduler.rs#L58-L73)**
```rust
pub fn update(&mut self) -> Option<Instant> {
    let now = Instant::now();

    // 弹出所有 deadline <= now 的定时器，发送对应事件
    while !self.timers.is_empty() && self.timers[0].deadline <= now {
        if let Some(timer) = self.timers.pop_front() {
            // 如果是周期性定时器（如 BlinkCursor），重新入队
            if let Some(interval) = timer.interval {
                self.schedule(timer.event.clone(), interval, true, timer.id);
            }
            // 通过 EventLoopProxy 发送自定义事件
            let _ = self.event_proxy.send_event(timer.event);
        }
    }

    // 返回最近一个未到期定时器的 deadline
    // 返回值被 about_to_wait 用来设置 WaitUntil
    self.timers.front().map(|timer| timer.deadline)
}
```

### 2.3 定时器如何入队

**[scheduler.rs#L76-L90](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/scheduler.rs#L76-L90)**

`schedule()` 按 deadline 升序插入 `VecDeque`，保证队首永远是最早到期的。

**关键 Topic 列表**（[scheduler.rs#L26-L32](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/scheduler.rs#L26-L32)）：

| Topic | 周期 | 生产者 | 事件内容 |
|-------|------|--------|----------|
| Frame | 单次 | `Display::request_frame()` | 对齐到 vblank 的下一个绘制槽 |
| BlinkCursor | 周期 | 光标闪烁逻辑 | 触发光标切换可见性 |
| BlinkTimeout | 单次 | 键盘/鼠标活动后 | 光标停止闪烁进入省电 |
| SelectionScrolling | 周期 | 鼠标在窗口外选择时 | 每 15ms 滚动终端 |
| DelayedSearch | 单次 | 搜索输入延迟 | 延迟搜索触发 |

### 2.4 唤醒时机设置示例

假设显示器 60Hz（16.667ms/帧），刚 swap_buffers 后调用 `request_frame()`：
1. `FrameTimer::compute_timeout()` 算出距下一个 vblank 相位点还有 12ms
2. `scheduler.schedule(Frame事件, 12ms, false, Frame+WindowId)`
3. 事件队列耗尽触发 `about_to_wait()`
4. `scheduler.update()` 发现队首 Frame 定时器还有 10ms 才到，返回该 Instant
5. `event_loop.set_control_flow(WaitUntil(该Instant))`
6. winit 阻塞到该时刻，超时后自动唤醒事件循环，产生 `StartCause::WaitCancelled` 或直接触发 `user_event`

---

## 三、Frame 事件回到窗口重绘的完整路径

这是最容易跳读的链路，共 **7 步**：

### Step 1: swap_buffers 后调度 Frame

**[display/mod.rs#L1040-L1044](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L1040-L1044)**
```rust
// 非 Wayland 平台才走这个路径
if !matches!(self.raw_window_handle, RawWindowHandle::Wayland(_)) {
    self.request_frame(scheduler);  // ← 调度 Frame 定时器
}
```

**注释特别说明**：在 swap_buffers **之后**才调度，这样 OpenGL 操作完成时间会被计入超时计算。

### Step 2: request_frame 计算 vblank 对齐延迟

**[display/mod.rs#L1435-L1458](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L1435-L1458)**
```rust
fn request_frame(&mut self, scheduler: &mut Scheduler) {
    self.window.has_frame = false;  // 标记当前帧槽已用掉

    // 读显示器刷新率（默认 60_000 mHz = 60Hz）
    let monitor_vblank_interval = 1_000_000.0 / refresh_rate_millihertz as f64;
    let monitor_vblank_interval = Duration::from_micros(...);

    // FrameTimer 计算对齐到 vblank 相位的等待时间
    let swap_timeout = self.frame_timer.compute_timeout(monitor_vblank_interval);

    // 把 Frame 事件排进调度器
    let timer_id = TimerId::new(Topic::Frame, window_id);
    let event = Event::new(EventType::Frame, window_id);
    scheduler.schedule(event, swap_timeout, false, timer_id);
}
```

### Step 3: Scheduler 到点后发送 user_event

**[scheduler.rs#L61-L69](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/scheduler.rs#L61-L69)**

deadline 到了就 `event_proxy.send_event(Event{EventType::Frame, Some(window_id)})`，这是跨线程安全的通道。

### Step 4: Processor::user_event 收到 Frame（快速路径）

**[event.rs#L442-L449](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/event.rs#L442-L449)**
```rust
// NOTE: This event bypasses batching to minimize input latency.
(EventType::Frame, Some(window_id)) => {
    if let Some(window_context) = self.windows.get_mut(window_id) {
        window_context.display.window.has_frame = true;  // ① 标记帧槽可用
        if window_context.dirty {                        // ② 如果内容有变化
            window_context.display.window.request_redraw();  // ③ 请求重绘
        }
    }
},
```

**关键点**：Frame 事件**不走** WindowContext 的 event_queue 批处理，直接在全局层处理，注释明确写 "bypasses batching to minimize input latency"。

### Step 5: Window::request_redraw 去重

**[display/window.rs#L260-L265](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/window.rs#L260-L265)**
```rust
pub fn request_redraw(&mut self) {
    if !self.requested_redraw {
        self.requested_redraw = true;
        self.window.request_redraw();  // 调 winit 的 request_redraw
    }
}
```

`requested_redraw` 标志防止一帧内重复请求多次。

### Step 6: winit 产生 RedrawRequested

winit 内部调度后，通过 `window_event()` 回调回传 `WindowEvent::RedrawRequested`。

### Step 7: Processor::window_event 收到 RedrawRequested → draw()

**[event.rs#L269-L282](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/event.rs#L269-L282)**
```rust
let is_redraw = matches!(event, WindowEvent::RedrawRequested);

window_context.handle_event(..., WinitEvent::WindowEvent { window_id, event });
// ↑ handle_event 里如果是 RedrawRequested，会排空 event_queue 并处理批量事件

if is_redraw {
    window_context.draw(&mut self.scheduler);  // ← 真的画！
}
```

### 路径总结

```
swap_buffers() 后
    │
    ▼ request_frame()
FrameTimer.compute_timeout() → 计算对齐延迟
    │
    ▼ scheduler.schedule(Frame 事件)
VecDeque 按 deadline 升序排队
    │
    ▼ about_to_wait() 触发 scheduler.update()
deadline 到了 → event_proxy.send_event(Frame)
    │
    ▼ Processor::user_event() 快速路径
has_frame = true; dirty? request_redraw()
    │
    ▼ winit 内部调度
    │
    ▼ Processor::window_event(RedrawRequested)
handle_event() 排空队列 → draw() 真正绘制
    │
    ▼ draw() 末尾 swap_buffers()
回到 Step 1 循环
```

---

## 四、显示更新与实际绘制的层级

整个系统分 **4 个层级**，每一层只做自己该做的事，绝不越界：

### 层级 1：事件入队（只记不改）

**位置**：[window_context.rs#L409-L423](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/window_context.rs#L409-L423)

```rust
match event {
    AboutToWait | RedrawRequested => { /* 继续往下处理 */ },
    event => {
        self.event_queue.push(event);  // ← 仅入队，立即返回
        return;
    },
}
```

**除了 AboutToWait 和 RedrawRequested，其他所有事件一律入队等批处理**。包括：
- 键盘、鼠标、触摸、IME
- 窗口 Resized、ScaleFactorChanged、Focused、Occluded
- 自定义 Event（如 BlinkCursor、ConfigReload 等）

### 层级 2：批量处理（改终端状态 + 改 DisplayUpdate 缓冲）

**触发时机**：收到 AboutToWait 或 RedrawRequested，且 event_queue 非空。

**位置**：[window_context.rs#L425-L473](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/window_context.rs#L425-L473)

```rust
// ① 获取终端锁
let mut terminal = self.terminal.lock();

// ② 构建 ActionContext（把 WindowContext 拆成引用视图）
let context = ActionContext { terminal: &mut terminal, display: &mut self.display, ... };

// ③ input::Processor 逐个处理事件
let mut processor = input::Processor::new(context);
for event in self.event_queue.drain(..) {
    processor.handle_event(event);
}
```

#### input::Processor 处理结果产出两类东西：

| 产出 | 存储位置 | 含义 |
|------|----------|------|
| 终端内容变更 | `terminal.grid()` | 直接写终端缓冲区，同时标 damage |
| 显示配置变更 | `display.pending_update` (DisplayUpdate) | **不直接生效，先缓存** |

**DisplayUpdate 结构**：[display/mod.rs#L304-L310](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L304-L310)
```rust
pub struct DisplayUpdate {
    pub dirty: bool,              // 是否有待处理更新
    dimensions: Option<PhysicalSize<u32>>,  // 窗口尺寸变化
    cursor_dirty: bool,           // 光标样式变化
    font: Option<Font>,           // 字体大小变化
}
```

这些字段在 input 处理时通过 `ActionContext` 的 setter 写入 `pending_update`，但不立即应用。

### 层级 3：DisplayUpdate 提交（改 SizeInfo + 排 GL 操作）

**触发**：批量处理结束后，如果 `pending_update.dirty` 为 true。

**位置**：[window_context.rs#L461-L473](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/window_context.rs#L461-L473)
```rust
if self.display.pending_update.dirty {
    Self::submit_display_update(&mut terminal, &mut self.display, ...);
    self.dirty = true;  // 提交后标脏，触发重绘
}
```

#### submit_display_update → display.handle_update()

**位置**：[display/mod.rs#L651-L737](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L651-L737)

```rust
pub fn handle_update<T>(&mut self, terminal, pty_resize_handle, ...) {
    let pending_update = mem::take(&mut self.pending_update);  // 取出并清空

    // ① 如果改了字体 / 光标样式 → 排进 pending_renderer_update（GL 层）
    if pending_update.font().is_some() || pending_update.cursor_dirty() {
        let renderer_update = self.pending_renderer_update.get_or_insert(...);
        renderer_update.clear_font_cache = true;
    }

    // ② 计算新 cell_width / cell_height（字体变化时）
    // ③ 计算新窗口尺寸（dimensions 变化时）
    // ④ 构建新 SizeInfo
    // ⑤ 如果行列数变了 → resize PTY + resize Terminal grid
    // ⑥ 如果尺寸变了 → 标记 pending_renderer_update.resize = true
    self.size_info = new_size;
}
```

**关键设计**：`handle_update()` **不做任何 GL 调用**。所有需要 GL context current 的操作都被延迟到 `pending_renderer_update`，在下一层处理。

注释说明原因（[display/mod.rs#L739-L741](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L739-L741)）：
> Wayland 等平台要求 resize 和其他 GL 操作必须在渲染前一刻执行，否则会锁定 back buffer 并用旧状态渲染，还会导致 resize 闪烁。

### 层级 4：实际绘制（GL 操作 + swap_buffers）

#### Step 4.1: WindowContext::draw() 预处理

**位置**：[window_context.rs#L366-L398](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/window_context.rs#L366-L398)
```rust
pub fn draw(&mut self, scheduler: &mut Scheduler) {
    self.display.window.requested_redraw = false;  // 清请求标志

    if self.occluded { return; }  // 窗口被遮挡，不画

    self.dirty = false;           // 清脏标志

    // ① 先执行上一层排队的 GL 操作
    self.display.process_renderer_update();

    // ② 视觉铃动画未结束 → 再请求下一帧
    if !self.display.visual_bell.completed() {
        if self.display.window.has_frame {
            self.display.window.request_redraw();
        } else {
            self.dirty = true;
        }
    }

    // ③ 拿终端锁 → 调 Display::draw()
    let terminal = self.terminal.lock();
    self.display.draw(terminal, scheduler, ...);
}
```

#### Step 4.2: process_renderer_update（真正改 GL 状态）

**位置**：[display/mod.rs#L744-L768](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L744-L768)
```rust
pub fn process_renderer_update(&mut self) {
    let renderer_update = match self.pending_renderer_update.take() {
        Some(u) => u, _ => return,  // 没待处理就返回
    };

    // ① Surface resize（需要 GL context）
    if renderer_update.resize {
        self.surface.resize(&self.context, width, height);
    }

    // ② make_current（确保我们在改正确的 GL context）
    self.make_current();

    // ③ 清字体缓存（GL 纹理操作）
    if renderer_update.clear_font_cache {
        self.reset_glyph_cache();
    }

    // ④ Renderer resize（更新 uniform、视口等）
    self.renderer.resize(&self.size_info);
}
```

#### Step 4.3: Display::draw() 真正上屏

**位置**：[display/mod.rs#L775-L1047](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L775-L1047)

```rust
pub fn draw<T: EventListener>(&mut self, mut terminal: MutexGuard<'_, Term<T>>, ...) {
    // ① process_renderer_update（再次确认，可能在 handle_update 之后才标脏）
    self.process_renderer_update();

    // ② 尽早释放终端锁！（减少 I/O 线程阻塞）
    //    收集完 RenderableContent 就 drop terminal
    //    ... damage 合并、选择计算、光标计算都在这里 ...

    // ③ make_current
    self.make_current();

    // ④ GL 绘制调用
    self.renderer.clear(&self.colors.background);
    self.renderer.draw_cells(...);        // 文字
    self.renderer.draw_rects(...);        // 光标/下划线
    self.renderer.draw_string(...);       // UI 文本

    // ⑤ 通知 winit 即将 present（macOS 需要）
    self.window.pre_present_notify();

    // ⑥ swap_buffers 上屏
    self.swap_buffers();

    // ⑦ X11: renderer.finish() 解决一帧延迟问题

    // ⑧ 调度下一帧（非 Wayland）
    self.request_frame(scheduler);

    // ⑨ 轮换 damage 缓冲区（新帧从空 damage 开始）
    self.damage_tracker.swap_damage();
}
```

### 四层总结

```
事件到达
  │
  ├─ 层级1: event_queue 入队（只记不改）[window_context.rs#L409-L423]
  │
  ▼ 收到 AboutToWait / RedrawRequested 且队列非空
  │
  ├─ 层级2: input::Processor 批量处理
  │     ├─ 改 terminal（直接生效 + 标 damage）
  │     └─ 改 display.pending_update（DisplayUpdate 缓冲）
  │
  ▼ pending_update.dirty?
  │
  ├─ 层级3: submit_display_update → handle_update()
  │     ├─ 改 SizeInfo（窗口/字体尺寸计算）
  │     ├─ resize PTY + Terminal grid
  │     └─ 排队 GL 操作到 pending_renderer_update
  │
  ▼ RedrawRequested 触发 draw()
  │
  └─ 层级4: 实际绘制
        ├─ process_renderer_update()（surface resize / make_current / 清字体缓存）
        ├─ draw_cells / draw_rects / draw_string（GL 调用）
        ├─ swap_buffers（上屏）
        └─ request_frame（调度下一帧，回到循环开始）
```

---

## 五、各状态标志的流转时机

### 5.1 dirty（终端内容变了需要重绘）

| 设置时机 | 清时机 |
|----------|--------|
| input::Processor 修改终端内容 | `WindowContext::draw()` 第 373 行 |
| DisplayUpdate 提交后（[window_context.rs#L472](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/window_context.rs#L472)） | |
| Wakeup 事件（PTY 有新数据） | |
| hint highlight 变化 | |

### 5.2 has_frame（是否有可用帧槽）

| 设置时机 | 清时机 |
|----------|--------|
| Frame 事件到达（[event.rs#L445](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/event.rs#L445)） | `request_frame()` 开头（[display/mod.rs#L1437](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L1437)） |
| 初始化为 true（Display 创建时） | |

**作用**：这是 Alacritty 自己的帧率节流阀。即使 dirty=true，如果还没到下一个 Frame 槽位（has_frame=false），也不会 request_redraw，避免超刷新率绘制。

### 5.3 requested_redraw（是否已向 winit 请求重绘）

| 设置时机 | 清时机 |
|----------|--------|
| `Window::request_redraw()` 内部去重后 | `WindowContext::draw()` 第 367 行 |

**作用**：一帧内多次调用 request_redraw 只实际调一次 winit API。

### 5.4 pending_update.dirty / pending_renderer_update

| 标志 | 层级 | 设置 | 消费 |
|------|------|------|------|
| pending_update.dirty | 2→3 | input 处理器写入 dimensions/font/cursor_dirty | `handle_update()` 中 `mem::take()` |
| pending_renderer_update | 3→4 | `handle_update()` 标记 resize / clear_font_cache | `process_renderer_update()` 中 `take()` |

---

## 六、Wayland 差异（不走 Frame 定时器）

Wayland 平台完全跳过 `request_frame()` 调度器路径：

- **[display/mod.rs#L1042](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L1042)**：`if !matches!(self.raw_window_handle, Wayland(_))` 判断跳过 Frame 调度
- 依赖 Wayland compositor 的 frame callback 来驱动 RedrawRequested
- `has_frame` 标志在 Wayland 上由 winit 的 frame callback 间接维护
- Damage 也走 EGL 的 `swap_buffers_with_damage()` 优化

---

## 七、完整链路单步追踪示例

场景：用户输入一个字符（a），从按键到上屏的完整路径。

```
① 用户按键盘
   ↓ winit 产生 WindowEvent::KeyboardInput
② Processor::window_event() [event.rs#L249]
   ├─ skip_window_event? No
   └─ window_context.handle_event(WinitEvent::WindowEvent{KeyboardInput})
③ WindowContext::handle_event() [window_context.rs#L401]
   ├─ 不是 AboutToWait / RedrawRequested
   └─ self.event_queue.push(event); return;   ← 入队，不处理
④ 事件队列耗尽，winit 调 about_to_wait() [event.rs#L466]
⑤ 每个窗口 handle_event(AboutToWait)
   ├─ 是 AboutToWait，且队列非空 → 继续
   ├─ terminal.lock()
   ├─ ActionContext 构建
   ├─ input::Processor::handle_event(KeyboardInput)
   │    ├─ 匹配键绑定？No（普通字符）
   │    ├─ Vi 模式？No
   │    ├─ build_sequence() 生成 "a" 字节序列
   │    └─ write_to_pty() 发送给 shell
   ├─ shell 还没返回输出，terminal 内容没变 → dirty 仍为 false
   ├─ pending_update.dirty? No → 不提交
   └─ dirty? No → 不 request_redraw
⑥ scheduler.update() → 暂无到期定时器 → ControlFlow::Wait
⑦ shell 处理 "a"，PTY 返回输出
⑧ PtyEventLoop 线程检测到 PTY 可读
   ├─ 解析终端输出，写入 terminal.grid()（通过 FairMutex）
   └─ event_proxy.send_event(Event{EventType::Terminal(Wakeup), Some(id)})
⑨ Processor::user_event(Wakeup) [event.rs#L409]
   ├─ window_context.dirty = true
   └─ has_frame? Yes → window_context.display.window.request_redraw()
⑩ winit 调度 RedrawRequested
⑪ Processor::window_event(RedrawRequested) [event.rs#L249]
    ├─ window_context.handle_event(RedrawRequested)
    │    ├─ 是 RedrawRequested，队列空 → return（跳过批处理）
    └─ is_redraw=true → window_context.draw()
⑫ WindowContext::draw() [window_context.rs#L366]
    ├─ requested_redraw = false
    ├─ dirty = false
    ├─ process_renderer_update()（这次没 GL 更新）
    └─ self.display.draw(terminal, scheduler, ...)
⑬ Display::draw() [display/mod.rs#L775]
    ├─ 收集 RenderableContent（包含新输出的 "a"）
    ├─ draw_cells() 把文字变成 GPU 绘制指令
    ├─ swap_buffers() 上屏
    ├─ request_frame(scheduler) → 排下一个 Frame 定时器
    └─ damage_tracker.swap_damage()
⑭ 用户看到屏幕上的 "a"
```
