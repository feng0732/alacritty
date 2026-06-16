# 剪贴板与选择 状态流转与交互流程

---

## 一、核心结论速览

### 1.1 鼠标释放 → 选区继续保留

| 操作 | 行为 | 选区状态 |
|------|------|----------|
| 鼠标释放 | 调用 `copy_selection(Selection)` 复制到主选区剪贴板 | 保留，继续高亮 |
| 点击别处 | 先 `clear_selection()`，再 `start_selection()` | 清除后重建 |
| 内容变化 | 相交则整体丢弃，不相交则保留 | 视情况而定 |

### 1.2 三种清除场景的处理策略

| 场景 | 代码模式 | 处理策略 |
|------|----------|----------|
| 内容清除（Above/Below/行清除） | `filter(!intersects_range)` | **整体丢弃**（相交就全丢） |
| 历史清除（ClearMode::Saved） | `filter(!intersects_range(..Line(0)))` | **整体丢弃（相交时），完全不相交则保留** |
| 行范围相交（行清除等） | `intersects_range` 只检查行 | **整体丢弃**（不裁剪） |
| 滚动跟随 | `rotate()` 方法 | **裁剪**（部分滚出则截断到边界） |

### 1.3 剪贴板交互的三条路径

| 路径 | 方向 | 触发方 | 关键代码 |
|------|------|--------|----------|
| 用户复制 | 选区 → 剪贴板 | 用户 | `copy_selection()` |
| 用户粘贴 | 剪贴板 → PTY | 用户 | `clipboard.load()` + `paste()` |
| 终端剪贴板请求（OSC 52） | 双向 | 终端程序 | `ClipboardStore` / `ClipboardLoad` 事件 |

---

## 二、鼠标释放后选区的状态

### 2.1 结论：选区继续保留

鼠标释放**不会清除**选区，只是将选区内容复制到剪贴板。选区仍然保留在终端状态中，视觉上继续高亮显示。

