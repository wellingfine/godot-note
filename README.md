# godot-note

个人学习笔记（AI含量80%），分为两个相互独立的部分：

| 部分 | 目录 | 内容 |
| --- | --- | --- |
| Godot 学习笔记 | [`godot/`](./godot) | Godot 引擎相关的笔记、示例脚本（GDScript / C#）、场景与节点用法 |
| C# 学习笔记 | [`csharp/`](./csharp) | C# 语言与 .NET 相关的笔记、示例代码、最佳实践 |

写入笔记的同时，如果有新增类目，需要更新目录

## 目录结构

```
godot-note/
├── godot/              # Godot 学习笔记（9 篇）
│   ├── signals/        信号与事件（3 篇 + 1 已合并）
│   ├── nodes/          节点与场景用法（4 篇）
│   ├── animation/      动画（1 篇）
│   ├── tooling/        工具链与 IDE（1 篇）
│   └── README.md       分部索引
├── csharp/             # C# 学习笔记（2 篇）
│   ├── basics/         语法基础（1 篇）
│   ├── tooling/        工具链与 IDE（1 篇）
│   └── README.md       分部索引
├── AGENT.md            # 维护约定（AI 写笔记时必须遵守）
└── README.md           # 本文件（全站总索引）
```

## 全部文章目录

### Godot（[分部索引 »](./godot)）

| 分类 | 笔记 | 内容 |
| --- | --- | --- |
| signals/ | [csharp-signal-basics.md](./godot/signals/csharp-signal-basics.md) ⭐ | `[Signal]` 从零到用：为什么需要信号、最小可用示例（声明/发射/订阅/退订）、`SignalName` 与 `EmitSignalXxx` 都是源码生成器产物、名字推导规则、发射方法是 `protected` 要包一层对外发射口、四条硬性规则（`GD0201/0202/0203/GD0001`）、Autoload 事件总线完整写法、IDE 标红与重命名的坑 |
| signals/ | [csharp-event-vs-godot-signal.md](./godot/signals/csharp-event-vs-godot-signal.md) | 用 `[Signal]` 还是原生 `event`：默认用 event，只在跨 GDScript、要编辑器连线、订阅内置节点事件时用信号；`BodyEntered +=` 写法；编辑器连线 + 代码订阅导致回调触发两次的坑 |
| signals/ | [csharp-signal-emit-pipeline.md](./godot/signals/csharp-signal-emit-pipeline.md) | 发射一次信号走了多远：dump 源码生成器真实产物、`int→Variant` 装箱 → 跨 C#/C++ → 引擎 → 拆箱的七步链路与开销、性能量级判断；C# 侧 `+=` 只是委托 `+=`，不走引擎 Connect，编辑器 / `IsConnected()` 看不到 |
| signals/ | ~~csharp-signal-source-generator.md~~ | 已合并进 `csharp-signal-basics.md`，仅留跳转说明 |
| nodes/ | [node-path-and-get-node.md](./godot/nodes/node-path-and-get-node.md) | C# 取节点的全部姿势：绝对路径与 `/root`、相对路径与 `..`、场景唯一名 `%`、`[Export]` 拖引用、`GetParent/GetChild/FindChild/Owner`、`GetTree().Root` 与 `CurrentScene`、分组；`GetNode` 返回 null 而非抛异常、`_Ready` 取兄弟节点的时序坑 |
| nodes/ | [timer-node-cooldown.md](./godot/nodes/timer-node-cooldown.md) | Timer 节点做射击冷却：`wait_time` + `one_shot` 配置、`IsStopped()` 当门锁、`Start/Stop/TimeLeft/Paused` 与 Timeout 信号；对比手动累加 delta 与常见踩坑 |
| nodes/ | [3d-forward-axis-and-basis.md](./godot/nodes/3d-forward-axis-and-basis.md) | 3D 方向约定：为什么「前」是 -Z；Basis 是旋转+缩放矩阵、轴取列；Basis/GlobalBasis、Position/GlobalPosition 对照表；`GlobalPosition += -GlobalBasis.Z.Normalized() * speed * delta` |
| nodes/ | [3d-local-space-and-parent-rotation.md](./godot/nodes/3d-local-space-and-parent-rotation.md) | 3D 局部坐标系与父子变换翻转：glb 导入的 180° 补偿旋转导致 Marker3D 的 -Z 朝后、子弹朝向反了；.tscn 的 Transform3D 行列读法、Use Local Space 排查、GlobalTransform 赋值 |
| animation/ | [animation-tree-oneshot.md](./godot/animation/animation-tree-oneshot.md) | AnimationTree 播放一次性动画：OneShot 的 0 号口常态 / 1 号口插播、动画名是 `库名/动画名`、`parameters/OneShot/request` 的 None/Fire/Abort、C# `Set()` 写法与命中→Mob→BatModel 调用链；Active 默认关闭的坑 |
| tooling/ | [vscode-godot-project-display.md](./godot/tooling/vscode-godot-project-display.md) | VS Code 里让 Godot 项目看着不乱：`.import` / `.uid` 为什么不能进 `.gitignore` 只能靠 `files.exclude` 隐藏；五种排除机制的分工；`explorer.fileNesting` 折叠；`.vscode` 被忽略时如何共享配置 |

### C#（[分部索引 »](./csharp)）

| 分类 | 笔记 | 内容 |
| --- | --- | --- |
| basics/ | [at-identifier-and-pattern-matching.md](./csharp/basics/at-identifier-and-pattern-matching.md) | `@` 转义标识符（`_UnhandledInput(InputEvent @event)` 里 `event` 是关键字）与 `is` 类型模式匹配：一次完成「判断类型 + 安全转换 + 绑定变量」，附 `switch` 模式匹配与 `when` 守卫写法 |
| tooling/ | [editorconfig-completion.md](./csharp/tooling/editorconfig-completion.md) | `.editorconfig` 控制 C# 补全生成的代码风格：override 补出 `=> base._Ready();` 的根因是 `csharp_style_expression_bodied_methods`、改法与其它影响补全的选项、`:silent/suggestion` 严重级别、改完要重启语言服务器；以及「打 `public override void` 途中没提示」的两层原因（只打 `override` + 空格才会触发成员列表） |

## 说明

- 两部分内容互不依赖，可按主题自由添加文件。
- 涉及 Godot 引擎或 C# 的代码片段、知识点，统一归入对应目录。
- 新增 / 修改 / 删除笔记后，**必须同步更新本文件的「全部文章目录」和对应分部 `README.md` 的索引**，详见 [`AGENT.md`](./AGENT.md)。
