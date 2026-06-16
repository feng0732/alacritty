# Alacritty TOML 配置监听实现梳理

---

## 一、参与监听流程的核心文件

| 文件 | 角色 |
|------|------|
| `alacritty/src/config/monitor.rs` | ConfigMonitor 结构体：创建 watcher 线程、防抖、路径匹配、发送 ConfigReload 事件 |
| `alacritty/src/event.rs` | Processor 在 `user_event()` 中处理 ConfigReload 事件：重载配置 + 判断 Monitor 重启 + 更新窗口 |
| `alacritty/src/config/mod.rs` | `reload()` / `parse_config()` / `load_imports()`：配置文件的完整解析与 import 递归 |
| `alacritty/src/config/ui_config.rs` | `UiConfig.config_paths` 字段：记录本次加载涉及的所有文件路径（含 import） |
| `alacritty/src/window_context.rs` | `update_config()`：将新配置差异应用到每个窗口 |
| `alacritty/src/main.rs` | `alacritty()` 函数：启动时创建 Processor → 创建 ConfigMonitor |

---

## 二、启动阶段：ConfigMonitor 的创建

入口在 `alacritty/src/main.rs` 的 `alacritty()` 函数，它调用 `Processor::new()`，后者在 `alacritty/src/event.rs:126-128` 中：

```rust
if config.live_config_reload() {
    config_monitor = ConfigMonitor::new(config.config_paths.clone(), event_loop.create_proxy());
}
```

前置条件：`general.live_config_reload` 为 true（默认值）。
传入参数：`config.config_paths`（所有被加载的配置文件路径，含 import），以及 `EventLoopProxy`。

---

## 三、ConfigMonitor::new() 实现

完整代码位于 `alacritty/src/config/monitor.rs:33-155`。

### 3.1 路径预处理（四步）

```
输入: paths = config.config_paths（来自 parse_config 的原始路径列表）
  │
  ├─ 1. paths.is_empty()? → 是则返回 None，不创建监听
  │
  ├─ 2. watched_hash = hash_paths(&paths)   ← 对原始路径计算哈希，保存到结构体
  │
  ├─ 3. paths.retain(|p| p.metadata().is_file())  ← 排除字符设备/socket
  │      注：metadata() 会跟随符号链接，软链指向的真实文件是普通文件则保留
  │
  └─ 4. 逐个 canonicalize（代码位于 alacritty/src/config/monitor.rs:49-57）：
         for i in 0..paths.len() {
             if let Ok(canonical_path) = paths[i].canonicalize() {
                 match paths[i].symlink_metadata() {
                     Ok(m) if m.is_symlink() => paths.push(canonical_path),
                     _                       => paths[i] = canonical_path,
                 }
             }
         }
         ├─ 若 paths[i] 是符号链接 → 保留原软链路径不变，将真实路径追加到 paths 末尾
         └─ 若 paths[i] 是普通文件 → 原地替换为 canonicalize 后的真实路径
```

**关键点**：`watched_hash` 在步骤 2 计算，基于的是**尚未被过滤和规范化的原始 paths**。后续步骤 3、4 会修改 `paths`，但不影响已保存的 `watched_hash`。

#### 3.1.1 paths 在预处理前后的具体差异

假设主配置 `~/.config/alacritty/alacritty.toml` 是软链，指向 `~/dotfiles/alacritty.toml`，并 import 了一个普通文件 `~/alacritty/theme.toml`：

| 阶段 | paths 内容 | 说明 |
|------|------------|------|
| 输入（config.config_paths） | `["~/.config/alacritty/alacritty.toml", "~/alacritty/theme.toml"]` | 都是规范化绝对路径，但未 canonicalize |
| 步骤 3（retain 后） | `["~/.config/alacritty/alacritty.toml", "~/alacritty/theme.toml"]` | 两个都是普通文件（软链跟随检查通过），均保留 |
| 步骤 4（canonicalize 后） | `["~/.config/alacritty/alacritty.toml", "/home/xxx/alacritty/theme.toml", "/home/xxx/dotfiles/alacritty.toml"]` | 第一个是软链→保留原路径+追加真实路径；第二个是普通文件→原地替换为真实路径 |

**核心差异**：符号链接的原始路径会被保留，真实路径追加在末尾；普通文件直接被真实路径替换。

### 3.2 创建 Watcher 线程

```rust
let (tx, rx) = mpsc::channel();
let mut watcher = RecommendedWatcher::new(tx.clone(), Config::default().with_poll_interval(1s))?;
```

