# Winit 窗口与事件链调用顺序详解

本文档沿着实际代码追踪 winit 事件链的每一步，重点讲清：
1. **显示更新与实际绘制的职责边界**（哪层只记状态、哪层计算、哪层动 GL）
2. **等待回调如何设置唤醒时机**（about_to_wait → ControlFlow::WaitUntil）
3. **Frame 事件怎样回到窗口重绘**（调度器 → user_event → request_redraw → RedrawRequested）

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

## 二、显示更新与实际绘制的职责边界（三层结构）

这是最容易读错的部分。系统用 **三层缓冲** 把"记状态"、"算布局"、"动 GL" 严格分开。

### 第一层：只记录待处理状态（DisplayUpdate）

**存储**：`display.pending_update`，类型 `DisplayUpdate`

**位置**：[display/mod.rs#L304-L339](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L304-L339)

```rust
pub struct DisplayUpdate {
    pub dirty: bool,              // 总开关：是否有待处理更新
    dimensions: Option<PhysicalSize<u32>>,  // 窗口新尺寸
    font: Option<Font>,           // 新字体配置
    cursor_dirty: bool,           // 光标样式是否变化
}
```

**三个 setter**（全是纯设值，零计算）：

```rust
pub fn set_dimensions(&mut self, dimensions: PhysicalSize<u32>) {
    self.dimensions = Some(dimensions);
    self.dirty = true;             // 同时打开总开关
}

pub fn set_font(&mut self, font: Font) {
    self.font = Some(font);
    self.dirty = true;
}

pub fn set_cursor_dirty(&mut self) {
    self.cursor_dirty = true;
    self.dirty = true;
}
```

**设置位置分布**：

| 设置点 | 代码位置 | 设置什么 |
|--------|----------|----------|
| 配置热重载（字体/光标厚度变化） | [window_context.rs#L272-L283](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/window_context.rs#L272-L283) | set_cursor_dirty / set_font |
| 配置热重载（padding/dynamic_padding 变化） | [window_context.rs#L295](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/window_context.rs#L295) | dirty = true |
| 窗口 Resized 事件 | input::Processor 的 Resized 分支 | set_dimensions |
| 缩放字体（IncreaseFontSize 等） | [event.rs#L926](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/event.rs#L926) | set_font |
| 弹出/关闭消息栏 | [event.rs#L940](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/event.rs#L940) | dirty = true |
| 进入/退出搜索模式 | [event.rs#L980](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/event.rs#L980) | dirty = true |
| 进入/退出 Vi 模式 | [event.rs#L1613](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/event.rs#L1613) | dirty = true |
| 收到 Message 事件 | [event.rs#L1865](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/event.rs#L1865) | dirty = true |
| 配置热重载（移除消息后） | [event.rs#L348](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/event.rs#L348) | dirty = true |

**关键特征**：
- ✅ 只改 `pending_update` 内部字段
- ✅ 不改 `size_info`、不改 terminal、不改 GL
- ✅ 不做任何计算（甚至连新的 cell 尺寸都不算）
- ✅ 调用后唯一副作用：`dirty = true`

---

### 第二层：应用显示更新（handle_update）—— 算布局、排 GL、但不动 GL

**触发点**：[window_context.rs#L461-L473](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/window_context.rs#L461-L473)

批量事件处理完后，如果 `pending_update.dirty` 为 true，就调用 `submit_display_update()` → `display.handle_update()`。

**核心函数**：`Display::handle_update()`

**位置**：[display/mod.rs#L651-L737](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L651-L737)

```rust
pub fn handle_update<T>(
    &mut self,
    terminal: &mut Term<T>,
    pty_resize_handle: &mut dyn OnResize,
    message_buffer: &MessageBuffer,
    search_state: &mut SearchState,
    config: &UiConfig,
) {
    // 取出 pending_update 并清空（消费）
    let pending_update = mem::take(&mut self.pending_update);

    // ① 先检查：字体/光标变化 → 排队 GL 字体缓存清理
    if pending_update.font().is_some() || pending_update.cursor_dirty() {
        let renderer_update = self.pending_renderer_update.get_or_insert(Default::default());
        renderer_update.clear_font_cache = true;  // ← 只排队，不执行
    }

    // ② 字体变化 → 更新 glyph_cache 字体大小 + 算新 cell 尺寸
    //    注意：update_font_size 只改 CPU 侧的字体元数据，不改 GL 纹理
    if let Some(font) = pending_update.font() {
        let cell_dimensions = Self::update_font_size(&mut self.glyph_cache, config, font);
        // cell_width, cell_height = ...
    }

    // ③ 尺寸变化 → 读新宽高
    if let Some(dimensions) = pending_update.dimensions() {
        // width, height = ...
    }

    // ④ 算 padding → 构建新 SizeInfo
    let padding = config.window.padding(self.window.scale_factor as f32);
    let mut new_size = SizeInfo::new(width, height, cell_width, cell_height, ...);

    // ⑤ 减去消息栏/搜索栏占的行
    new_size.reserve_lines(message_bar_lines + search_lines);

    // ⑥ 设窗口 resize 增量（纯窗口系统调用，非 GL）
    if config.window.resize_increments {
        self.window.set_resize_increments(PhysicalSize::new(cell_width, cell_height));
    }

    // ⑦ 行列数变化 → resize PTY + resize Terminal grid
    if self.size_info.screen_lines() != new_size.screen_lines()
        || self.size_info.columns() != new_size.columns()
    {
        pty_resize_handle.on_resize(new_size.into());  // ← 通知 PTY 改尺寸
        terminal.resize(new_size);                     // ← 改终端网格大小
        self.damage_tracker.resize(...);               // ← 改 damage 跟踪器大小
    }

    // ⑧ 尺寸有变化 → 排队 GL resize
    if new_size != self.size_info {
        let renderer_update = self.pending_renderer_update.get_or_insert(Default::default());
        renderer_update.resize = true;                 // ← 只排队，不执行
    }

    // ⑨ 更新 size_info（这是本函数最核心的产出）
    self.size_info = new_size;
}
```

#### 本层产出物

| 产出 | 类型 | 去哪了 |
|------|------|--------|
| 新的 `self.size_info` | SizeInfo | 直接写入 Display，后续绘制用 |
| `terminal.resize()` | 终端网格 | 直接改 terminal |
| `pty.on_resize()` | PTY 尺寸 | 通知子进程窗口大小变了 |
| `pending_renderer_update` | Option<RendererUpdate> | 排队到下一层，不立即执行 |

#### 本层**绝对不做**的事

- ❌ 不调用任何 GL 函数（`gl*`、`egl*`）
- ❌ 不 `make_current()`（不激活 GL 上下文）
- ❌ 不 `surface.resize()`（不动 GL 绘制表面）
- ❌ 不清理字体纹理（`reset_glyph_cache` 要到 GL 层才做）

> **为什么要分这一层？** 代码注释 [display/mod.rs#L739-L741](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L739-L741) 说得很清楚：
>
> Wayland 等平台要求 resize 和其他 GL 操作必须在**渲染前一刻**执行。否则会锁定 back buffer，导致用旧状态渲染，还会产生 resize 闪烁。
>
> 所以 layout 计算（handle_update）和 GL 操作（process_renderer_update）必须分开。

---

### 第三层：GL 更新 + 实际绘制（含状态边界详析）

这一层是整个渲染管线的终点，也是状态最复杂的一层。它又分两步，都在 `WindowContext::draw()` 里按顺序执行。

关键认知：**draw() 不只是"收集内容和上屏"，它是渲染记录的"消费+更新"中心**——上一帧累积的 damage、光标位置、提示高亮等渲染状态，都在这一帧被消费并轮换。

---

#### Step 3.0：WindowContext::draw() 入口状态清理

**函数**：[window_context.rs#L366-L398](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/window_context.rs#L366-L398)

进入 Display::draw() 之前，先做 4 件事：

```rust
pub fn draw(&mut self, scheduler: &mut Scheduler) {
    // ① 清请求重绘标志（去重用）
    self.display.window.requested_redraw = false;

    // ② 窗口被遮挡 → 直接返回，不画
    if self.occluded {
        return;
    }

    // ③ 清 dirty 标志（本帧就是来画脏内容的）
    self.dirty = false;

    // ④ 先执行排队的 GL 更新（第二步 handle_update 排进来的）
    self.display.process_renderer_update();

    // ⑤ 视觉铃动画未结束 → 再请求一帧（连续动画）
    if !self.display.visual_bell.completed() {
        if self.display.window.has_frame {
            self.display.window.request_redraw();
        } else {
            self.dirty = true;
        }
    }

    // ⑥ 拿终端锁 → 调用 Display::draw()
    let terminal = self.terminal.lock();
    self.display.draw(terminal, scheduler, &self.message_buffer, &self.config, &mut self.search_state);
}
```

**注意第 ⑤ 步**：`visual_bell.completed()` 不只是读取，它有副作用——如果动画已经完成，会把 `start_time` 重置为 `None`（[bell.rs#L31-L41](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/bell.rs#L31-L41)）。

---

#### Step 3.1：process_renderer_update——执行排队的 GL 更新

**入口**：[window_context.rs#L376](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/window_context.rs#L376)

**函数**：[display/mod.rs#L744-L768](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L744-L768)

```rust
pub fn process_renderer_update(&mut self) {
    let renderer_update = match self.pending_renderer_update.take() {
        Some(u) => u,
        _ => return,   // 没待处理的就直接返回
    };

    // ① Surface resize（需要 GL context，因为 surface 是 GL 绘制表面）
    if renderer_update.resize {
        self.surface.resize(&self.context, width, height);
    }

    // ② 确保当前窗口的 GL context 是激活的
    self.make_current();

    // ③ 清字体缓存（涉及 GL 纹理删除，必须 context current）
    if renderer_update.clear_font_cache {
        self.reset_glyph_cache();  // 内部用 renderer.with_loader() 调用 GL
    }

    // ④ Renderer resize（更新 uniform、视口矩阵等 GL 状态）
    self.renderer.resize(&self.size_info);
}
```

**调用时机**：在 `WindowContext::draw()` 的**最开头**，在收集渲染内容之前。

**被消费的状态**：`pending_renderer_update` 被 `take()` 消费掉，变为 `None`。

**被更新的状态**：
- `surface`（GL 绘制表面尺寸）
- `glyph_cache`（字体纹理缓存，若 clear_font_cache）
- `renderer`（GL uniform / 视口）

---

#### Step 3.2：Display::draw()——渲染状态的消费与轮换

**入口**：[window_context.rs#L391](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/window_context.rs#L391)

**函数**：[display/mod.rs#L775-L1047](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L775-L1047)

`Display::draw()` 内部按顺序做 4 大类事，每一类都涉及特定渲染状态的消费或更新。下面逐项拆解：

---

##### 阶段 A：收集内容 + 消费终端脏区

**代码位置**：[display/mod.rs#L783-L815](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L783-L815)

```rust
// ① 收集 RenderableContent（终端网格 → 可渲染单元迭代器）
let mut content = RenderableContent::new(config, self, &terminal, search_state);
let mut grid_cells = Vec::new();
for cell in &mut content {
    grid_cells.push(cell);
}
let selection_range = content.selection_range();
// ...

// ② 【消费】终端 damage → 合并到帧 damage
match terminal.damage() {
    TermDamage::Full => self.damage_tracker.frame().mark_fully_damaged(),
    TermDamage::Partial(damaged_lines) => {
        for damage in damaged_lines {
            self.damage_tracker.frame().damage_line(damage);
        }
    },
}

// ③ 【消费】终端 damage 标志 → 清空
terminal.reset_damage();

// ④ 尽早释放终端锁！（减少 I/O 线程阻塞时间）
drop(terminal);
```

**状态流转**：

| 状态 | 操作 | 位置 |
|------|------|------|
| `terminal.damage` | **消费**：读取 Full/Partial，逐行合并到 damage_tracker | [display/mod.rs#L804-L811](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L804-L811) |
| `terminal.damage` | **清空**：`reset_damage()` 重置，下一帧从零开始累积 | [display/mod.rs#L812](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L812) |
| `damage_tracker.frame()` | **更新**：累加终端 damage | 同上 |
| `terminal` 锁 | **释放**：收集完就 drop，不再持有 | [display/mod.rs#L815](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L815) |

---

##### 阶段 B：校验提示高亮 + 计算本帧 damage

**代码位置**：[display/mod.rs#L817-L833](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L817-L833)

```rust
// ① 【校验/可能清空】提示高亮（如果所在区域是脏的）
self.validate_hint_highlights(display_offset);

// ② UI 元素（视觉铃 / 提示 / 搜索栏）强制全 damage
let requires_full_damage = self.visual_bell.intensity() != 0.
    || self.hint_state.active()
    || search_state.regex().is_some();
if requires_full_damage {
    self.damage_tracker.frame().mark_fully_damaged();
    self.damage_tracker.next_frame().mark_fully_damaged();
}

// ③ 【消费+更新】Vi 光标位置 → 计算 damage（新旧位置都标记）
self.damage_tracker.damage_vi_cursor(vi_cursor_viewport_point);

// ④ 【消费+更新】选择范围 → 计算 damage（新旧范围都标记）
self.damage_tracker.damage_selection(selection_range, display_offset);
```

**重点 1：validate_hint_highlights 做了什么**

**函数**：[display/mod.rs#L1395-L1432](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L1395-L1432)

这是最容易被忽略的状态更新。它检查高亮的提示（鼠标悬停 / Vi 光标）是否落在了脏区里，如果是就清除掉：

```rust
fn validate_hint_highlights(&mut self, display_offset: usize) {
    let frame = self.damage_tracker.frame();
    let hints = [
        (&mut self.highlighted_hint, &mut self.highlighted_hint_age, true),     // 鼠标高亮
        (&mut self.vi_highlighted_hint, &mut self.vi_highlighted_hint_age, false), // Vi光标高亮
    ];

    for (hint, hint_age, reset_mouse) in hints {
        // 跳过本帧刚创建的高亮（age == 1）
        *hint_age += 1;
        if *hint_age == 1 { continue; }

        // 高亮所在区域与本帧 damage 相交 → 清除高亮 + 全脏
        if frame.intersects(start, end) {
            if reset_mouse {
                self.window.set_mouse_cursor(CursorIcon::Default);
            }
            frame.mark_fully_damaged();
            *hint = None;
        }
    }
}
```

**为什么要这么做？** 终端内容变化后，原位置的字符/链接可能已经不存在了，高亮就无效了。通过与 damage 做相交检测，可以在内容变化时自动清除失效的高亮。

**状态流转**：

| 状态 | 操作 | 条件 |
|------|------|------|
| `highlighted_hint` | **可能被设为 None** | 所在区域是脏的且 age > 1 |
| `vi_highlighted_hint` | **可能被设为 None** | 所在区域是脏的且 age > 1 |
| `highlighted_hint_age` | **递增 1** | 每次 draw 都加 |
| `vi_highlighted_hint_age` | **递增 1** | 每次 draw 都加 |
| `damage_tracker.frame()` | **可能被标全脏** | 清除高亮时 |

**重点 2：damage_vi_cursor 的"新旧交换"模式**

**函数**：[damage.rs#L76-L90](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/damage.rs#L76-L90)

```rust
pub fn damage_vi_cursor(&mut self, mut vi_cursor: Option<Point<usize>>) {
    // 把新位置存到参数里，旧位置从 old_vi_cursor 拿出来
    mem::swap(&mut self.old_vi_cursor, &mut vi_cursor);

    // 旧位置 → 标记 damage（光标移走了，原来的位置需要重绘）
    if let Some(vi_cursor) = self.old_vi_cursor {
        self.frame().damage_point(vi_cursor);
    }

    // 新位置 → 标记 damage（光标到了新位置，需要重绘）
    if let Some(vi_cursor) = vi_cursor {
        self.frame().damage_point(vi_cursor);
    }
}
```

**设计模式**：DamageTracker 保存**上一帧**的光标/选择位置，每帧和新位置交换，然后把新旧位置都标脏。这样光标移动时原来的位置和新位置都会被重绘。

`damage_selection` 用的是完全相同的模式（[damage.rs#L106-L135](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/damage.rs#L106-L135)）。

**状态流转**：

| 状态 | 操作 |
|------|------|
| `damage_tracker.old_vi_cursor` | **与输入值交换**（本帧位置 → old，old → 被消费） |
| `damage_tracker.old_selection` | **与输入值交换** |
| `damage_tracker.frame()` | **更新**：累加新旧光标/选择的 damage |

---

##### 阶段 C：GL 绘制

**代码位置**：[display/mod.rs#L835-L1028](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L835-L1028)

```rust
// ① 激活本窗口的 GL context
self.make_current();

// ② 清屏
self.renderer.clear(background_color, config.window_opacity());

// ③ 画文字网格（含提示高亮下划线）
self.renderer.draw_cells(&size_info, glyph_cache, cells);

// ④ 画矩形（光标 / 下划线 / 删除线 / 视觉铃 / 消息栏背景）
self.renderer.draw_rects(&size_info, &metrics, rects);

// ⑤ 画文字（消息栏文字 / 搜索栏 / 行号指示器）
self.renderer.draw_string(point, fg, bg, text.chars(), ...);

// ⑥ （调试模式）画 damage 矩形高亮
if self.damage_tracker.debug {
    // ...
    self.renderer.draw_rects(&self.size_info, &metrics, rects);
}
```

这一阶段**只消费，不更新业务状态**。所有 GL 调用都是向 GPU 发送绘制指令，不改变 CPU 侧的渲染记录。

---

##### 阶段 D：上屏 + 帧调度 + damage 轮换

**代码位置**：[display/mod.rs#L1030-L1047](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L1030-L1047)

```rust
// ① macOS 预备 present 通知
self.window.pre_present_notify();

// ② 【上屏】swap_buffers —— 前后缓冲交换，新帧真正显示出来
self.swap_buffers();

// ③ X11: renderer.finish() 消除一帧延迟
//    （X11 swap_buffers 不阻塞，下一条 GL 命令才阻塞）

// ④ 【调度】非 Wayland：排下一帧 Frame 事件
if !matches!(self.raw_window_handle, RawWindowHandle::Wayland(_)) {
    self.request_frame(scheduler);
}

// ⑤ 【轮换】damage 缓冲区 —— 下一帧从零开始累积 damage
self.damage_tracker.swap_damage();
```

**重点：swap_damage 的双缓冲机制**

**函数**：[damage.rs#L58-L63](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/damage.rs#L58-L63)

```rust
pub fn swap_damage(&mut self) {
    let screen_lines = self.screen_lines;
    let columns = self.columns;
    self.frame().reset(screen_lines, columns);  // 重置当前帧（下标0）
    self.frames.swap(0, 1);                      // 交换 0 和 1
}
```

**双缓冲设计**：
- `frames[0]` = 当前帧（本帧正在累积 damage 的那个）
- `frames[1]` = 下一帧（备用）
- `frame()` 始终返回 `&mut frames[0]`
- `next_frame()` 返回 `&mut frames[1]`

**swap_damage 做了什么**：
1. 先重置 `frames[0]`（清成全干净状态）
2. 交换下标 0 和 1 → 原来的 frames[1] 变成新的当前帧，原来的 frames[0] 变成备用

**为什么要双缓冲？** 因为有些 UI 元素（搜索栏、视觉铃）需要**下一帧也全脏**（[display/mod.rs#L826-L827](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L826-L827)），它们通过 `next_frame()` 提前给下一帧标脏。

**状态流转**：

| 状态 | 操作 |
|------|------|
| `window.has_frame` | **设为 false**（在 request_frame 内部） |
| `damage_tracker.frames` | **双缓冲轮换**：当前帧重置，前后台交换 |
| `scheduler` | **新增** Frame 定时器（非 Wayland） |

---

#### 绘制阶段所有状态变化总表

把整个第三层所有被消费/更新的状态汇总如下：

| 状态 | 操作类型 | 阶段 | 代码位置 |
|------|----------|------|----------|
| `requested_redraw` | 清零 | 入口 | [window_context.rs#L367](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/window_context.rs#L367) |
| `dirty` | 清零 | 入口 | [window_context.rs#L373](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/window_context.rs#L373) |
| `pending_renderer_update` | 消费（take） | Step 3.1 | [display/mod.rs#L745](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L745) |
| `surface` | 更新（resize） | Step 3.1 | [display/mod.rs#L753](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L753) |
| `glyph_cache` | 重置（GL纹理） | Step 3.1 | [display/mod.rs#L761](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L761) |
| `renderer` 视口 | 更新 | Step 3.1 | [display/mod.rs#L764](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L764) |
| `visual_bell.start_time` | 可能清零 | 入口（completed()） | [bell.rs#L31-L41](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/bell.rs#L31-L41) |
| `terminal.damage` | 消费 + 清零 | 阶段A | [display/mod.rs#L804-L812](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L804-L812) |
| `terminal` 锁 | 释放 | 阶段A | [display/mod.rs#L815](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L815) |
| `highlighted_hint` | 可能清空 | 阶段B | [display/mod.rs#L1429](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L1429) |
| `vi_highlighted_hint` | 可能清空 | 阶段B | 同上 |
| `highlighted_hint_age` | 递增 | 阶段B | [display/mod.rs#L1410](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L1410) |
| `vi_highlighted_hint_age` | 递增 | 阶段B | 同上 |
| `old_vi_cursor` | 新旧交换 | 阶段B | [damage.rs#L77](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/damage.rs#L77) |
| `old_selection` | 新旧交换 | 阶段B | [damage.rs#L111](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/damage.rs#L111) |
| `damage_tracker.frame()` | 累加 damage | 阶段A+B+C | 多处 |
| `has_frame` | 设 false | 阶段D | [display/mod.rs#L1437](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L1437) |
| `damage_tracker.frames` | 双缓冲轮换 | 阶段D | [damage.rs#L61-L62](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/damage.rs#L61-L62) |
| Scheduler | 新增 Frame 定时器 | 阶段D | [display/mod.rs#L1457](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/display/mod.rs#L1457) |

**不改的状态（只读）**：`terminal.grid()`（内容读取）、`size_info`、`config`、`colors`、`glyph_cache`（绘制时只读，重置除外）、`metrics`。

---

#### draw() 的真实职责

draw() 不是"只负责收集内容和上屏"，它是**渲染状态的一帧结算点**。可以用"三消费两更新一排"来概括：

- **三消费**：消费终端 dirty、消费终端 damage、消费 pending_renderer_update
- **两更新**：更新 damage tracker（含光标/选择新旧交换）、更新提示高亮校验（可能清除）
- **一排**：排下一帧（request_frame + swap_damage）

但有一条清晰的红线：**draw() 不改业务状态**（terminal 内容、size_info、config 都是只读或加锁读）。它只消费和更新**渲染相关**的状态，产出新的一帧图像。

---

### 三层结构总览

```
┌─────────────────────────────────────────────────────┐
│  第一层：DisplayUpdate (pending_update)              │
│  只记状态：dimensions / font / cursor_dirty / dirty  │
│  零计算、零副作用、不动 GL                           │
└────────────────────────┬────────────────────────────┘
                         │ pending_update.dirty == true
                         ▼
┌─────────────────────────────────────────────────────┐
│  第二层：handle_update()                             │
│  ✅ 算：cell 尺寸 / SizeInfo / padding / 行列数       │
│  ✅ 改：terminal.resize / pty.on_resize              │
│  ✅ 排：pending_renderer_update（GL 操作队列）        │
│  ❌ 不动 GL / 不 make_current / 不 resize surface     │
└────────────────────────┬────────────────────────────┘
                         │ RedrawRequested 触发 draw()
                         ▼
┌─────────────────────────────────────────────────────┐
│  第三层：process_renderer_update() + draw()          │
│  3.1 process_renderer_update                         │
│    - surface.resize / make_current / 清字体纹理       │
│    - renderer.resize（uniform/视口）                 │
│  3.2 draw()                                          │
│    - 收集内容 + 合并 damage                          │
│    - GL 绘制（cells/rects/strings）                  │
│    - swap_buffers 上屏                               │
│    - 调度下一帧（request_frame）                     │
└─────────────────────────────────────────────────────┘
```

---

## 三、等待回调设置唤醒时机（Scheduler + about_to_wait）

### 3.1 触发链

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

**注意顺序**：先排空窗口的事件队列（可能产生新的定时器），再更新调度器，保证 deadline 准确。

### 3.2 Scheduler::update() 内部机制

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

### 3.3 定时器入队规则

**[scheduler.rs#L76-L90](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/scheduler.rs#L76-L90)**

`schedule()` 按 deadline 升序插入 `VecDeque`，保证队首永远是最早到期的。

**关键 Topic 列表**（[scheduler.rs#L26-L32](file:///d:/fz/0601/solo-dogfeeding/code/337-alacritty/alacritty/src/scheduler.rs#L26-L32)）：

| Topic | 周期 | 生产者 | 事件内容 |
|-------|------|--------|----------|
| Frame | 单次 | `Display::request_frame()` | 对齐到 vblank 的下一个绘制槽 |
| BlinkCursor | 周期 | 光标闪烁逻辑 | 触发光标切换可见性 |
| BlinkTimeout | 单次 | 键盘/鼠标活动后重置 | 光标停止闪烁进入省电 |
| SelectionScrolling | 周期 | 鼠标在窗口外选择时 | 每 15ms 滚动终端 |
| DelayedSearch | 单次 | 搜索输入延迟 | 延迟搜索触发 |

### 3.4 唤醒时机示例

假设显示器 60Hz（16.667ms/帧），刚 swap_buffers 后调用 `request_frame()`：
1. `FrameTimer::compute_timeout()` 算出距下一个 vblank 相位点还有 12ms
2. `scheduler.schedule(Frame事件, 12ms, false, Frame+WindowId)`
3. 事件队列耗尽触发 `about_to_wait()`
4. `scheduler.update()` 发现队首 Frame 定时器还有 10ms 才到，返回该 Instant
5. `event_loop.set_control_flow(WaitUntil(该Instant))`
6. winit 阻塞到该时刻，超时后自动唤醒事件循环

---

## 四、Frame 事件回到窗口重绘的完整路径

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

### 5.4 两层 pending 标志对比

| 标志 | 层级 | 设置者 | 消费者 | 设置后做了什么 |
|------|------|--------|--------|----------------|
| pending_update.dirty | 1→2 | input 处理器 / config / window 事件 | handle_update() | 只设标志，什么都不做 |
| pending_renderer_update | 2→3 | handle_update() | process_renderer_update() | 只排队 GL 操作，不执行 |

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
   └─ self.event_queue.push(event); return;   ← 入第一层：仅记录
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
    ├─ process_renderer_update()（这次没 GL 更新，直接返回）
    └─ self.display.draw(terminal, scheduler, ...)
⑬ Display::draw() [display/mod.rs#L775]
    ├─ 收集 RenderableContent（包含新输出的 "a"）
    ├─ 合并 damage
    ├─ make_current()
    ├─ draw_cells() 把文字变成 GPU 绘制指令
    ├─ swap_buffers() 上屏
    ├─ request_frame(scheduler) → 排下一个 Frame 定时器
    └─ damage_tracker.swap_damage()
⑭ 用户看到屏幕上的 "a"
```
