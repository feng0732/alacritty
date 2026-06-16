# 剪贴板与选择 状态流转与交互流程

## 一、鼠标释放后选区的状态

### 1.1 结论：选区继续保留

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

### 1.2 释放后的状态

```
鼠标按下 → 拖动 → 鼠标释放
   ↓        ↓        ↓
start   update   copy_selection
selection selection (Selection类型)
   ↓        ↓        ↓
选区创建  选区更新  文本写入剪贴板
                  选区仍然保留 ✓
```

### 1.3 选区保留期间的行为

释放后选区继续存在，直到：
- 用户在其他位置点击（`clear_selection` 后开始新选择）
- 终端内容发生变化（见第二章）
- 用户显式触发 `ClearSelection` 动作
- 切换到/退出 alt screen

在 VI 模式下，释放后还可以通过键盘移动光标来继续扩展选区，由 `vi_mode_recompute_selection()` 更新：[term/mod.rs#L870-L881](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L870-L881)

---

## 二、终端内容变化导致选区清除的完整场景

选区清除策略遵循一个原则：**如果文本内容发生变化导致选区位置失效，就清除或过滤选区**。

### 2.1 完全清除选区（设置为 None）

| 场景 | 代码位置 | 触发原因 |
|------|----------|----------|
| 列数变化 | [term/mod.rs#L682](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L682) | 窗口宽度调整，列对齐失效 |
| 切换主/辅屏幕 | [term/mod.rs#L733](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L733) | `swap_alt()` 切换 grid，选区属于旧屏幕 |
| 清除全部屏幕 | [term/mod.rs#L1803](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L1803) | `ClearMode::All`，所有内容被清除 |
| 终端重置 | [term/mod.rs#L1847](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L1847) | `reset_state()`，终端完全重置 |

### 2.2 部分清除（过滤相交范围）

如果只有部分行的内容变化，采用「过滤相交」策略：只清除与变化范围相交的选区部分，不相交的保留。

使用 `intersects_range(range)` 判断是否相交，不相交则保留：

```rust
self.selection = self.selection.take().filter(|s| !s.intersects_range(range));
```

| 场景 | 代码位置 | 清除范围 |
|------|----------|----------|
| 清除行内部分内容 | [term/mod.rs#L1657](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L1657) | 当前行 `cursor.line..=cursor.line` |
| 清除光标以上内容 | [term/mod.rs#L1773](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L1773) | `Line(0)..=cursor.line` |
| 清除光标以下内容 | [term/mod.rs#L1786](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L1786) | `cursor.line..Line(screen_lines)` |
| 清除历史记录 | [term/mod.rs#L1811](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L1811) | `..Line(0)`（历史区内的部分） |

### 2.3 选区跟随滚动（旋转而非清除）

当滚动发生时，选区不会被清除，而是通过 `rotate()` 方法跟随内容一起移动。如果选区被滚出可视区域，则返回 `None`（被清除）。

| 场景 | 代码位置 | 行为 |
|------|----------|------|
| 行数变化（窗口高度调整） | [term/mod.rs#L686-L689](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L686-L689) | 调用 `rotate` 调整选区位置 |
| 向下滚动（内容下移） | [term/mod.rs#L751-L752](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L751-L752) | `scroll_down_relative` 中旋转 |
| 向上滚动（内容上移） | [term/mod.rs#L778](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L778) | `scroll_up_relative` 中旋转 |

`rotate()` 方法的行为：[selection.rs#L137-L191](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/selection.rs#L137-L191)
- 选区起止点在滚动范围内 → 跟随移动
- 选区部分滚出 → 截断到边界
- 选区完全滚出 → 返回 `None`（清除）

### 2.4 不清除选区的操作

注意：以下行内编辑操作**不会**触发选区清除：
- `insert_blank()` — 插入空白字符
- `delete_chars()` — 删除字符
- `erase_chars()` — 擦除字符

这些操作只修改行内单元格内容，但不清除选区。这是设计选择还是潜在 bug 需进一步确认。

---

## 三、粘贴读取流程

### 3.1 用户触发的粘贴（UI → 剪贴板 → PTY）

这是最常见的粘贴场景：用户按 Ctrl+Shift+V 或通过菜单触发粘贴。

**调用链：**

```
用户按键/菜单
    │
    ▼
Action::Paste / Action::PasteSelection
    │
    ▼
input/mod.rs: Action.execute()
    │
    │  // 从剪贴板读取
    │  let text = ctx.clipboard_mut().load(ClipboardType::Clipboard);
    │  ctx.paste(&text, true);
    ▼
event.rs: paste(text, bracketed)
    │
    │  // 根据模式处理
    │  ├─ 搜索模式 → 作为搜索输入
    │  ├─ 内联搜索 → 作为搜索输入
    │  ├─ 括号粘贴模式 → 加 \x1b[200~ ... \x1b[201~ 包裹
    │  └─ 普通模式 → 换行符替换为 \r
    ▼
write_to_pty(...)  →  发送给终端进程
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

- 粘贴实现：[event.rs#L1369-L1410](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty/src/event.rs#L1369-L1410)
  - 搜索模式下：作为搜索字符输入
  - 括号粘贴模式：用 `\x1b[200~` 和 `\x1b[201~` 包裹文本，并过滤 `\x1b` 和 `\x03`
  - 普通模式：将换行符替换为回车 `\r`

### 3.2 剪贴板读取的平台行为

`clipboard.load(ty)` 的行为：[clipboard.rs#L70-L83](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty/src/clipboard.rs#L70-L83)

```rust
pub fn load(&mut self, ty: ClipboardType) -> String {
    let clipboard = match (ty, &mut self.selection) {
        (ClipboardType::Selection, Some(provider)) => provider,
        _ => &mut self.clipboard,
    };
    match clipboard.get_contents() {
        Err(err) => {
            debug!("Unable to load text from clipboard: {err}");
            String::new()
        },
        Ok(text) => text,
    }
}
```

关键点：
- `Selection` 类型在无 selection 剪贴板的平台（macOS/Windows）上，**回退到系统剪贴板**
- 读取失败返回空字符串，不报错

---

## 四、终端发起的剪贴板读写请求

这是终端程序（如 vim、tmux）通过转义序列（OSC 52）主动操作剪贴板的机制。

### 4.1 整体流程

```
终端程序发送 OSC 52 转义序列
    │
    ▼
VTE 解析 → 调用 term 的 clipboard_store/load 方法
    │
    ▼
终端层：检查 osc52 配置权限
    │
    ▼
发送 Event::ClipboardStore / ClipboardLoad 事件
    │
    ▼
UI 层事件循环处理事件
    │
    ├─ Store：调用 clipboard.store(ty, content)
    └─ Load：调用 clipboard.load(ty)，然后将结果写回 PTY
```

### 4.2 剪贴板写入（终端 → 剪贴板）

**触发**：终端程序发送 OSC 52 序列，如 `\x1b]52;c;<base64编码内容>\x07`

**处理流程：**

1. 终端层接收并解码：[term/mod.rs#L1705-L1721](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L1705-L1721)
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
       
       // Base64 解码
       if let Ok(bytes) = Base64.decode(base64) {
           if let Ok(text) = String::from_utf8(bytes) {
               self.event_proxy.send_event(Event::ClipboardStore(clipboard_type, text));
           }
       }
   }
   ```

2. UI 层处理事件：[event.rs#L1902-L1905](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty/src/event.rs#L1902-L1905)
   ```rust
   TerminalEvent::ClipboardStore(clipboard_type, content) => {
       if self.ctx.terminal.is_focused {
           self.ctx.clipboard.store(clipboard_type, content);
       }
   },
   ```

**安全机制：**
- `osc52` 配置控制权限：`OnlyCopy` / `OnlyPaste` / `CopyPaste` / `Disabled`
- 只有窗口获得焦点时才执行写入
- 内容通过 Base64 编码传输

### 4.3 剪贴板读取（剪贴板 → 终端）

**触发**：终端程序发送 OSC 52 读取序列，如 `\x1b]52;c;?\x07`

**处理流程：**

1. 终端层发起请求：[term/mod.rs#L1726-L1747](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty_terminal/src/term/mod.rs#L1726-L1747)
   ```rust
   fn clipboard_load(&mut self, clipboard: u8, terminator: &str) {
       // 权限检查
       if !matches!(self.config.osc52, Osc52::OnlyPaste | Osc52::CopyPaste) {
           return;
       }
       
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

**设计特点：**
- 终端层只发请求，不直接访问剪贴板（保持平台无关）
- 通过 `Arc<dyn Fn(&str) -> String>` 闭包传递格式化逻辑
- 结果作为转义序列写回 PTY，终端程序接收后自行处理

### 4.4 OSC 52 剪贴板类型映射

| OSC 52 字符 | ClipboardType | 含义 |
|------------|---------------|------|
| `'c'` | Clipboard | 系统剪贴板 |
| `'p'` / `'s'` | Selection | X11 主选区 |

### 4.5 与用户复制的对比

| 维度 | 用户复制（Copy 动作） | OSC 52 终端写入 |
|------|----------------------|-----------------|
| 触发方 | 用户 | 终端程序 |
| 数据源 | 终端选区内容 | 转义序列中的 Base64 |
| 权限控制 | 无（用户主动操作） | `osc52` 配置 + 焦点检查 |
| 写入目标 | 根据 Action 类型决定 | 根据转义序列参数决定 |

---

## 五、完整的状态流转图

```
┌───────────────────────────────────────────────────────────────┐
│                        选区状态流转                             │
└───────────────────────────────────────────────────────────────┘

None (无选区)
  │
  ├─ 鼠标单击 / 双击 / 三击 / Ctrl+单击
  │  start_selection(ty, point, side)
  ▼
Some(Selection)  ←───────┐
  │                      │
  ├─ 鼠标拖动            │
  │  update_selection()  │
  ▼                      │
选区范围更新             │
  │                      │
  ├─ 鼠标释放            │
  │  copy_selection(Selection)  │
  │  （选区仍保留）       │
  │                      │
  ├─ VI 模式光标移动      │
  │  vi_mode_recompute_selection()
  ▼                      │
选区自动扩展             │
  │                      │
  └─ 清除触发 ───────────┘
     clear_selection() → None
       │
       ├─ 点击空白处
       ├─ 列数变化
       ├─ 切换 alt screen
       ├─ 清屏 / 重置
       ├─ 内容变化（行清除等）
       └─ 显式 ClearSelection 动作

┌───────────────────────────────────────────────────────────────┐
│                       剪贴板数据流                              │
└───────────────────────────────────────────────────────────────┘

  选区文本 ──copy_selection()──▶ 剪贴板
                                (Selection 或 Clipboard)

  剪贴板 ────load() + paste()──▶ PTY（终端进程）
  (用户触发粘贴)

  终端进程 ──OSC 52 Store──▶ 剪贴板
  (通过转义序列)

  剪贴板 ───OSC 52 Load───▶ 终端进程
  (通过转义序列回写)
```

---

## 六、关键边界与注意事项

### 6.1 Selection 剪贴板的平台差异

- **X11/Wayland**：有独立的 Selection 剪贴板（PRIMARY）
- **macOS/Windows**：没有 Selection 剪贴板，selection 字段为 `None`
  - `store(Selection)` 时直接返回（不写入任何地方）
  - `load(Selection)` 时回退到系统剪贴板（走 `_` 分支）

### 6.2 save_to_clipboard 配置

当 `selection.save_to_clipboard = true` 时，复制到 Selection 类型时会**同时复制到系统剪贴板**：[event.rs#L750-L752](file:///d:/fz/0601/solo-dogfeeding/code/341-alacritty/alacritty/src/event.rs#L750-L752)

```rust
if ty == ClipboardType::Selection && self.config.selection.save_to_clipboard {
    self.clipboard.store(ClipboardType::Clipboard, text.clone());
}
```

### 6.3 空选择的特殊处理

选择可以处于「存在但为空」的状态：
- `is_empty()` 返回 true
- `copy_selection()` 会跳过，不执行复制
- `clear_selection()` 不会标记 dirty（无需重绘）

典型场景：鼠标刚按下还没拖动时。

### 6.4 焦点安全检查

OSC 52 剪贴板操作有焦点检查：只有终端窗口获得焦点时才执行。这是安全措施，防止后台终端程序随意读写用户剪贴板。
