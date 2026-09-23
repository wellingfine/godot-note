# Godot [Signal] vs C# 原生 event

出处：Godot 4 + C# 项目实践（如 `GdQuestBasic3d/Bullet.cs` 的子弹碰撞处理）

## 结论（C# 项目的默认选择）

在 Godot 4 的 C# 开发中，**优先用 C# 原生 `event`（`Action` / `EventHandler`）**，
只有在下列三种情况才用 Godot 的 `[Signal]`：

1. 需要与 **GDScript** 交互（GDScript 只能连 Godot 信号，听不到 C# event）
2. 需要在**编辑器节点面板可视化连线**、做节点树解耦
3. 订阅的是 **Godot 内置节点事件**（`BodyEntered`、`Timeout`、`Pressed` 等），
   此时用 C# 的事件语法 `node.BodyEntered += Handler;` 订阅即可，
   不需要在编辑器连线，也不需要写字符串方法名

## 对比

| 维度 | Godot `[Signal]` | C# `event` |
| --- | --- | --- |
| 性能 | 较慢（C# ↔ C++ 引擎层跨语言绑定 + 反射） | 极快（纯 C# 委托，无跨语言开销） |
| 类型安全 | 较弱（参数变动/强转易运行时异常） | 极强（编译期检查） |
| 重构友好度 | 改名要手动改字符串或 `SignalName` | IDE 一键重命名，全项目同步 |
| 跨语言兼容 | 完全兼容（GDScript 可连可触发） | 不兼容（GDScript 无法监听） |
| 编辑器集成 | 支持（节点面板连线） | 不支持（无法可视化连线） |

## 写法对照

### Godot 信号（自己声明、编辑器连线）

```csharp
[Signal]
public delegate void HealthChangedEventHandler(int current, int max);

// 触发：字符串名，改名时编译器不会提醒
EmitSignal(SignalName.HealthChanged, hp, maxHp);
```

编辑器里连到脚本方法后，方法名是字符串匹配：

```csharp
private void _on_health_changed(int current, int max) { }  // 改名/改签名 = 运行时才炸
```

### C# 原生 event（推荐）

```csharp
public event Action<int, int> HealthChanged;

// 触发：编译期检查
HealthChanged?.Invoke(hp, maxHp);

// 订阅
mob.HealthChanged += (cur, max) => GD.Print($"{cur}/{max}");
```

要点：

- 触发前用 `?.Invoke()`，没人订阅时不会抛异常
- 订阅方销毁前记得 `-=` 取消订阅，否则被持有者拖住不释放
- 需要传事件对象时用 `EventHandler<T>`（T 继承 `EventArgs`），简单场景用 `Action`

### 内置节点事件：用 C# 事件语法订阅（不开编辑器连线）

```csharp
// Bullet.cs
public override void _Ready() {
  BodyEntered += OnBodyEntered;      // Area3D 内置信号，C# 侧就是个 event
}

private void OnBodyEntered(Node3D body) {
  if (body is Mob mob) {
    mob.takeDamage();
  }
  QueueFree();
}
```

这样既拿到内置信号的精确检测（比 `GetOverlappingBodies()` 轮询更不容易穿透），
又保留了 C# 的类型检查和重命名能力。若同时还在编辑器连了一次，
回调会被触发**两次**，注意别重复连线。

## 踩坑

- 编辑器连线 + 代码里 `+=` 订阅同一个信号 = 回调执行两次
- 删掉脚本方法后，编辑器里那条连线不会自动消失，实例化时会报找不到方法
- `[Signal]` 委托名必须以 `EventHandler` 结尾，否则 Godot 识别不了
- 信号参数个数/类型改了，编辑器连线不会跟着改，运行时才报参数不匹配
- C# event 对 GDScript 完全不可见，混合语言项目里对外接口仍要用 `[Signal]`
