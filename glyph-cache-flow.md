# 字形栅格化与缓存：缓存清理时机 & 图集满载边界 核准分析

## 一、字体尺寸调整后的缓存清理时机

### 1.1 两步分离设计

`GlyphCache` 将"更新字体参数"和"清空缓存"拆成两个独立方法：

- [update_font_size()](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/glyph_cache.rs#L283-L305)：仅更新 `font_offset`/`glyph_offset`/`font_key`/`metrics` 等字段，**不清空** `cache: HashMap`，**不调用** `loader.clear()`
- [reset_glyph_cache()](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/glyph_cache.rs#L272-L277)：调用 `loader.clear()` 清空 Atlas → 清空 HashMap → 调用 `load_common_glyphs()` 预加载

源码注释已明确标注（[glyph_cache.rs:281](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/glyph_cache.rs#L281)）：

> NOTE: To reload the renderers's fonts `Self::reset_glyph_cache` should be called afterwards.

### 1.2 调用时序（Display 层编排）

字体尺寸调整涉及**三个时间点**，分布在两次不同的方法调用中：

#### 时间点 1：`Display::handle_update()` — 无 GL 操作

[display/mod.rs:651-682](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/display/mod.rs#L651-L682)

```
handle_update(terminal, pty, ..., config)
│
├─ pending_update.font().is_some() || cursor_dirty() ?
│    └─ 设置 pending_renderer_update.clear_font_cache = true
│       （标记"稍后需要清缓存"，但不立即执行）
│
└─ pending_update.font().is_some() ?
     └─ Display::update_font_size(&mut glyph_cache, config, font)
         │
         └─ glyph_cache.update_font_size(font)
             ├─ 更新 font_offset / glyph_offset
             ├─ compute_font_keys() 重新加载字体 → 新 FontKey
             ├─ load_font_metrics() → 新 Metrics
             └─ 更新 self.font_size / font_key / bold_key / ... / metrics
             （此时 HashMap 中仍是旧字号的 Glyph，Atlas 中仍是旧纹理数据）
```

**关键**：`handle_update()` 的注释明确声明 ([display/mod.rs:647](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/display/mod.rs#L647))：

> XXX: this function must not call to any OpenGL related tasks. Renderer updates are performed in `Self::process_renderer_update` right before drawing.

所以在 `handle_update()` 中**只更新 CPU 侧参数**（font keys、metrics），**不触碰 GL 资源**。此时 GlyphCache 处于一种**不一致状态**：HashMap 里缓存的是旧字号字形数据，但 `font_key`/`font_size` 等字段已经指向新字号。

#### 时间点 2：`Display::process_renderer_update()` — 执行 GL 操作

[display/mod.rs:744-768](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/display/mod.rs#L744-L768)

```
process_renderer_update()
│
├─ renderer_update.resize ?  → surface.resize()
│
├─ make_current()  （确保 GL 上下文正确）
│
├─ renderer_update.clear_font_cache ?
│    └─ self.reset_glyph_cache()
│        │
│        └─ renderer.with_loader(|mut api| {
│               cache.reset_glyph_cache(&mut api)
│           })
│           │
│           ├─ loader.clear()    → Atlas::clear_atlas()
│           │     → 遍历 Vec<Atlas> 中所有 Atlas 调用 clear()
│           │       （只重置 row_extent/row_baseline/row_tallest，不释放纹理）
│           │     → current_atlas = 0
│           │
│           ├─ self.cache = Default::default()   → 清空 HashMap
│           │
│           └─ self.load_common_glyphs(loader)
│               → 4 种字体 × ASCII 32..126 = 380 次缓存量
│                 （这些字形会写入 atlas[0]，即第一个 Atlas）
│
└─ renderer.resize(&size_info)
```

### 1.3 为什么必须分离？

**原因**：Wayland 等平台要求 GL 操作必须在渲染前执行，否则会锁住 back buffer 导致用旧状态渲染/闪烁。因此：

1. `handle_update()` 在事件处理阶段执行（可以任意时机）→ 只改 CPU 数据
2. `process_renderer_update()` 在绘制帧之前执行 → 集中处理 GL 操作

如果在 `update_font_size()` 内直接清缓存，就会触发 `loader.clear()` 和 `load_common_glyphs()` 中的 `glTexSubImage2D` 调用，违反平台约束。

### 1.4 中间不一致状态的窗口期

从 `handle_update()` 到 `process_renderer_update()` 之间存在一帧的窗口期，此时：
- `glyph_cache.font_key` 等已更新为新字号
- `glyph_cache.cache` 中仍是旧字号的 Glyph 条目

但由于 `GlyphKey` 包含 `size` 字段，而 `GlyphCache::get()` 构造的 key 使用 `self.font_size`（已更新），所以**旧缓存不会被命中**（因为 size 不同），相当于缓存自动"失效"。不过旧条目仍占据 HashMap 内存，直到 `reset_glyph_cache()` 清空。

**注意**：如果在此窗口期有渲染帧（理论上不应发生，因为 process_renderer_update 在 draw 之前），`glyph_cache.get()` 会因缓存未命中而走 rasterizer 路径，此时使用的是新字号，得到的是正确字形，但会写入尚未清空的旧 HashMap，且 Atlas 纹理中旧数据仍在。这不会导致崩溃，但会浪费 Atlas 空间。

### 1.5 GPU 重置场景

[display/mod.rs:580-605](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/display/mod.rs#L580-L605)

GPU 重置后：
1. 重建 GL 上下文 + Renderer（旧 Atlas 纹理随 Renderer 销毁而 `glDeleteTextures`）
2. `self.reset_glyph_cache()` → 在**全新 Renderer** 上执行 `loader.clear()`（清空新 Atlas 的 row 指针）→ 清空 HashMap → `load_common_glyphs()`

这里**不需要**调用 `update_font_size()`，因为 font_key/metrics 等在 CPU 侧没有丢失，只有 GL 资源需要重建。

---

## 二、图集满载后新纹理创建的边界情况

### 2.1 正常 Full 路径的递归流程

[atlas.rs:251-287](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/atlas.rs#L251-L287)

```
Atlas::load_glyph(active_tex, atlas, current_atlas, rasterized)
│
├─ atlas[current_atlas].insert(rasterized, active_tex)
│   │
│   ├─ Ok(glyph) → 直接返回
│   │
│   ├─ Err(GlyphTooLarge) → 返回零尺寸 Glyph（静默不可见）
│   │
│   └─ Err(Full)
│       │
│       ├─ current_atlas += 1
│       │
│       ├─ current_atlas == atlas.len() ?
│       │   ├─ 是 → Atlas::new(ATLAS_SIZE, is_gles_context)
│       │   │       → gl::GenTextures + gl::TexImage2D (1024×1024 RGBA)
│       │   │       → atlas.push(new)
│       │   │       → active_tex = 0
│       │   │
│       │   └─ 否 → 跳过创建，复用已存在的 Atlas
│       │            （clear_atlas 重置指针但保留 Vec 容量，所以
│       │             之前创建过的 Atlas 仍在 Vec 中可复用）
│       │
│       └─ 递归调用 Atlas::load_glyph(active_tex, atlas, current_atlas, rasterized)
│           → 在新/复用的 Atlas 上重试 insert()
```

### 2.2 关键边界：Atlas::new 的 GL 失败

[Atlas::new()](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/atlas.rs#L73-L110) 执行以下 GL 调用，**全都不检查返回值**：

```rust
gl::GenTextures(1, &mut id);          // 若失败，id 保持 0
gl::BindTexture(gl::TEXTURE_2D, id);  // 绑定 id=0 无实际效果
gl::TexImage2D(..., size, size, ...); // 向 id=0 写入 1024×1024 RGBA
// ... TexParameteri ...
gl::BindTexture(gl::TEXTURE_2D, 0);
```

**OpenGL 的静默失败语义**：

| GL 调用 | 失败时行为 |
|---------|-----------|
| `glGenTextures` | 不生成纹理名，`id` 保持 0 |
| `glBindTexture(0)` | 解绑当前纹理，不影响任何对象 |
| `glTexImage2D` 对 id=0 | GL 无效操作（undefined behavior / silently ignored） |

当 GPU 显存耗尽时：
1. `GenTextures` 可能返回 `id=0`（或一个"名称"但后续操作静默失败）
2. `TexImage2D` 在无效目标上被忽略
3. `Atlas::new()` 仍然返回一个 `Atlas { id: 0, ... }` 结构体
4. `atlas.push(new)` 将这个 `id=0` 的 Atlas 加入 Vec
5. 递归调用 `load_glyph` → `atlas[current_atlas].insert()` 在 `id=0` 的 Atlas 上尝试
6. **此时 `insert()` 不会报错**——`room_in_row()` 只看尺寸，不看 id 是否有效
7. `insert_inner()` 中 `glTexSubImage2D` 写入 `id=0` → GL 静默忽略
8. 返回的 `Glyph { tex_id: 0, ... }` 中 `tex_id=0` 不是有效纹理
9. **渲染时该字形不显示**，但不会崩溃

### 2.3 递归不会无限——但有特殊终止条件

递归的终止条件有两个：

1. **正常终止**：新 Atlas 的 `insert()` 成功 → `Ok(glyph)` 返回
2. **静默失败终止**：新 Atlas `id=0` 时 `insert()` 仍返回 `Ok`（GL 不报错），递归 1 层就结束

**不会无限递归的原因**：`Atlas::new()` 每次创建的 Atlas 初始 `row_extent=0, row_baseline=0`，是一个"空" Atlas，`insert()` 对空 Atlas 一定成功（只要 `glyph.width <= 1024 && glyph.height <= 1024`），所以递归最多 1 层。

**唯一的递归风险**：如果字形本身 `width > 1024 || height > 1024`，则 `insert()` 返回 `GlyphTooLarge`，走零尺寸 Glyph 分支——这与 Full 分支无关，不会导致递归。

### 2.4 Vec<Atlas> 的增长与收缩

| 场景 | Vec<Atlas> 行为 |
|------|----------------|
| 初始创建 | `vec![Atlas::new(1024, ...)]` → 长度 1 |
| Atlas 满载 | `atlas.push(new)` → 长度 +1 |
| `clear_atlas()` | 遍历所有 Atlas 调用 `clear()`，**不改变 Vec 长度**；`current_atlas=0` 归零 |
| `reset_glyph_cache()` | `loader.clear()` → `clear_atlas()` → 同上，Vec 不缩短 |
| Renderer 销毁 | `Drop for Glsl3Renderer` 不显式 drop Atlas；但 `Vec<Atlas>` 随 Renderer drop → 每个 Atlas 的 `Drop` 调用 `glDeleteTextures` |

**累积效应**：如果运行期间多次触发 Full 导致新增 Atlas，之后即使 `clear_atlas()` 重置了指针，Vec 中的额外 Atlas 也不会被回收。只有 Renderer 整体重建（如 GPU reset）时才会释放。

**复用效应**：`clear_atlas()` 后 `current_atlas=0`，后续写入从 `atlas[0]` 开始。当 `atlas[0]` 再次 Full 后 `current_atlas=1`，此时 `1 < atlas.len()`，不会创建新 Atlas，而是复用之前已创建的 `atlas[1]`（其 GL 纹理仍在，row 指针已重置）。

### 2.5 GlyphTooLarge 的精确条件

[atlas.rs:124-126](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/atlas.rs#L124-L126)

```rust
if glyph.width > self.width || glyph.height > self.height {
    return Err(AtlasInsertError::GlyphTooLarge);
}
```

条件是 `width > 1024 || height > 1024`（严格大于）。也就是说：
- 1024×1024 的字形**可以**插入（`==` 不触发 GlyphTooLarge）
- 1025×1 的字形**不可**插入

返回的零尺寸 Glyph ([atlas.rs:274-285](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/atlas.rs#L274-L285))：

```rust
Glyph {
    tex_id: atlas[*current_atlas].id,  // 引用当前 Atlas 的纹理 ID
    multicolor: false,
    top: 0, left: 0, width: 0, height: 0,
    uv_bot: 0., uv_left: 0., uv_width: 0., uv_height: 0.,
}
```

注意 `tex_id` 指向的是**当前** Atlas（不是 0），所以如果后续 Batch 按 tex_id 排序，这个 Glyph 仍会绑定到正确的纹理，只是渲染出零尺寸（不可见）。

### 2.6 room_in_row 的精确判断

[atlas.rs:225-231](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/atlas.rs#L225-L231)

```rust
pub fn room_in_row(&self, raw: &RasterizedGlyph) -> bool {
    let next_extent = self.row_extent + raw.width;
    let enough_width = next_extent <= self.width;
    let enough_height = raw.height < (self.height - self.row_baseline);
    enough_width && enough_height
}
```

高度判断使用**严格小于** `<`，而非 `<=`。这意味着如果 `raw.height == height - row_baseline`（字形高度恰好等于剩余垂直空间），判定为**放不下**，会触发 `advance_row()`。这是有意为之——避免字形贴底溢出。

---

## 三、总结对比

### 缓存清理时机

| 场景 | update_font_size | reset_glyph_cache | 间隔 |
|------|:---:|:---:|------|
| 字体配置变更 | `handle_update()` 中立即执行 | `process_renderer_update()` 中渲染前执行 | 同帧内，但分两个方法 |
| GPU 重置 | 不调用（CPU 参数未丢失） | 重建 Renderer 后立即调用 | 无间隔 |
| 初始化 | `GlyphCache::new()` 隐含 | `reset_glyph_cache()` 在 `Display::new()` 末尾调用 | 无间隔 |

### 图集满载边界

| 情况 | 行为 | 后果 |
|------|------|------|
| 正常 Full | 创建新 Atlas → 递归 insert → 成功 | 正常渲染 |
| 复用旧 Atlas | current_atlas < len → 跳过创建 → 递归 insert | 复用已分配纹理，无额外开销 |
| GPU 显存耗尽 | `GenTextures` 返回 id=0 → `TexImage2D` 静默失败 → insert "成功" | Glyph.tex_id=0，渲染时不可见，不崩溃 |
| 字形 > 1024px | `GlyphTooLarge` → 零尺寸 Glyph | 静默不可见，不崩溃，不递归 |
| `clear_atlas()` | 重置 row 指针，current_atlas=0 | Vec 长度不变，已有纹理可复用 |