线程名为 `"config watcher"`，内部执行：

1. **收集所有唯一父目录**，去重后对每个目录调用 `watcher.watch(parent, RecursiveMode::NonRecursive)`
   - 监听目录而非文件，因为部分平台的原子保存会先删除再创建文件
   - `NonRecursive`：不递归监听子目录

2. **事件循环**：从 `rx` 接收 notify 事件，经过防抖后向主线程发送 `ConfigReload`

### 3.3 防抖算法

代码位于 `alacritty/src/config/monitor.rs:92-141`。

常量：`DEBOUNCE_DELAY = 10ms`

```
loop {
    if debouncing_deadline is None {
        event = rx.recv()                      // 阻塞等待第一个事件
        debouncing_deadline = now + 10ms       // 启动倒计时
        存入 received_events
    } else {
        event = rx.recv_timeout(remaining)     // 带超时等待
        if 收到事件 → 存入 received_events
        if Timeout {
            debouncing_deadline = None          // 重置
            if 事件路径与 paths 有交集 {
                send Event::new(ConfigReload(paths[0]), None)
            }
        }
    }
}
```

- 第一个事件到达时才开始 10ms 窗口
- 窗口内的后续事件被累积
- 超时后一次性检查所有累积事件的路径

### 3.4 事件过滤与路径匹配

**接受的事件**：`EventKind::Any` | `EventKind::Create(_)` | `EventKind::Modify(_)` | `EventKind::Other`

**忽略的事件**：
- `Modify(Name(From | Both))`：文件被移走（编辑器原子保存的一部分）
- `EventKind::Remove(_)`：文件删除
- `EventKind::Other + info == "shutdown"`：关机信号，退出循环

**路径匹配**：超时后检查 `received_events` 中所有路径是否与 `paths`（规范化的监听列表）有交集。

**事件发送**：始终发送 `Event::new(EventType::ConfigReload(paths[0].clone()), None)`。
- 无论变更的是主配置还是 import 文件，都发送主配置路径触发完整重载
- `paths[0]` 的具体含义取决于主配置是否为符号链接：
  - 主配置是普通文件 → `paths[0]` = canonicalize 后的真实路径
  - 主配置是软链 → `paths[0]` = **原始软链路径**（真实路径被追加到 paths 末尾）

#### 3.4.1 父目录监听范围

watcher 实际监听的是 `paths` 中每个路径的**父目录**（去重后），而非文件本身。对于软链主配置，会同时监听：
- 软链所在目录（如 `~/.config/alacritty/`）
- 真实文件所在目录（如 `~/dotfiles/`）

这意味着无论编辑器修改软链路径还是真实路径的文件，事件都能被捕获。

#### 3.4.2 路径匹配的双通道

由于软链的 `paths` 中同时包含软链路径和真实路径，notify 事件携带的路径无论等于哪一个，都能通过 `any(|path| paths.contains(&path))` 的匹配检查。这兼容了不同编辑器的保存策略：
- 部分编辑器写入软链 → 事件路径为软链路径
- 部分编辑器跟随软链写入真实文件 → 事件路径为真实路径
- 部分编辑器先删后建真实文件 → 事件路径为真实路径

### 3.5 关机机制

代码位于 `alacritty/src/config/monitor.rs:158-168`。

```rust
pub fn shutdown(self) {
    let mut event = NotifyEvent::new(EventKind::Other);
    event = event.set_info("shutdown");
    self.shutdown_tx.send(Ok(event));
    self.thread.join();   // 同步等待线程退出
}
```

通过 `shutdown_tx`（与 `rx` 同一个 channel 的发送端）发送带 `"shutdown"` 标记的事件，线程识别后 break 退出循环。

---

## 四、主线程处理 ConfigReload 事件

代码位于 `alacritty/src/event.rs:343-371`。

```rust
(EventType::ConfigReload(path), _) => {
    // 1. 清除所有窗口的配置错误消息
    for window_context in self.windows.values_mut() {
        window_context.message_buffer.remove_target(LOG_TARGET_CONFIG);
    }

    // 2. 重新加载配置
    if let Ok(config) = config::reload(&path, &mut self.cli_options) {
        self.config = Rc::new(config);

        // 3. 判断是否需要重启 Monitor
        if let Some(monitor) = self.config_monitor.take() {
            let paths = &self.config.config_paths;
            self.config_monitor = if monitor.needs_restart(paths) {
                monitor.shutdown();
                ConfigMonitor::new(paths.clone(), self.proxy.clone())
            } else {
                Some(monitor)
            };
        }

        // 4. 逐窗口应用新配置
        for window_context in self.windows.values_mut() {
            window_context.update_config(self.config.clone());
        }
    }
}
```

