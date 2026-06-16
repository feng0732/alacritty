# Alacritty 字形栅格化与缓存代码分析

## 一、模块结构概览

字形栅格化与缓存系统主要分布在 `alacritty/src/renderer/text/` 目录下，核心模块关系如下：

| 文件 | 职责 |
|------|------|
| [glyph_cache.rs](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/glyph_cache.rs) | 字形缓存核心，管理 `GlyphKey → Glyph` 的 HashMap 缓存，封装栅格化调用 |
| [atlas.rs](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/atlas.rs) | 纹理图集（Texture Atlas）管理，将栅格化字形上传到 GPU 纹理 |
| [mod.rs](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/mod.rs) | 模块入口，定义 `TextRenderer`/`TextRenderApi`/`LoadGlyph` 等 trait |
| [builtin_font.rs](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/builtin_font.rs) | 内置字体，手工绘制框线字符、Powerline 符号等 |
| [glsl3.rs](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/glsl3.rs) | OpenGL 3.3 渲染器实现 |
| [gles2.rs](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/gles2.rs) | OpenGL ES 2.0 渲染器实现 |

---

## 二、主流程分析

### 2.1 初始化流程

#### 入口：Display 初始化
[display/mod.rs:412-445](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/display/mod.rs#L412-L445)

```
1. 创建 Rasterizer（来自 crossfont 库）
2. 根据 DPI 缩放调整字体大小
3. GlyphCache::new(rasterizer, &font)  → 加载四种字体变体（Regular/Bold/Italic/BoldItalic）
4. 创建 Renderer（自动选择 GLSL3 或 GLES2）
5. renderer.with_loader(|api| glyph_cache.reset_glyph_cache(api))
   → 清空 Atlas + 预加载 ASCII 常用字符（32~126）
```

#### GlyphCache::new 详细过程
[glyph_cache.rs:82-99](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/glyph_cache.rs#L82-L99)

核心调用 `compute_font_keys()`：
1. **加载 Regular 字体**：通过 `load_regular_font()` 加载，失败时回退到系统默认字体
2. **加载 Bold/Italic/BoldItalic**：使用 `load_or_regular` 闭包，如果变体描述与 Regular 相同则复用，否则尝试加载，加载失败也回退到 Regular

### 2.2 字形获取（缓存命中/未命中）

核心方法：[GlyphCache::get()](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/glyph_cache.rs#L200-L245)

```
┌─────────────────────────────────────────────────────────┐
│                  glyph_cache.get(glyph_key)             │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
               ┌──────────────────────┐
               │  检查 HashMap 缓存    │  cache: HashMap<GlyphKey, Glyph>
               └──────────────────────┘
                    │           │
                 命中         未命中
                    │           │
                    ▼           ▼
              返回 Glyph    ┌─────────────────────────────┐
                           │ 尝试内置字体（框线字符等）    │ builtin_box_drawing 开关
                           └─────────────────────────────┘
                                    │           │
                                成功生成      非内置字符
                                    │           │
                                    ▼           ▼
                           load_glyph()   rasterizer.get_glyph(glyph_key)
                                                │
                                           ┌────┴────┐
                                           │  成功   │ 失败(MissingGlyph)
                                           ▼         ▼
                                     load_glyph()  加载缺省字形('\0')
                                           │         │
                                           └────┬────┘
                                                ▼
                                    插入缓存 HashMap 并返回
```

### 2.3 字形加载到 GPU Atlas

`load_glyph()` 方法 [glyph_cache.rs:248-269](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/glyph_cache.rs#L248-L269) 在上传 GPU 之前做三件事：

1. **应用 glyph_offset**：`glyph.left/top += offset`
2. **减去 descent**：`glyph.top -= metrics.descent`（使基线对齐）
3. **零宽字符修正**：`glyph.left += metrics.average_advance`（将锚点右移一个单元格）

最终调用 `loader.load_glyph(&glyph)`，由 `LoaderApi` 委托给 `Atlas::load_glyph()`。

#### Atlas 插入策略
[atlas.rs:118-140](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/atlas.rs#L118-L140)

Atlas 采用**行优先**的矩形装箱算法：
```
(0,0) ──────────────────────────► x
  │ ┌────┬────┬────┬───────────┐
  │ │ 1  │ 2  │ 3  │ 4         │  row_tallest = max(row heights)
  │ ├────┼────┼────┼───────────┤  row_baseline = 累积行高
  │ │ 5  │ 6  │ 7  │ 8  │ 9    │
  │ ├────┼────┼────┼────┴──────┤
  ▼ │ 10 │    │    │          │  ATLAS_SIZE = 1024 x 1024
  y └────┴────┴────┴──────────┘
```

插入时：
1. `room_in_row()` 检查当前行宽度和剩余高度
2. 放不下则 `advance_row()` 切换到下一行
3. 切换后仍放不下则当前 Atlas 满，返回 `AtlasInsertError::Full`
4. 通过 `insert_inner()` 调用 `glTexSubImage2D` 上传像素数据

### 2.4 渲染时的字形流转

渲染调用链：
```
Renderer::draw_cells()
  → TextRenderer::draw_cells()        [mod.rs:58-69]
    → TextRenderApi::draw_cell()      [mod.rs:135-172]
      1. 根据 cell flags 选择 font_key（Regular/Bold/Italic/BoldItalic）
      2. Tab/Hidden 字符替换为空格
      3. glyph_cache.get(glyph_key, self, show_missing=true)
      4. add_render_item(cell, glyph)  → 加入渲染批次（Batch）
         - 若 tex_id 变化则先 render_batch() 刷出
         - 若 Batch 满（0x10000 项）也刷出
```

---

## 三、辅助路径

### 3.1 内置字体（Box Drawing / Powerline）

[builtin_font.rs:23-47](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/builtin_font.rs#L23-L47)

`builtin_glyph()` 处理两类 Unicode 字符：

| 范围 | 内容 | 处理方式 |
|------|------|----------|
| U+2500 ~ U+259F | 框线、方块元素、阴影、象限 | `box_drawing()` 手工绘制 |
| U+1FB00 ~ U+1FB3B | 六分仪（Sextants） | `box_drawing()` 手工绘制 |
| U+1FB82 ~ U+1FB8B | 八分块扩展 | `box_drawing()` 手工绘制 |
| U+E0B0 ~ U+E0B3 | Powerline 三角/箭头 | `powerline_drawing()` 手工绘制 |

**设计要点**：
- 内置字体确保这些字符**完全覆盖单元格**，避免字体渲染导致的线条断裂
- 支持配置开关 `builtin_box_drawing`（默认 `true`）
- 对内置字形会**反向减去 glyph_offset**，以抵消 `load_glyph()` 中的加回操作，保证精准对齐

绘制能力：
- 直线（水平/垂直虚线、单线/双线/轻重线组合）
- 对角线（Xiaolin Wu 反锯齿直线算法）
- 圆角（`draw_rounded_corner` 圆形近似）
- 块填充（1/8 ~ 7/8 精度的水平/垂直分块）
- 阴影（4 级灰度）
- 象限分块（4 象限）、六分仪分块（2×3 网格）

### 3.2 字体回退机制

#### 主字体加载回退
[glyph_cache.rs:165-180](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/glyph_cache.rs#L165-L180)

```rust
fn load_regular_font(...) -> Result<FontKey, crossfont::Error> {
    match rasterizer.load_font(description, size) {
        Ok(font) => Ok(font),
        Err(err) => {
            error!("{err}");
            // 回退到 Font::default().normal() 对应的系统默认字体
            let fallback_desc = Self::make_desc(Font::default().normal(), ...);
            rasterizer.load_font(&fallback_desc, size)
        },
    }
}
```

默认字体平台差异（[font.rs:103-113](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/config/font.rs#L103-L113)）：
- Linux: `"monospace"`
- macOS: `"Menlo"`
- Windows: `"Consolas"`

#### 字重/斜体变体回退
[glyph_cache.rs:139-145](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/glyph_cache.rs#L139-L145)

`load_or_regular` 闭包逻辑：
- 若变体描述 `==` regular_desc → 直接复用 Regular 的 FontKey
- 否则尝试 `rasterizer.load_font(&desc, size).unwrap_or(regular)` → 失败也回退到 Regular

#### 单个字形缺失回退
[glyph_cache.rs:227-239](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/glyph_cache.rs#L227-L239)

当 `rasterizer.get_glyph()` 返回 `RasterizerError::MissingGlyph(rasterized)` 时：
1. 使用 `'\0'`（NUL 字符）作为**统一的缺失字形缓存键**
2. 先查缓存，命中则直接返回（避免重复加载）
3. 未命中则将 rasterizer 返回的缺省字形（通常是方框/问号）加载到 Atlas
4. 以 `'\0'` 为 key 存入缓存，后续所有缺失字形共享这一个条目

### 3.3 字体尺寸调整

[glyph_cache.rs:283-305](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/glyph_cache.rs#L283-L305)

`update_font_size(font)` 执行：
1. 更新 `font_offset` / `glyph_offset`
2. 重新 `compute_font_keys()`（因为不同字号下 crossfont 内部可能重新生成 FontKey）
3. 重新加载字体度量 `metrics`
4. 更新 `builtin_box_drawing` 开关

**注意**：`update_font_size()` **不会自动清空缓存**，必须随后调用 `reset_glyph_cache()`，由 [display/mod.rs:628-678](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/display/mod.rs#L628-L678) 的 `update_font_size()` 包装函数确保两者配合使用。

### 3.4 常用字形预加载

[glyph_cache.rs:312-317](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/glyph_cache.rs#L312-L317)

`load_common_glyphs()` 在每次 `reset_glyph_cache()` 后自动调用：
- 对 Regular/Bold/Italic/BoldItalic **四种字体**分别加载 ASCII 32~126（共 95 个）字符
- 确保首次渲染英文文本时不会触发大量的缓存未命中

### 3.5 多 Atlas 管理

[atlas.rs:251-287](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/atlas.rs#L251-L287)

`Atlas::load_glyph()` 处理图集扩展：
- 当当前 Atlas 返回 `Full` 时，`current_atlas += 1`
- 若索引超出已有 Atlas 数量，则新建一个 `ATLAS_SIZE(1024)²` 的纹理
- 递归调用自身在新 Atlas 上重试
- `clear_atlas()` 会将所有 Atlas 的 row 指针重置，但不删除纹理（复用）

### 3.6 GLES 上下文兼容

[atlas.rs:158-174](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/atlas.rs#L158-L174)

GLES 不支持将 RGB 数据直接上传到 RGBA 纹理，因此在 GLES 上下文下：
- `BitmapBuffer::Rgb` 会被逐像素转换为 RGBA（补 Alpha=255）
- 使用 `Cow::Owned` 持有转换后的缓冲区
- GL 桌面环境下直接 `Cow::Borrowed` 零拷贝使用

---

## 四、失败处理逻辑

### 4.1 字形过大（GlyphTooLarge）

[atlas.rs:124-126](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/atlas.rs#L124-L126) + [atlas.rs:274-285](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/atlas.rs#L274-L285)

当 `glyph.width > ATLAS_SIZE || glyph.height > ATLAS_SIZE`（> 1024px）时：
- 返回一个**零尺寸 Glyph**（`width=height=0`，UV 全为 0）
- 渲染时该字形相当于不可见，不会崩溃但会缺失显示

### 4.2 图集全满（Full）

正常情况下不会发生，因为会自动创建新 Atlas。只有在**极端场景**下（内存耗尽导致无法创建新纹理）才会在新 Atlas 创建后仍然 insert 失败，此时递归会持续下去直到 Atlas 插入成功或栈溢出（理论上的边界情况）。

### 4.3 栅格化通用错误

[glyph_cache.rs:240](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/glyph_cache.rs#L240)

```rust
Err(_) => self.load_glyph(loader, Default::default()),
```

除 `MissingGlyph` 之外的所有 `RasterizerError`（例如字体内部错误）：
- 使用 `RasterizedGlyph::default()`（全零/空 buffer）
- 加载后存入缓存，避免每次触发都走错误路径

### 4.4 GPU 重置恢复

[renderer/mod.rs:281-302](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/mod.rs#L281-L302) + [display/mod.rs:587-605](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/display/mod.rs#L587-L605)

通过 `GL_KHR_robustness` 扩展检测 GPU 重置：
1. `Renderer::was_context_reset()` 调用 `glGetGraphicsResetStatus()`
2. 若检测到重置，Display 会：
   - 销毁旧 Renderer，创建新 Renderer
   - 调用 `reset_glyph_cache()` 清空并重建所有字形缓存
   - 标记全屏为 damaged 以触发完整重绘

### 4.5 配置缺失字重/斜体的降级

[font.rs:124-129](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/config/font.rs#L124-L129)

`SecondaryFontDescription::desc()` 处理用户未配置 bold/italic 家族名：
- `family` 未配置时，从 normal 字体的 family 继承
- `style` 未配置时为 `None`，后续由 `make_desc()` 用 Slant/Weight 描述方式匹配

---

## 五、关键数据结构

### GlyphCache
```rust
pub struct GlyphCache {
    cache: HashMap<GlyphKey, Glyph, RandomState>,  // 字形缓存（ahash 高性能哈希）
    rasterizer: Rasterizer,                         // crossfont 栅格器
    font_key / bold_key / italic_key / bold_italic_key: FontKey,
    font_size: Size,
    font_offset: Delta<i8>,     // 单元格级偏移
    glyph_offset: Delta<i8>,    // 字形级偏移
    metrics: Metrics,
    builtin_box_drawing: bool,
}
```

### Glyph（GPU 侧描述）
```rust
pub struct Glyph {
    pub tex_id: GLuint,      // 所在 Atlas 纹理 ID
    pub multicolor: bool,    // RGBA（彩色 emoji）vs RGB（单色）
    pub top/left/width/height: i16,   // 字形像素包围盒
    pub uv_bot/uv_left/uv_width/uv_height: f32,  // 纹理归一化坐标
}
```

### Atlas
```rust
pub struct Atlas {
    id: GLuint,              // OpenGL 纹理 ID
    width/height: i32,       // 固定 1024
    row_extent: i32,         // 当前行已用宽度
    row_baseline: i32,       // 当前行起始 Y
    row_tallest: i32,        // 当前行最高字形高度
    is_gles_context: bool,   // 是否 GLES 上下文（影响 RGB→RGBA 转换）
}
```

---

## 六、完整调用链总结

```
Display::new()
├─ Rasterizer::new()
├─ GlyphCache::new(rasterizer, font)
│   └─ compute_font_keys()
│       ├─ load_regular_font() [可能回退到默认字体]
│       └─ load_or_regular() × 3 [Bold/Italic/BoldItalic]
├─ Renderer::new() [GLSL3 或 GLES2]
└─ glyph_cache.reset_glyph_cache(loader_api)
    ├─ Atlas::clear_atlas()
    ├─ cache.clear()
    └─ load_common_glyphs()
        └─ load_glyphs_for_font() × 4
            └─ get(GlyphKey{32..126}) [预填充]

每帧渲染:
Renderer::draw_cells(cells)
└─ for cell in cells:
    TextRenderApi::draw_cell(cell)
    ├─ 选择 font_key（Regular/Bold/Italic/BoldItalic）
    ├─ Tab/Hidden → ' '
    ├─ glyph_cache.get(glyph_key, self, show_missing=true)
    │   ├─ cache hit → 直接返回
    │   ├─ cache miss
    │   │   ├─ builtin_box_drawing && box_drawing_char → builtin_glyph()
    │   │   └─ else → rasterizer.get_glyph()
    │   │       ├─ Ok → load_glyph()
    │   │       ├─ Err(MissingGlyph) → 缺省'\0'字形
    │   │       └─ Err(_) → Default::default() 空字形
    │   └─ cache.insert(glyph_key, glyph)
    └─ add_render_item()
        ├─ tex_id 变化 → render_batch() 刷出
        ├─ 加入 Batch
        └─ Batch 满 → render_batch() 刷出
```
