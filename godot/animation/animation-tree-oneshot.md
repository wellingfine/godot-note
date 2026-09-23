# AnimationTree 播放一次性动画（OneShot）

出处：`GdQuestBasic3d/mob/bat/bat_model.tscn` + `mob/bat/BatModel.cs` + `Mob.cs`（蝙蝠受击动画）

## 场景结构

```
bat_model (glb 实例，挂 BatModel.cs)
├─ AnimationPlayer          autoplay = "Idle"，libraries/custom = { "aa": ... }
└─ AnimationTree            tree_root = AnimationNodeBlendTree
                            anim_player = NodePath("../AnimationPlayer")
```

BlendTree 内部连线（对应 .tscn 的 `node_connections`）：

```
output ← OneShot
OneShot 端口 0 ← Animation   （Idle，常态循环）
OneShot 端口 1 ← Animation 2 （custom/aa，一次性受击/扇翅动画）
```

要点：

- **动画名 = 库名/动画名**。动画存在 `custom` 这个 AnimationLibrary 里，所以全名是 `custom/aa`，不是 `aa`
- **OneShot 的 0 号口是常态动画**，1 号口是要插播的那段；播完自动回到 0 号口
- `mix_mode = 1` 表示 Blend（过渡），不是 Add

## C# 触发方式

```csharp
// BatModel.cs
private const string OneShotRequestPath = "parameters/OneShot/request";

private AnimationTree _animationTree;

public override void _Ready() {
  _animationTree = GetNode<AnimationTree>("AnimationTree");
  _animationTree.Active = true;   // 关键：默认关着，不开树不接管播放
}

public void PlayOneShotAnimation() {
  _animationTree.Set(OneShotRequestPath, (int)AnimationNodeOneShot.OneShotRequest.Fire);
}
```

调用链（子弹命中 → 怪物受击 → 播动画）：

```csharp
// Bullet.cs
private void OnBodyEntered(Node3D body) {
  if (body is Mob mob) mob.takeDamage();
  Destroy();
}

// Mob.cs
private BatModel _batModel;
public override void _Ready() => _batModel = GetNode<BatModel>("bat_model");
public void takeDamage() => _batModel?.PlayOneShotAnimation();
```

## request 参数取值

| 值 | 枚举 | 作用 |
| --- | --- | --- |
| 0 | `OneShotRequest.None` | 无操作（引擎播完会自动复位到这里） |
| 1 | `OneShotRequest.Fire` | 触发，播放 1 号口动画一次 |
| 2 | `OneShotRequest.Abort` | 中断，立刻回到 0 号口 |

读当前是否在播：`(bool)_animationTree.Get("parameters/OneShot/active")`。

## 常用参数路径

```csharp
_animationTree.Set("parameters/OneShot/request", 1);   // 触发
_animationTree.Set("parameters/playback", ...);        // StateMachine 用 travel()
_animationTree.Get("parameters/OneShot/active");       // 是否正在播
```

若是 `AnimationNodeStateMachine`（而不是 BlendTree），则用：

```csharp
_animationTree.Set("parameters/playback", playback);   // 或强转后
((AnimationNodeStateMachinePlayback)_animationTree.Get("parameters/playback")).Travel("hit");
```

## 踩坑

- **AnimationTree 的 Active 默认关闭**：不设 `_animationTree.Active = true`（或检查器里勾 Active），树完全不工作，画面只有 AnimationPlayer 的 autoplay
- **不要同时用 AnimationPlayer.Play()**：树激活后会接管播放，手动 Play 会被立刻覆盖/打断
- 参数路径写错（比如拼错节点名）**不会报错，也不会有任何效果**，排查时先打印 `_animationTree.Get("parameters/OneShot/request")`
- `GetNode<AnimationTree>("AnimationTree")` 依赖节点名，改名后编译器查不出来 —— 要么保持名字稳定，要么用 `[Export] private AnimationTree _tree;` 在检查器里拖引用
- BlendTree 里节点名带空格（默认 `Animation 2`）时，参数路径也要带空格，建议重命名为无空格名
- OneShot 播完自动回到常态动画，不需要手动再触发一次；连点会把 request 反复置 1，表现为动画从头重播