**步骤 2（config::reload）** 的内部流程：

`alacritty/src/config/mod.rs:150-159` → `load_from()` → `read_config()` → `parse_config()`

```
parse_config(path, config_paths, recursion_limit)
  ├─ config_paths.push(path)              ← 记录此文件路径
  ├─ deserialize_config(path)             ← 读文件 + 去 BOM + YAML 兼容 + TOML 解析 → Value
  ├─ load_imports(&config, path, ...)     ← 递归解析 import 列表
  │    ├─ imports(&config, base_path)     ← 从 Value 中提取 import 数组
  │    ├─ normalize_import()              ← ~/ 前缀展开、相对路径补全
  │    └─ 对每个 import: parse_config(..., recursion_limit - 1)
  └─ serde_utils::merge(imports, config)  ← import 在前，当前文件在后（右偏合并）
```

**`config_paths` 的构成**：按 DFS 前序遍历收集所有被加载的文件路径。主配置文件排在第一个。

`config_paths` 中每条路径的形态：
- 主配置路径：即传入 `config::reload()` 的参数。如果主配置是软链，此处记录的是**软链路径**（因为 `parse_config` 直接 `push(path.to_owned())`，不做 canonicalize）。
- Import 路径：先经 `normalize_import`（alacritty/src/config/mod.rs:318-333）处理，展开 `~/`、相对路径补全为绝对路径，但**不调用 canonicalize**。因此如果 import 本身指向软链，此处记录的仍是软链路径。

每次 reload 都**完整重走**这条链路，产生全新的 `config_paths` 列表。

---

## 五、监听重启判断：needs_restart 的真实逻辑

### 5.1 函数定义

代码位于 `alacritty/src/config/monitor.rs:170-176`。

```rust
/// Check if the config monitor needs to be restarted.
///
/// This checks the supplied list of files against the monitored files to determine if a
/// restart is necessary.
pub fn needs_restart(&self, files: &[PathBuf]) -> bool {
    Self::hash_paths(files).is_none_or(|hash| Some(hash) == self.watched_hash)
}
```

### 5.2 逐步推演

`is_none_or(f)` 的语义（Rust 标准库 `Option` 方法）：
- 若 `Option` 为 `None` → 返回 `true`
- 若 `Option` 为 `Some(v)` → 返回 `f(v)`

所以 `needs_restart` 的返回值：

| `hash_paths(files)` 结果 | `Some(hash) == self.watched_hash` | 返回值 | 含义 |
|--------------------------|-----------------------------------|--------|------|
| `None`（文件数 > 1024） | — | `true` | 无法比较 → 重启 |
| `Some(h)`，h == watched_hash | `true` | `true` | **路径未变 → 重启** |
| `Some(h)`，h != watched_hash | `false` | `false` | **路径变了 → 不重启** |

### 5.3 与注释/调用处语义的矛盾

调用处注释写的是：
```rust
// Restart config monitor if imports changed.
```

但代码的实际行为是：
- 路径**相同** → `needs_restart` 返回 `true` → 执行 `monitor.shutdown()` + `ConfigMonitor::new()` → **重启**
- 路径**不同**（import 增删） → `needs_restart` 返回 `false` → `Some(monitor)` → **不重启**

函数名和注释表达的意图是 "import 变了就重启"，但代码实现了**相反**的逻辑。

### 5.4 对三种场景的实际影响

| 场景 | config_paths 变化 | needs_restart | 实际行为 | 预期行为 |
|------|-------------------|---------------|----------|----------|
| A. 改主配置内容，import 不变 | 不变 | `true` | 重启 Monitor（浪费） | 不需要重启 |
| B. 改 import 文件内容 | 不变 | `true` | 重启 Monitor（浪费） | 不需要重启 |
| C. 主配置增/删 import 条目 | 变了 | `false` | 不重启 Monitor | 应当重启 |

场景 C 的后果：
- **新增 import**：新文件不在 watcher 的监听列表中 → 修改该文件不会触发热重载
- **删除 import**：旧文件仍在监听列表中 → 修改该文件仍会触发重载（但已无效果）

### 5.5 hash_paths 的实现

代码位于 `alacritty/src/config/monitor.rs:179-197`。

