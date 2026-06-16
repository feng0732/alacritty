# Alacritty 滚动缓冲（Scrollback Buffer）实现分析

## 一、整体架构概览

Alacritty 的滚动缓冲实现分布在三个层次，职责分明：

```
┌─────────────────────────────────────────────┐
│  UI 层 (alacritty crate)                     │  用户滚动输入 → Grid::scroll_display
├─────────────────────────────────────────────┤
│  Terminal 层 (alacritty_terminal/term)       │  PTY 输出解析 → scroll_up/scroll_down
├─────────────────────────────────────────────┤
│  Grid 层 (alacritty_terminal/grid)           │  行存储与移动 → Storage 环形缓冲区
└─────────────────────────────────────────────┘
```

核心数据结构关系：
- [Grid](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/mod.rs#L110-L138) 持有 `Storage<T>`，是终端内容的顶层容器
- [Storage](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/storage.rs#L33-L53) 是环形缓冲区，底层用 `Vec<Row<T>>` 存储行数据
- [Row](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/row.rs#L17-L25) 是单行数据，持有 `Vec<T>` (T=Cell) 和 `occ` 脏标记

---

## 二、入口：滚动缓冲的创建与配置

### 2.1 配置入口

配置文件通过 [scrolling.rs](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty/src/config/scrolling.rs#L1-L53) 定义 `Scrolling` 结构体，核心字段是 `history: ScrollingHistory(u32)`，默认值 10000 行，上限 100000 行（`MAX_SCROLLBACK_LINES`）。

### 2.2 Grid 初始化

[Grid::new](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/mod.rs#L141-L151) 接收 `max_scroll_limit` 参数，但 **scrollback 不预分配**：

```rust
pub fn new(lines: usize, columns: usize, max_scroll_limit: usize) -> Grid<T> {
    Grid {
        raw: Storage::with_capacity(lines, columns),  // 只分配可见行
        max_scroll_limit,
        display_offset: 0,
        // ...
    }
}
```

[Storage::with_capacity](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/storage.rs#L67-L76) 只初始化 `visible_lines` 数量的行，scrollback 区域在运行时按需动态增长。

### 2.3 Terminal 初始化

[Term::new](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/term/mod.rs#L410-L445) 创建两个 Grid：
- **主 Grid**：`Grid::new(num_lines, num_cols, history_size)` — 带 scrollback
- **备选 Grid**：`Grid::new(num_lines, num_cols, 0)` — 无 scrollback（alt screen buffer）

`scroll_region` 初始化为 `Line(0)..Line(screen_lines)`，即整个视口。

---

## 三、核心协作：Scrollback 数据如何流转

### 3.1 数据写入触发滚动（PTY 输出路径）

这是 scrollback 产生的根本原因：终端输出文本导致行向上滚动，旧行进入历史。

**触发链路：**

```
PTY 读取字节 → vte::ansi::Parser 解析 → Term Handler 方法
```

1. [EventLoop::pty_read](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/event_loop.rs#L103-L171) 从 PTY 读取原始字节
2. 调用 `state.parser.advance(&mut **terminal, &buf[..unprocessed])` 将字节送入 VTE 解析器
3. 解析器调用 Term 实现的 Handler trait 方法

**关键触发点 — linefeed 换行：**

[Term::linefeed](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/term/mod.rs#L1423-L1433)：

```rust
fn linefeed(&mut self) {
    let next = self.grid.cursor.point.line + 1;
    if next == self.scroll_region.end {
        self.scroll_up(1);      // ← 光标在滚动区底部，触发向上滚动
    } else if next < self.screen_lines() {
        self.grid.cursor.point.line += 1;
    }
}
```

**另一个触发点 — wrapline 自动换行：**

[Term::wrapline](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/term/mod.rs#L960-L980)：当光标在行末且需要换行时，如果光标在滚动区底部，同样调用 `linefeed()` → `scroll_up(1)`。

### 3.2 scroll_up — 行进入 scrollback 的核心

[Term::scroll_up](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/term/mod.rs#L1485-L1488) 直接委托给 [Term::scroll_up_relative](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/term/mod.rs#L770-L790)：

```rust
fn scroll_up_relative(&mut self, origin: Line, mut lines: usize) {
    lines = cmp::min(lines, (self.scroll_region.end - self.scroll_region.start).0 as usize);
    let region = origin..self.scroll_region.end;

    // 1. 同步滚动选区
    self.selection = self.selection.take().and_then(|s| s.rotate(self, &region, lines as i32));

    // 2. 委托 Grid 执行行移动
    self.grid.scroll_up(&region, lines);

    // 3. 同步 vi_mode 光标
    // ...
    self.mark_fully_damaged();
}
```

**Grid::scroll_up 做了什么：**

[Grid::scroll_up](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/mod.rs#L252-L307) 是性能关键路径，分两种策略：

**策略 A — region.start == 0（全屏滚动，最常见场景）：**

```rust
// 1. 扩展 scrollback 空间
self.increase_scroll_limit(positions);

// 2. 将滚动区上方的固定行（region.start 之前的行）swap 到目标位置
for i in (0..region.start.0).rev().map(Line::from) {
    self.raw.swap(i, i + positions);
}

// 3. 旋转整个环形缓冲区（O(1) 操作，只改 zero 指针）
self.raw.rotate(-(positions as isize));

// 4. 将底部固定行 swap 回来
let screen_lines = self.screen_lines() as i32;
for i in (region.end.0..screen_lines).rev().map(Line::from) {
    self.raw.swap(i, i - positions);
}
```

**策略 B — 子区域滚动（scroll_region 不从顶部开始）：**
不涉及 scrollback，只在可见区域内用 swap 移动行：

```rust
for i in (region.start.0..region.end.0 - positions as i32).map(Line::from) {
    self.raw.swap(i, i + positions);
}
```

两种策略最后都重置被腾出的行：

```rust
for i in (region.end.0 - positions as i32..region.end.0).map(Line::from) {
    self.raw[i].reset(&self.cursor.template);
}
```

### 3.3 increase_scroll_limit — scrollback 动态增长

[Grid::increase_scroll_limit](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/mod.rs#L175-L180)：

```rust
fn increase_scroll_limit(&mut self, count: usize) {
    let count = min(count, self.max_scroll_limit - self.history_size());
    if count != 0 {
        self.raw.initialize(count, self.columns);
    }
}
```

[Storage::initialize](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/storage.rs#L126-L138) 负责**按需扩展**底层 Vec：

```rust
pub fn initialize(&mut self, additional_rows: usize, columns: usize) {
    if self.len + additional_rows > self.inner.len() {
        self.rezero();                    // 先将 zero 归零（旋转到物理起点）
        let realloc_size = self.inner.len() + max(additional_rows, MAX_CACHE_SIZE);
        self.inner.resize_with(realloc_size, || Row::new(columns));
    }
    self.len += additional_rows;          // 逻辑长度增长
}
```

关键点：`MAX_CACHE_SIZE = 1000`，每次分配至少多 1000 行的缓冲空间，避免频繁重新分配。`len` 是逻辑长度，`inner.len()` 是物理容量，两者可以不同——`len <= inner.len()`。

### 3.4 Storage 环形缓冲区索引机制

[Storage](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/storage.rs#L33-L53) 的核心设计：

- `inner: Vec<Row<T>>` — 物理存储
- `zero: usize` — 环形缓冲区的"逻辑起点"偏移，代表终端底部行在物理数组中的位置
- `len: usize` — 逻辑有效行数（scrollback + 可见行）
- `visible_lines: usize` — 可见行数

**索引映射 [compute_index](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/storage.rs#L220-L234)：**

```rust
fn compute_index(&self, requested: Line) -> usize {
    let positive = -(requested - self.visible_lines).0 as usize - 1;
    let zeroed = self.zero + positive;
    if zeroed >= self.inner.len() { zeroed - self.inner.len() } else { zeroed }
}
```

`Line` 是有符号 i32：`Line(0)` = 可见区最底行，`Line(-(history_size))` = scrollback 最顶行。通过 `zero` 偏移 + 取模实现 O(1) 的旋转操作。

**旋转操作：**
- [rotate](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/storage.rs#L180-L185)：`self.zero = (self.zero as isize + count + len as isize) as usize % len` — O(1)
- [rotate_down](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/storage.rs#L193-L195)：`self.zero = (self.zero + count) % self.inner.len()` — O(1)

**rezero [归零](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/storage.rs#L238-L245)：** 当需要扩展 Vec 或截断时，先调用 `inner.rotate_left(self.zero)` 把逻辑起始移到物理位置 0，然后 `self.zero = 0`。

### 3.5 display_offset — 视口偏移的阅读侧

[Grid::display_offset](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/mod.rs#L134) 表示用户向上滚动了多少行。0 表示视口在底部（最新内容），正值表示用户在查看历史。

**用户滚动触发：** 通过 [Grid::scroll_display](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/mod.rs#L163-L173)：

```rust
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

`display_offset` 始终被 clamp 在 `[0, history_size()]` 范围内。

**UI 层调用入口：** [input/mod.rs](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty/src/input/mod.rs#L269) 中鼠标滚轮、键盘绑定等触发 `ctx.scroll(Scroll::Delta(...))`，最终调用 `terminal.scroll_display(scroll)`。

**scroll_up 中的 display_offset 联动：** 当用户正在查看历史时（`display_offset != 0`），新输出会让视口"钉住"：

```rust
// Grid::scroll_up 中
if self.display_offset != 0 {
    self.display_offset = min(self.display_offset + positions, self.max_scroll_limit);
}
```

这确保用户查看历史时，新内容不会把视口拉回底部。

### 3.6 scroll_down — 向下滚动的反向操作

[Grid::scroll_down](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/mod.rs#L191-L247) 同样分两种策略：

**无 scrollback 时（max_scroll_limit == 0）：** 先 swap 底部固定行、再 rotate_down、清空新行、swap 顶部固定行回来。

**有 scrollback 时：** 子区域旋转，只用 swap 逐行移动，不触及历史区域。

### 3.7 resize 时的 scrollback 处理

[Grid::resize](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/resize.rs#L14-L36) 按行/列分别处理。

**增长行数 [grow_lines](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/resize.rs#L43-L69)：** 窗口变高时，从 scrollback 中"拉回"行填充新增的可见区域：

```rust
fn grow_lines<D>(&mut self, target: usize) {
    let lines_added = target - self.lines;
    self.raw.grow_visible_lines(target);    // 扩展 Storage
    self.lines = target;
    let history_size = self.history_size();
    let from_history = min(history_size, lines_added);
    if from_history != lines_added {
        let delta = lines_added - from_history;
        self.scroll_up(&(Line(0)..Line(target as i32)), delta);
    }
    self.cursor.point.line += from_history;
    self.display_offset = self.display_offset.saturating_sub(lines_added);
    self.decrease_scroll_limit(lines_added);   // scrollback 缩小
}
```

**缩减行数 [shrink_lines](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/resize.rs#L78-L98)：** 窗口变矮时，底部内容被推入 scrollback：

```rust
fn shrink_lines<D>(&mut self, target: usize) {
    let required_scrolling = (self.cursor.point.line.0 as usize + 1).saturating_sub(target);
    if required_scrolling > 0 {
        self.scroll_up(&(Line(0)..Line(self.lines as i32)), required_scrolling);
        self.cursor.point.line = min(self.cursor.point.line, Line(target as i32 - 1));
    }
    self.raw.rotate((self.lines - target) as isize);
    self.raw.shrink_visible_lines(target);
    self.lines = target;
}
```

---

## 四、收尾动作：Scrollback 的收缩与清理

### 4.1 shrink_lines — 逻辑收缩（延迟释放）

[Storage::shrink_lines](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/storage.rs#L107-L114)：

```rust
pub fn shrink_lines(&mut self, shrinkage: usize) {
    self.len -= shrinkage;    // 只减逻辑长度
    if self.inner.len() > self.len + MAX_CACHE_SIZE {
        self.truncate();      // 缓存超限才真正释放
    }
}
```

关键设计：**延迟释放**。减小 scrollback 不立即释放内存，只要 `inner.len() - len <= MAX_CACHE_SIZE`（1000行），物理内存保持不变。这避免了反复增减 scrollback 时的频繁分配/释放。

### 4.2 truncate — 物理截断

[Storage::truncate](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/storage.rs#L118-L122)：

```rust
pub fn truncate(&mut self) {
    self.rezero();                 // 先旋转回物理起点
    self.inner.truncate(self.len); // 截断到逻辑长度
}
```

先归零再截断，确保截断的是真正的"无效"行。

### 4.3 update_history — 运行时调整 scrollback 大小

[Grid::update_history](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/mod.rs#L154-L161)：

```rust
pub fn update_history(&mut self, history_size: usize) {
    let current_history_size = self.history_size();
    if current_history_size > history_size {
        self.raw.shrink_lines(current_history_size - history_size);
    }
    self.display_offset = min(self.display_offset, history_size);
    self.max_scroll_limit = history_size;
}
```

### 4.4 clear_history — 清空 scrollback

[Grid::clear_history](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/mod.rs#L383-L389)：

```rust
pub fn clear_history(&mut self) {
    self.raw.shrink_lines(self.history_size());
    self.display_offset = 0;
}
```

### 4.5 Grid::reset — 完全重置

[Grid::reset](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/mod.rs#L336-L352) 在终端重置时调用，先 `clear_history()` 清空 scrollback，再重置所有可见行。

### 4.6 decrease_scroll_limit — 显式缩小 scrollback

[Grid::decrease_scroll_limit](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/mod.rs#L182-L188) 在 `grow_lines` 中被调用：

```rust
fn decrease_scroll_limit(&mut self, count: usize) {
    let count = min(count, self.history_size());
    if count != 0 {
        self.raw.shrink_lines(min(count, self.history_size()));
        self.display_offset = min(self.display_offset, self.history_size());
    }
}
```

---

## 五、Grid 内存布局总览

根据 [Grid 文档注释](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/mod.rs#L81-L107)，环形缓冲区的逻辑布局：

```
┌─────────────────────────┐  <-- max_scroll_limit + lines
│      UNINITIALIZED      │  物理分配但逻辑无效的区域（MAX_CACHE_SIZE 缓冲）
├─────────────────────────┤  <-- self.raw.inner.len()
│      RESIZE BUFFER      │  物理分配但 len 不包含的区域
├─────────────────────────┤  <-- self.history_size() + lines
│     SCROLLUP REGION     │  scrollback 历史（从 display_offset 之上）
├─────────────────────────┤v lines
│     VISIBLE  REGION     │  当前可见区域
├─────────────────────────┤^ <-- display_offset
│    SCROLLDOWN REGION    │  display_offset 之下的历史（用户滚动时可见）
└─────────────────────────┘  <-- zero (环形缓冲区的逻辑底部)
```

`Line` 索引约定：
- `Line(0)` = 可见区最底行
- `Line(screen_lines - 1)` = 可见区最顶行
- `Line(-1)` = scrollback 中最近的一行
- `Line(-(history_size))` = scrollback 中最远的一行

---

## 六、关键协作关系图

```
                    PTY 输出
                       │
                       ▼
               EventLoop::pty_read
                       │
                       ▼
               vte::ansi::Parser
                       │
              ┌────────┼────────┐
              ▼        ▼        ▼
          Term::input  Term::linefeed  Term::reverse_index
              │           │               │
              │    (cursor at bottom)  (cursor at top)
              │           │               │
              ▼           ▼               ▼
          Term::wrapline  Term::scroll_up  Term::scroll_down
                       │                    │
                       ▼                    ▼
              Term::scroll_up_relative   Term::scroll_down_relative
                  │  │  │                     │  │  │
                  │  │  └── mark_fully_damaged  │  │  └── mark_fully_damaged
                  │  └── selection.rotate      │  └── selection.rotate
                  ▼                            ▼
           Grid::scroll_up              Grid::scroll_down
           │  │  │                      │  │  │
           │  │  └── reset new lines    │  │  └── reset new lines
           │  └── raw.rotate            │  └── raw.rotate_down
           ▼                             ▼
     increase_scroll_limit          raw.swap (子区域)
           │
           ▼
     Storage::initialize
     (动态增长 len，必要时 rezero+resize)

     ═══════════════════════════════════════

         用户鼠标/键盘滚动
                │
                ▼
     input::ActionContext::scroll
                │
                ▼
     Term::scroll_display
                │
                ▼
     Grid::scroll_display (修改 display_offset)
                │
                ▼
     display_iter() → 渲染时根据 display_offset
     读取 scrollback 区域的内容

     ═══════════════════════════════════════

       resize / 配置变更 / 终端重置
                │
      ┌─────────┼──────────┐
      ▼         ▼          ▼
  grow_lines  shrink_lines  update_history / clear_history / reset
      │         │          │
      ▼         ▼          ▼
  decrease_   scroll_up +  Storage::shrink_lines
  scroll_     rotate +      → truncate (延迟释放)
  limit       shrink_visible
```

---

## 七、核心设计要点总结

| 设计要点 | 实现手法 | 代码位置 |
|---------|---------|---------|
| 环形缓冲区 | `Storage.zero` 偏移 + 取模索引，旋转 O(1) | [storage.rs#L220-L234](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/storage.rs#L220-L234) |
| 延迟内存释放 | `len`（逻辑长度）< `inner.len()`（物理容量），差值达 MAX_CACHE_SIZE 才 truncate | [storage.rs#L107-L114](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/storage.rs#L107-L114) |
| 批量预分配 | `initialize` 一次分配 `max(additional, MAX_CACHE_SIZE)` 额外行 | [storage.rs#L126-L138](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/storage.rs#L126-L138) |
| 全屏旋转优化 | `region.start == 0` 时用 rotate + swap 固定行，避免逐行移动 | [grid/mod.rs#L272-L306](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/mod.rs#L272-L306) |
| 动态 scrollback 增长 | `scroll_up` 时按需调用 `increase_scroll_limit`，不预分配 | [grid/mod.rs#L175-L180](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/mod.rs#L175-L180) |
| 视口锚定 | `display_offset != 0` 时新输出不拉回视口，只增大 offset | [grid/mod.rs#L267-L269](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/mod.rs#L267-L269) |
| 行脏标记优化 | `Row.occ` 追踪修改过的单元格数，reset 时只遍历 `[0..occ]` | [row.rs#L91-L110](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/row.rs#L91-L110) |
| 高效行交换 | `Storage::swap` 利用 `size_of::<Row<T>>() == 4 * usize` 的不变量做 qword 级交换 | [storage.rs#L153-L176](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/grid/storage.rs#L153-L176) |
| Alt Screen 隔离 | 备选 Grid 的 `max_scroll_limit = 0`，无 scrollback | [term/mod.rs#L416](file:///d:/fz/0601/solo-dogfeeding/code/336-alacritty/alacritty_terminal/src/term/mod.rs#L416) |
