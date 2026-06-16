# Alacritty IME 提交流向与光标互斥深度分析

> 本文档所有源码定位均使用 **Markdown 相对链接**（仓库根目录相对路径 + 行号锚点）。在 GitHub 仓库页面直接点击链接即可跳转到对应代码；本地 IDE 中按配置支持的文件导航方式复核。

---

## 1. IME Commit 的完整分流路径

### 1.1 入口：所有 IME Commit 统一走 `paste()`

**定位证据**：[alacritty/src/event.rs#L2018-L2024](alacritty/src/event.rs#L2018-L2024)

```rust
WindowEvent::Ime(ime) => match ime {
    Ime::Commit(text) => {
        *self.ctx.dirty = true;
        // Don't use bracketed paste for single char input.
        self.ctx.paste(&text, text.chars().count() > 1);
        self.ctx.update_cursor_blinking();
    },
```

所有 IME 确认文本，不论当前处于何种状态，都调用同一个 `paste()` 方法。`paste()` 内部实现了三路分流逻辑。

### 1.2 `paste()` 的三路分流

**定位证据**：[alacritty/src/event.rs#L1369-L1411](alacritty/src/event.rs#L1369-L1411)

```
paste(text, bracketed)
  │
  ├─ 分支1: search_active() == true
  │    → for c in text.chars() { search_input(c) }
  │    → 字符逐个送入搜索正则
  │
  ├─ 分支2: inline_search_state.char_pending == true
  │    → inline_search_input(text)
  │    → 取首个字符作为内联搜索目标
  │
  ├─ 分支3: bracketed && TermMode::BRACKETED_PASTE
  │    → on_terminal_input_start()
  │    → 写 \x1b[200~ + 过滤文本 + \x1b[201~
  │
  └─ 分支4: 普通模式
       → on_terminal_input_start()
       → 写原始文本到 PTY
```

### 1.3 分支1：搜索模式下的 IME Commit

**定位证据**：[alacritty/src/event.rs#L1370-L1373](alacritty/src/event.rs#L1370-L1373)

```rust
if self.search_active() {
    for c in text.chars() {
        self.search_input(c);
    }
}
```

完整路径：`Ime::Commit` → `paste()` → `search_input(c)逐字符` → `update_search()` → 更新 DFA

关键细节：
- IME Commit 可能一次提交多个字符（如词组"你好"），`paste()` 将其逐字符拆分后每个字符调用 `search_input()`
- `search_input()` 内部对字符做了过滤，只接受可打印字符和退格。见 [alacritty/src/event.rs#L1078-L1087](alacritty/src/event.rs#L1078-L1087)
- 此时 `bracketed` 参数被忽略——搜索模式不关心 bracketed paste 语义
- 这是合理的：搜索栏是 Alacritty 自己的 UI，不是终端应用程序，不需要 bracketed paste 协议

### 1.4 分支2：内联搜索模式下的 IME Commit

**定位证据**：[alacritty/src/event.rs#L1374-L1375](alacritty/src/event.rs#L1374-L1375)

```rust
else if self.inline_search_state.char_pending {
    self.inline_search_input(text);
}
```

完整路径：`Ime::Commit` → `paste()` → `inline_search_input(text)` → 取第一个字符 → `inline_search_next()`

**定位证据**：[alacritty/src/event.rs#L1465-L1478](alacritty/src/event.rs#L1465-L1478)

```rust
fn inline_search_input(&mut self, text: &str) {
    // Ignore input with empty text, like modifier keys.
    let c = match text.chars().next() {
        Some(c) => c,
        None => return,
    };

    self.inline_search_state.char_pending = false;
    self.inline_search_state.character = Some(c);
    self.window().set_ime_inhibitor(ImeInhibitor::VI, true);

    // Immediately move to the captured character.
    self.inline_search_next();
}
```

关键细节：
- **只取第一个字符**：即使 IME 提交了多字符词组，内联搜索只使用首字符
- `char_pending` 被设为 `false`：内联搜索只接受一次输入，之后立即跳转
- **VI 抑制器被重新设置**：之前在内联搜索等待输入时被清除，现在重新启用
- 这意味着用 IME 输入内联搜索字符时，IME 先被启用（key release 清除 VI 抑制器），Commit 后又被禁用

### 1.5 分支3+4：普通终端模式下的 IME Commit

**定位证据**：[alacritty/src/event.rs#L1376-L1410](alacritty/src/event.rs#L1376-L1410)

```rust
else if bracketed && self.terminal().mode().contains(TermMode::BRACKETED_PASTE) {
    // bracketed paste 路径
} else {
    // 普通 paste 路径
}
```

IME Commit 传入的 `bracketed` 参数是 `text.chars().count() > 1`（见入口 [alacritty/src/event.rs#L2022](alacritty/src/event.rs#L2022)）：

| IME 提交 | 字符数 | bracketed | 实际行为 |
|----------|--------|-----------|---------|
| 单个汉字"你" | 1 | `false` | 直接写入 PTY，不包裹 bracketed paste 序列 |
| 词组"你好" | 2 | `true` | 若终端支持 BRACKETED_PASTE 则包裹 `\x1b[200~...\x1b[201~` |
| 词组"你好" + 终端不支持 BRACKETED_PASTE | 2 | `true` | 走 else 分支，`\r\n` → `\r`，直接写入 |

设计意图：
- 单字符 IME 提交等同于键盘直接输入一个字符，不应被 bracketed paste 包裹
- 多字符 IME 提交（词组输入）等同于粘贴行为，应用 bracketed paste 协议告知应用程序

---

## 2. Preedit 活跃时普通终端光标的隐藏机制

光标隐藏并非在事件处理层完成，而是在 **渲染层** 通过 `CursorShape::Hidden` 实现。存在 **两条独立的隐藏路径**，分别作用于不同光标。

### 2.1 路径A：终端主光标的隐藏

**定位证据**：[alacritty/src/display/content.rs#L52-L62](alacritty/src/display/content.rs#L52-L62)

```rust
// Find terminal cursor shape.
let cursor_shape = if terminal_content.cursor.shape == CursorShape::Hidden
    || display.cursor_hidden
    || search_state.regex().is_some()
    || display.ime.preedit().is_some()   // ← 关键：preedit 活跃时强制 Hidden
{
    CursorShape::Hidden
} else if !term.is_focused && config.cursor.unfocused_hollow {
    CursorShape::HollowBlock
} else {
    terminal_content.cursor.shape
};
```

四重隐藏条件（任一为 true 即隐藏）：

| 条件 | 含义 |
|------|------|
| `terminal_content.cursor.shape == CursorShape::Hidden` | 应用程序主动设置了光标隐藏（如 vim 普通模式） |
| `display.cursor_hidden` | 光标因闪烁超时或打字而隐藏 |
| `search_state.regex().is_some()` | 搜索模式活跃时隐藏终端光标 |
| `display.ime.preedit().is_some()` | **IME 预编辑活跃时隐藏终端光标** |

当 `cursor_shape` 被设为 `CursorShape::Hidden` 后，`RenderableCursor::rects()` 中：

**定位证据**：[alacritty/src/display/cursor.rs#L29-L34](alacritty/src/display/cursor.rs#L29-L34)

```rust
match self.shape() {
    CursorShape::Beam => beam(x, y, height, thickness, self.color()),
    CursorShape::Underline => underline(x, y, width, height, thickness, self.color()),
    CursorShape::HollowBlock => hollow(x, y, width, height, thickness, self.color()),
    _ => CursorRects::default(),  // Hidden/Block → 返回空迭代器，无矩形渲染
}
```

`CursorShape::Hidden` 匹配到 `_` 分支，`CursorRects::default()` 产生零个矩形 → **光标完全不被渲染**。

### 2.2 路径B：搜索栏光标的隐藏

**定位证据**：[alacritty/src/display/mod.rs#L929-L937](alacritty/src/display/mod.rs#L929-L937)

```rust
// Add cursor to search bar if IME is not active.
if self.ime.preedit().is_none() {
    let fg = config.colors.footer_bar_foreground();
    let shape = CursorShape::Underline;
    let cursor_width = NonZeroU32::new(1).unwrap();
    let cursor =
        RenderableCursor::new(Point::new(line, column), shape, fg, cursor_width);
    rects.extend(cursor.rects(&size_info, config.cursor.thickness()));
}
```

搜索栏光标（Underline 形状）在 preedit 活跃时 **不被添加到渲染矩形列表** 中。

### 2.3 Preedit 光标替代终端光标的完整替换链

当 IME preedit 活跃时，终端光标被隐藏，取而代之的是 IME 自身的光标。整个替换链如下（对应完整路径见各定位证据链接）：

```
Preedit 活跃
  │
  ├─ 终端主光标: content.rs 第55行 → CursorShape::Hidden
  │                           → cursor.rs 第33行 → 空 Rects（不渲染）
  │
  ├─ 搜索栏光标: display/mod.rs 第930行 → 条件不成立 → 跳过 Underline 光标创建
  │
  └─ IME 替代光标: draw_ime_preview() 第1193-1209行
       ├─ 多字符选中 → CursorShape::HollowBlock
       ├─ 单字符/无选中 → CursorShape::Beam
       └─ 渲染到 preedit 文本所在位置
```

### 2.4 IME 替代光标的渲染逻辑

**定位证据**：[alacritty/src/display/mod.rs#L1193-L1213](alacritty/src/display/mod.rs#L1193-L1213)

```rust
let ime_popup_point = match preedit.cursor_end_offset {
    Some(cursor_end_offset) => {
        // Use hollow block when multiple characters are changed at once.
        let (shape, width) = if let Some(width) =
            NonZeroU32::new((cursor_end_offset.0 - cursor_end_offset.1) as u32)
        {
            (CursorShape::HollowBlock, width)
        } else {
            (CursorShape::Beam, NonZeroU32::new(1).unwrap())
        };

        let cursor_column = Column(
            (end.column.0 as isize - cursor_end_offset.0 as isize + 1).max(0) as usize,
        );
        let cursor_point = Point::new(point.line, cursor_column);
        let cursor = RenderableCursor::new(cursor_point, shape, fg, width);
        rects.extend(cursor.rects(&size_info, config.cursor.thickness()));
        cursor_point
    },
    _ => end,
};
```

光标位置计算的数值含义（见 [alacritty/src/display/mod.rs#L1204-L1207](alacritty/src/display/mod.rs#L1204-L1207)）：

- `end` 是可见预编辑文本的末尾列号
- `cursor_end_offset.0` 是从预编辑文本末尾到光标起始位置的字符宽度距离
- `end - cursor_end_offset.0 + 1` 计算出光标起始列

示例：预编辑文本 "你好世界"（4个宽字符），光标在第2-3字符（"好世"被选中）：
- `end` = cursor_point.column + 8（4个宽字符×2列）
- `cursor_end_offset` = (6, 2)（从选中起始到末尾6列宽，从选中末尾到文本末尾2列宽）
- `cursor_column` = (cursor_point.column + 8) - 6 + 1 = cursor_point.column + 3

---

## 3. 两种光标隐藏机制的设计对比

| 特性 | 终端主光标 | 搜索栏光标 |
|------|-----------|-----------|
| 代码位置 | [alacritty/src/display/content.rs#L52-L62](alacritty/src/display/content.rs#L52-L62) | [alacritty/src/display/mod.rs#L929-L937](alacritty/src/display/mod.rs#L929-L937) |
| 隐藏时机 | RenderableContent 构造时 | draw() 渲染时 |
| 隐藏方式 | shape 强制为 Hidden，rects 产生空迭代 | 不创建 RenderableCursor |
| 影响范围 | 同时隐藏了 Block/Beam/Underline/HollowBlock 所有形状 | 只影响 Underline 搜索栏光标 |
| 恢复机制 | preedit 变 None → shape 恢复原始值 | preedit 变 None → 条件成立 → 重新创建 |
| 与其他隐藏条件的关系 | 与 cursor_hidden、搜索模式、应用隐藏共享同一判断 | 仅受 preedit 控制 |

为什么终端光标不用"不创建"的方式？因为终端光标的形状和位置是从 `TerminalContent` 中提取的，在 `RenderableContent` 迭代过程中已经构建了光标对象。将其 shape 设为 Hidden 是最小侵入的修改方式，不影响迭代逻辑。

为什么搜索栏光标不用"Hidden shape"的方式？因为搜索栏光标是在 `draw()` 中独立创建的，直接控制创建与否更简洁，避免了创建一个无用的 Hidden 光标再被跳过的开销。

---

## 4. 完整时序：一次 IME 输入的光标状态变化

### 4.1 普通终端模式下输入"你好"

```
时间线                     事件                        终端光标    搜索栏光标    IME光标
─────────────────────────────────────────────────────────────────────────────────
T0  正常状态                                           可见(Beam)    -          -
T1  按 'n' 键                                          可见(Beam)    -          -
T2  Ime::Preedit("n", Some((0,1)))                     Hidden        -          Beam
T3  按 'i' 键                                          Hidden        -          Beam
T4  Ime::Preedit("ni", Some((0,2)))                    Hidden        -          Beam
T5  候选窗口选择"你"                                     Hidden        -          -
T6  Ime::Commit("你")  → paste("你", false) → PTY      可见(Beam)    -          -
T7  Ime::Preedit("", None) → preedit 清空              可见(Beam)    -          -
```

注意 T6-T7 的顺序：Commit 和 Preedit("") 是两个独立事件。Commit 先到达，此时 preedit 仍为 `Some`，终端光标仍为 Hidden。随后 Preedit("") 到达，preedit 被清除为 None，下一帧渲染时光标恢复。

### 4.2 搜索模式下用 IME 输入搜索词

```
时间线                     事件                        终端光标    搜索栏光标    IME光标
─────────────────────────────────────────────────────────────────────────────────
T0  搜索活跃，光标在搜索栏                              Hidden      Underline    -
T1  Ime::Preedit("n", Some((0,1)))                     Hidden        -          Beam
T2  Ime::Preedit("ni", Some((0,2)))                    Hidden        -          Beam
T3  Ime::Commit("你")  → paste("你", false)            Hidden        -          -
    → search_active() → search_input('你')
T4  Ime::Preedit("", None) → preedit 清空              Hidden      Underline    -
```

搜索模式下终端光标始终为 Hidden（因为 `search_state.regex().is_some()` 为 true），preedit 仅影响搜索栏光标的可见性。

---

## 5. inline_search 中 IME 抑制器的精细时序

内联搜索场景下 IME 抑制器的状态变化较为复杂：

```
时间线    事件                                    VI抑制器    IME状态
─────────────────────────────────────────────────────────────────
T0  Vi 模式活跃                               ON         禁用
T1  触发 InlineSearchForward action            ON         禁用
    → char_pending = true
T2  key release (触发内联搜索的按键松开)         OFF        启用
    → set_ime_inhibitor(VI, false)
T3  用户通过 IME 输入字符                      OFF        启用+预编辑
T4  Ime::Commit("X")                           OFF        启用
    → paste("X", false)
    → inline_search_input("X")
      → char_pending = false
      → character = Some('X')
      → set_ime_inhibitor(VI, true)  ← 重新禁用
      → inline_search_next()         ← 跳转
T5  IME 被禁用                                 ON         禁用
```

**关键时序 T2** 的定位证据：[alacritty/src/input/keyboard.rs#L31-L34](alacritty/src/input/keyboard.rs#L31-L34)

```rust
if key.state == ElementState::Released {
    if self.ctx.inline_search_state().char_pending {
        self.ctx.window().set_ime_inhibitor(ImeInhibitor::VI, false);
    }
    self.key_release(key, mode, mods);
    return;
}
```

这确保了内联搜索等待字符输入时 IME 被启用，用户可以用 IME 输入搜索字符。一旦字符被接收，IME 立即被重新禁用（见 `inline_search_input()` 中的 [alacritty/src/event.rs#L1474](alacritty/src/event.rs#L1474)）。

同时，[alacritty/src/input/keyboard.rs#L23-L26](alacritty/src/input/keyboard.rs#L23-L26) 的键盘互斥：

```rust
// IME input will be applied on commit and shouldn't trigger key bindings.
if self.ctx.display().ime.preedit().is_some() {
    return;
}
```

preedit 活跃期间所有按键直接返回，防止 IME 组合过程中的按键穿透到终端绑定或终端应用。

---

## 6. 源码定位速查表（仓库相对路径 + 行号链接）

| 分析点 | 相对链接 | 功能 |
|--------|---------|------|
| IME Commit 入口 | [alacritty/src/event.rs#L2018-L2024](alacritty/src/event.rs#L2018-L2024) | winit Ime::Commit → paste() |
| paste() 三路分流 | [alacritty/src/event.rs#L1369-L1411](alacritty/src/event.rs#L1369-L1411) | 搜索/内联搜索/终端三路 |
| search_input 调用点 | [alacritty/src/event.rs#L1370-L1373](alacritty/src/event.rs#L1370-L1373) | 搜索模式逐字符接收 IME 提交 |
| inline_search_input | [alacritty/src/event.rs#L1374-L1375](alacritty/src/event.rs#L1374-L1375) | 内联搜索调用入口 |
| inline_search_input 实现 | [alacritty/src/event.rs#L1465-L1478](alacritty/src/event.rs#L1465-L1478) | 取首字符 + 重新禁用 VI IME |
| bracketed paste 路径 | [alacritty/src/event.rs#L1376-L1389](alacritty/src/event.rs#L1376-L1389) | 多字符 IME 走 bracketed paste |
| 普通 paste 路径 | [alacritty/src/event.rs#L1390-L1410](alacritty/src/event.rs#L1390-L1410) | 单字符或非 bracketed 模式 |
| 终端光标 preedit 隐藏 | [alacritty/src/display/content.rs#L52-L62](alacritty/src/display/content.rs#L52-L62) | CursorShape::Hidden 判定 |
| Hidden 形状产生空 Rects | [alacritty/src/display/cursor.rs#L29-L34](alacritty/src/display/cursor.rs#L29-L34) | match 分支 _ → CursorRects::default() |
| 搜索栏光标 preedit 隐藏 | [alacritty/src/display/mod.rs#L929-L937](alacritty/src/display/mod.rs#L929-L937) | preedit 时不创建 Underline 光标 |
| IME 替代光标渲染 | [alacritty/src/display/mod.rs#L1193-L1213](alacritty/src/display/mod.rs#L1193-L1213) | Beam / HollowBlock 与列计算 |
| 键盘 preedit 互斥 | [alacritty/src/input/keyboard.rs#L23-L26](alacritty/src/input/keyboard.rs#L23-L26) | preedit 活跃时跳过按键处理 |
| 内联搜索 key release 启用 IME | [alacritty/src/input/keyboard.rs#L31-L34](alacritty/src/input/keyboard.rs#L31-L34) | key release 清除 VI 抑制器 |