```rust
fn hash_paths(files: &[PathBuf]) -> Option<u64> {
    const MAX_PATHS: usize = 1024;
    if files.len() > MAX_PATHS { return None; }

    let mut sorted_files = [None; MAX_PATHS];    // 栈上固定数组
    for (i, file) in files.iter().enumerate() {
        sorted_files[i] = Some(file);
    }
    sorted_files.sort_unstable();                 // 排序消除顺序影响

    let mut hasher = DefaultHasher::new();
    Hash::hash_slice(&sorted_files, &mut hasher); // 对排序后的路径计算哈希
    Some(hasher.finish())
}
```

- 文件数超 1024 → 返回 None（无法比较，保守重启）
- 排序后哈希 → 路径顺序不影响判断
- 栈分配 → 避免堆分配开销

### 5.6 watched_hash 与 config_paths 的一致性

- `watched_hash`：在 `alacritty/src/config/monitor.rs:40` 计算，输入是 `ConfigMonitor::new()` 接收的**原始 `paths` 参数**（即 `config.config_paths` 的 clone）
- `needs_restart` 的 `files` 参数：来自重载后的 `self.config.config_paths`

两者都是 `parse_config` 中 `config_paths.push(path.to_owned())` 收集的原始路径，未经过 canonicalize，因此**哈希具有可比性**。

---

## 六、路径处理细节详解：软链主配置、真实路径、导入文件监听

### 6.1 路径流经的关键节点

```
config.config_paths (parse_config 收集的原始路径)
        │
        │ 传入 ConfigMonitor::new()
        ▼
┌─────────────────────────────────────────────────────────┐
│  ConfigMonitor::new() 路径预处理                        │
│  1. hash_paths(原始 paths) → watched_hash（供重启判断） │
│  2. retain：过滤非文件                                   │
│  3. canonicalize 循环：                                  │
│     ├─ 软链 → 保留原路径，追加真实路径                   │
│     └─ 普通文件 → 替换为真实路径                         │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
            paths（传入 watcher 线程的最终列表）
                           │
             ┌─────────────┴──────────────┐
             ▼                            ▼
   父目录去重后交给 notify 监听    路径匹配时用 paths.contains()
             │                            │
             ▼                            ▼
        操作系统事件             any(|p| paths.contains(p))
                                          │
                                          ▼
                               发送 ConfigReload(paths[0])
                                          │
                                          ▼
                           主线程 config::reload(paths[0])
                                          │
                                          ▼
                           parse_config(paths[0]) → 新的 config_paths
```

### 6.2 主配置为软链时 paths[0] 的完整流转

设主配置 `~/.config/alacritty/alacritty.toml` 是符号链接，指向真实文件 `~/dotfiles/alacritty.toml`。

| 阶段 | paths[0] 的值 | 说明 |
|------|---------------|------|
| 1. 启动时 config_paths[0] | `~/.config/alacritty/alacritty.toml` | parse_config 直接 push 传入路径 |
| 2. ConfigMonitor::new() 输入 | `~/.config/alacritty/alacritty.toml` | 与上相同 |
| 3. watched_hash 计算 | 基于 `~/.config/alacritty/alacritty.toml` | 步骤 2 完成，未修改 |
| 4. canonicalize 循环后 | `~/.config/alacritty/alacritty.toml`（**不变**） | 检测到是软链，保留原路径，真实路径被 push 到 paths 末尾 |
| 5. watcher 线程内 paths[0] | `~/.config/alacritty/alacritty.toml` | 软链路径 |
| 6. 触发重载时发送的路径 | `~/.config/alacritty/alacritty.toml` | `EventType::ConfigReload(paths[0].clone())` |
| 7. 主线程 reload 接收的路径 | `~/.config/alacritty/alacritty.toml` | 软链路径 |
| 8. reload 后新的 config_paths[0] | `~/.config/alacritty/alacritty.toml` | parse_config 直接 push 软链路径 |
| 9. needs_restart 比较 | 新 config_paths hash == watched_hash | 相等 → 返回 true → 重启 Monitor |

**结论**：主配置为软链时，重载路径全程是软链路径，与真实路径无关。reload 时 `fs::read_to_string(软链路径)` 会自动跟随符号链接读取真实文件，因此结果正确。

### 6.3 真实路径监听的作用范围

真实路径（canonicalize 后）被加入 `paths` 列表后，有两个作用：

1. **路径匹配**：notify 事件中的路径如果是真实路径（例如编辑器直接写入真实文件），也能通过 `paths.contains()` 检查。
2. **父目录监听**：真实路径的父目录会被加入 watcher 的监听列表，使得直接修改真实文件所在目录的事件也能被捕获。

