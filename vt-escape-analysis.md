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
2. 所有与参数相关的 action 都会设置 `self.ignoring = true`：
   - **`action_csi_dispatch`**：最终 dispatch 时，若仍有未 push 的最后一个参数且 is_full
   - **`action_hook`**：DCS hook 时
   - **`action_collect`**：中间字节满时（见 §2.5）
   - **`action_subparam`**：收到冒号 `:` 时
   - **`action_param`**：收到分号 `;` 时
   - **`action_paramnext`**：收到数字字节时

3. `action_csi_dispatch` 的完整逻辑：
   ```rust
   fn action_csi_dispatch(&mut self, performer: &mut P, byte: u8) {
       if self.params.is_full() {
           self.ignoring = true;     // 参数总数超限，标记忽略
       } else {
           self.params.push(self.param);  // 推入最后一个未完成参数
       }
       performer.csi_dispatch(
           self.params(),
           self.intermediates(),
           self.ignoring,   // ← 以 Perform trait 的 `_ignore` 参数名传递
           byte as char,
       );
       self.state = State::Ground
   }
   ```

4. **Performer 层面直接丢弃**：在 `impl Perform for Performer` 的 `csi_dispatch` 中，该参数被重命名为 `has_ignored_intermediates`，并作为第一道守卫：
   ```rust
   fn csi_dispatch(
       &mut self, params: &Params, intermediates: &[u8],
       has_ignored_intermediates: bool, action: char,
   ) {
       if has_ignored_intermediates || intermediates.len() > 2 {
           unhandled!();   // debug 日志
           return;         // ← 直接返回，不进入具体指令匹配
       }
       // 下面才是具体的 ('H', [])、('m', []) 等 match
   }
   ```

所以**参数超限的序列根本不会到达 Handler**，它在 Performer 的 `csi_dispatch` 入口就被拦截了。之前文档说"仍然回调 Handler"是错误的——`has_ignored_intermediates` 直接触发 `unhandled!()` + `return`，不会往下走。

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

### 2.5 私有标记字节与真正中间字节的区分

CSI/DCS 序列中有两类特殊字节容易被混淆，它们共用 `intermediates[]` 存储但语义和状态机行为完全不同：

| 分类 | 字节范围 | 典型字符 | 语义 |
|------|---------|---------|------|
| **真正中间字节** | `0x20–0x2F` | 空格、`!`、`"`、`#`、`$`、`%`、`'`、`(`、`)` 等 | VT 规范的中间字节，出现在参数前或参数后 |
| **私有标记字节** | `0x3C–0x3F` | `<`、`=`、`>`、`?` | DEC 私有参数指示符，只能出现在参数**之前** |

**两者都通过 `action_collect` 写入同一个 `intermediates[]` 数组**，共享 `MAX_INTERMEDIATES = 2` 的上限，但在状态机中的处理路径完全不同。

#### 常量与存储

```rust
// vte/src/lib.rs
const MAX_INTERMEDIATES: usize = 2;

pub struct Parser {
    intermediates: [u8; MAX_INTERMEDIATES], // [u8; 2]，中间字节和私有标记共用
    intermediate_idx: usize,
    // ...
}
```

#### action_collect：两类字节的共同入口

```rust
// vte/src/lib.rs
fn action_collect(&mut self, byte: u8) {
    if self.intermediate_idx == MAX_INTERMEDIATES {
        self.ignoring = true;     // ← 第三次及之后 collect 触发忽略
    } else {
        self.intermediates[self.intermediate_idx] = byte;
        self.intermediate_idx += 1;
    }
}
```

无论是 `0x20–0x2F` 还是 `0x3C–0x3F`，只要被 `action_collect` 处理，就写入同一个数组。超限后统一设 `ignoring = true`。

#### 状态相关行为：同一字节在不同状态下命运完全不同

以 `?`（0x3F）为例，它在不同 CSI 状态下的处理：

**`CsiEntry` 状态**（刚收到 `ESC [`，尚未收到任何参数或中间字节）：