相关代码：[input/mod.rs#L696-L723](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty/src/input/mod.rs#L696-L723)

```rust
fn on_mouse_release(&mut self, button: MouseButton) {
    // ... 鼠标模式处理 ...

    // 停止选择滚动定时器
    let timer_id = TimerId::new(Topic::SelectionScrolling, self.ctx.window().id());
    self.ctx.scheduler_mut().unschedule(timer_id);

    if let MouseButton::Left | MouseButton::Right = button {
        // 复制选区到剪贴板 — 仅此而已，不清除选区
        self.ctx.copy_selection(ClipboardType::Selection);
    }
}
```

### 2.2 释放后的状态流转

```
鼠标按下 → 拖动 → 鼠标释放
   ↓        ↓        ↓
start   update   copy_selection
selection selection (Selection类型)
   ↓        ↓        ↓
选区创建  选区更新  文本写入剪贴板
                  选区仍然保留 ✓
```

### 2.3 选区保留期间的行为

释放后选区继续存在，直到：
- 用户在其他位置点击（`clear_selection` 后开始新选择）
- 终端内容发生变化（见第三章）
- 用户显式触发 `ClearSelection` 动作
- 切换到/退出 alt screen

在 VI 模式下，释放后还可以通过键盘移动光标来继续扩展选区：

```rust
fn vi_mode_recompute_selection(&mut self) {
    if !self.mode.contains(TermMode::VI) { return; }
    if let Some(selection) = self.selection.as_mut().filter(|s| !s.is_empty()) {
        selection.update(self.vi_mode_cursor.point, Side::Left);
        selection.include_all();
    }
}
```
[term/mod.rs#L870-L881](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L870-L881)

---

## 三、选区清除的精确策略分析

### 3.1 相交判断的核心逻辑

所有「部分清除」场景都使用同一模式：

```rust
self.selection = self.selection.take().filter(|s| !s.intersects_range(range));
```

`intersects_range` 的实现：[selection.rs#L228-L249](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/selection.rs#L228-L249)

```rust
pub fn intersects_range<R: RangeBounds<Line>>(&self, range: R) -> bool {
    let mut start = self.region.start.point.line;
    let mut end = self.region.end.point.line;
    if start > end { mem::swap(&mut start, &mut end); }

    let range_top = ...; // 解析范围上界
    let range_bottom = ...; // 解析范围下界

    // 关键：只要行范围有任何重叠，就返回 true
    range_bottom >= start && range_top <= end
}
```

**关键特点：**
- 只检查**行级别**的相交，不检查列
- 只要选区的行范围与被清除范围有任何重叠，就视为相交
- `filter(!intersects)` 意味着：**相交 → 整体丢弃，不相交 → 完整保留**
- **没有任何裁剪逻辑**，不会保留不相交的部分

### 3.2 场景一：内容清除（clear_screen）

**代码位置**：[term/mod.rs#L1750-L1818](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L1750-L1818)

| 清除模式 | 清除范围 | 选区处理 |
|----------|----------|----------|
| `Above` | `Line(0)..=cursor.line` | 相交则整体丢弃 |
| `Below` | `cursor.line..Line(screen_lines)` | 相交则整体丢弃 |
| `All` | 全部 | 直接设为 `None`（全部丢弃） |

**示例：**
```
选区覆盖行 2-5
执行清除光标以上内容（光标在行 3）
清除范围：行 0-3
选区行 2-5 与范围 0-3 相交 → 整个选区被丢弃
即使行 4-5 没被清除，选区也整体消失
```

代码：
```rust
// ClearMode::Above
let range = Line(0)..=cursor.line;
self.selection = self.selection.take().filter(|s| !s.intersects_range(range));

// ClearMode::Below  
let range = cursor.line..Line(screen_lines as i32);
self.selection = self.selection.take().filter(|s| !s.intersects_range(range));

// ClearMode::All
self.selection = None;  // 直接全丢
```

### 3.3 场景二：历史清除（ClearMode::Saved）

**代码位置**：[term/mod.rs#L1805-L1812](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L1805-L1812)

```rust
// ClearMode::Saved
self.grid.clear_history();
self.selection = self.selection.take().filter(|s| !s.intersects_range(..Line(0)));
```

范围 `..Line(0)` 表示所有 `< Line(0)` 的行，即历史记录区。

**策略：**
- 选区只要有任何部分在历史区（line < 0）→ **整体丢弃**
- 选区完全在可视区（所有 line >= 0）→ **完整保留**

**示例：**
```
选区覆盖行 -3 到 2（跨历史区和可视区）
清除历史记录
选区与 ..Line(0) 相交 → 整个选区被丢弃
即使可视区部分（行 0-2）没被清除，选区也整体消失
```

### 3.4 场景三：行范围相交（行清除等）

**代码位置**：[term/mod.rs#L1656-L1657](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L1656-L1657)

```rust
// clear_line
let range = self.grid.cursor.point.line..=self.grid.cursor.point.line;
self.selection = self.selection.take().filter(|s| !s.intersects_range(range));
```

**策略：整体丢弃**

即使只是清除一行的部分内容（如 Left/Right 模式），也会检查**整行**是否相交，相交则整个选区丢弃。

**极端示例：**
```
选区覆盖行 0-10，是一个很大的选区
光标在行 5，执行清除行右侧内容（只清除行 5 的部分列）
范围是行 5..=行 5
选区与行 5 相交 → 整个选区（行 0-10）被丢弃
即使行 0-4 和 6-10 的内容完全没变
```

### 3.5 唯一的裁剪场景：滚动跟随（rotate）

`rotate()` 是唯一会**裁剪**选区而非整体丢弃的场景：[selection.rs#L137-L191](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/selection.rs#L137-L191)

```rust
pub fn rotate<D: Dimensions>(mut self, ...) -> Option<Selection> {
    // ... 移动起止点 ...

    // 起点滚出上边界 → 裁剪到上边界
    if start.point.line < range_top && range_top != 0 {
        if self.ty != SelectionType::Block {
            start.point.column = Column(0);
            start.side = Side::Left;
        }
        start.point.line = range_top;  // 裁剪
    }

    // 终点滚出下边界 → 裁剪到下边界
    if end.point.line >= range_bottom {
        if self.ty != SelectionType::Block {
            end.point.column = dimensions.last_column();
            end.side = Side::Right;
        }
        end.point.line = range_bottom - 1;  // 裁剪
    }

    // 完全滚出或起止交叉 → 返回 None（丢弃）
    if end.point.line < start.point.line { return None; }

    Some(self)
}
```

**裁剪逻辑：**
- 选区部分滚出上边界 → 起点裁剪到区域顶部，列设为 0（非 Block 模式）
- 选区部分滚出下边界 → 终点裁剪到区域底部，列设为最后一列（非 Block 模式）
- 选区完全滚出或起止点交叉 → 整体丢弃

### 3.6 清除策略总表

| 操作 | 策略 | 相交处理 | 不相交处理 | 代码 |
|------|------|----------|------------|------|
| 列数变化 | 整体丢弃 | - | - | `= None` |
| 切换 alt screen | 整体丢弃 | - | - | `= None` |
| Clear All | 整体丢弃 | - | - | `= None` |
| 终端重置 | 整体丢弃 | - | - | `= None` |
| Clear Above | 整体丢弃 | 丢弃 | 保留 | `filter(!intersects)` |
| Clear Below | 整体丢弃 | 丢弃 | 保留 | `filter(!intersects)` |
| Clear Line | 整体丢弃 | 丢弃 | 保留 | `filter(!intersects)` |
| Clear Saved（历史） | 整体丢弃 | 丢弃 | 保留 | `filter(!intersects)` |
| 向上/向下滚动 | 裁剪跟随 | 裁剪 | 跟随 | `rotate()` |
| 行数变化 | 裁剪跟随 | 裁剪 | 跟随 | `rotate()` |
| insert_blank / delete_chars / erase_chars | **不触碰** | - | - | 无选区代码 |

### 3.7 行内编辑操作：完全不触碰选区状态

以下三个行内编辑函数**不包含任何选区相关代码**，选区状态不受影响：

| 函数 | 代码位置 | 功能 | 选区代码 |
|------|----------|------|----------|
| `insert_blank(count)` | [term/mod.rs#L1187-L1212](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L1187-L1212) | 在光标处插入空白，右侧单元格右移 | 无 |
| `delete_chars(count)` | [term/mod.rs#L1538-L1564](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L1538-L1564) | 删除光标处字符，右侧单元格左移 | 无 |
| `erase_chars(count)` | [term/mod.rs#L1519-L1535](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L1519-L1535) | 擦除光标处的字符，单元格清空但位置不变 | 无 |

**关键区分**：不是「做了相交判断但未命中」，而是**根本没有做任何选区检查**。

对比 `clear_line()`：[term/mod.rs#L1635-L1658](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L1635-L1658)
```rust
fn clear_line(&mut self, mode: ansi::LineClearMode) {
    // ... 清除单元格 ...
    let range = self.grid.cursor.point.line..=self.grid.cursor.point.line;
    self.selection = self.selection.take().filter(|s| !s.intersects_range(range));
    //  ↑ clear_line 有显式的选区相交检查
}
```

而 `insert_blank` / `delete_chars` / `erase_chars` 的函数体中，**完全没有 `self.selection` 的任何读写**。它们只操作 `self.grid` 中的单元格数据和 `self.damage` 中的损伤标记，选区状态完全不被触及。

**影响**：当光标在选区内执行这些行内编辑操作时，选区坐标不会更新，可能导致选区高亮与实际内容错位。这是代码中的已知行为，与 `clear_line` 的处理策略不一致。

---

## 四、粘贴读取流程

### 4.1 用户触发的粘贴（UI → 剪贴板 → PTY）

**完整调用链：**

```
用户按 Ctrl+Shift+V
    │
    ▼
Action::Paste 匹配触发
    │
    ▼
input/mod.rs: Action.execute()  [L327-L330]
    │
    ├─ let text = ctx.clipboard_mut().load(ClipboardType::Clipboard);
    │  └─ clipboard.rs: load(ty)  [L70-L83]
    │      ├─ Selection 类型 + 有 selection 剪贴板 → 用 selection
    │      └─ 其他情况 → 用系统剪贴板
    │
    └─ ctx.paste(&text, true);
        └─ event.rs: paste(text, bracketed)  [L1369-L1410]
            ├─ 搜索模式 → 作为搜索字符输入
            ├─ 括号粘贴模式 → \x1b[200~ + 过滤(\x1b,\x03) + \x1b[201~
            └─ 普通模式 → \n → \r 替换
                │
                ▼
        write_to_pty(payload) → 发送给终端进程
```

**关键代码：**

- 触发点：[input/mod.rs#L327-L334](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty/src/input/mod.rs#L327-L334)
  ```rust
  Action::Paste => {
      let text = ctx.clipboard_mut().load(ClipboardType::Clipboard);
      ctx.paste(&text, true);
  },
  Action::PasteSelection => {
      let text = ctx.clipboard_mut().load(ClipboardType::Selection);
      ctx.paste(&text, true);
  },
  ```

- 剪贴板读取：[clipboard.rs#L70-L83](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty/src/clipboard.rs#L70-L83)
  ```rust
  pub fn load(&mut self, ty: ClipboardType) -> String {
      let clipboard = match (ty, &mut self.selection) {
          (ClipboardType::Selection, Some(provider)) => provider,
          _ => &mut self.clipboard,  // Windows/macOS 走这里
      };
      match clipboard.get_contents() {
          Err(err) => String::new(),
          Ok(text) => text,
      }
  }
  ```

- 粘贴实现：[event.rs#L1369-L1410](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty/src/event.rs#L1369-L1410)

### 4.2 平台差异

| 平台 | Selection 剪贴板 | load(Selection) 行为 | store(Selection) 行为 |
|------|-----------------|----------------------|----------------------|
| X11 | 有（PRIMARY） | 读取 PRIMARY | 写入 PRIMARY |
| Wayland | 有 | 读取 selection 剪贴板 | 写入 selection 剪贴板 |
| macOS | 无 | 回退到系统剪贴板 | 直接返回（不写入） |
| Windows | 无 | 回退到系统剪贴板 | 直接返回（不写入） |

---

## 五、终端发起的剪贴板请求（OSC 52）

### 5.1 整体架构

终端程序（如 vim、tmux）可以通过 OSC 52 转义序列主动操作剪贴板。终端核心层只发事件，不直接访问剪贴板（保持平台无关）。

```
终端程序（vim/tmux）
    │
    │  发送 \x1b]52;c;<base64>\x07 (写) 或 \x1b]52;c;?\x07 (读)
    ▼
VTE 解析器
    │
    ▼
term/mod.rs: clipboard_store/load()
    │
    │  检查 osc52 配置权限
    │  发送 Event::ClipboardStore / ClipboardLoad
    ▼
event.rs: 事件循环处理
    │
    ├─ Store → clipboard.store(ty, content)
    └─ Load → clipboard.load(ty) → 格式化为 OSC 52 回写 → write_to_pty
```

### 5.2 剪贴板写入（终端 → 剪贴板）

**触发序列：** `\x1b]52;c;<base64内容>\x07`

**处理流程：**

1. 终端层接收：[term/mod.rs#L1705-L1721](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L1705-L1721)
   ```rust
   fn clipboard_store(&mut self, clipboard: u8, base64: &[u8]) {
       // 权限检查
       if !matches!(self.config.osc52, Osc52::OnlyCopy | Osc52::CopyPaste) {
           return;
       }
       
       let clipboard_type = match clipboard {
           b'c' => ClipboardType::Clipboard,
           b'p' | b's' => ClipboardType::Selection,
           _ => return,
       };
       
       // Base64 解码后发送事件
       if let Ok(bytes) = Base64.decode(base64) {
           if let Ok(text) = String::from_utf8(bytes) {
               self.event_proxy.send_event(Event::ClipboardStore(clipboard_type, text));
           }
       }
   }
   ```

2. UI 层处理：[event.rs#L1902-L1905](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty/src/event.rs#L1902-L1905)
   ```rust
   TerminalEvent::ClipboardStore(clipboard_type, content) => {
       if self.ctx.terminal.is_focused {  // 安全检查：只有焦点窗口才执行
           self.ctx.clipboard.store(clipboard_type, content);
       }
   },
   ```

### 5.3 剪贴板读取（剪贴板 → 终端）

**触发序列：** `\x1b]52;c;?\x07`

**处理流程：**

1. 终端层发起请求：[term/mod.rs#L1726-L1747](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L1726-L1747)
   ```rust
   fn clipboard_load(&mut self, clipboard: u8, terminator: &str) {
       // 权限检查
       if !matches!(self.config.osc52, Osc52::OnlyPaste | Osc52::CopyPaste) {
           return;
       }
       
       // 发送事件，附带格式化闭包
       self.event_proxy.send_event(Event::ClipboardLoad(
           clipboard_type,
           Arc::new(move |text| {
               let base64 = Base64.encode(text);
               format!("\x1b]52;{};{}{}", clipboard as char, base64, terminator)
           }),
       ));
   }
   ```

2. UI 层处理并回写：[event.rs#L1907-L1911](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty/src/event.rs#L1907-L1911)
   ```rust
   TerminalEvent::ClipboardLoad(clipboard_type, format) => {
       if self.ctx.terminal.is_focused {
           let text = format(self.ctx.clipboard.load(clipboard_type).as_str());
           self.ctx.write_to_pty(text.into_bytes());
       }
   },
   ```

### 5.4 安全机制的层级分布

配置检查、焦点检查和剪贴板访问分布在不同的代码层级，职责边界清晰：

```
┌────────────────────────────────────────────────────────────┐
│  UI 层 (alacritty crate)                                    │
│                                                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ event.rs: 事件循环                                   │   │
│  │  ┌─ 焦点检查 ───────────────────────────────────┐    │   │
│  │  │ ClipboardStore: if is_focused { store() }    │    │   │
│  │  │ ClipboardLoad: if is_focused { load()+写回 } │    │   │
│  │  └──────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────┘   │
│                         │                                  │
│  ┌──────────────────────▼─────────────────────────────┐   │
│  │ clipboard.rs: 平台剪贴板访问                         │   │
│  │  - store(ty, text):  实际写入系统剪贴板              │   │
│  │  - load(ty):        实际读取系统剪贴板              │   │
│  │  - 处理 X11/Wayland/macOS/Windows 平台差异          │   │
│  └─────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────┘
                            ▲
                            │  Event::ClipboardStore/Load
                            │
┌────────────────────────────────────────────────────────────┐
│  终端核心层 (alacritty_terminal crate)                      │
│                                                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ term/mod.rs: 终端状态机                               │   │
│  │  ┌─ 配置权限检查 ─────────────────────────────┐      │   │
│  │  │ clipboard_store: 检查 config.osc52         │      │   │
│  │  │   (OnlyCopy|CopyPaste 才放行)              │      │   │
│  │  │ clipboard_load:  检查 config.osc52         │      │   │
│  │  │   (OnlyPaste|CopyPaste 才放行)             │      │   │
│  │  └────────────────────────────────────────────┘      │   │
│  │                                                      │   │
│  │  is_focused 字段：焦点状态存储（不做检查，只存值）   │   │
│  └─────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────┘
```

**层级职责边界总结：**

| 检查/操作 | 所在层级 | 代码位置 | 说明 |
|----------|----------|----------|------|
| osc52 配置权限检查 | **终端核心层** | [term/mod.rs#L1705-L1709](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L1705-L1709)、[L1726-L1730](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L1726-L1730) | 不通过就不发事件 |
| is_focused 状态存储 | **终端核心层** | [term/mod.rs#L270](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L270) | 只存值，不做检查逻辑 |
| 焦点检查 | **UI 层** | [event.rs#L1902-L1905](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty/src/event.rs#L1902-L1905)、[L1907-L1911](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty/src/event.rs#L1907-L1911) | 事件处理时判断 |
| 实际剪贴板读写 | **UI 层** | [clipboard.rs](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty/src/clipboard.rs) | 平台相关代码 |
| is_focused 状态更新 | **UI 层** | [event.rs#L1985-L1986](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty/src/event.rs#L1985-L1986) | 窗口事件触发 |

**OSC 52 两层检查的执行顺序：**

```
终端程序发 OSC 52 序列
    │
    ▼
终端核心层：clipboard_store/load()
    │
    ├─ osc52 配置权限检查 → 不通过 → 直接返回（不发事件）
    │      [term/mod.rs]
    │
    └─ 通过 → 发送 Event::ClipboardStore/Load
            │
            ▼
UI 层：事件循环处理
    │
    ├─ is_focused 焦点检查 → 不通过 → 直接返回（不访问剪贴板）
    │      [event.rs]
    │
    └─ 通过 → 调用 clipboard.store/load()
            │
            ▼
UI 层：实际平台剪贴板访问
    [clipboard.rs]
```

### 5.5 用户复制/粘贴 vs OSC 52 的安全检查差异

| 检查项 | 用户复制/粘贴 | OSC 52 写入/读取 |
|--------|--------------|-----------------|
| osc52 配置权限 | **不检查**（用户主动操作） | **检查**（终端层入口） |
| is_focused 焦点 | **不检查**（用户直接操作） | **检查**（UI 层入口） |
| 剪贴板访问 | 直接调用 clipboard.store/load | 事件触发后调用 |

用户复制/粘贴代码位置：
- `copy_selection()` — [event.rs#L744-L754](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty/src/event.rs#L744-L754)：**无任何检查**，直接 `store`
- `Action::Paste` — [input/mod.rs#L327-L330](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty/src/input/mod.rs#L327-L330)：**无任何检查**，直接 `load` + `paste`

这是合理的：用户主动操作默认可信，终端程序发起的操作需要严格限制。

### 5.6 与用户复制/粘贴的对比

| 维度 | 用户复制 | OSC 52 写入 | 用户粘贴 | OSC 52 读取 |
|------|----------|-------------|----------|-------------|
| 触发方 | 用户 | 终端程序 | 用户 | 终端程序 |
| 数据源 | 选区内容 | 转义序列 | 剪贴板 | 剪贴板 |
| 目标 | 剪贴板 | 剪贴板 | PTY | PTY |
| 权限 | 无 | osc52 + 焦点 | 无 | osc52 + 焦点 |
| 数据格式 | 纯文本 | Base64 编码 | 纯文本 | Base64 编码后封装为 OSC |

---

## 六、关键边界与设计权衡

### 6.1 相交即丢弃的设计选择

为什么内容变化时采用「相交即整体丢弃」而不是裁剪？

- **简单高效**：行级相交判断 O(1)，裁剪需要复杂的范围计算
- **避免不一致**：如果只清除部分行，保留的选区可能与实际内容错位
- **用户预期**：内容变化后，原来的选区语义上已失效

代价：用户体验上偶尔会出现「我只清除了一行，怎么整个选区都没了」的困惑。

### 6.2 滚动时裁剪的合理性

滚动时内容只是位置移动，没有实际清除，所以：
- 选区应该跟随内容移动
- 部分滚出时裁剪到边界是合理的
- 完全滚出后丢弃也是合理的

### 6.3 OSC 52 的分层设计

终端核心层不直接访问剪贴板：
- 保持终端层平台无关，可独立测试
- 安全检查（焦点、配置）在 UI 层统一处理
- 通过事件和闭包传递格式化逻辑，解耦层间依赖

### 6.4 空选择的处理

选择可以处于「存在但为空」的状态：
- `is_empty()` 返回 true
- `copy_selection()` 跳过不执行
- `clear_selection()` 不标记 dirty

这种设计避免了频繁创建销毁 `Option<Selection>`，保持选择对象的连续性。