真实路径**不参与**：
- `watched_hash` 计算（hash 在 canonicalize 之前就完成了）
- 重载事件发送（始终使用 paths[0]）
- `config_paths` 记录（parse_config 不做 canonicalize）

### 6.4 导入文件的监听与主配置软链的关系

导入文件的路径在 `load_imports` → `normalize_import` 中处理：

```rust
normalize_import(base_config_path, import_path):
  ├─ ~/xxx → home_dir/xxx
  ├─ 相对路径 → base_config_path.parent()/相对路径
  └─ 返回规范化后的绝对路径（不调用 canonicalize）
```

然后 parse_config 直接将此路径 `push` 到 `config_paths`。

**关键关系**：
- import 路径与主配置是否为软链**无关**。import 的 base 是主配置文件所在目录，normalize_import 不跟随主配置的软链去找真实目录。
- 示例：主配置 `~/.config/alacritty/alacritty.toml` 是软链（指向 `~/dotfiles/`），其中写 `import = ["theme.toml"]`。normalize_import 会解析为 `~/.config/alacritty/theme.toml`，**不会**解析为 `~/dotfiles/theme.toml`。
- 如果 import 文件本身是软链，进入 ConfigMonitor 后会被同样处理：保留软链路径 + 追加真实路径，双重监听。

### 6.5 各层路径形态汇总

| 层面 | 主配置（软链） | 主配置（普通文件） | import 文件（软链） | import 文件（普通文件） |
|------|----------------|--------------------|--------------------|------------------------|
| config_paths | 软链路径 | 规范化绝对路径 | 软链路径 | 规范化绝对路径 |
| watched_hash 输入 | 软链路径 | 规范化绝对路径 | 软链路径 | 规范化绝对路径 |
| paths（watcher 内匹配用） | [软链路径, 真实路径] | [真实路径] | [软链路径, 真实路径] | [真实路径] |
| 监听父目录 | 软链目录 + 真实目录 | 真实目录 | 软链目录 + 真实目录 | 真实目录 |
| 重载发送路径 | 软链路径 | 真实路径 | —（用主配置 paths[0]） | — |

---

## 七、三种变更场景的完整时序

### 场景 A：修改主配置内容（import 列表不变）

```
编辑器保存 alacritty.toml
  │
  ├─ 文件系统 Modify 事件
  ├─ Watcher 线程收到，防抖 10ms
  ├─ 路径匹配 → paths 中有主配置文件
  └─ 通过 EventLoopProxy 发送 ConfigReload(paths[0])
       │
       └─ 主线程 Processor::user_event()
            ├─ config::reload(&paths[0])
            │    └─ parse_config() 重新解析所有 imports
            │    └─ config_paths 与上次相同
            ├─ needs_restart(paths) → true（哈希相等）→ shutdown + new（浪费但无害）
            └─ 各窗口 update_config() → 差异更新 → 重绘
```

### 场景 B：修改 import 文件内容

```
编辑器保存 theme.toml（被 alacritty.toml import 的文件）
  │
  ├─ 文件系统 Modify 事件
  ├─ Watcher 线程收到，防抖 10ms
  ├─ 路径匹配 → paths 中有该 import 文件（规范化后的路径）
  └─ 发送 ConfigReload(paths[0])   ← 仍然发送主配置路径
       │
       └─ 主线程 config::reload(&主配置路径)
            └─ 完整重解析整个配置树（含所有 import）
            └─ theme.toml 的变更被正确合并到最终配置中
```

**要点**：import 文件修改触发的是**完整配置树重载**，而非仅重载该 import。因为 `ConfigReload` 始终携带主配置路径，`reload()` 从主配置开始走完整个 `parse_config` → `load_imports` 链路。

### 场景 C：修改 import 列表（增/删 import 条目）

```
编辑器修改 alacritty.toml 的 general.import，新增 ~/theme.toml
  │
  ├─ 文件系统 Modify 事件 → Watcher → ConfigReload(paths[0])
  │
  └─ 主线程 config::reload()
       ├─ parse_config() 发现新增 import → 加载 ~/theme.toml
       ├─ config_paths 变了（多了一个路径）
       ├─ needs_restart(paths)
       │    └─ hash_paths(新路径列表) != watched_hash → 返回 false
       │    └─ 不重启 Monitor
       │
       ├─ 配置内容已正确更新（包含新 import 的内容）
       │
       └─ 但新 import 文件不在 watcher 的监听列表中
          └─ 后续修改 ~/theme.toml 不会触发热重载
```

