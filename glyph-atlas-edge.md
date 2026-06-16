# 图集高度边界核准：与满载扩展、过大字形处理的精确关系

## 一、三处高度判定的比较运算符

Atlas 高度相关判定分布在三个方法中，比较运算符**不一致**：

### 1.1 GlyphTooLarge 判定 — 严格大于 `>`

[atlas.rs:124-126](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/atlas.rs#L124-L126)

```rust
if glyph.width > self.width || glyph.height > self.height {
    return Err(AtlasInsertError::GlyphTooLarge);
}
```

含义：`height > 1024` 才算 GlyphTooLarge。`height == 1024` **通过**此检查。

### 1.2 room_in_row 高度判定 — 严格小于 `<`

[atlas.rs:225-231](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/atlas.rs#L225-L231)

```rust
pub fn room_in_row(&self, raw: &RasterizedGlyph) -> bool {
    let next_extent = self.row_extent + raw.width;
    let enough_width = next_extent <= self.width;       // <=  包含等于
    let enough_height = raw.height < (self.height - self.row_baseline);  // <  不含等于

    enough_width && enough_height
}
```

宽度用 `<=`（包含边界），高度用 `<`（不含边界）。

含义：`height < 1024 - row_baseline` 才算放得下。当 `height == 1024 - row_baseline` 时，**判定为放不下**。

代码注释（[atlas.rs:22](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/atlas.rs#L22)）明确写了这个语义：

```
│ 10  │     │     │     │     │ <- Empty spaces; can be filled while
│     │     │     │     │     │    glyph_height < height - row_baseline
```

### 1.3 advance_row 判定 — 小于等于 `<= 0`

[atlas.rs:234-238](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/atlas.rs#L234-L238)

```rust
pub fn advance_row(&mut self) -> Result<(), AtlasInsertError> {
    let advance_to = self.row_baseline + self.row_tallest;
    if self.height - advance_to <= 0 {
        return Err(AtlasInsertError::Full);
    }
    self.row_baseline = advance_to;
    ...
}
```

含义：`1024 - advance_to <= 0` 即 `advance_to >= 1024` 时报 Full。剩余 0 像素也不允许推进。

---

## 二、height == 1024 时的完整路径推演

### 2.1 空 Atlas 插入 height=1024 的字形

Atlas 初始状态：`row_extent=0, row_baseline=0, row_tallest=0`

```
步骤 1: GlyphTooLarge 检查
  1024 > 1024 → false → 通过

步骤 2: room_in_row 检查
  宽度: 0 + width <= 1024 → 取决于 width，假设 width <= 1024 → true
  高度: 1024 < (1024 - 0) → 1024 < 1024 → false
  结果: 放不下 → 进入 advance_row

步骤 3: advance_row
  advance_to = 0 + 0 = 0
  1024 - 0 = 1024, 1024 <= 0 → false → Ok
  新状态: row_baseline=0, row_extent=0, row_tallest=0（未变！）

步骤 4: 再次 room_in_row 检查
  同步骤 2 → 仍然 false → 返回 Err(Full)
```

**结论：height=1024 在空 Atlas 上也放不下，返回 Full。**

### 2.2 满 Atlas 满载扩展路径（load_glyph 中的递归）

[atlas.rs:251-287](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/atlas.rs#L251-L287)

当 `insert()` 返回 `Full` 时，`load_glyph` 的处理：

```
1. atlas[0].insert(glyph{height=1024}) → Full
2. current_atlas += 1 → current_atlas = 1
3. current_atlas == atlas.len() (1 == 1) → 创建新 Atlas
   Atlas::new(1024, ...) → 全新空 Atlas
   atlas.push(new) → Vec 长度 = 2
4. 递归调用 load_glyph(active_tex, atlas, current_atlas=1, rasterized)
5. atlas[1].insert(glyph{height=1024}) → 同 2.1 推演 → Full
6. current_atlas += 1 → current_atlas = 2
7. current_atlas == atlas.len() (2 == 2) → 又创建新 Atlas
   atlas.push(new) → Vec 长度 = 3
8. 递归调用 load_glyph(active_tex, atlas, current_atlas=2, rasterized)
   ... 无限继续 ...
```

**结论：height=1024 的字形导致 `load_glyph` 无限递归，最终栈溢出。**

这是一个**潜在 bug**——代码中的"死区"：字形既不被 GlyphTooLarge 拦截（`>` 不含等于），也永远无法被任何 Atlas 容纳（`<` 不含等于）。

### 2.3 与 width == 1024 的对比（不对称性）

| 维度 | GlyphTooLarge | room_in_row | height=1024 可插入？ |
|------|:---:|:---:|:---:|
| **宽度** | `width > 1024` | `row_extent + width <= 1024` | **可以**（row_extent=0 时 `0+1024 <= 1024` 为 true） |
| **高度** | `height > 1024` | `height < 1024 - row_baseline` | **不可以**（`1024 < 1024` 为 false） |

宽度判定用 `<=`（包容边界），高度判定用 `<`（排斥边界）。这导致：
- 最大可插入宽度 = 1024（= ATLAS_SIZE）
- 最大可插入高度 = 1023（= ATLAS_SIZE - 1）

---

## 三、各边界值的完整分类

设 `H = ATLAS_SIZE = 1024`，空 Atlas（row_baseline=0）：

| 字形高度 | GlyphTooLarge? | room_in_row(空Atlas)? | 最终结果 |
|:---:|:---:|:---:|:---:|
| 0 | ✗ (`0 > 1024` = false) | ✓ (`0 < 1024`) | 正常插入 |
| 1 ~ 1023 | ✗ | ✓ | 正常插入 |
| **1024** | ✗ (`1024 > 1024` = false) | ✗ (`1024 < 1024` = false) | **Full → 无限递归** |
| 1025+ | ✓ (`1025 > 1024` = true) | 不执行 | **GlyphTooLarge → 零尺寸 Glyph** |

已有行数据时（row_baseline=B > 0, row_tallest=T > 0）：

| 字形高度 | GlyphTooLarge? | room_in_row? | advance_row 后? | 最终结果 |
|:---:|:---:|:---:|:---:|:---:|
| ≤ H-B-T-1 | ✗ | ✓ (当前行宽度够时) | — | 正常插入 |
| H-B-T | ✗ | ✗ (`H-B-T < H-B` = false) | 新 row_baseline=B+T, `H-B-T < H-B-T` = false | **Full → 新 Atlas** |
| H-B | ✗ | ✗ | 新 row_baseline=B+T, 检查 `H-B < H-B-T` | **Full → 新 Atlas** |
| H (=1024) | ✗ | ✗ | 无论 B 为何值，`H < H-B` 恒 false | **Full → 无限递归** |
| > H | ✓ | — | — | **GlyphTooLarge** |

**关键**：只有 `height == 1024` 这个精确值才会触发无限递归。对于 `height <= 1023`，即使当前 Atlas 放不下，在新创建的空 Atlas 上一定能放（因为 `height < 1024 - 0 = 1024` 为 true）。对于 `height >= 1025`，GlyphTooLarge 拦截，返回零尺寸 Glyph。

---

## 四、实际发生概率分析

### 4.1 正常终端使用

终端字体通常 8~24pt，DPI 缩放后字形高度约 10~50px。远低于 1024px 阈值，**不可能触发**。

### 4.2 极端字体大小

要使字形高度达到 1024px，需要约 700~800pt 的字体大小。用户不太可能这样配置，但技术上可以。

### 4.3 高 DPI + 大字号的组合

在 4x DPI 缩放下，200pt 字体的字形可能接近 1024px。但 Alacritty 会在 `Display::update_font_size` 中将 `font.size` 乘以 `scale_factor`，所以实际传给 crossfont 的 size 已经是缩放后的值。

### 4.4 特殊字形

某些 emoji 或彩色字形可能在较高字号下产生大尺寸位图。但 1024px 仍然是极端情况。

---

## 五、advance_row 的精确边界

[atlas.rs:234-238](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/atlas.rs#L234-L238)

```rust
let advance_to = self.row_baseline + self.row_tallest;
if self.height - advance_to <= 0 {
    return Err(AtlasInsertError::Full);
}
```

| advance_to 值 | `1024 - advance_to` | `<= 0`? | 结果 |
|:---:|:---:|:---:|:---:|
| 0 | 1024 | ✗ | Ok，row_baseline=0 |
| 1 ~ 1023 | 1023 ~ 1 | ✗ | Ok，row_baseline=advance_to |
| 1024 | 0 | ✓ | **Full** |
| > 1024 | < 0 | ✓ | **Full**（理论上不应出现） |

**注意**：当 `advance_to = 1023` 时，`advance_row` 成功，`row_baseline = 1023`。此时 `room_in_row` 要求 `height < (1024 - 1023) = 1`，即只有 `height = 0` 的字形能放入——实际上无法利用。

当 `advance_to = 1024` 时（恰好用完全部高度），`advance_row` 返回 Full，不会再推进——这是正确的。

---

## 六、insert 内部的完整控制流图

```
insert(glyph)
│
├─ glyph.width > 1024 || glyph.height > 1024 ?
│   └─ 是 → GlyphTooLarge（零尺寸 Glyph）
│
├─ room_in_row(glyph) ?
│   │  宽度: row_extent + glyph.width <= 1024
│   │  高度: glyph.height < (1024 - row_baseline)
│   │
│   ├─ 是 → insert_inner(glyph) → Ok(Glyph)
│   │
│   └─ 否 → advance_row()
│       │
│       ├─ 1024 - (row_baseline + row_tallest) <= 0 ?
│       │   └─ 是 → Full（向上传播到 load_glyph）
│       │
│       └─ 否 → row_baseline += row_tallest
│               row_extent = 0, row_tallest = 0
│               │
│               └─ room_in_row(glyph) 再次检查 ?
│                   ├─ 是 → insert_inner(glyph) → Ok(Glyph)
│                   └─ 否 → Full（向上传播到 load_glyph）
```

---

## 七、load_glyph 的递归与终止条件

[atlas.rs:251-287](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/atlas.rs#L251-L287)

```
load_glyph(active_tex, atlas, current_atlas, rasterized)
│
├─ atlas[current_atlas].insert(rasterized)
│   │
│   ├─ Ok(glyph) → 返回（递归终止 ✓）
│   │
│   ├─ Err(GlyphTooLarge) → 返回零尺寸 Glyph（递归终止 ✓）
│   │
│   └─ Err(Full)
│       ├─ current_atlas += 1
│       ├─ current_atlas == atlas.len() ?
│       │   └─ 是 → Atlas::new() + push
│       └─ 递归: load_glyph(active_tex, atlas, current_atlas, rasterized)
```

### 正常终止条件

对于 `height <= 1023` 的字形，新创建的空 Atlas 一定能容纳（`height < 1024`），递归 1 层即终止。

### 异常情况：height == 1024

新创建的空 Atlas 仍无法容纳（`1024 < 1024` 为 false），再次返回 Full → 再次创建 → 再次 Full → **无限递归，直到栈溢出**。

### 根因分析

两处判定运算符的**间隙**：

```
GlyphTooLarge:  height > 1024  →  拦截 [1025, +∞)
room_in_row:    height < 1024  →  允许 [0, 1023]

                  1024 恰好落入两者之间的死区
```

若 GlyphTooLarge 改为 `height >= 1024`（`>=`），或 room_in_row 高度改为 `<=`，则可消除此死区。当前代码保持了注释中的设计意图（`glyph_height < height - row_baseline`），但未在 GlyphTooLarge 侧对齐这一约束。

---

## 八、总结

| 边界问题 | 精确结论 |
|---------|---------|
| Atlas 最大可插入高度 | **1023px**（不是 1024），因为 room_in_row 用 `<` |
| Atlas 最大可插入宽度 | **1024px**，因为 room_in_row 宽度用 `<=` |
| height=1024 的行为 | 通过 GlyphTooLarge 检查，但永远无法放入任何 Atlas → Full → **无限递归** |
| height=1025+ 的行为 | GlyphTooLarge → 零尺寸 Glyph，不递归 |
| height≤1023 且当前 Atlas 放不下 | Full → 创建新 Atlas → 新 Atlas 能容纳 → 递归 1 层终止 |
| advance_row 剩余 0px | `1024 - advance_to <= 0` → Full，正确拦截 |
| 宽高判定不对称 | 宽度 `<=` 包容、高度 `<` 排斥，这是代码注释中的显式设计 |
