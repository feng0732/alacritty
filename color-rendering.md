# Alacritty 颜色主题到渲染输出链路分析

## 概述

Alacritty 的颜色系统从配置到最终 GPU 渲染输出，经历了 **6 个核心层次** 的转换。同时，单元格样式（Flags）分为两条并行的渲染管线——**文本绘制管线**和**矩形绘制管线**。

```
配置层 (config::Colors)
    ↓ 转换
显示颜色列表 (display::color::List)
    ↓ 结合
终端调色板 (term::color::Colors) + 单元格 (term::cell::Cell)
    ↓ 计算
可渲染单元格 (display::content::RenderableCell)
    ├─────────────────────────────────────────────────────┐
    ↓ 文本绘制管线                           ↓ 矩形绘制管线
GPU 文本实例数据 (InstanceData)             线条收集 (RenderLines)
    ↓                                        ↓
文本着色器 (text.f.glsl)                    矩形数据 (RenderRect)
                                             ↓
                                          矩形着色器 (rect.f.glsl)
```

---

## 第一层：颜色配置 (config/color.rs)

### 核心结构体：`Colors`

位于 [alacritty/src/config/color.rs](alacritty/src/config/color.rs#L8-L24)

这是用户配置层的颜色主题入口，从配置文件 (alacritty.toml) 反序列化而来。

```rust
pub struct Colors {
    pub primary: PrimaryColors,          // 主色（前景/背景）
    pub cursor: InvertedCellColors,      // 光标颜色
    pub vi_mode_cursor: InvertedCellColors, // Vi 模式光标颜色
    pub selection: InvertedCellColors,   // 选区颜色
    pub normal: NormalColors,            // 标准 ANSI 8 色
    pub bright: BrightColors,            // 亮色 8 色
    pub dim: Option<DimColors>,          // 暗色 8 色（可选）
    pub indexed_colors: Vec<IndexedColor>, // 自定义索引色（16-255）
    pub search: SearchColors,            // 搜索高亮颜色
    pub hints: HintColors,               // 提示（Hint）颜色
    // ...
}
```

### 关键颜色类型

1. **`Rgb`** - 具体的 RGB 颜色值
   - 字段：`r: u8`, `g: u8`, `b: u8`
   - 支持 `Mul<f32>` 运算（用于计算 dim 颜色）

2. **`CellRgb`** - 单元格引用色
   ```rust
   pub enum CellRgb {
       CellForeground,  // 引用单元格前景色
       CellBackground,  // 引用单元格背景色
       Rgb(Rgb),        // 具体 RGB 值
   }
   ```
   用于光标、选区、搜索等可以"反转"或"继承"单元格颜色的场景。

---

## 第二层：显示颜色列表 (display/color.rs)

### 核心结构体：`List`

位于 [alacritty/src/display/color.rs](alacritty/src/display/color.rs#L18-L19)

```rust
pub struct List([Rgb; COUNT]);  // COUNT = 269
```

这是渲染时使用的颜色查找表，包含 269 个颜色槽位。

### 颜色索引分布

| 索引范围 | 描述 |
|---------|------|
| 0..16 | 命名 ANSI 颜色（normal + bright，共 16 个） |
| 16..232 | 6×6×6 颜色立方体（216 色） |
| 232..256 | 灰度渐变（24 级） |
| 256 | Foreground（前景色） |
| 257 | Background（背景色） |
| 258 | Cursor（光标色） |
| 259..267 | Dim 暗色（8 个，半开区间不含右端） |
| 267 | Bright foreground（亮前景） |
| 268 | Dim background（暗背景） |

### 从配置到 List 的转换

位于 [List::from](alacritty/src/display/color.rs#L21-L32)：

```rust
impl From<&'_ Colors> for List {
    fn from(colors: &Colors) -> List {
        let mut list = List([Rgb::default(); COUNT]);
        list.fill_named(colors);     // 填充配置直接定义的命名色（不含 Cursor / DimBackground）
        list.fill_cube(colors);      // 填充颜色立方体（16-231）
        list.fill_gray_ramp(colors); // 填充灰度（232-255）
        list
    }
}
```

### fill_named 实际填充的槽位

位于 [fill_named](alacritty/src/display/color.rs#L34-L89)，实际写入的命名色分组如下：

| 分组 | 命名色 | 数量 |
|------|--------|-----|
| Normals | Black, Red, Green, Yellow, Blue, Magenta, Cyan, White | 8 |
| Brights | BrightBlack..BrightWhite | 8 |
| Bright foreground | BrightForeground | 1 |
| Foreground/Background | Foreground, Background | 2 |
| Dims | DimForeground, DimBlack..DimWhite | 9 |

**未被 fill_named 填充的命名色槽位**：

| 槽位索引 | 命名色 | 说明 |
|---------|--------|------|
| 258 | Cursor | 光标颜色不进入配置转换；显示层使用时优先读取终端层 `term.colors[Cursor]`，没有则回退到配置 `config.colors.cursor` |
| 268 | DimBackground | 终端调色板注释中预留的暗背景槽位；当前代码没有在 `fill_named` 中写入，也没有在渲染路径中引用 |

**Dim 颜色计算策略**：
- 如果配置了 `dim` 字段，使用配置值
- 否则，使用 `DIM_FACTOR = 0.66` 乘以 normal 颜色自动计算

---

## 第三层：终端调色板与单元格

### 3.1 终端调色板：`term::color::Colors`

位于 [alacritty_terminal/src/term/color.rs](alacritty_terminal/src/term/color.rs#L8-L21)

```rust
pub struct Colors([Option<Rgb>; COUNT]);  // COUNT = 269
```

终端调色板的 269 个槽位定义（来自源码注释）：

| 索引范围 | 描述 |
| -------- | ---- |
| 0..16 | 命名 ANSI 颜色 |
| 16..232 | 颜色立方体 |
| 233..256 | 灰度渐变 |
| 256 | Foreground |
| 257 | Background |
| 258 | Cursor |
| 259..267 | Dim 颜色（半开区间，8 个槽位） |
| 267 | Bright foreground |
| 268 | Dim background |

与 `display::color::List` 结构相同，但存储的是 `Option<Rgb>`：
- `None`：使用默认颜色（回退到显示层 `display.colors` 即配置值）
- `Some(rgb)`：被 OSC 4/10/11 等转义序列动态修改过的颜色

这样设计的目的是让终端运行时可以动态修改颜色，同时保留配置中的默认值作为回退。

**终端调色板与显示列表的槽位差异**：
- **258 (Cursor)**：两端都存在，但显示层 `fill_named` 不填充它。光标颜色在终端层通过 OSC 动态设置（[alacritty_terminal/src/term/mod.rs](alacritty_terminal/src/term/mod.rs#L1666)），显示层使用时优先读取终端层的值，没有则回退到配置 `config.colors.cursor`（[alacritty/src/display/content.rs](alacritty/src/display/content.rs#L120-L121)）。
- **268 (DimBackground)**：仅在终端调色板注释中预留；当前代码没有在显示层 `fill_named` 中写入，也没有在渲染路径中引用。

### 3.2 终端单元格：`term::cell::Cell`

位于 [alacritty_terminal/src/term/cell.rs](alacritty_terminal/src/term/cell.rs#L134-L140)

```rust
pub struct Cell {
    pub c: char,           // 字符
    pub fg: Color,         // 前景色
    pub bg: Color,         // 背景色
    pub flags: Flags,      // 样式标志位
    pub extra: Option<Arc<CellExtra>>, // 额外存储（零宽字符、下划线颜色、超链接）
}
```

### 3.3 颜色枚举：`vte::ansi::Color`

来自外部 `vte` crate，表示终端单元格的颜色类型：

```rust
pub enum Color {
    Named(NamedColor),  // 命名颜色索引（Foreground, Background, Black, Red, ...）
    Indexed(u8),        // 256 色索引
    Spec(Rgb),          // 24 位真彩色（直接 RGB 值）
}
```

这是颜色的**逻辑表示**，不包含具体的 RGB 值，需要在渲染时解析。

### 3.4 单元格样式标志：`Flags`

位于 [alacritty_terminal/src/term/cell.rs](alacritty_terminal/src/term/cell.rs#L12-L36)

使用 bitflags 表示单元格的文本样式。Flags 按渲染管线分组如下：

#### 文本绘制管线处理的 Flags：

| 标志位 | 含义 | 处理方式 |
|-------|------|---------|
| `BOLD` | 粗体 | 选择粗体字体 |
| `ITALIC` | 斜体 | 选择斜体字体 |
| `BOLD_ITALIC` | 粗斜体 | 选择粗斜体字体 |
| `HIDDEN` | 隐藏 | 替换为空格字符 |
| `WIDE_CHAR` | 宽字符 | 标记给着色器 |
| `WIDE_CHAR_SPACER` | 宽字符占位 | 跳过渲染 |
| `DIM` | 暗淡 | 前景色乘以 DIM_FACTOR |
| `INVERSE` | 反转 | 交换 fg/bg |

#### 矩形绘制管线处理的 Flags：

| 标志位 | 含义 | 处理方式 |
|-------|------|---------|
| `UNDERLINE` | 下划线 | 收集为 RenderLine → RenderRect |
| `DOUBLE_UNDERLINE` | 双下划线 | 收集为两条 RenderLine |
| `STRIKEOUT` | 删除线 | 收集为 RenderLine |
| `UNDERCURL` | 波浪下划线 | 收集为 RenderLine（波浪线） |
| `DOTTED_UNDERLINE` | 点状下划线 | 收集为 RenderLine（点线） |
| `DASHED_UNDERLINE` | 虚线下划线 | 收集为 RenderLine（虚线） |
| `ALL_UNDERLINES` | 所有下划线 | 组合掩码 |

---

## 第四层：可渲染内容 (display/content.rs)

这一层是**颜色计算的核心**，将逻辑颜色（Color 枚举）转换为具体的 RGB 值。

### 4.1 `RenderableContent`

位于 [alacritty/src/display/content.rs](alacritty/src/display/content.rs#L27-L38)

```rust
pub struct RenderableContent<'a> {
    terminal_content: TerminalContent<'a>,  // 终端内容迭代器
    config: &'a UiConfig,                   // UI 配置
    colors: &'a List,                       // 显示颜色列表
    // ...
}
```

它是一个迭代器，产出 `RenderableCell`。

### 4.2 `RenderableCell`

位于 [alacritty/src/display/content.rs](alacritty/src/display/content.rs#L188-L198)

```rust
pub struct RenderableCell {
    pub character: char,
    pub point: Point<usize>,
    pub fg: Rgb,        // 已计算的前景 RGB
    pub bg: Rgb,        // 已计算的背景 RGB
    pub bg_alpha: f32,  // 背景透明度
    pub underline: Rgb, // 下划线颜色
    pub flags: Flags,
    // ...
}
```

### 4.3 颜色计算流程

颜色计算发生在 `RenderableCell::new` 中（[content.rs:209-299](alacritty/src/display/content.rs#L209-L299)），按以下顺序进行：

#### 步骤 1：基础前景色计算

[compute_fg_rgb](alacritty/src/display/content.rs#L326-L370)

```rust
fn compute_fg_rgb(content: &RenderableContent<'_>, fg: Color, flags: Flags) -> Rgb {
    match fg {
        Color::Spec(rgb) => {
            // 真彩色：直接使用，DIM 时乘以 DIM_FACTOR
        }
        Color::Named(ansi) => {
            // 命名色：考虑 BOLD/DIM 标志
            // - draw_bold_text_with_bright_colors + BOLD → 亮色
            // - DIM → 暗色
        }
        Color::Indexed(idx) => {
            // 索引色：同样考虑 BOLD/DIM 调整索引
        }
    }
}
```

**优先级逻辑**：
1. 如果是真彩色 (`Color::Spec`)，直接使用 RGB 值（DIM 时变暗）
2. 如果是命名色 + BOLD + `draw_bold_text_with_bright_colors` → 使用亮色
3. 如果是命名色 + DIM → 使用暗色
4. 索引色 0-7 + BOLD → 偏移 +8（变为亮色）
5. 索引色 8-15 + DIM（且不 draw_bold） → 偏移 -8

#### 步骤 2：基础背景色计算

[compute_bg_rgb](alacritty/src/display/content.rs#L373-L380)

```rust
fn compute_bg_rgb(content: &RenderableContent<'_>, bg: Color) -> Rgb {
    match bg {
        Color::Spec(rgb) => rgb.into(),
        Color::Named(ansi) => content.color(ansi as usize),
        Color::Indexed(idx) => content.color(idx as usize),
    }
}
```

背景色计算相对简单，不考虑 BOLD/DIM 标志。

#### 步骤 3：INVERSE 反转

如果单元格有 `INVERSE` 标志，交换前景和背景色，并将背景 alpha 设为 1.0。

#### 步骤 4：选区/搜索/Hint 颜色叠加

按优先级从低到高：
1. **Hint 高亮** - 键盘提示标签
2. **选区颜色** - 文本选中
3. **搜索匹配** - 搜索结果高亮

使用 `compute_cell_rgb` 函数应用 `CellRgb` 颜色：

```rust
fn compute_cell_rgb(cell_fg, cell_bg, bg_alpha, fg: CellRgb, bg: CellRgb) {
    let old_fg = replace(cell_fg, fg.color(*cell_fg, *cell_bg));
    *cell_bg = bg.color(old_fg, *cell_bg);
    // 如果背景不是 CellBackground，设置 alpha = 1.0
}
```

这里的 `CellRgb::color(foreground, background)` 方法实现了颜色的"继承"语义：
- `CellForeground` → 使用当前前景色
- `CellBackground` → 使用当前背景色
- `Rgb(rgb)` → 使用具体 RGB 值

#### 步骤 5：透明度计算

[compute_bg_alpha](alacritty/src/display/content.rs#L388-L396)

- 如果背景是 `Color::Named(NamedColor::Background)`（默认背景色），alpha = 0（不绘制背景）
- 如果启用 `transparent_background_colors`，使用窗口透明度
- 否则 alpha = 1.0

#### 步骤 6：下划线颜色

优先使用单元格的下划线颜色（如果设置了），否则使用前景色。

---

## 第五层：GPU 渲染批处理 - 双管线并行

### 5.0 绘制总流程总览

位于 [alacritty/src/display/mod.rs](alacritty/src/display/mod.rs#L775-L1009)

```rust
pub fn draw<T: EventListener>(&mut self, ...) {
    // 1. 生成可渲染内容
    let mut content = RenderableContent::new(...);
    let mut grid_cells: Vec<RenderableCell> = content.collect();
    
    // 2. 清屏
    self.renderer.clear(background_color, config.window_opacity());
    
    // 3. 创建线条收集器
    let mut lines = RenderLines::new();
    
    // 4. 文本绘制 + 线条收集（并行进行）
    let cells = grid_cells.into_iter().map(|cell| {
        lines.update(&cell);  // 收集下划线/删除线等线条样式
        cell              // 传递给文本绘制
    });
    self.renderer.draw_cells(&size_info, glyph_cache, cells);
    
    // 5. 线条转换为矩形
    let mut rects = lines.rects(&metrics, &size_info);
    
    // 6. 追加光标矩形
    rects.extend(cursor.rects(&size_info, config.cursor.thickness()));
    
    // 7. 追加视觉响铃矩形
    // 8. 追加消息栏矩形
    
    // 9. 一次性绘制所有矩形
    self.renderer.draw_rects(&size_info, &metrics, rects);
    
    // 10. 交换缓冲区
    self.swap_buffers();
}
```

### 5.1 文本渲染管线

#### 5.1.1 渲染器架构

位于 [alacritty/src/renderer/mod.rs](alacritty/src/renderer/mod.rs#L89-L93)

```rust
pub struct Renderer {
    text_renderer: TextRendererProvider,  // Gles2 或 Glsl3
    rect_renderer: RectRenderer,          // 矩形渲染（光标、下划线等）
}
```

#### 5.1.2 文本渲染流程

```
Renderer::draw_cells
    → TextRenderer::draw_cells
        → TextRenderApi::draw_cell
            → GlyphCache::get（获取字形）
            → TextRenderApi::add_render_item
                → Batch::add_item（写入 InstanceData）
                → 批次满了 render_batch
```

#### 5.1.3 文本渲染中 Flags 处理

位于 [alacritty/src/renderer/text/mod.rs](alacritty/src/renderer/text/mod.rs#L134-L173)

```rust
fn draw_cell(&mut self, mut cell: RenderableCell, glyph_cache: &mut GlyphCache, size_info: &SizeInfo) {
    // 根据 BOLD/ITALIC 选择字体
    let font_key = match cell.flags & Flags::BOLD_ITALIC {
        Flags::BOLD_ITALIC => glyph_cache.bold_italic_key,
        Flags::ITALIC => glyph_cache.italic_key,
        Flags::BOLD => glyph_cache.bold_key,
        _ => glyph_cache.font_key,
    };
    
    // HIDDEN 字符替换为空格
    let hidden = cell.flags.contains(Flags::HIDDEN);
    if cell.character == '\t' || hidden {
        cell.character = ' ';
    }
    
    // 获取字形并添加到批次
    let glyph_key = GlyphKey { font_key, size: glyph_cache.font_size, character: cell.character };
    let glyph = glyph_cache.get(glyph_key, self, true);
    self.add_render_item(&cell, &glyph, size_info);
}
```

**文本渲染中处理的 Flags**：
- `BOLD`/`ITALIC`/`BOLD_ITALIC` → 选择不同字体
- `HIDDEN` → 替换为空格
- `WIDE_CHAR` → 标记到 `RenderingGlyphFlags::WIDE_CHAR`

**文本渲染中不处理的 Flags**：
- `UNDERLINE`/`STRIKEOUT`/`UNDERCURL` 等线条样式 → 由矩形管线处理

#### 5.1.4 GPU 实例数据：`InstanceData`

位于 [alacritty/src/renderer/text/glsl3.rs](alacritty/src/renderer/text/glsl3.rs#L284-L318)

```rust
#[repr(C)]
struct InstanceData {
    col: u16, row: u16,        // 单元格坐标
    left: i16, top: i16,       // 字形偏移
    width: i16, height: i16,   // 字形尺寸
    uv_left: f32, uv_bot: f32, // UV 坐标
    uv_width: f32, uv_height: f32,
    
    // 前景色
    r: u8, g: u8, b: u8,
    cell_flags: RenderingGlyphFlags,  // COLORED / WIDE_CHAR
    
    // 背景色 + alpha
    bg_r: u8, bg_g: u8, bg_b: u8, bg_a: u8,
}
```

在 `Batch::add_item` 中，颜色从 `RenderableCell` 直接复制到 `InstanceData`：

```rust
self.instances.push(InstanceData {
    r: cell.fg.r,
    g: cell.fg.g,
    b: cell.fg.b,
    // ...
    bg_r: cell.bg.r,
    bg_g: cell.bg.g,
    bg_b: cell.bg.b,
    bg_a: (cell.bg_alpha * 255.0) as u8,
});
```

### 5.2 矩形渲染管线

#### 5.2.1 线条收集：`RenderLines`

位于 [alacritty/src/renderer/rects.rs](alacritty/src/renderer/rects.rs#L158-L227)

```rust
pub struct RenderLines {
    inner: HashMap<Flags, Vec<RenderLine>, RandomState>,
}
```

**线条收集过程**（[update](alacritty/src/renderer/rects.rs#L180-L189)）：

```rust
pub fn update(&mut self, cell: &RenderableCell) {
    self.update_flag(cell, Flags::UNDERLINE);
    self.update_flag(cell, Flags::DOUBLE_UNDERLINE);
    self.update_flag(cell, Flags::STRIKEOUT);
    self.update_flag(cell, Flags::UNDERCURL);
    self.update_flag(cell, Flags::DOTTED_UNDERLINE);
    self.update_flag(cell, Flags::DASHED_UNDERLINE);
}
```

**单个标志更新逻辑**（[update_flag](alacritty/src/renderer/rects.rs#L191-L226)）：

```rust
fn update_flag(&mut self, cell: &RenderableCell, flag: Flags) {
    if !cell.flags.contains(flag) {
        return;
    }
    
    // 颜色选择：删除线使用前景色，其他使用下划线颜色
    let color = if flag.contains(Flags::STRIKEOUT) { cell.fg } else { cell.underline };
    
    // 宽字符处理
    let mut end = cell.point;
    if cell.flags.contains(Flags::WIDE_CHAR) {
        end.column += 1;
    }
    
    // 合并连续同色同样式的线段
    if let Some(line) = self.inner.get_mut(&flag).and_then(|lines| lines.last_mut()) {
        if color == line.color
            && cell.point.column == line.end.column + 1
            && cell.point.line == line.end.line
        {
            line.end = end;  // 延长现有线段
            return;
        }
    }
    
    // 开始新线段
    let line = RenderLine { start: cell.point, end, color };
    self.inner.entry(flag).or_default().push(line);
}
```

**合并优化**：相邻、同色、同样式、同行的单元格会被合并为一条 `RenderLine`，从而减少生成的 `RenderRect` 数量和上传到 GPU 的顶点数据量。矩形渲染器按 `RectKind` 分组后，每组仅调用一次 `glDrawArrays`，draw call 数量恒为 ≤4，与合并无关。

#### 5.2.2 线段数据结构：`RenderLine`

位于 [alacritty/src/renderer/rects.rs](alacritty/src/renderer/rects.rs#L36-L41)

```rust
pub struct RenderLine {
    pub start: Point<usize>,
    pub end: Point<usize>,
    pub color: Rgb,
}
```

#### 5.2.3 线段转矩形：`RenderLine::rects`

位于 [alacritty/src/renderer/rects.rs](alacritty/src/renderer/rects.rs#L54-L119)

```rust
pub fn rects(&self, flag: Flags, metrics: &Metrics, size: &SizeInfo) -> Vec<RenderRect> {
    // 根据不同样式计算位置、厚度、类型
    let (position, thickness, ty) = match flag {
        Flags::DOUBLE_UNDERLINE => {
            // 双下划线：两条线，分别在 25% 和 75% 位置
            let top_pos = 0.25 * metrics.descent;
            let bottom_pos = 0.75 * metrics.descent;
            // 先添加第一条线
            // ...
            (bottom_pos, metrics.underline_thickness, RectKind::Normal)
        },
        // 波浪线：占用整个 descent 区域，使用波浪线着色器
        Flags::UNDERCURL => (metrics.descent, metrics.descent.abs(), RectKind::Undercurl),
        // 下划线：使用字体的 underline_position
        Flags::UNDERLINE => (metrics.underline_position, metrics.underline_thickness, RectKind::Normal),
        // 点线：占用整个 descent 区域，使用点线着色器
        Flags::DOTTED_UNDERLINE => (metrics.descent, metrics.descent.abs(), RectKind::DottedUnderline),
        // 虚线：使用字体的 underline_position，使用虚线着色器
        Flags::DASHED_UNDERLINE => (metrics.underline_position, metrics.underline_thickness, RectKind::DashedUnderline),
        // 删除线：使用字体的 strikeout_position
        Flags::STRIKEOUT => (metrics.strikeout_position, metrics.strikeout_thickness, RectKind::Normal),
        _ => unimplemented!(),
    };
}
```

**位置计算公式**（[create_rect](alacritty/src/renderer/rects.rs#L121-L156)）：

```rust
fn create_rect(...) -> RenderRect {
    let start_x = start.column.0 as f32 * size.cell_width();
    let end_x = (end.column.0 + 1) as f32 * size.cell_width();
    let width = end_x - start_x;
    
    let line_bottom = (start.line as f32 + 1.) * size.cell_height();
    let baseline = line_bottom + descent;
    let y = (baseline - position - thickness / 2.).round();
    
    RenderRect::new(
        start_x + size.padding_x(),
        y + size.padding_y(),
        width,
        thickness,
        color,
        1.,
    )
}
```

#### 5.2.4 矩形类型：`RectKind`

位于 [alacritty/src/renderer/rects.rs](alacritty/src/renderer/rects.rs#L43-L52)

```rust
#[repr(u8)]
pub enum RectKind {
    Normal = 0,           // 普通矩形（直线）
    Undercurl = 1,        // 波浪线
    DottedUnderline = 2,  // 点线
    DashedUnderline = 3,  // 虚线
    NumKinds = 4,
}
```

每种类型对应不同的着色器程序，通过预编译时通过 `#define` 控制绘制逻辑。

#### 5.2.5 矩形渲染器：`RectRenderer`

位于 [alacritty/src/renderer/rects.rs](alacritty/src/renderer/rects.rs#L247-L398)

```rust
pub struct RectRenderer {
    vao: GLuint,
    vbo: GLuint,
    programs: [RectShaderProgram; 4],  // 4 个着色器程序
    vertices: [Vec<Vertex>; 4],      // 4 种类型的顶点数据
}
```

**矩形绘制流程**（[draw](alacritty/src/renderer/rects.rs#L320-L370)）：

```rust
pub fn draw(&mut self, size_info: &SizeInfo, metrics: &Metrics, rects: Vec<RenderRect>) {
    // 按类型分类顶点
    self.vertices.iter_mut().for_each(|v| v.clear());
    for rect in &rects {
        Self::add_rect(&mut self.vertices[rect.kind as usize], half_width, half_height, rect);
    }
    
    // 逆序绘制（普通矩形最后绘制，在最上层）
    for rect_kind in (RectKind::Normal as u8..RectKind::NumKinds as u8).rev() {
        let vertices = &mut self.vertices[rect_kind as usize];
        if vertices.is_empty() { continue; }
        
        // 使用对应类型的着色器程序
        let program = &self.programs[rect_kind as usize];
        gl::UseProgram(program.id());
        program.update_uniforms(size_info, metrics);
        
        // 上传顶点数据
        gl::BufferData(gl::ARRAY_BUFFER, ...);
        
        // 绘制
        gl::DrawArrays(gl::TRIANGLES, 0, vertices.len() as i32);
    }
}
```

#### 5.2.6 光标矩形转换：`IntoRects`

位于 [alacritty/src/display/cursor.rs](alacritty/src/display/cursor.rs#L11-L90)

```rust
pub trait IntoRects {
    fn rects(self, size_info: &SizeInfo, thickness: f32) -> CursorRects;
}

impl IntoRects for RenderableCursor {
    fn rects(self, size_info: &SizeInfo, thickness: f32) -> CursorRects {
        let x = point.column.0 as f32 * size_info.cell_width() + size_info.padding_x();
        let y = point.line as f32 * size_info.cell_height() + size_info.padding_y();
        
        match self.shape() {
            CursorShape::Beam => beam(x, y, height, thickness, self.color()),
            CursorShape::Underline => underline(x, y, width, height, thickness, self.color()),
            CursorShape::HollowBlock => hollow(x, y, width, height, thickness, self.color()),
            _ => CursorRects::default(),
        }
    }
}
```

**光标形状对应的矩形**：
- `Beam` → 1 个矩形（左侧竖线）
- `Underline` → 1 个矩形（底部横线）
- `HollowBlock` → 4 个矩形（四条边框）
- `Block` → 不生成矩形（Block 光标通过改变单元格颜色实现）

---

## 第六层：GLSL 着色器

### 6.1 文本着色器

#### 6.1.1 顶点着色器 (text.v.glsl)

将 `InstanceData` 中的属性传递给片段着色器，包括：
- `fg`：前景色（rgba，a 通道用作 colored 标志）
- `bg`：背景色（rgba）
- `TexCoords`：纹理坐标

#### 6.1.2 片段着色器 (text.f.glsl)

位于 [alacritty/res/glsl3/text.f.glsl](alacritty/res/glsl3/text.f.glsl)

多通道渲染，通过 `renderingPass` 控制：

```glsl
void main() {
    if (renderingPass == 0) {
        // Pass 0: 绘制背景色
        if (bg.a == 0.0) discard;
        FRAG_COLOR = vec4(bg.rgb * bg.a, bg.a); // 预乘 alpha
        return;
    }
    
    if (colored) {
        // 彩色字形（如 emoji）：直接使用纹理颜色
        FRAG_COLOR = texture(mask, TexCoords);
    } else {
        // 普通文本：使用前景色 × 灰度蒙版
        vec3 textColor = texture(mask, TexCoords).rgb;
        FRAG_COLOR = vec4(fg.rgb, 1.0);
    }
}
```

#### 6.1.3 文本混合模式

使用 `SRC1_COLOR` / `ONE_MINUS_SRC1_COLOR` 进行双源混合（Dual Source Blending），实现亚像素渲染（subpixel rendering）。

### 6.2 矩形着色器

#### 6.2.1 顶点着色器 (rect.v.glsl)

位于 [alacritty/res/rect.v.glsl](alacritty/res/rect.v.glsl)

```glsl
void main() {
    color = aColor;
    gl_Position = vec4(aPos.x, aPos.y, 0.0, 1.0);
}
```

简单的顶点着色器，直接传递位置和颜色。

#### 6.2.2 片段着色器 (rect.f.glsl)

位于 [alacritty/res/rect.f.glsl](alacritty/res/rect.f.glsl)

4 种绘制模式通过 `#define` 控制：

```glsl
// 波浪线绘制函数
color_t draw_undercurl(float_t x, float_t y) {
  // 使用余弦函数绘制波浪
  float_t undercurl = undercurlPosition / 2. * cos((x + 0.5) * 2.
                    * PI / cellWidth) + undercurlPosition - 1.;
  
  float_t undercurlTop = undercurl + max((underlineThickness - 1.), 0.) / 2.;
  float_t undercurlBottom = undercurl - max((underlineThickness - 1.), 0.) / 2.;
  
  // 计算到曲线的距离，用于抗锯齿
  float_t dst = max(y - undercurlTop, max(undercurlBottom - y, 0.));
  float_t alpha = 1. - dst * dst;  // 1/x² 抗锯齿
  
  return vec4(color.rgb, alpha);
}

// 点线绘制函数（抗锯齿版本）
color_t draw_dotted_aliased(float_t x, float_t y) {
  // 计算圆点中心和半径
  float_t dotNumber = floor(x / underlineThickness);
  float_t radius = underlineThickness / 2.;
  float_t centerY = underlinePosition - 1.;
  
  // 计算到两个圆点的距离
  float_t distanceLeft = sqrt(pow(x - leftCenter, 2.) + pow(y - centerY, 2.));
  float_t alpha = max(1. - (min(distanceLeft, distanceRight) - radius), 0.));
  
  return vec4(color.rgb, alpha);
}

// 虚线绘制函数
color_t draw_dashed(float_t x) {
  float_t halfDashLen = floor(cellWidth / 4. + 0.5);
  float_t alpha = 1.;
  
  // 中间留空隙
  if (x > halfDashLen - 1. && x < cellWidth - halfDashLen) {
    alpha = 0.;
  }
  
  return vec4(color.rgb, alpha);
}

void main() {
  float_t x = floor(mod(gl_FragCoord.x - paddingX, cellWidth));
  float_t y = floor(mod(gl_FragCoord.y - paddingY, cellHeight));
  
#if defined(DRAW_UNDERCURL)
  FRAG_COLOR = draw_undercurl(x, y);
#elif defined(DRAW_DOTTED)
  if (underlineThickness < 2.) {
    FRAG_COLOR = draw_dotted(x, y);
  } else {
    FRAG_COLOR = draw_dotted_aliased(x, y);
  }
#elif defined(DRAW_DASHED)
  FRAG_COLOR = draw_dashed(x);
#else
  FRAG_COLOR = color;  // 普通直线
#endif
}
```

#### 6.2.3 着色器编译时选择

位于 [alacritty/src/renderer/rects.rs](alacritty/src/renderer/rects.rs#L437-L446)

```rust
impl RectShaderProgram {
    pub fn new(shader_version: ShaderVersion, kind: RectKind) -> Result<Self, ShaderError> {
        let header = match kind {
            RectKind::Undercurl => Some("#define DRAW_UNDERCURL\n"),
            RectKind::DottedUnderline => Some("#define DRAW_DOTTED\n"),
            RectKind::DashedUnderline => Some("#define DRAW_DASHED\n"),
            _ => None,
        };
        let program = ShaderProgram::new(shader_version, header, RECT_SHADER_V, RECT_SHADER_F)?;
        // ...
    }
}
```

---

## 完整调用链总结

### 初始化阶段

```
配置文件 (alacritty.toml)
  → serde 反序列化
    → config::Colors
      → Display::new() 中 List::from(&config.colors)
        → display::color::List (269 个 RGB)
```

### 运行时阶段

```
终端接收转义序列 (VTE 解析)
  → Term::set_color / 等方法修改 term::color::Colors (Option<Rgb>)
  → Cell 的 fg/bg 被设置为 Color 枚举值
  → Cell 的 flags 被设置为样式标志位
```

### 每帧渲染阶段

```
Display::draw()
  ├─ RenderableContent::new(config, display, term, search_state)
  │   ├─ 迭代器遍历每个 Cell
  │   └─ RenderableCell::new()
  │       ├─ compute_fg_rgb()   // 计算前景 RGB（处理 BOLD/DIM）
  │       ├─ compute_bg_rgb()   // 计算背景 RGB
  │       ├─ 处理 INVERSE 反转
  │       ├─ 处理选区/搜索/Hint 叠加
  │       └─ 计算 bg_alpha
  │
  ├─ renderer.clear()        // 清屏
  │
  ├─ 创建 RenderLines
  │
  ├─ 文本绘制管线
  │   └─ renderer.draw_cells()
  │       └─ 遍历 cells：
  │           ├─ lines.update(&cell)  // 收集线条样式（6 种）
  │           └─ TextRenderApi::draw_cell()
  │               ├─ 根据 BOLD/ITALIC 选择字体
  │               ├─ HIDDEN 替换为空格
  │               ├─ 获取字形
  │               └─ 写入 InstanceData → GPU 绘制
  │
  ├─ 矩形绘制管线
  │   ├─ lines.rects() → RenderRect（位置、厚度、类型）
  │   ├─ cursor.rects() → 光标 RenderRect
  │   ├─ 视觉响铃 RenderRect
  │   ├─ 消息栏 RenderRect
  │   └─ renderer.draw_rects()
  │       └─ 按 RectKind 分类顶点
  │       └─ 逆序绘制 4 种类型
  │           └─ 使用对应着色器（直线/波浪/点/虚线）
  │           └─ glDrawArrays
  │
  └─ swap_buffers()  // 交换缓冲区
```

### 颜色查找回退机制

在 [RenderableContent::color](alacritty/src/display/content.rs#L104-L106) 中：

```rust
pub fn color(&self, color: usize) -> Rgb {
    self.terminal_content.colors[color].map(Rgb).unwrap_or(self.colors[color])
}
```

- 优先使用终端动态修改的颜色（`term::color::Colors`）
- 如果是 `None`，回退到配置的默认颜色（`display::color::List`）

---

## 关键设计要点

1. **双层调色板设计**：终端层 (`Option<Rgb>`) + 显示层 (`Rgb`)，支持运行时动态修改颜色且不丢失默认值

2. **Color 枚举的三层抽象**：
   - `Color::Named` - 语义颜色（前景/背景/黑/红...）
   - `Color::Indexed` - 256 色调色板索引
   - `Color::Spec` - 直接 RGB 值

3. **CellRgb 的反转语义**：支持"使用单元格前景/背景色"的引用，实现光标/选区的灵活配色

4. **双渲染管线设计**：
   - **文本管线**处理字符本身（BOLD/ITALIC/HIDDEN 等）
   - **矩形管线**处理装饰线条（下划线/删除线/波浪线等）
   - 分离关注点，各自优化

5. **RenderLines 合并优化**：连续同色同样式的单元格合并为单条线段，减少 `RenderRect` 数量和上传到 GPU 的顶点数据量（矩形渲染器的 draw call 数量恒为 ≤4，与合并无关）

6. **4 种 RectKind 对应 4 个预编译着色器**：通过 `#define` 控制绘制逻辑，避免运行时分支

7. **批处理渲染**：使用 Instance Draw 减少 draw call 数量，提升性能

8. **多通道渲染**：背景和文本分开渲染，支持亚像素抗锯齿

9. **背景透明优化**：默认背景色不绘制（alpha=0），减少 GPU 填充率开销

10. **着色器抗锯齿**：波浪线使用距离场（SDF 近似）实现平滑边缘

11. **绘制顺序保证**：矩形在文本之后绘制，确保线条在文字上层

12. **逆序绘制策略**：矩形按类型逆序绘制，普通矩形最后绘制确保在最上层
