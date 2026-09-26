# Godot 学习笔记

本目录用于记录 Godot 引擎的学习内容，建议按主题分子目录，例如：

- `nodes/` — 节点与场景用法（Node2D、Control、CharacterBody2D 等）
- `animation/` — 动画（AnimationPlayer / AnimationTree、OneShot、StateMachine）
- `scripts/` — 示例脚本（GDScript 或 C#）
- `signals/` — 信号与事件
- `shaders/` — 着色器
- `tooling/` — 工具链与 IDE（VS Code 配置、导入文件、编辑器偏好）
- `misc/` — 其他零散知识点

## 笔记索引

- `signals/csharp-signal-basics.md` — **（入门首选）** Godot C# 信号从零到用：为什么需要信号（打死怪→玩家加分，对比直接 `GetNode` 抓玩家的耦合问题）、最小可用示例（声明/发射/订阅/退订）、`[Signal]` 只是编译期标记而 `SignalName` / `MobDied` event / `EmitSignalMobDied` 都是源码生成器产物、名字推导规则（`MobDiedEventHandler` → `MobDied` → `EmitSignalMobDied`）、发射方法是 `protected` 所以必须包一层对外发射口、四条硬性规则（`GD0201/0202/0203/GD0001`）、Autoload 事件总线完整写法、IDE 标红与重命名的坑、动手验证小实验、术语小抄
- `nodes/timer-node-cooldown.md` — Timer 节点做射击冷却：场景里 `wait_time` + `one_shot` 配置、`IsStopped()` 当门锁的写法、`Start/Stop/TimeLeft/Paused` 与 Timeout 信号；对比手动累加 delta 的写法与常见踩坑
- `nodes/3d-local-space-and-parent-rotation.md` — 3D 局部坐标系与父子变换翻转（glb 导入的 180° 补偿旋转导致 Marker3D 的 -Z 朝后、子弹朝向反了；含 .tscn Transform3D 行列读法、Use Local Space 排查、GlobalTransform 赋值要点）
- `nodes/3d-forward-axis-and-basis.md` — Godot 3D 方向约定：为什么"前"是 -Z（右手系 / Camera3D 看 -Z / LookAt 对齐 -Z）；Basis 是旋转+缩放矩阵、轴取列；Basis 与 GlobalBasis、Position 与 GlobalPosition 对照表；`GlobalPosition += -GlobalBasis.Z.Normalized() * speed * delta` 写法与 Normalized 的必要性
- `signals/csharp-event-vs-godot-signal.md` — Godot 4 C# 里该用 `[Signal]` 还是 C# `event`：C# 项目默认用原生 event（性能/类型安全/可重构），只在跨 GDScript、要编辑器连线、订阅内置节点事件时用信号；`BodyEntered +=` 的 C# 事件语法写法与"编辑器连线 + 代码订阅导致回调触发两次"等踩坑
- `nodes/node-path-and-get-node.md` — Godot C# 取节点的全部姿势：`/root/Game/ScoreLabel` 绝对路径的真实树形与 `/root` 是什么、相对路径与 `..`、`%场景唯一名`（只在所属场景内有效、不能跨场景）、`[Export]` 拖引用、`GetParent/GetChild/FindChild/Owner`、`GetTree().Root` 与 `CurrentScene`、分组 `GetFirstNodeInGroup`；`GetNode` 找不到是返回 null（不抛异常）而非报错、`GetNodeOrNull` 的区别；`_Ready` 取兄弟节点的时序坑与选型表
- `tooling/vscode-godot-project-display.md` — VS Code 里让 Godot 项目看着不乱：`.import` / `.uid` 是什么、为什么**不能**进 `.gitignore` 只能靠 `files.exclude` 隐藏；`files.exclude` / `search.exclude` / `files.watcherExclude` / `explorer.excludeGitIgnore` / `.gitignore` 五种机制的分工；`explorer.fileNesting` 折叠替代隐藏的写法；`.vscode` 被忽略时如何共享配置（`.vscode/*` + `!.vscode/settings.json`，父目录整体忽略时 `!` 反选无效）
- `animation/animation-tree-oneshot.md` — AnimationTree 播放一次性动画：BlendTree 里 OneShot 的 0 号口常态 / 1 号口插播、动画名是 `库名/动画名`（custom/aa）、`parameters/OneShot/request` 的 None/Fire/Abort 三种取值、C# `Set()` 写法与子弹命中→Mob→BatModel 的调用链；含"Active 默认关闭导致树不工作"等踩坑

在此添加你的 Godot 笔记与示例。


# 注意

AI含量95%