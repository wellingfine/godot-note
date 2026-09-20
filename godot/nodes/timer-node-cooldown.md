# Timer 节点与冷却（Cooldown）用法

出处：`GdQuestBasic3d/Player.cs` + `player.tscn`（第一人称射击 Demo）

## 场景中的配置

```
[node name="Timer" type="Timer" parent="."]
wait_time = 0.5
one_shot = true
```

- **`Wait Time`**：计时时长（秒）
- **`One Shot`**：勾选后倒计时到 0 就自动**停下**，不会循环
- **`Autostart`**：勾选后进入场景树自动开始，一般不勾
- **`Process Callback`**：`Idle`（默认）/ `Physics`，决定按哪种帧计时

## 核心套路：`IsStopped()` 当"门锁"

```csharp
private Timer _timer;

public override void _Ready() {
  _timer = GetNode<Timer>("Timer");
}

public void Shoot() {
  // 没按住射击键，或者计时器还在跑 -> 直接返回（冷却中）
  if (!(Input.IsActionPressed("shoot") && _timer.IsStopped())) {
    return;
  }

  _timer.Start();   // 重新计时，0.5s 后自动停下，届时 IsStopped() 才变回 true

  Node3D bullet = BulletPrefab.Instantiate<Node3D>();
  BulletSpawnerNode.AddChild(bullet);
  bullet.GlobalTransform = BulletSpawnerNode.GlobalTransform;
}
```

要点：

- `IsStopped()` 返回 `true` 表示**当前没在计时**（初始状态、或 One Shot 已跑完）
- 上面这段逻辑放在 `_PhysicsProcess` 里每帧调用，等于"按住不放时每 0.5 秒发射一发"
- `_timer.Start()` 写在超时后才能再次通过判断，天然形成节奏
- 若 Timer 正在运行，`Start()` 会**重置**剩余时间（也可以设之前手动 `Stop()`）

## 常用 API

| 成员 | 说明 |
| --- | --- |
| `Start(double timeSec = -1)` | 开始（可选临时覆盖 wait_time）；已运行时调用会重置倒计时 |
| `Stop()` | 停止，`IsStopped()` 立刻变 true |
| `IsStopped()` | 是否没在跑。未进树（例如 `_Ready` 之前）时恒为 true |
| `TimeLeft` | 剩余时间，用 Cook 节点显示冷却进度条就靠它 |
| `Paused` / `SetPaused(bool)` | 暂停而不清零（注意 `Node.ProcessMode` 也会受影响） |
| `Timeout` 信号 | 倒计时归零时发出（One Shot 的发出一次，非 One Shot 的周期重复） |

## Timeout 信号用法（C#）

```csharp
_timer.Timeout += OnTimerTimeout;   // 或直接 += () => { ... }

private void OnTimerTimeout() {
  GD.Print("冷却结束，可以再射了");
}
```

Godot 4 也可以在编辑器里把 Timeout 连到脚本方法，信号面板会自动生成 `public void OnTimerTimeout()`。

## 另一种写法：不用 Timer，手动累加 delta

```csharp
private float _shootCooldown;

public override void _PhysicsProcess(double delta) {
  _shootCooldown = Mathf.MoveToward(_shootCooldown, 0f, (float)delta);
  if (Input.IsActionPressed("shoot") && _shootCooldown == 0f) {
    _shootCooldown = 0.5f;
    ShootOnce();
  }
}
```

对比：

- **Timer 节点**：可视化、`TimeLeft` 便于 UI、不用自己管 delta；代价是多一个节点
- **手动累加**：轻量、代码自包含、帧率无关；代价是逻辑写在 Process 里，UI 取进度要自己暴露字段

## 踩坑

- `One Shot` 忘了勾 -> Timer 一直循环，`IsStopped()` 永远 false，射不出第二发？
  实际是每帧都能通过判断，反而变成连发
- `_Ready()` 里没 `GetNode` 就直接用 -> `Timer` 未进树，`IsStopped()` 恒 true，冷却失效
- Timer 是 Node，被 `QueueFree` 或场景重新实例化后要重新取引用
- 想让冷却随暂停停止，注意 `Timer` 的 `ProcessMode`（默认 Inherit）
