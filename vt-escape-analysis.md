# VT 转义解析：CUP 换算、参数默认值与同步更新兜底

## 1. CUP 行列换算的完整路径

CUP（Cursor Position，`CSI H` / `CSI f`）的参数从 PTY 字节流到 Term 内部坐标，经历了四个阶段的换算。

### 1.1 阶段一：Parser 累积原始参数

当 Parser 状态机处于 `CsiParam` 状态时，每收到一个数字字节（0x30-0x39），调用：

```rust
// vte/src/lib.rs
fn action_paramnext(&mut self, byte: u8) {
    if self.params.is_full() {
        self.ignoring = true;
    } else {
        self.param = self.param.saturating_mul(10);
        self.param = self.param.saturating_add((byte - b'0') as u16);
    }
}
```

- 用**饱和运算**（`saturating_mul` / `saturating_add`）防止 `u16` 溢出，超过 65535 的参数被钳位到 65535
- 参数值暂存在 `self.param` 中，尚未推入 `Params`

当收到分号 `;`（0x3B）时，调用 `action_param()` 把 `self.param` 推入 `Params`，然后重置 `self.param = 0`。

### 1.2 阶段二：Params 的扁平存储

```rust
// vte/src/params.rs
pub(crate) const MAX_PARAMS: usize = 32;

pub struct Params {
    subparams: [u8; MAX_PARAMS],   // 每个参数起始位置记录子参数个数
    params: [u16; MAX_PARAMS],     // 扁平数组，参数与子参数连续存放
    current_subparams: u8,
    len: usize,
}
```

以 `CSI 1;2H` 为例，解析完成后 `params` 内部状态：

| 字段 | 值 |
|------|---|
| `params[..]` | `[1, 2, 0, 0, ...]` |
| `subparams[..]` | `[1, 1, 0, 0, ...]`（第 0 位=1 表示参数 0 有 1 个子参数，第 1 位=1 同理） |
| `len` | `2` |

`ParamsIter` 依据 `subparams[index]` 的值切割 `params` 数组，每次 `next()` 返回 `&[u16]` 切片：

```rust
// vte/src/params.rs
fn next(&mut self) -> Option<Self::Item> {
    let num_subparams = self.params.subparams[self.index];
    let param = &self.params.params[self.index..self.index + num_subparams as usize];
    self.index += num_subparams as usize;
    Some(param)
}
```

所以 `CSI 1;2H` 迭代产生两个切片：`&[1]`、`&[2]`。

### 1.3 阶段三：Performer 的参数默认值与 1→0 换算

```rust
// vte/src/ansi.rs，csi_dispatch 方法内
let mut params_iter = params.iter();

let mut next_param_or = |default: u16| match params_iter.next() {
    Some(&[param, ..]) if param != 0 => param,
    _ => default,
};

// CUP 的匹配分支
('H', []) | ('f', []) => {
    let y = next_param_or(1) as i32;
    let x = next_param_or(1) as usize;
    handler.goto(y - 1, x - 1);
},
```

**参数默认值的关键逻辑**在 `next_param_or` 闭包中：

1. 调用 `params_iter.next()` 获取下一个参数切片
2. 如果切片存在且第一个子参数 **非零**（`param != 0`），返回该值
3. 如果切片不存在（参数缺失）或参数为零（`param == 0`），返回 `default`

**零值被视为缺失**：这是 VT 规范的体现——ANSI 参数 0 在大多数指令中与"缺失"等价，因为 VT 参数是 1-based 的。`CSI 0;0H` 等同于 `CSI H`，都定位到 (1,1)。

**1→0 换算**：`handler.goto(y - 1, x - 1)`。因为 `next_param_or(1)` 保证 `y >= 1`、`x >= 1`，所以 `y - 1 >= 0`（`i32` 安全）、`x - 1 >= 0`（`usize` 安全），不需要 `saturating_sub`。

### 1.4 阶段四：Term::goto 中的 ORIGIN 模式偏移与越界钳位

```rust
// alacritty_terminal/src/term/mod.rs:1156
fn goto(&mut self, line: i32, col: usize) {
    let line = Line(line);
    let col = Column(col);

    let (y_offset, max_y) = if self.mode.contains(TermMode::ORIGIN) {
        (self.scroll_region.start, self.scroll_region.end - 1)
    } else {
        (Line(0), self.bottommost_line())
    };

    self.damage_cursor();
    self.grid.cursor.point.line = cmp::max(cmp::min(line + y_offset, max_y), Line(0));
    self.grid.cursor.point.column = cmp::min(col, self.last_column());
    self.damage_cursor();
    self.grid.cursor.input_needs_wrap = false;
}
```

