# Godot 学习笔记

本目录用于记录 Godot 引擎的学习内容，建议按主题分子目录，例如：

- `nodes/` — 节点与场景用法（Node2D、Control、CharacterBody2D 等）
- `animation/` — 动画（AnimationPlayer / AnimationTree、OneShot、StateMachine）
- `scripts/` — 示例脚本（GDScript 或 C#）
- `signals/` — 信号与事件
- `shaders/` — 着色器
- `misc/` — 其他零散知识点

## 笔记索引

- `nodes/timer-node-cooldown.md` — Timer 节点做射击冷却：场景里 `wait_time` + `one_shot` 配置、`IsStopped()` 当门锁的写法、`Start/Stop/TimeLeft/Paused` 与 Timeout 信号；对比手动累加 delta 的写法与常见踩坑
- `nodes/3d-local-space-and-parent-rotation.md` — 3D 局部坐标系与父子变换翻转（glb 导入的 180° 补偿旋转导致 Marker3D 的 -Z 朝后、子弹朝向反了；含 .tscn Transform3D 行列读法、Use Local Space 排查、GlobalTransform 赋值要点）
- `nodes/3d-forward-axis-and-basis.md` — Godot 3D 方向约定：为什么"前"是 -Z（右手系 / Camera3D 看 -Z / LookAt 对齐 -Z）；Basis 是旋转+缩放矩阵、轴取列；Basis 与 GlobalBasis、Position 与 GlobalPosition 对照表；`GlobalPosition += -GlobalBasis.Z.Normalized() * speed * delta` 写法与 Normalized 的必要性
- `signals/csharp-event-vs-godot-signal.md` — Godot 4 C# 里该用 `[Signal]` 还是 C# `event`：C# 项目默认用原生 event（性能/类型安全/可重构），只在跨 GDScript、要编辑器连线、订阅内置节点事件时用信号；`BodyEntered +=` 的 C# 事件语法写法与"编辑器连线 + 代码订阅导致回调触发两次"等踩坑
- `animation/animation-tree-oneshot.md` — AnimationTree 播放一次性动画：BlendTree 里 OneShot 的 0 号口常态 / 1 号口插播、动画名是 `库名/动画名`（custom/aa）、`parameters/OneShot/request` 的 None/Fire/Abort 三种取值、C# `Set()` 写法与子弹命中→Mob→BatModel 的调用链；含"Active 默认关闭导致树不工作"等踩坑

在此添加你的 Godot 笔记与示例。
