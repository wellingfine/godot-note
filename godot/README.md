# Godot 学习笔记

Godot 4（C# 为主）的学习记录，按主题分子目录存放。新增笔记请同步更新本文件的「笔记索引」。

## 目录结构（现状）

```
godot/
├── nodes/      节点与场景用法（4 篇）
├── signals/    信号与事件（3 篇）
├── animation/  动画（1 篇）
├── tooling/    工具链与 IDE（1 篇）
└── README.md   本文件
```

预留类目（有内容时再建）：`scripts/`（示例脚本）、`shaders/`、`physics/`、`input/`、`ui/`、`export/`、`misc/`。

## 笔记索引

### signals/ — 信号与事件

| 笔记 | 内容 |
| --- | --- |
| **[csharp-signal-basics.md](./signals/csharp-signal-basics.md)** ⭐ | `[Signal]` 从零到用：为什么需要信号（打死怪→玩家加分，对比直接 `GetNode` 的耦合）、最小可用示例（声明/发射/订阅/退订）、`[Signal]` 只是编译期标记而 `SignalName` / `MobDied` event / `EmitSignalMobDied` 都是源码生成器产物、名字推导规则、发射方法是 `protected` 所以要包一层对外发射口、四条硬性规则（`GD0201/0202/0203/GD0001`）、Autoload 事件总线完整写法、IDE 标红与重命名的坑、动手验证小实验、术语小抄 |
| [csharp-event-vs-godot-signal.md](./signals/csharp-event-vs-godot-signal.md) | C# 项目该用 `[Signal]` 还是原生 `event`：默认用 event（性能/类型安全/可重构），只在跨 GDScript、要编辑器连线、订阅内置节点事件时用信号；`BodyEntered +=` 的 C# 事件写法，以及「编辑器连线 + 代码订阅导致回调触发两次」的坑 |
| [csharp-signal-emit-pipeline.md](./signals/csharp-signal-emit-pipeline.md) | 发射一次信号走了多远：`dotnet build /p:EmitCompilerGeneratedFiles=true` dump 源码生成器真实产物（`SignalName` / `backing_` 委托 / `EmitSignalXxx` / `RaiseGodotClassSignalCallbacks`）、`int→Variant` 装箱 → 跨 C#/C++ → 引擎 → 拆箱 → 委托 Invoke 的七步链路与每步开销、性能量级判断（一秒几次随便用、一帧几十次换 event）；**修正**：C# 的 `+=` 只是 `backing_ += value`，不走引擎 Connect，所以编辑器/`IsConnected()` 看不到 |

### nodes/ — 节点与场景用法

| 笔记 | 内容 |
| --- | --- |
| [node-path-and-get-node.md](./nodes/node-path-and-get-node.md) | C# 取节点的全部姿势：`/root/Game/ScoreLabel` 绝对路径的真实树形与 `/root` 是什么、相对路径与 `..`、场景唯一名 `%`（只在所属场景内有效）、`[Export]` 拖引用、`GetParent/GetChild/FindChild/Owner`、`GetTree().Root` 与 `CurrentScene`、分组 `GetFirstNodeInGroup`；`GetNode` 找不到是返回 null 而非抛异常、`GetNodeOrNull` 的区别、`_Ready` 取兄弟节点的时序坑与选型表 |
| [timer-node-cooldown.md](./nodes/timer-node-cooldown.md) | Timer 节点做射击冷却：场景里 `wait_time` + `one_shot` 配置、`IsStopped()` 当门锁的写法、`Start/Stop/TimeLeft/Paused` 与 Timeout 信号；对比手动累加 delta 的写法与常见踩坑 |
| [3d-forward-axis-and-basis.md](./nodes/3d-forward-axis-and-basis.md) | 3D 方向约定：为什么「前」是 -Z（右手系 / Camera3D 看 -Z / LookAt 对齐 -Z）；Basis 是旋转+缩放矩阵、轴取列；Basis 与 GlobalBasis、Position 与 GlobalPosition 对照表；`GlobalPosition += -GlobalBasis.Z.Normalized() * speed * delta` 及 Normalized 的必要性 |
| [3d-local-space-and-parent-rotation.md](./nodes/3d-local-space-and-parent-rotation.md) | 3D 局部坐标系与父子变换翻转：glb 导入的 180° 补偿旋转导致 Marker3D 的 -Z 朝后、子弹朝向反了；含 .tscn 的 Transform3D 行列读法、Use Local Space 排查、GlobalTransform 赋值要点 |

### animation/ — 动画

| 笔记 | 内容 |
| --- | --- |
| [animation-tree-oneshot.md](./animation/animation-tree-oneshot.md) | AnimationTree 播放一次性动画：BlendTree 里 OneShot 的 0 号口常态 / 1 号口插播、动画名是 `库名/动画名`（`custom/aa`）、`parameters/OneShot/request` 的 None/Fire/Abort 三种取值、C# `Set()` 写法与子弹命中→Mob→BatModel 的调用链；含「Active 默认关闭导致树不工作」等踩坑 |

### tooling/ — 工具链与 IDE

| 笔记 | 内容 |
| --- | --- |
| [vscode-godot-project-display.md](./tooling/vscode-godot-project-display.md) | VS Code 里让 Godot 项目看着不乱：`.import` / `.uid` 是什么、为什么**不能**进 `.gitignore` 而只能靠 `files.exclude` 隐藏；`files.exclude` / `search.exclude` / `files.watcherExclude` / `explorer.excludeGitIgnore` / `.gitignore` 五种机制的分工；`explorer.fileNesting` 折叠替代隐藏；`.vscode` 被忽略时如何共享配置 |

## 推荐阅读顺序

1. `signals/csharp-signal-basics.md` — 入门首选，⭐
2. `signals/csharp-event-vs-godot-signal.md` — 信号 vs event 选型
3. `signals/csharp-signal-emit-pipeline.md` — 想深挖底层和性能再看这篇
4. `nodes/node-path-and-get-node.md` — 拿得到节点才谈别的
5. `nodes/timer-node-cooldown.md` — 第一个完整小功能
6. `animation/animation-tree-oneshot.md` — 接上表现层
7. `nodes/3d-forward-axis-and-basis.md` → `nodes/3d-local-space-and-parent-rotation.md` — 3D 方向：先看基础约定，再看排查实录

## 按问题速查

| 我想… | 看哪篇 |
| --- | --- |
| 搞懂 `[Signal]` 的声明、发射、订阅 | signals/csharp-signal-basics.md |
| 决定用 `[Signal]` 还是 C# `event` | signals/csharp-event-vs-godot-signal.md |
| 想知道信号发射的底层链路 / 性能开销 | signals/csharp-signal-emit-pipeline.md |
| 取不到节点 / 路径怎么写 | nodes/node-path-and-get-node.md |
| 做攻击冷却、技能 CD | nodes/timer-node-cooldown.md |
| 播一次攻击/受击动画 | animation/animation-tree-oneshot.md |
| 子弹或角色朝向反了 | nodes/3d-local-space-and-parent-rotation.md |
| 「前」到底是哪个轴、Basis 怎么用 | nodes/3d-forward-axis-and-basis.md |
| VS Code 里 Godot 项目太乱 | tooling/vscode-godot-project-display.md |

## 维护约定

- 新笔记放进对应子目录；没有合适类目就新建目录，并在本文件登记。
- 每篇加进「笔记索引」对应表格，用一句话说清「解决什么问题」，关键词（报错码、API 名）尽量写进去，方便全文搜索。