```rust
// vte/src/lib.rs:187
fn advance_csi_entry<P: Perform>(&mut self, performer: &mut P, byte: u8) {
    match byte {
        0x20..=0x2F => {
            self.action_collect(byte);
            self.state = State::CsiIntermediate   // 真正中间字节 → CsiIntermediate
        },
        0x30..=0x39 => {
            self.action_paramnext(byte);
            self.state = State::CsiParam
        },
        0x3A => {
            self.action_subparam();
            self.state = State::CsiParam
        },
        0x3B => {
            self.action_param();
            self.state = State::CsiParam
        },
        0x3C..=0x3F => {
            self.action_collect(byte);              // 私有标记字节 → action_collect
            self.state = State::CsiParam            // 然后转入 CsiParam（不是 CsiIgnore！）
        },
        0x40..=0x7E => self.action_csi_dispatch(performer, byte),
        // ...
    }
}
```

**关键**：`0x3C–0x3F` 在 `CsiEntry` 中是合法的！它们通过 `action_collect` 存入 `intermediates[]`，然后转入 `CsiParam`（因为私有标记后面通常跟数字参数）。这就是 `CSI ?2026h` 能正常工作的原因。

**`CsiParam` 状态**（已经收到过数字参数）：

```rust
// vte/src/lib.rs:238
fn advance_csi_param<P: Perform>(&mut self, performer: &mut P, byte: u8) {
    match byte {
        0x20..=0x2F => {
            self.action_collect(byte);              // 真正中间字节仍然合法
            self.state = State::CsiIntermediate
        },
        0x30..=0x39 => self.action_paramnext(byte),
        0x3A => self.action_subparam(),
        0x3B => self.action_param(),
        0x3C..=0x3F => self.state = State::CsiIgnore,  // 私有标记出现在参数之后→非法！
        0x40..=0x7E => self.action_csi_dispatch(performer, byte),
        // ...
    }
}
```

**关键**：同样的 `0x3C–0x3F`，在 `CsiParam` 中转入 `CsiIgnore`——私有标记**只能在参数之前出现**，参数之后出现则整个序列被丢弃。

**`CsiIntermediate` 状态**（已经收到过中间字节）：

```rust
// vte/src/lib.rs:227
fn advance_csi_intermediate<P: Perform>(&mut self, performer: &mut P, byte: u8) {
    match byte {
        0x20..=0x2F => self.action_collect(byte),          // 继续累积中间字节
        0x30..=0x3F => self.state = State::CsiIgnore,      // 任何非中间字节→忽略
        0x40..=0x7E => self.action_csi_dispatch(performer, byte),
        // ...
    }
}
```

**关键**：`CsiIntermediate` 中，`0x30–0x3F` **全部**转入 `CsiIgnore`（包括数字和私有标记），因为中间字节之后只允许更多中间字节或终结字节。

#### 完整的状态 × 字节行为表

| 字节范围 | CsiEntry | CsiParam | CsiIntermediate | CsiIgnore |
|---------|----------|----------|-----------------|-----------|
| `0x20–0x2F`（真正中间字节） | `action_collect` → CsiIntermediate | `action_collect` → CsiIntermediate | `action_collect` | 静默丢弃 |
| `0x30–0x39`（数字） | `action_paramnext` → CsiParam | `action_paramnext` | → **CsiIgnore** | 静默丢弃 |
| `0x3A`（`:`） | `action_subparam` → CsiParam | `action_subparam` | → **CsiIgnore** | 静默丢弃 |
| `0x3B`（`;`） | `action_param` → CsiParam | `action_param` | → **CsiIgnore** | 静默丢弃 |
| `0x3C–0x3F`（`< = > ?`） | `action_collect` → **CsiParam** ✅ | → **CsiIgnore** ❌ | → **CsiIgnore** ❌ | 静默丢弃 |
| `0x40–0x7E`（终结字节） | `action_csi_dispatch` | `action_csi_dispatch` | `action_csi_dispatch` | → Ground（不 dispatch） |

