# 剪贴板与选择 代码分析

## 一、模块职责边界

### 1.1 层次结构总览

```
┌─────────────────────────────────────────────────────┐
│  UI 层 (alacritty crate)                            │
│  ┌──────────────┐  ┌──────────────────────────────┐ │
│  │ clipboard.rs │  │ event.rs / input/mod.rs      │ │
│  │  平台封装    │  │  协调控制（选择→剪贴板）     │ │
│  └──────┬───────┘  └───────────────┬──────────────┘ │
└─────────┼───────────────────────────┼────────────────┘
          │                           │
┌─────────▼───────────────────────────▼────────────────┐
│  终端核心层 (alacritty_terminal crate)               │
│  ┌──────────────────┐  ┌───────────────────────────┐ │
│  │ selection.rs     │  │ term/mod.rs               │ │
│  │  选择数据模型    │  │  终端状态与选择操作       │ │
│  └──────────────────┘  └─────────────┬─────────────┘ │
│                                       │               │
│                              ┌────────▼──────────┐    │
│                              │ event.rs          │    │
│                              │  事件定义         │    │
│                              └───────────────────┘    │
└──────────────────────────────────────────────────────┘
```

### 1.2 各模块职责

#### `alacritty_terminal/src/selection.rs` — 选择数据模型

**职责**：纯数据结构与算法，不关心 UI 和平台。

- `Selection` 结构体：表示一个选择区域，包含类型和起止锚点
- `SelectionType` 枚举：4 种选择模式（Simple / Block / Semantic / Lines）
- `SelectionRange` 结构体：归一化后的选择范围（start ≤ end）
- `Anchor` 结构体：点 + 侧边（Left/Right），用于精确定位选择边界

核心能力：
- `new` / `update`：创建和更新选择
- `to_range`：将选择转换为归一化的网格坐标范围
- `rotate`：选择随滚动区域旋转时的坐标变换
- `is_empty`：判断选择是否为空
- `include_all`：扩展选择边以包含完整单元格

#### `alacritty_terminal/src/term/mod.rs` — 终端状态中的选择

**职责**：将选择作为终端状态的一部分管理。

- `selection: Option<Selection>` 字段：终端当前的选择状态
- `selection_to_string()`：将选择内容转换为文本字符串
- 各种终端操作触发的选择清除逻辑（滚动、清屏、切换屏幕等）

#### `alacritty/src/clipboard.rs` — 剪贴板平台封装

**职责**：跨平台剪贴板访问，屏蔽平台差异。

- `Clipboard` 结构体：持有系统剪贴板和 X11 主选区（selection）
- `store(ty, text)`：根据 `ClipboardType` 写入对应剪贴板
- `load(ty)`：从对应剪贴板读取内容

平台差异：
- **X11**：有两个剪贴板 — `Clipboard`（系统剪贴板）和 `Selection`（主选区，鼠标中键粘贴）
- **Wayland**：通过 wayland 协议创建两个剪贴板
- **macOS / Windows**：只有系统剪贴板，selection 为 `None`

#### `alacritty/src/event.rs` — 选择与剪贴板协调层

**职责**：连接 UI 事件与终端状态，管理选择的生命周期。

核心方法：
- `start_selection(ty, point, side)`：开始新选择
- `update_selection(point, side)`：更新选择终点
- `clear_selection()`：清除选择
- `toggle_selection(ty, point, side)`：切换选择模式
- `copy_selection(ty)`：复制选择内容到剪贴板
- `selection_is_empty()`：判断选择是否为空

#### `alacritty/src/input/mod.rs` — 输入触发层

**职责**：处理鼠标/键盘/触摸输入，触发选择操作。

- 鼠标点击 → 开始选择
- 鼠标拖动 → 更新选择
- 鼠标释放 → 复制到 selection 剪贴板
- 双击 → 语义选择
- 三击 → 行选择
- Ctrl+点击 → 块选择

#### `alacritty/src/config/selection.rs` — 选择配置

**职责**：用户可配置的选择相关参数。

- `semantic_escape_chars`：语义选择的分隔字符
- `save_to_clipboard`：选择时是否同时保存到系统剪贴板

---

## 二、状态变化流程