要让新 import 的文件也被监听，需要重启 Alacritty。

---

## 八、窗口级配置应用

代码位于 `alacritty/src/window_context.rs:261-333`。

`update_config()` 通过 `mem::replace` 保留旧配置，逐一对比新旧差异：

| 配置项 | 对比逻辑 | 变更时操作 |
|--------|----------|------------|
| IPC 窗口覆盖 | 无条件 | `window_config.override_config_rc()` |
| 显示层 | 无条件 | `display.update_config()` |
| 终端选项 | 无条件 | `terminal.set_options(term_options())` |
| 光标厚度 | `abs(old - new) > f32::EPSILON` | `set_cursor_dirty()` |
| 字体 | `old.font != new.font` | 重加载字体，保留运行时字号 |
| 窗口主题 | 无条件 | `set_theme()` |
| padding / resize_increments | 直接比较 | 标脏 |
| 窗口标题 | 三变量判断表 | 条件设置 |
| 透明度/模糊 | 无条件 | `set_transparent()` / `set_blur()` |
| 提示字母表 | 无条件 | `update_alphabet()` |
| 光标闪烁 | 无条件 | 发送 `CursorBlinkingChange` |

标题判断表（来自源码注释）：

```
│ CLI 指定标题 │ dynamic_title │ 当前标题 == 旧配置标题 ││ 设置新标题? │
│      Y      │       _       │          _            ││     N      │
│      N      │       Y       │          Y            ││     Y      │
│      N      │       Y       │          N            ││     N      │
│      N      │       N       │          _            ││     Y      │
```

---

## 九、完整数据流总览

```
                          ┌──────────────────────────────────────────────┐
                          │            启动阶段                          │
                          │                                              │
  main() → Processor::new(config)                                        │
              │                                                          │
              ├─ ConfigMonitor::new(config.config_paths, proxy)          │
              │    ├─ hash_paths(原始 paths) → watched_hash              │
              │    ├─ 过滤非文件 + canonicalize + 符号链接               │
              │    └─ 创建 watcher 线程，监听所有父目录                  │
              │                                                          │
              └─ config: Rc<UiConfig>                                    │
                          │                                              │
                          └──────────────────────────────────────────────┘

                          ┌──────────────────────────────────────────────┐
                          │            运行阶段                          │
                          │                                              │
  文件系统变更                                                    │
      │                                                          │
      ▼                                                          │
  Watcher 线程                                                   │
  ├─ 收到 notify 事件                                            │
  ├─ 防抖 10ms                                                   │
  ├─ 路径匹配                                                    │
  └─ EventLoopProxy.send(ConfigReload(paths[0]))  ──►  主线程     │
                                                      │          │
                                            Processor::user_event()  │
                                            ├─ config::reload()      │
                                            │    └─ 完整重解析        │
                                            │       含所有 imports    │
                                            ├─ needs_restart()?      │
                                            │    ├─ true → 重启      │
                                            │    └─ false → 保留     │
                                            └─ 窗口 update_config() │
                                                          │          │
                                                          ▼          │
                                              Display / Term 更新    │
                                              dirty = true → 重绘    │
                          └──────────────────────────────────────────────┘
```

---

## 十、设计要点

1. **目录监听策略**：watcher 监听父目录，应用层匹配文件路径，兼容原子保存
2. **10ms 防抖**：累积窗口内所有事件，超时后一次性判断
3. **始终发送主配置路径**：任何文件变更都走 `paths[0]` 触发完整重载
4. **完整配置树重解析**：reload 从主配置开始递归处理所有 import
5. **Rc 共享 + 写时克隆**：多窗口共享同一份配置，更新时创建新的 Rc
6. **差异更新**：`update_config` 对比新旧配置，仅处理变更项
7. **符号链接双重监听**：软链文件同时监听原始路径和真实路径，兼容不同编辑器保存策略
8. **路径哈希与顺序无关**：先排序再哈希，`config_paths` 顺序变化不影响判断
9. **软链主配置全程保留软链路径**：paths[0]、重载发送路径、config_paths 全程使用软链路径，`fs::read_to_string` 自动跟随
10. **Import 路径不跟随主配置软链**：normalize_import 基于软链所在目录解析相对 import，不跟随到真实目录
11. **真实路径仅用于匹配与监听**：canonicalize 后的真实路径不参与 hash、重载发送和 config_paths 记录