**核心规则**：
- 私有标记 `0x3C–0x3F` **只在 `CsiEntry` 中合法**——它们必须紧跟在 `ESC [` 之后、任何数字参数之前
- 真正中间字节 `0x20–0x2F` 在 `CsiEntry`、`CsiParam`、`CsiIntermediate` 中都合法
- 一旦进入 `CsiIntermediate`，除了更多中间字节和终结字节，**一切**都导致 `CsiIgnore`

#### 实例对比

**合法**：`CSI ?25h`（显示光标）

```
CsiEntry:
  0x3F '?' → action_collect(0x3F), intermediates=[0x3F], state=CsiParam
CsiParam:
  0x32 '2' → action_paramnext, param=2
  0x35 '5' → action_paramnext, param=25
  0x68 'h' → action_csi_dispatch
→ csi_dispatch(params=[25], intermediates=[0x3F], ignore=false, action='h')
→ Performer 匹配 ('h', [b'?']) → set_private_mode(ShowCursor)
```

**非法**：`CSI 25?h`（参数之后的 `?`）

```
CsiEntry:
  0x32 '2' → action_paramnext, param=2, state=CsiParam
CsiParam:
  0x35 '5' → action_paramnext, param=25
  0x3F '?' → state=CsiIgnore    ← 参数之后出现私有标记→忽略！
CsiIgnore:
  0x68 'h' → state=Ground       ← 终结字节，回到 Ground，不 dispatch
→ 整个序列被丢弃
```

**合法**：`CSI ?1$m`（虽然 vte 无对应 Handler，但解析路径合法）

```
CsiEntry:
  0x3F '?' → action_collect(0x3F), intermediates=[0x3F], state=CsiParam
CsiParam:
  0x31 '1' → action_paramnext, param=1
  0x24 '$' → action_collect(0x24), intermediates=[0x3F,0x24], state=CsiIntermediate
CsiIntermediate:
  0x6D 'm' → action_csi_dispatch
→ csi_dispatch(params=[1], intermediates=[0x3F,0x24], ignore=false, action='m')
→ Performer 无匹配 → unhandled!()
```

#### CsiIgnore 态的处理

```rust
// vte/src/lib.rs:216
fn advance_csi_ignore<P: Perform>(&mut self, performer: &mut P, byte: u8) {
    match byte {
        0x00..=0x17 | 0x19 | 0x1C..=0x1F => performer.execute(byte), // C0 控制仍执行
        0x20..=0x3F => (),       // 中间字节/参数/私有标记全部静默丢弃
        0x40..=0x7E => self.state = State::Ground, // 终结字节：回 Ground，**不 dispatch**
        0x7F => (),              // DEL 忽略
        _ => self.anywhere(performer, byte),        // CAN/SUB/ESC 等全局处理
    }
}
```

#### 丢弃路径总结

| 路径 | 触发条件 | 丢弃位置 |
|------|---------|---------|
| 1 | `action_collect` 超过 2 次（`ignoring=true`），最终走到终结字节 `0x40–0x7E` | `action_csi_dispatch` 仍被调用 → Performer.csi_dispatch：`if has_ignored_intermediates { return; }` |
| 2 | `CsiParam` 收到 `0x3C–0x3F`（私有标记出现在参数之后）→ `CsiIgnore` | `advance_csi_ignore` 中终结字节只回 Ground，**完全不调用** `csi_dispatch` |
| 3 | `CsiIntermediate` 收到 `0x30–0x3F`（数字或私有标记出现在中间字节之后）→ `CsiIgnore` | 同上 |

DCS 序列（`DcsEntry` / `DcsParam` / `DcsIntermediate` / `DcsIgnore`）遵循完全相同的状态机逻辑，其中 `DcsEntry` 对 `0x3C–0x3F` 同样是 `action_collect` → `DcsParam`。

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
// vte/src/ansi.rs:389
const SYNC_ESCAPE_LEN: usize = 8;
const BSU_CSI: [u8; SYNC_ESCAPE_LEN] = *b"\x1b[?2026h";
const ESU_CSI: [u8; SYNC_ESCAPE_LEN] = *b"\x1b[?2026l";