| 步骤 | 操作 | 防护 |
|------|------|------|
| ORIGIN 判断 | 若 `ORIGIN` 模式开启，行坐标相对 `scroll_region.start` 偏移 | — |
| 行钳位 | `cmp::max(cmp::min(line + y_offset, max_y), Line(0))` | 上限 `max_y`，下限 `0` |
| 列钳位 | `cmp::min(col, self.last_column())` | 上限 `columns - 1` |

其中 `bottommost_line()` = `Line(screen_lines - 1)`，`last_column()` = `Column(columns - 1)`，定义在 [grid/mod.rs#L498](file:///d:/fz/0601/solo-dogfeeding/code/333-alacritty/alacritty_terminal/src/grid/mod.rs#L498) 和 [grid/mod.rs#L510](file:///d:/fz/0601/solo-dogfeeding/code/333-alacritty/alacritty_terminal/src/grid/mod.rs#L510)。

### 1.5 完整数值追踪示例

以 80×24 终端下 `CSI 5;100H` 为例：

```
Parser: params = [5, 100]
  ↓
Performer:
  y = next_param_or(1) → 5     (5 != 0, 使用原值)
  x = next_param_or(1) → 100   (100 != 0, 使用原值)
  handler.goto(5 - 1, 100 - 1) = goto(4, 99)
  ↓
Term::goto(line=4, col=99):
  ORIGIN 模式关闭: y_offset=Line(0), max_y=Line(23)
  grid.cursor.point.line = max(min(Line(4)+Line(0), Line(23)), Line(0)) = Line(4)
  grid.cursor.point.column = min(Column(99), Column(79)) = Column(79)  ← 钳位！
```

再以 `CSI H`（无参数）为例：

```
Parser: params = []  (未收到任何数字字节，params 为空)
  ↓
Performer:
  y = next_param_or(1) → 1   (params_iter.next() = None, 使用默认值 1)
  x = next_param_or(1) → 1   (同上)
  handler.goto(1 - 1, 1 - 1) = goto(0, 0)
  ↓
Term::goto(line=0, col=0):
  grid.cursor.point.line = Line(0)
  grid.cursor.point.column = Column(0)
  → 光标回到左上角
```

---

## 2. CSI 参数默认值机制

### 2.1 `next_param_or` 闭包的完整语义

```rust
// vte/src/ansi.rs，csi_dispatch 内部
let mut next_param_or = |default: u16| match params_iter.next() {
    Some(&[param, ..]) if param != 0 => param,
    _ => default,
};
```

这个闭包是**局部变量**，每次 `csi_dispatch` 调用时创建，捕获 `params_iter` 的可变引用。关键行为：

| 输入 | 结果 | 说明 |
|------|------|------|
| `params_iter` 返回 `Some(&[3])` | `3` | 正常非零参数 |
| `params_iter` 返回 `Some(&[0])` | `default` | **零参数视为缺失** |
| `params_iter` 返回 `Some(&[3, 1])` | `3` | 子参数取第一个（`[param, ..]` 匹配首个） |
| `params_iter` 返回 `None` | `default` | 参数缺失 |

### 2.2 各指令的默认值差异

不同指令对同一个 `next_param_or` 传入不同的 default：

| 指令 | Action | 参数 | 默认值 | 代码 |
|------|--------|------|--------|------|
| CUP | `H`/`f` | row, col | 1, 1 | `next_param_or(1)` |
| CUU | `A` | 行数 | 1 | `next_param_or(1)` |
| CUD | `B`/`e` | 行数 | 1 | `next_param_or(1)` |
| CUF | `C`/`a` | 列数 | 1 | `next_param_or(1)` |
| CUB | `D` | 列数 | 1 | `next_param_or(1)` |
| CNL | `E` | 行数 | 1 | `next_param_or(1)` |
| CHA | `G`/`` ` `` | 列号 | 1 | `next_param_or(1)` |
| VPA | `d` | 行号 | 1 | `next_param_or(1)` |
| ICH | `@` | 数量 | 1 | `next_param_or(1)` |
| DCH | `P` | 数量 | 1 | `next_param_or(1)` |
| EL | `K` | 模式 | 0 | `next_param_or(0)` ← 注意不同 |
| ED | `J` | 模式 | 0 | `next_param_or(0)` |
| DA | `c` | 请求 | 0 | `next_param_or(0)` |
| DSR | `n` | 状态 | 0 | `next_param_or(0)` |

**EL/ED 的默认值是 0**（清除光标到行末/屏末），而不是 1——这说明 `next_param_or` 的 default 参数需要根据指令语义定制，不是统一的。

### 2.3 Params 的容量上限

```rust
// vte/src/params.rs
pub(crate) const MAX_PARAMS: usize = 32;
```

`Params` 内部使用固定大小数组 `[u16; 32]`。当参数或子参数总数超过 32 时：

1. `Params::is_full()` 返回 `true`
2. `action_param` / `action_subparam` / `action_paramnext` 设置 `self.ignoring = true`
3. `action_csi_dispatch` 在回调前也检查 `is_full`：
   ```rust
   fn action_csi_dispatch(&mut self, performer, byte) {
       if self.params.is_full() {
           self.ignoring = true;
       } else {
           self.params.push(self.param);
       }
       performer.csi_dispatch(self.params(), self.intermediates(), self.ignoring, byte as char);
   }
   ```
4. Performer 的 `csi_dispatch` 收到 `has_ignored_intermediates = true`，但**仍然回调 Handler**——只是 Handler 可以选择忽略

### 2.4 子参数（Subparameter）的处理

当收到冒号 `:`（0x3A）时，Parser 调用 `action_subparam`：

```rust
fn action_subparam(&mut self) {
    if self.params.is_full() {
        self.ignoring = true;
    } else {
        self.params.extend(self.param);  // extend 而非 push
        self.param = 0;
    }
}
```

`Params::extend` 与 `push` 的区别：
- `push`：设置 `current_subparams = 0`，下一个值开始新参数
- `extend`：递增 `current_subparams`，下一个值是当前参数的子参数

例如 `CSI 38:2:255:0:128 m`（RGB 颜色）产生：
- `params` = `[38, 2, 255, 0, 128, ...]`
- `subparams` = `[5, 0, 0, 0, 0, ...]`（第 0 位=5 表示参数 0 有 5 个子参数）
- `ParamsIter` 返回单个切片 `&[38, 2, 255, 0, 128]`

在 `next_param_or` 中，`Some(&[38, ..]) if 38 != 0` 匹配成功，取 `38`。后续子参数（2、255、0、128）需要通过 `params_iter.next()` 返回的完整切片来访问。

---

## 3. 同步更新兜底的完整流程

同步更新（Synchronized Update）协议允许应用一次性提交大块屏幕更新，避免中间帧闪烁。Alacritty 实现了三层兜底。

### 3.1 正常流程：BSU → 缓冲 → ESU

**BSU 到达**（`CSI ?2026h`）：

```
Parser 状态机:
  0x1B → Escape
  0x5B → CsiEntry, reset_params()
  0x3F → action_collect(0x3F), state=CsiParam   ← '?' 是 0x3F
  0x32 → action_paramnext('2')
  0x30 → action_paramnext('0')
  0x32 → action_paramnext('2')
  0x36 → action_paramnext('6')
  0x68 → action_csi_dispatch()  ← 'h' = 0x68

Performer.csi_dispatch(params, intermediates=[0x3F], action='h'):
  匹配 ('h', [b'?'])
  → param = 2026
  → 检测到 NamedPrivateMode::SyncUpdate:
     self.state.sync_state.timeout.set_timeout(SYNC_UPDATE_TIMEOUT)  // 150ms
     self.terminated = true   ← 关键！让 Parser 提前停止
  → handler.set_private_mode(PrivateMode::new(2026))
     → Term::set_private_mode → NamedPrivateMode::SyncUpdate => ()  // Term 不做操作
```

**`terminated = true` 的效果**：

```rust
// vte/src/ansi.rs，Processor::advance
pub fn advance<H>(&mut self, handler: &mut H, bytes: &[u8]) {
    let mut processed = 0;
    while processed != bytes.len() {
        if self.state.sync_state.timeout.pending_timeout() {
            processed += self.advance_sync(handler, &bytes[processed..]);
        } else {
            let mut performer = Performer::new(&mut self.state, handler);
            processed +=
                self.parser.advance_until_terminated(&mut performer, &bytes[processed..]);
            // ↑ advance_until_terminated 在每个字节后检查 performer.terminated()
            // 当 terminated=true 时，返回已消费的字节数，退出
        }
    }
    // 下一次循环进入 advance_sync 分支
}
```

**缓冲阶段**（`advance_sync`）：

```rust
fn advance_sync<H>(&mut self, handler: &mut H, bytes: &[u8]) -> usize {
    if self.state.sync_state.buffer.len() + bytes.len() >= SYNC_BUFFER_SIZE - 1 {
        // 兜底一：缓冲溢出，强制终止同步更新
        self.stop_sync_internal(handler, None);
        let mut performer = Performer::new(&mut self.state, handler);
        self.parser.advance_until_terminated(&mut performer, bytes)
    } else {
        self.state.sync_state.buffer.extend(bytes);
        self.advance_sync_csi(handler, bytes.len());
        bytes.len()
    }
}
```

**ESU 到达**（`CSI ?2026l`）在缓冲中被检测到：

```rust
fn advance_sync_csi<H>(&mut self, handler: &mut H, new_bytes: usize) {
    // 在缓冲区的特定窗口内搜索 0x1B
    let search_buffer = &self.state.sync_state.buffer[start_offset..end_offset];
    let mut bsu_offset = None;
    for index in memchr::memchr_iter(0x1B, search_buffer).rev() {
        let offset = start_offset + index;
        let escape = &self.state.sync_state.buffer[offset..offset + SYNC_ESCAPE_LEN];
        if escape == BSU_CSI {  // \x1b[?2026h → 刷新超时
            self.state.sync_state.timeout.set_timeout(SYNC_UPDATE_TIMEOUT);
            bsu_offset = Some(offset);
        } else if escape == ESU_CSI {  // \x1b[?2026l → 终止同步更新
            self.stop_sync_internal(handler, bsu_offset);
            break;
        }
    }
}
```

**`stop_sync_internal` 的处理**：

```rust
fn stop_sync_internal<H>(&mut self, handler: &mut H, bsu_offset: Option<usize>) {
    let buffer = mem::take(&mut self.state.sync_state.buffer);
    let offset = bsu_offset.unwrap_or(buffer.len());
    // 把缓冲的字节重新喂给 Parser 正常解析
    let mut performer = Performer::new(&mut self.state, handler);
    self.parser.advance(&mut performer, &buffer[..offset]);
    self.state.sync_state.buffer = buffer;

    match bsu_offset {
        Some(bsu_offset) => {
            // 缓冲中有新 BSU：保留 BSU 之后的部分，继续同步模式
            let new_len = self.state.sync_state.buffer.len() - bsu_offset;
            self.state.sync_state.buffer.copy_within(bsu_offset.., 0);
            self.state.sync_state.buffer.truncate(new_len);
        },
        None => {
            // 没有 BSU：完全退出同步模式
            handler.unset_private_mode(NamedPrivateMode::SyncUpdate.into());
            self.state.sync_state.timeout.clear_timeout();
            self.state.sync_state.buffer.clear();
        },
    }
}
```

### 3.2 兜底一：缓冲溢出

```rust
// vte/src/ansi.rs
const SYNC_BUFFER_SIZE: usize = 0x20_0000;  // 2 MiB

fn advance_sync<H>(&mut self, handler: &mut H, bytes: &[u8]) -> usize {
    if self.state.sync_state.buffer.len() + bytes.len() >= SYNC_BUFFER_SIZE - 1 {
        self.stop_sync_internal(handler, None);  // 强制终止
        // 当前这批字节走正常解析
        let mut performer = Performer::new(&mut self.state, handler);
        self.parser.advance_until_terminated(&mut performer, bytes)
    } else {
        // 正常缓冲
        self.state.sync_state.buffer.extend(bytes);
        self.advance_sync_csi(handler, bytes.len());
        bytes.len()
    }
}
```

溢出时 `stop_sync_internal(handler, None)` 会：
1. 将已缓冲的字节全部正常解析
2. 调用 `handler.unset_private_mode(SyncUpdate)` 通知 Term 退出同步模式
3. 清空超时和缓冲

### 3.3 兜底二：超时到期

**超时设置**：BSU 到达时 `timeout.set_timeout(SYNC_UPDATE_TIMEOUT)`，即 150ms 后到期。

**EventLoop 中的超时检测**：

```rust
// alacritty_terminal/src/event_loop.rs:227
'event_loop: loop {
    let handler = state.parser.sync_timeout();
    let timeout = handler.sync_timeout()
        .map(|st| st.saturating_duration_since(Instant::now()));

    events.clear();
    if let Err(err) = self.poll.wait(&mut events, timeout) { ... }

    // poll 超时返回（无事件且无消息）
    if events.is_empty() && self.rx.peek().is_none() {
        state.parser.stop_sync(&mut *self.terminal.lock());
        self.event_proxy.send_event(Event::Wakeup);
        continue;
    }
    // ...
}
```

这里的机制是：

1. **`StdSyncHandler::sync_timeout()`** 返回 `Option<Instant>`（超时到期时刻）
2. **EventLoop 计算** `timeout = saturating_duration_since(Instant::now())`——剩余时间
3. **`poll.wait(events, timeout)`** 以剩余时间为超时参数阻塞等待 I/O 事件
4. **如果 150ms 内没有任何 I/O 事件或消息**：`events.is_empty()` 且 `self.rx.peek().is_none()`，说明同步更新超时
5. **调用 `state.parser.stop_sync()`**：强制终止同步更新，缓冲内容全部解析

注意：`StdSyncHandler::pending_timeout()` 只检查 `self.timeout.is_some()`，不检查是否过期。过期检查由 EventLoop 的 poll 超时机制完成。在 `Processor::advance` 中，`pending_timeout()` 返回 `true` 时进入 `advance_sync` 缓冲分支；过期后不会自动停止缓冲，需要等 EventLoop 调用 `stop_sync`。

### 3.4 兜底三：嵌套 BSU

如果同步更新期间又收到新的 BSU：

```rust
fn advance_sync_csi<H>(&mut self, handler: &mut H, new_bytes: usize) {
    // ...
    let mut bsu_offset = None;
    for index in memchr::memchr_iter(0x1B, search_buffer).rev() {
        let offset = start_offset + index;
        let escape = &self.state.sync_state.buffer[offset..offset + SYNC_ESCAPE_LEN];
        if escape == BSU_CSI {
            self.state.sync_state.timeout.set_timeout(SYNC_UPDATE_TIMEOUT);  // 刷新超时
            bsu_offset = Some(offset);
        } else if escape == ESU_CSI {
            self.stop_sync_internal(handler, bsu_offset);
            break;
        }
    }
}
```

搜索是**从后向前**的（`.rev()`），所以：
- 如果缓冲末尾同时有 BSU 和 ESU，先找到 ESU
- `stop_sync_internal(handler, bsu_offset)` 会将 BSU 之前的缓冲正常解析，BSU 之后的保留在缓冲中继续同步模式

### 3.5 Term 对 SyncUpdate 的处理

```rust
// alacritty_terminal/src/term/mod.rs:1992
NamedPrivateMode::SyncUpdate => (),
```

**Term 不对 SyncUpdate 做任何操作**——它只是个空操作。同步更新的实际效果体现在：
- Processor 层面：**延迟解析**，避免中间帧
- EventLoop 层面：**延迟 Wakeup 通知**

```rust
// alacritty_terminal/src/event_loop.rs:166
if state.parser.sync_bytes_count() < processed && processed > 0 {
    self.event_proxy.send_event(Event::Wakeup);
}
```

只有在同步更新期间处理的非同步字节数超过同步缓冲字节数时，才会发送 Wakeup。由于同步更新期间所有字节都进入缓冲、不被处理（`sync_bytes_count` 持续增长），Wakeup 通常不会发出，直到 `stop_sync` 清空缓冲。

### 3.6 三层兜底总结

| 兜底层 | 触发条件 | 代码位置 | 行为 |
|--------|---------|---------|------|
| ESU 正常终止 | 缓冲中检测到 `\x1b[?2026l` | `advance_sync_csi` | 回放缓冲、退出同步模式 |
| 缓冲溢出 | `buffer.len() + bytes.len() >= 2MiB` | `advance_sync` | 强制 `stop_sync_internal(None)` |
| 超时到期 | 150ms 内无新 I/O 事件 | EventLoop poll 超时 | `stop_sync()` + Wakeup |
| 嵌套 BSU | 缓冲中检测到新 `\x1b[?2026h` | `advance_sync_csi` | 刷新超时、保留 BSU 后数据 |