### 2.1 选择的生命周期

```
None (无选择)
  │
  │  start_selection()
  ▼
Some(Selection { ty, region })  (选择进行中)
  │
  │  update_selection()  ←── 鼠标移动/拖动
  ▼
选择范围变化
  │
  │  copy_selection()    ←── 鼠标释放 / 显式复制
  ▼
文本写入剪贴板
  │
  │  clear_selection()   ←── 终端内容变化 / 点击空白 / ...
  ▼
None (无选择)
```

### 2.2 选择创建与更新的状态细节

**开始选择** — `start_selection()` [event.rs#L789-L794](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty/src/event.rs#L789-L794)

```rust
fn start_selection(&mut self, ty: SelectionType, point: Point, side: Side) {
    self.terminal.selection = Some(Selection::new(ty, point, side));
    *self.dirty = true;
    self.copy_selection(ClipboardType::Selection);  // 立即复制初始空选择？
}
```

注意：初始选择可能是空的（起止点相同），`copy_selection` 内部会检查内容非空才执行。

**更新选择** — `update_selection()` [event.rs#L767-L787](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty/src/event.rs#L767-L787)

- 取出当前选择
- 处理消息栏边界（选择不超出最后一行）
- 更新选择终点
- VI 模式下同步移动 vi 光标并调用 `include_all()`
- 重新放回终端状态，标记 dirty

**切换选择类型** — `toggle_selection()` [event.rs#L796-L809](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty/src/event.rs#L796-L809)

- 如果已有同类非空选择 → 清除选择
- 如果已有不同类非空选择 → 切换类型并复制
- 否则 → 开始新选择

### 2.3 选择类型与行为差异

| 类型 | 行为 | 触发方式 |
|------|------|----------|
| Simple | 精确跟踪单元格，支持跨行 | 单击拖动 |
| Block | 矩形区域选择，列对齐 | Ctrl+单击拖动 |
| Semantic | 自动扩展到语义边界（词、括号） | 双击 |
| Lines | 自动扩展到整行 | 三击 |

在 `to_range()` 中，不同类型有不同的范围计算逻辑：
- Simple / Block：根据 `Side` 精确计算边界
- Semantic：调用 `semantic_search_left/right` 扩展
- Lines：调用 `line_search_left/right` 扩展到行首行尾

---

## 三、结束阶段逻辑

### 3.1 选择结束的触发场景

**主动结束（复制）**：

1. **鼠标左键/右键释放** — [input/mod.rs#L719-L722](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty/src/input/mod.rs#L719-L722)
   - 释放时调用 `copy_selection(ClipboardType::Selection)`
   - 同时停止选择滚动定时器

2. **显式复制操作** — [input/mod.rs#L323-L325](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty/src/input/mod.rs#L323-L325)
   - `Action::Copy` → 复制到系统剪贴板
   - `Action::CopySelection` → 复制到主选区（X11）

3. **选择开始时** — 新选择创建时立即复制（初始可能为空）
   - `start_selection()` 末尾调用 `copy_selection(Selection)`

**被动结束（清除）**：

终端内容变化导致选择失效时会被清除，主要发生在 [term/mod.rs](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs)：

| 场景 | 位置 | 说明 |
|------|------|------|
| 列数变化 | L682 | 窗口宽度调整导致列数改变 |
| 切换主/辅屏幕 | L733 | alt screen 切换 |
| 清除屏幕 | L1803 | 清屏操作 |
| 终端重置 | L1847 | 终端完全重置 |
| 滚动区域内容旋转 | L689 / L778 | 滚动时选择跟随移动 |
| 行内编辑操作 | L1657, L1773, L1786 | 删除字符、插入字符等改变行内容时清除相交的选择 |
| 历史记录清除 | L1811 | 清除历史记录时清除历史区内的选择 |

### 3.2 复制到剪贴板的详细逻辑

`copy_selection()` [event.rs#L744-L754](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty/src/event.rs#L744-L754)

```rust
fn copy_selection(&mut self, ty: ClipboardType) {
    let text = match self.terminal.selection_to_string().filter(|s| !s.is_empty()) {
        Some(text) => text,
        None => return,
    };

    if ty == ClipboardType::Selection && self.config.selection.save_to_clipboard {
        self.clipboard.store(ClipboardType::Clipboard, text.clone());
    }
    self.clipboard.store(ty, text);
}
```

关键行为：
1. **空选择不复制**：`selection_to_string()` 返回 `None` 或空字符串时直接返回
2. **双写入机制**：当 `save_to_clipboard = true` 且目标类型是 Selection 时，同时写入系统剪贴板
3. **选择文本提取**：根据选择类型（Block / Lines / 其他）采用不同的文本拼接方式

### 3.3 选择滚动的结束

当鼠标在选择过程中移出窗口上下边界时，会触发自动滚动。释放鼠标时结束滚动：

```
鼠标按下 + 拖动到边界 → 启动滚动定时器（SELECTION_SCROLLING_INTERVAL）
鼠标释放 → unschedule 定时器，停止滚动
```

相关代码：[input/mod.rs#L716-L717](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty/src/input/mod.rs#L716-L717)

### 3.4 清除选择的副作用

`clear_selection()` [event.rs#L760-L765](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty/src/event.rs#L760-L765)

```rust
fn clear_selection(&mut self) {
    let selection = self.terminal.selection.take();
    *self.dirty |= selection.is_some_and(|s| !s.is_empty());
}
```

- 只有非空选择被清除时才标记 dirty（需要重绘）
- 空选择清除不产生视觉变化

---

## 四、剪贴板与选择的协作关系

### 4.1 数据流方向

```
鼠标/键盘输入
    │
    ▼
input/mod.rs (输入处理器)
    │  start_selection / update_selection / clear_selection
    ▼
event.rs (事件协调层)
    │  操作 terminal.selection
    ▼
term/mod.rs (终端状态)
    │  selection_to_string()
    ▼
event.rs (copy_selection)
    │  store / load
    ▼
clipboard.rs (平台剪贴板)
```

### 4.2 关键交互点

1. **选择 ↔ 剪贴板**：
   - 选择是**数据源**，剪贴板是**数据目标**
   - 选择状态存在于终端核心层，剪贴板访问在 UI 层
   - 两者通过 `copy_selection()` 桥接

2. **选择 ↔ 终端内容**：
   - 选择依赖终端网格内容（提取文本）
   - 终端内容变化会导致选择失效（清除）
   - 选择坐标随终端滚动而旋转（`rotate` 方法）

3. **剪贴板 ↔ 终端**：
   - 终端可以通过 `Event::ClipboardStore` 请求写入剪贴板
   - 终端可以通过 `Event::ClipboardLoad` 请求读取剪贴板内容
   - 这是终端转义序列（如 OSC 52）的支持机制

### 4.3 Selection 与 Clipboard 的概念区分

容易混淆的两个概念：

| 概念 | 层级 | 含义 |
|------|------|------|
| `Selection` (选择) | 终端层 | 终端网格中一块选中的文本区域 |
| `ClipboardType::Selection` | 剪贴板层 | X11 主选区（PRIMARY selection），一种剪贴板 |

**关系**：当用户在终端中选择文本（`Selection`）后，文本会被复制到 X11 的主选区（`ClipboardType::Selection`），这是 X11 平台的惯例。

---

## 五、设计特点与权衡

### 5.1 分层设计的优点

- **终端核心层无平台依赖**：`selection.rs` 纯算法，可独立测试
- **UI 层封装平台差异**：`clipboard.rs` 用条件编译屏蔽平台区别
- **协调层职责单一**：`event.rs` 专注于状态转换和副作用触发

### 5.2 选择的「延迟计算」设计

`Selection` 只存原始锚点，不存计算后的范围：
- 优点：状态精简，更新时只需修改一个锚点
- 缺点：每次使用都要调用 `to_range()` 重新计算
- 权衡：选择更新频率远高于读取频率，适合写优化

### 5.3 空选择的处理

选择可以处于「存在但为空」的状态（如刚点击还没拖动）：
- `is_empty()` 判断空选择
- `copy_selection()` 中空选择不执行复制
- `clear_selection()` 中空选择清除不触发重绘

这种设计避免了频繁创建销毁 `Option<Selection>`，保持选择对象的连续性。