fn advance_sync_csi<H>(&mut self, handler: &mut H, new_bytes: usize)
where
    H: Handler,
{
    // 精确计算搜索窗口，处理跨 chunk 的转义序列
    let buffer_len = self.state.sync_state.buffer.len();

    // start_offset：向前回退 7 字节，覆盖"上一批末尾 + 新字节开头"跨边界的情况
    let start_offset = (buffer_len - new_bytes).saturating_sub(SYNC_ESCAPE_LEN - 1);

    // end_offset：最后一个可以容纳完整 8 字节转义序列的起始位置
    let end_offset = buffer_len.saturating_sub(SYNC_ESCAPE_LEN - 1);

    let search_buffer = &self.state.sync_state.buffer[start_offset..end_offset];

    // 从后向前搜索 0x1B（ESC）
    let mut bsu_offset = None;
    for index in memchr::memchr_iter(0x1B, search_buffer).rev() {
        let offset = start_offset + index;
        let escape = &self.state.sync_state.buffer[offset..offset + SYNC_ESCAPE_LEN];

        if escape == BSU_CSI {
            self.state.sync_state.timeout.set_timeout(SYNC_UPDATE_TIMEOUT);
            bsu_offset = Some(offset);
        } else if escape == ESU_CSI {
            self.stop_sync_internal(handler, bsu_offset);
            break;
        }
    }
}
```

**精确匹配边界的设计意图**：

- **`start_offset`** = `(buffer_len - new_bytes).saturating_sub(7)`
  - `buffer_len - new_bytes` 是新字节写入前的长度，即新字节的起始位置
  - 向前回退 7 字节（`SYNC_ESCAPE_LEN - 1`），确保能检测到**跨批次边界**的 BSU/ESU
  - 例如：上一批结尾有 3 字节 `\x1b[?`，本批开头 5 字节 `2026h`，合起来就是完整的 BSU
  - `saturating_sub` 保证不会下溢到负数

- **`end_offset`** = `buffer_len.saturating_sub(7)`
  - 从缓冲区末尾回退 7 字节，保证搜索到的每个 ESC 后面都有至少 7 字节可以凑成完整的 8 字节转义序列
  - 换句话说：搜索 `ESC in buffer[start_offset .. end_offset]`，对每个匹配的 ESC 执行 `buffer[offset .. offset+8]` 比较是安全的，不会越界

- **搜索策略**：`memchr_iter(0x1B, search_buffer).rev()` — 从后向前找 ESC
  - 因为 BSU 和 ESU 都是 8 字节固定序列，找到 ESC 后检查后 7 字节即可
  - 从后向前可以先找到最近（末尾）的 ESU/BSU，更符合实际时序

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

### 3.6 四层兜底总结

| 兜底层 | 触发条件 | 代码位置 | 行为 |
|--------|---------|---------|------|
| ESU 正常终止 | 精确匹配 `\x1b[?2026l`（8 字节） | `advance_sync_csi` | 回放缓冲、退出同步模式 |
| 精确匹配边界 | 搜索窗口 `[start_offset, end_offset)`，`memchr` 从后向前搜 ESC | `advance_sync_csi` | start_offset 回退 7 字节覆盖跨 chunk，end_offset 回退 7 字节保证不越界 |
| 缓冲溢出 | `buffer.len() + bytes.len() >= 2MiB` | `advance_sync` | 强制 `stop_sync_internal(None)`，当前批次走正常解析 |
| 超时到期 | 150ms 内无新 I/O 事件 | EventLoop poll 超时 | `stop_sync()` + Wakeup |
| 嵌套 BSU | 精确匹配 `\x1b[?2026h`（8 字节） | `advance_sync_csi` | 刷新超时、保留 BSU 后数据继续缓冲 |

---

## 4. DCS 旁路：hook / put / unhook 与 CSI 的完整对比

DCS（Device Control String，`ESC P ... ST` 或 `ESC P ... ESC \`）是另一类转义序列，用于终端与设备之间传递任意数据。DCS 与 CSI 共享相似的参数解析阶段（Entry/Param/Intermediate），但在**旁路阶段（Passthrough）、Ignore 行为、C0 控制处理**上存在本质差异。

### 4.1 DCS 生命周期概览

```
ESC P (0x50)  →  DcsEntry
    │  (参数/中间字节/私有标记解析，与 CSI 完全同构)
    ▼
action_hook  →  DcsPassthrough  ← 旁路阶段：逐字节 put()
    │  (接收 ST = ESC \ 或 CAN/SUB = 0x18/0x1A 或 8-bit ST = 0x9C)
    ▼
performer.unhook()  →  Ground / Escape
```

### 4.2 DCS 与 CSI 的状态 × 字节行为对比

两者的 Entry/Param/Intermediate 阶段在参数和中间字节的处理上几乎一致，但在 **C0 控制**、**Ignore 实现**、**终结行为**上差异显著。

#### C0 控制处理对比

C0 控制指 `0x00–0x17 | 0x19 | 0x1C–0x1F`（NUL、BEL、BS、HT、LF、CR、ESC 除外的 C0 控制字符）。

| 状态 | CSI 行为 | DCS 行为 |
|------|---------|---------|
| CsiEntry / DcsEntry | `performer.execute(byte)` ✅ | **`()`**（静默丢弃）❌ |
| CsiParam / DcsParam | `performer.execute(byte)` ✅ | **`()`**（静默丢弃）❌ |
| CsiIntermediate / DcsIntermediate | `performer.execute(byte)` ✅ | **`()`**（静默丢弃）❌ |
| CsiIgnore | `performer.execute(byte)` ✅ | **交给 anywhere()**（仅 CAN/SUB/ESC 生效） |
| CsiPassthrough（不存在） | — | — |
| DcsPassthrough | — | `performer.put(byte)`（**字节被当作数据传递**） |

**关键**：CSI 在所有非 Ignore 状态下都执行 C0 控制，但 DCS 在 Entry/Param/Intermediate 阶段**完全静默丢弃 C0 控制**。进入 `DcsPassthrough` 后，C0 控制**不 execute 而是 put**——作为数据字节的一部分传给 `handler.put(byte)`。

DCS Passthrough 中的 C0 控制处理：

```rust
// vte/src/lib.rs
fn advance_dcs_passthrough<P: Perform>(&mut self, performer: &mut P, byte: u8) {
    match byte {
        0x00..=0x17 | 0x19 | 0x1C..=0x7E => performer.put(byte),  // ← C0 控制被 put，不是 execute
        0x18 | 0x1A => {                    // CAN (0x18) / SUB (0x1A)
            performer.unhook();             // 先终止 DCS
            performer.execute(byte);        // 再执行 CAN/SUB
            self.state = State::Ground;
        },
        0x1B => {                           // ESC
            performer.unhook();             // 先终止 DCS
            self.reset_params();
            self.state = State::Escape;     // 进入 Escape（准备接收 ST 或下一个序列）
        },
        0x7F => (),                         // DEL 丢弃
        0x9C => {                           // 8-bit ST (String Terminator)
            performer.unhook();
            self.state = State::Ground;
        },
        _ => (),
    }
}
```

#### Ignore 状态处理对比

**CsiIgnore** 有独立的 advance 方法，行为精细：

```rust
// vte/src/lib.rs
fn advance_csi_ignore<P: Perform>(&mut self, performer: &mut P, byte: u8) {
    match byte {
        0x00..=0x17 | 0x19 | 0x1C..=0x1F => performer.execute(byte), // C0 控制仍执行
        0x20..=0x3F => (),       // 参数/中间字节静默丢弃
        0x40..=0x7E => self.state = State::Ground, // 终结字节：回 Ground，不 dispatch
        0x7F => (),
        _ => self.anywhere(performer, byte),
    }
}
```

**DcsIgnore** 没有独立的 advance 方法，在 `change_state` 中直接交给 `anywhere()`：

```rust
// vte/src/lib.rs:168
fn change_state<P: Perform>(&mut self, performer: &mut P, byte: u8) {
    match self.state {
        // ...
        State::DcsIgnore => self.anywhere(performer, byte),  // ← 直接 anywhere！
        // ...
    }
}

fn anywhere<P: Perform>(&mut self, performer: &mut P, byte: u8) {
    match byte {
        0x18 | 0x1A => {            // CAN / SUB
            performer.execute(byte);
            self.state = State::Ground;
        },
        0x1B => {                   // ESC
            self.reset_params();
            self.state = State::Escape;
        },
        _ => (),                    // 所有其他字节静默丢弃
    }
}
```

**Ignore 差异总结**：

| 方面 | CsiIgnore | DcsIgnore |
|------|-----------|-----------|
| 处理方式 | 独立 `advance_csi_ignore` 方法 | `change_state` 中直接调 `anywhere()` |
| C0 控制 | `0x00-0x1F` 全部 execute | 仅 CAN (0x18)/SUB (0x1A) execute，其余丢弃 |
| `0x20-0x3F` | 静默丢弃 | 静默丢弃（anywhere 默认分支） |
| `0x40-0x7E`（终结字节） | 回 Ground | **不回 Ground！** anywhere 对这些字节返回 `()` |
| ESC | anywhere 分支 → Escape | anywhere 分支 → Escape |
| DEL (0x7F) | 丢弃 | anywhere 对 0x7F 返回 `()` → 丢弃 |
| 8-bit ST (0x9C) | 丢弃 | anywhere 对 0x9C 返回 `()` → 丢弃 |

**关键发现**：DcsIgnore 下收到 `0x40-0x7E`（终结字节）**不会**回到 Ground——DCS 的 Ignore 状态没有"终结字节跳出"机制，只有 CAN/SUB（回 Ground）或 ESC（回 Escape）能跳出。这是因为 DCS 的正确终结是 ST（`ESC \`），而不是单个终结字符。

### 4.3 hook / put / unhook 回调详解

DCS 旁路阶段有三个 Perform trait 回调：

```rust
// vte/src/lib.rs - Perform trait
fn hook(&mut self, _params: &Params, _intermediates: &[u8], _ignore: bool, _action: char) {}
fn put(&mut self, _byte: u8) {}
fn unhook(&mut self) {}
```

Performer 中的实现（`vte/src/ansi.rs`）全部是 **unhandled debug 日志**：

```rust
// vte/src/ansi.rs:1311
fn hook(&mut self, params: &Params, intermediates: &[u8], ignore: bool, action: char) {
    debug!(
        "[unhandled hook] params={:?}, ints: {:?}, ignore: {:?}, action: {:?}",
        params, intermediates, ignore, action
    );
}

fn put(&mut self, byte: u8) {
    debug!("[unhandled put] byte={:?}", byte);
}

fn unhook(&mut self) {
    debug!("[unhandled unhook]");
}
```

Alacritty 的 Term **没有实现 DCS 处理**——所有 DCS 序列在 Performer 层面就被丢弃。

#### hook（DCS 起始）

`action_hook` 触发时机：DcsEntry / DcsParam / DcsIntermediate 中收到终结字节 `0x40-0x7E`。

```rust
// vte/src/lib.rs:465
fn action_hook<P: Perform>(&mut self, performer: &mut P, byte: u8) {
    if self.params.is_full() {
        self.ignoring = true;
    } else {
        self.params.push(self.param);
    }
    performer.hook(self.params(), self.intermediates(), self.ignoring, byte as char);
    self.state = State::DcsPassthrough;  // ← hook 之后自动进入旁路
}
```

hook 之后状态机转入 `DcsPassthrough`，后续字节开始逐字节 `put`。

#### put（DCS 旁路数据）

`advance_dcs_passthrough` 中，`0x00-0x17 | 0x19 | 0x1C-0x7E` 全部走 `performer.put(byte)`。这包括：
- 可打印字符 `0x20-0x7E`
- C0 控制字符 `0x00-0x17 | 0x19 | 0x1C-0x1F`（**不是 execute，是 put**）

#### unhook（DCS 终结）

三个终结路径：

| 终结方式 | 字节 | 行为 |
|---------|------|------|
| CAN/SUB | `0x18` / `0x1A` | `unhook()` → `execute(byte)` → Ground |
| ESC → ST | `0x1B` → `0x5C` | DCS Passthrough 中 `0x1B` → `unhook()` → reset_params → Escape 状态；然后 Escape 状态中 `0x5C` (`\`) 触发 `esc_dispatch`（空操作 ST）→ Ground |
| 8-bit ST | `0x9C` | `unhook()` → Ground |

### 4.4 DCS 完整实例追踪

合法序列：`ESC P 0;1|data\x1b\`

```
Ground:
  0x1B ESC → reset_params(), state=Escape
Escape:
  0x50 P → reset_params(), state=DcsEntry
DcsEntry:
  0x30 '0' → action_paramnext, param=0, state=DcsParam
DcsParam:
  0x3B ';' → action_param, params=[0], state=DcsParam
  0x31 '1' → action_paramnext, param=1
  0x7C '|' → action_hook(params=[0,1], intermediates=[], ignore=false, action='|')
            → state=DcsPassthrough
DcsPassthrough:
  0x64 'd' → performer.put(0x64)
  0x61 'a' → performer.put(0x61)
  0x74 't' → performer.put(0x74)
  0x61 'a' → performer.put(0x61)
  0x1B ESC → performer.unhook(), reset_params(), state=Escape
Escape:
  0x5C '\' → performer.esc_dispatch([], false, 0x5C)  // ST 空操作
            → state=Ground
```

非法序列：`ESC P 0?1|...`（私有标记出现在参数之后）

```
Escape → DcsEntry → DcsParam:
  0x30 '0' → action_paramnext
  0x3F '?' → state=DcsIgnore  ← 参数后的私有标记非法！
DcsIgnore (走 anywhere):
  0x31 '1' → anywhere → _ => ()  ← 静默丢弃
  0x7C '|' → anywhere → _ => ()  ← 静默丢弃（**不会回 Ground！**）
  0x64 'd' → anywhere → _ => ()  ← 继续丢弃
  0x1B ESC → anywhere → reset_params(), state=Escape  ← 只有 ESC 能跳出 DcsIgnore
```

### 4.5 完整差异对比表

| 维度 | CSI | DCS |
|------|-----|-----|
| 入口 | ESC `[` (0x5B) | ESC `P` (0x50) |
| 参数/中间字节/私有标记解析 | 与 DCS 完全同构 | 与 CSI 完全同构 |
| Entry/Param/Intermediate 的 C0 控制 | `performer.execute(byte)` | **静默丢弃** |
| 旁路阶段 | 不存在（dispatch 即终结） | `DcsPassthrough` 状态，逐字节 `put()` |
| Passthrough 的 C0 控制 | — | **`put(byte)`**（不 execute） |
| Passthrough 的可打印字符 | — | `put(byte)` |
| Ignore 实现 | 独立 `advance_csi_ignore`，精细处理 | 直接走 `anywhere()`，仅 CAN/SUB/ESC 有意义 |
| Ignore 中终结字节 `0x40-0x7E` | 回 Ground | **不回 Ground**，静默丢弃 |
| Ignore 中 C0 控制 | 全部 execute | 仅 CAN/SUB execute |
| 终结方式 | 单个终结字节 `0x40-0x7E` → `csi_dispatch` → Ground | ST（ESC `\`）/ CAN / SUB / 0x9C → `unhook` → Ground |
| 终结回调 | `csi_dispatch`（一次性，带参数） | `hook` + 多次 `put` + `unhook`（流式） |
| Term 实现 | 全量 Handler trait，30+ 方法 | 全部未实现，Performer 层 debug 日志丢弃 |
