# Godot 学习笔记

本目录用于记录 Godot 引擎的学习内容，建议按主题分子目录，例如：

- `nodes/` — 节点与场景用法（Node2D、Control、CharacterBody2D 等）
- `scripts/` — 示例脚本（GDScript 或 C#）
- `signals/` — 信号与事件
- `shaders/` — 着色器
- `misc/` — 其他零散知识点

## 笔记索引

- `nodes/timer-node-cooldown.md` — Timer 节点做射击冷却：场景里 `wait_time` + `one_shot` 配置、`IsStopped()` 当门锁的写法、`Start/Stop/TimeLeft/Paused` 与 Timeout 信号；对比手动累加 delta 的写法与常见踩坑
- `nodes/3d-local-space-and-parent-rotation.md` — 3D 局部坐标系与父子变换翻转（glb 导入的 180° 补偿旋转导致 Marker3D 的 -Z 朝后、子弹朝向反了；含 .tscn Transform3D 行列读法、Use Local Space 排查、GlobalTransform 赋值要点）
- `nodes/3d-forward-axis-and-basis.md` — Godot 3D 方向约定：为什么"前"是 -Z（右手系 / Camera3D 看 -Z / LookAt 对齐 -Z）；Basis 是旋转+缩放矩阵、轴取列；Basis 与 GlobalBasis、Position 与 GlobalPosition 对照表；`GlobalPosition += -GlobalBasis.Z.Normalized() * speed * delta` 写法与 Normalized 的必要性

在此添加你的 Godot 笔记与示例。
