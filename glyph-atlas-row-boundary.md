# 图集行边界分类：当前行与换行后两套判断的代码级梳理

## 一、变量约定

为表述清晰，先约定符号：

| 符号 | 代码字段 | 含义 |
|:---:|------|------|
| `H` | `self.width/height` | 图集尺寸，固定 1024 |
| `B` | `self.row_baseline` | 当前行基线 Y 坐标（行顶部） |
| `T` | `self.row_tallest` | 当前行已插入字形的最大高度 |
| `E` | `self.row_extent` | 当前行已使用宽度 |
| `gh` | `glyph.height` | 待插入字形高度 |
| `gw` | `glyph.width` | 待插入字形宽度 |

**状态间关系**：
- 当前行已用高度 = `T`（行高由最高字形决定）
- 当前行底部 Y 坐标 = `B + T`
- 当前行基线到纹理底部的距离 = `H - B`
- 下一行基线到纹理底部的距离 = `H - (B + T)`

---

## 二、insert 完整控制流（代码级）

[atlas.rs:119-140](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/atlas.rs#L119-L140)

```rust
pub fn insert(&mut self, glyph: &RasterizedGlyph, active_tex: &mut u32)
    -> Result<Glyph, AtlasInsertError>
{
    // ① 入口：GlyphTooLarge 门槛
    if glyph.width > self.width || glyph.height > self.height {
        return Err(AtlasInsertError::GlyphTooLarge);
    }

    // ② 第一套判断：当前行能不能放
    if !self.room_in_row(glyph) {
        self.advance_row()?;    // ③ 过渡：尝试换行
    }

    // ④ 第二套判断：新行能不能放
    if !self.room_in_row(glyph) {
        return Err(AtlasInsertError::Full);
    }

    // ⑤ 写入
    Ok(self.insert_inner(glyph, active_tex))
}
```

控制流结构：`GlyphTooLarge → room_in_row① → advance_row → room_in_row② → insert_inner`

两次调用的是**同一个** `room_in_row` 方法，但因为 `advance_row()` 可能改变了 `row_baseline`，所以两次的判断基准不同。

---

## 三、第一套判断：当前行可容纳性

### 3.1 room_in_row 的判定式

[atlas.rs:225-231](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/atlas.rs#L225-L231)

```rust
pub fn room_in_row(&self, raw: &RasterizedGlyph) -> bool {
    let next_extent = self.row_extent + raw.width;
    let enough_width = next_extent <= self.width;       // 宽度：<= （包容边界）
    let enough_height = raw.height < (self.height - self.row_baseline);  // 高度：< （排斥边界）
    enough_width && enough_height
}
```

**宽度判定**：`E + gw <= H`
- 边界值 `gw = H - E` 时：判定为**能放下**（`<=` 包含等于）

**高度判定**：`gh < H - B`
- 边界值 `gh = H - B` 时：判定为**放不下**（`<` 不含等于）
- 注意：这里比较的是"字形高度"和"从基线到纹理底部的总距离"，**不是**"当前行剩余高度"。因为一行内所有字形共享同一基线，高度各异，行高由最高者决定，所以不存在"行的剩余高度"这种概念。

### 3.2 当前行可容纳的字形高度范围

| gh 范围 | 高度判定 | 说明 |
|:---:|:---:|------|
| `gh <= H - B - 1` | ✅ 通过 | 字形放得下 |
| `gh == H - B` | ❌ 不通过 | 恰好等于 → 排斥 |
| `gh >= H - B + 1` | ❌ 不通过 | 超出底部 |

最大可插入高度 = `H - B - 1`，单位 px。

### 3.3 当前行可容纳的字形宽度范围

| gw 范围 | 宽度判定 | 说明 |
|:---:|:---:|------|
| `gw <= H - E` | ✅ 通过 | 宽度足够 |
| `gw == H - E` | ✅ 通过 | 恰好等于 → 包容 |
| `gw >= H - E + 1` | ❌ 不通过 | 超出右侧 |

最大可插入宽度 = `H - E`。

---

## 四、过渡判断：能不能换行

### 4.1 advance_row 的判定式

[atlas.rs:234-245](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/atlas.rs#L234-L245)

```rust
pub fn advance_row(&mut self) -> Result<(), AtlasInsertError> {
    let advance_to = self.row_baseline + self.row_tallest;
    if self.height - advance_to <= 0 {       // <= 0 就 Full
        return Err(AtlasInsertError::Full);
    }
    self.row_baseline = advance_to;   // 新基线 = 旧行底部
    self.row_extent = 0;
    self.row_tallest = 0;
    Ok(())
}
```

判定式：`H - (B + T) <= 0` → Full

即：`B + T >= H` 时不能换行。

### 4.2 换行的边界

| `B + T` 范围 | 能否换行 | 说明 |
|:---:|:---:|------|
| `B + T <= H - 1` | ✅ 能 | 还有至少 1px 空间 |
| `B + T == H` | ❌ 不能 | 恰好用满 → Full |
| `B + T >= H + 1` | ❌ 不能 | 理论上不应出现（越界了） |

换行后新基线 `B' = B + T`，新行状态 `T' = 0, E' = 0`。

**注意**：`advance_row` 只检查"下一行的起点是否还在纹理内"，**不检查**字形能不能放进新行。字形放不放得下交给第二次 `room_in_row`。

---

## 五、第二套判断：新行可容纳性

换行成功后，第二次调用 `room_in_row`，此时基线已更新为 `B' = B + T`，行内状态重置为 `T' = 0, E' = 0`。

### 5.1 新行的高度判定

判定式：`gh < H - B'` = `gh < H - (B + T)`

**新行可容纳的字形高度范围**：

| gh 范围 | 高度判定 |
|:---:|:---:|
| `gh <= H - (B + T) - 1` | ✅ 通过 |
| `gh == H - (B + T)` | ❌ 不通过 |
| `gh >= H - (B + T) + 1` | ❌ 不通过 |

新行最大可插入高度 = `H - (B + T) - 1`。

### 5.2 新行的宽度判定

判定式：`E' + gw <= H` = `0 + gw <= H` = `gw <= H`

由于 GlyphTooLarge 检查已经保证 `gw <= H`（GlyphTooLarge 是 `>` 才拦截），所以**第二次 `room_in_row` 的宽度检查恒为 true**。

---

## 六、两套判断的过渡关系

### 6.1 高度方向的过渡

第一次高度阈值：`H - B`（从当前基线到底部）
第二次高度阈值：`H - (B + T)`（从新基线到底部）

因为 `T >= 0`，所以 `H - (B + T) <= H - B` → **第二次阈值 ≤ 第一次阈值**。

高度方向的三条区域：

```
纹理顶部 (y=0)
  │
  │  区域 A：gh < H - (B + T)
  │    → 两次高度检查都通过
  │    → 如果宽度第一次也通过 → 直接插入当前行
  │    → 如果宽度第一次不通过 → 换行 → 插入新行
  │
  ├── H - (B + T)  ——— 第二道门槛（更严格）
  │
  │  区域 B：H - (B + T) <= gh < H - B
  │    → 第一次高度检查通过（当前行能放下）
  │    → 但如果触发换行（宽度不够），第二次高度检查不通过
  │    → 结果：若宽度够 → 成功；若宽度不够 → Full
  │
  ├── H - B  ——— 第一道门槛（更宽松）
  │
  │  区域 C：gh >= H - B
  │    → 第一次高度检查不通过
  │    → 尝试换行
  │    → 第二次高度检查必然也不通过（阈值更小）
  │    → 结果：Full
  │
纹理底部 (y=H)
```

### 6.2 宽度方向的过渡

第一次宽度阈值：`H - E`（当前行剩余宽度）
第二次宽度阈值：`H`（新行整行宽度）

因为 `E >= 0`，所以 `H - E <= H` → **第二次宽度阈值 ≥ 第一次宽度阈值**。

但由于 GlyphTooLarge 已经保证 `gw <= H`，所以第二次宽度检查恒成立。宽度方向只有第一次检查可能失败。

### 6.3 失败路径总览

```
当前行放不下 (room_in_row① false)
        │
        ├─ 不能换行 (advance_row 失败) → Full
        │
        └─ 能换行 (advance_row 成功)
            │
            └─ 新行还放不下 (room_in_row② false)
                │
                ├─ 宽度原因？ → 不可能（GlyphTooLarge 已保证 gw <= H）
                │
                └─ 高度原因 → gh >= H - (B + T)
                    ├─ 若 gh > H → 已被 GlyphTooLarge 拦截，走不到这里
                    └─ 若 H - (B + T) <= gh <= H → 走到 Full，但还没到 GlyphTooLarge
```

---

## 七、完整边界分类表（按字形高度分组）

设 `H = 1024`，当前 Atlas 非空（`B > 0, T > 0`）：

| gh 区间 | GlyphTooLarge | 第一次高度 | 能否换行 | 第二次高度 | 最终结果 |
|:---:|:---:|:---:|:---:|:---:|:---:|
| `gh > H` (≥1025) | ✅ 拦截 | — | — | — | **GlyphTooLarge → 零尺寸** |
| `gh == H` (=1024) | ❌ 通过 | ❌ 失败 | 取决于 B+T | ❌ 失败 | **Full → 新 Atlas**（无限递归，见 §7.2） |
| `H - B < gh < H` | ❌ 通过 | ❌ 失败 | 取决于 B+T | ❌ 失败 | **Full → 新 Atlas** |
| `gh == H - B` | ❌ 通过 | ❌ 失败 | 取决于 B+T | ❌ 失败 | **Full → 新 Atlas** |
| `H - (B+T) < gh < H - B` | ❌ 通过 | ✅ 通过 | — | — | 若宽度够 → **成功**；若宽度不够 → 换行 → **Full** |
| `gh == H - (B+T)` | ❌ 通过 | ✅ 通过 | — | — | 若宽度够 → **成功**；若宽度不够 → 换行 → **Full** |
| `gh < H - (B+T)` | ❌ 通过 | ✅ 通过 | — | — | 宽度够 → **成功**；宽度不够 → 换行 → **成功** |

### 7.1 三类 Full 的区别

| Full 触发场景 | 原因 | 后续行为（load_glyph 中） |
|------|------|------|
| `advance_row` 失败 | `B + T >= H`，没地方放下一行了 | 新 Atlas → 成功插入 |
| `room_in_row②` 高度失败（gh < H） | 字形太高但没超过纹理尺寸，新行也放不下 | 新 Atlas → 成功插入 |
| `room_in_row②` 高度失败（gh == H） | 字形恰好等于纹理尺寸，任何 Atlas 都放不下 | 新 Atlas → 还是 Full → 无限递归 |

### 7.2 gh == H 的死区详解

**唯一会触发无限递归的情形**：字形高度恰好等于 Atlas 尺寸（1024px）。

递归路径：
```
atlas[0].insert(gh=1024) → Full
current_atlas += 1 → 1
atlas.len() == 1 → 创建 atlas[1]（空 Atlas）
atlas[1].insert(gh=1024) → Full（空 Atlas 也放不下，因 gh < H 为 false）
current_atlas += 1 → 2
atlas.len() == 2 → 创建 atlas[2]
atlas[2].insert(gh=1024) → Full
... 无限继续 ...
```

根本原因：GlyphTooLarge 用 `>`，room_in_row 高度用 `<`，两者之间留了 `gh == H` 这一个像素的死区。

---

## 八、容易混淆的点总结

### 混淆点 1："当前行剩余高度"的说法不准确

没有"当前行的剩余高度"这种说法。一行内所有字形共享同一条基线，字形可以有不同高度，行高由最高字形动态决定。`room_in_row` 的高度判断是在问：**这个字形会不会伸出纹理底部**（`gh < H - B`），而不是"行内还剩多少高度"。

### 混淆点 2：第一次高度通过 ≠ 换行后高度也通过

如果第一次 `room_in_row` 因**宽度**不够而失败，触发了 `advance_row`，那么第二次 `room_in_row` 的高度阈值变小了（基线往下移了 T px），第一次高度通过不代表第二次也通过。

例：`H=1024, B=0, T=800, gh=300`
- 第一次高度：`300 < 1024-0 = 1024` → ✅ 通过
- 因宽度不够换行
- 第二次高度：`300 < 1024-800 = 224` → ❌ 不通过 → Full

### 混淆点 3：advance_row 的 Full ≠ room_in_row② 的 Full

- `advance_row` 返回 Full：连下一行的起点都超出纹理了，彻底没空间
- `room_in_row②` 返回 Full：下一行有空间，但这个字形塞不下（字形太高）

两者虽然都返回 `Full`，但含义不同。前者是"图集没空间了"，后者是"这个字形太高，当前图集容不下"。

### 混淆点 4：GlyphTooLarge 和 Full 的分界

- GlyphTooLarge：字形**本身尺寸**超过纹理（`gh > H`），换任何 Atlas 都没用
- Full：字形尺寸不超过纹理，但**当前 Atlas 剩余空间**放不下
- 死区 `gh == H`：名义上没超过纹理（`>` 不含等于），但实际上任何 Atlas 都放不下，表现为 Full 递归

---

## 九、行装箱算法的内存效率

由于行高由最高字形决定，混排不同高度的字形会浪费空间：

```
┌───────────────────────────┐
│ ┌───┐                     │
│ │   │ ┌─┐ ┌─┐ ┌───┐       │  ← 行高 = 最高字形(左1)
│ │   │ │ │ │ │ │   │       │     矮字形上方空白被浪费
│ └───┘ └─┘ └─┘ └───┘       │
├───────────────────────────┤
│ 行基线 B，行高 T           │
└───────────────────────────┘
```

这也意味着如果一行内先插入了矮字形，后插入了更高的字形，`row_tallest` 会被撑高，这行的有效高度增加，但之前已插入的矮字形仍然在原位置，不会重新排列。

`insert_inner` 中更新 `row_tallest` 的代码（[atlas.rs:200-210](file:///d:/fz/0601/solo-dogfeeding/code/335-alacritty/alacritty/src/renderer/text/atlas.rs#L200-L210)）：

```rust
// Advance in row.
self.row_extent += width as i32;

// Update row tallest glyph if necessary.
if height as i32 > self.row_tallest {
    self.row_tallest = height as i32;
}
```

即 `row_tallest` 是**动态单调增长**的，只有插入更高的字形才会更新。当 `advance_row` 时，才用这个最终的 `row_tallest` 计算下一行基线。
