# Godot C# 取节点的几种方式（从 `/root/Game/ScoreLabel` 说起）

## 1. 绝对路径：从 SceneTree 的根开始

```csharp
_scoreLabel = GetNode<Label>("/root/Game/ScoreLabel");
```

路径开头的 `/root` 是**场景树的根（Window）**，不是项目根目录。运行时真实的树长这样：

```
root              ← GetTree().Root，一个 Window
└── Game          ← 主场景的根节点（game.tscn 里 [node name="Game"]）
    ├── Level
    ├── env
    ├── Player
    ├── Mob
    ├── MobSpawner
    └── ScoreLabel
```

所以 `/root/Game/ScoreLabel` 的读法是：先到树根，再进主场景根节点 `Game`，再找它的 `ScoreLabel`。

注意点：

- `Game` 是**场景里根节点的名字**，`game.tscn` 是文件名，两者可以不一样。改名 `.tscn` 文件不会改节点名，反过来也一样。
- autoload 的单例节点同样直接挂在 `/root` 下（如 `/root/GameEvents`）；而本文的 `/root/Game` 是**主场景根节点**。两者都在 `/root` 下，取的时候别把主场景当成 autoload。

## 2. 相对路径：相对当前节点（本项目其余写法）

```csharp
_camera   = GetNode<Camera3D>("Camera3D");   // Player.cs：直接子节点
_timer    = GetNode<Timer>("Timer");         // Player.cs：直接子节点
_batModel = GetNode<BatModel>("BatModel");   // Mob.cs：直接子节点
_animationTree = GetNode<AnimationTree>("AnimationTree");  // BatModel.cs
```

- 相对 `this` 解析，只在当前节点的子树里找（除非用 `..`）。
- `..` 回到父节点：`GetNode<Node>("../Sibling")`。
- 多级：`GetNode<Node>("Level/CSGBox3D")`。
- 以 `/` 开头就变成绝对路径（第 1 节），`../` 一路往上也能当成"半个绝对路径"用。

## 3. 场景唯一名 `%`：同一场景内最省心

编辑器里选中节点 → 右键 → **Access as Unique Name**（节点名旁边出现 `%` 图标，`.tscn` 里写成 `unique_name_in_owner = true`），然后：

```csharp
_scoreLabel = GetNode<Label>("%ScoreLabel");
```

- 只在**包含它的那个场景内部**有效：把场景实例化到别的场景里，外部脚本用 `%` 是拿不到的。**不能跨场景**，这是它和 `/root/...` 的根本区别。
- 之后随便挪位置、改层级都不失效，只有改名或取消唯一名才失效。
- 本项目 `ScoreLabel` 在 `game.tscn` 里，而取它的 `Player` 是 `player.tscn` 实例化的 —— 跨场景了，所以这里只能用绝对路径 / 导出引用 / 组。

## 4. `[Export]` 在检查器里拖引用（推荐跨场景用）

```csharp
[Export] public PackedScene BulletPrefab;
[Export] public Node3D BulletSpawnerNode;
```

`Player.cs` 里的现成例子。适合：跨场景引用、要引用预制体（`PackedScene`）、想要"改场景结构时编辑器报错提示"。

代价：字段名改了之后场景里存的旧值会失效（本项目 `Player.cs` 注释里就记了这个坑），所以顺手加空判断。

## 5. 在树里"找"：父子、递归、组

```csharp
GetParent<Game>();                       // 直接父节点（C# 泛型版，类型不对返回 null）
GetChild<Node3D>(0);                     // 第 i 个子节点
FindChild("ScoreLabel", recursive: true); // 按名字递归搜（深度优先，只找第一个）
Owner                                     // 场景所有者（通常是当前场景的根节点）

GetTree().Root                            // == "/root"，返回 Window
GetTree().CurrentScene                     // 当前主场景根节点，== /root/Game
GetTree().CurrentScene.GetNode<Label>("ScoreLabel");  // 不用硬编码场景名

// 分组：节点 AddToGroup("player") 后
GetTree().GetFirstNodeInGroup("player");   // Godot 4.1+
GetTree().GetNodesInGroup("player");       // 返回 Godot.Collections.Array<Node>
```

用 `GetTree().CurrentScene` 代替 `/root/Game` 的好处：主场景换名字、换成另一个场景时代码不用改。

## 6. 找不到时到底会怎样

- `GetNode<T>(path)`：引擎打印 `Node not found: ...` 错误，**返回 `null`**（C# 侧不会当场抛异常），空引用会推迟到真正用它的地方才炸。
- `GetNodeOrNull<T>(path)`：安静地返回 `null`，不刷错误日志，适合"有就配置、没有就跳过"的可选引用。
- 想主动断言：`GD.PushError(...)` 或在 `_Ready` 里判空后 `return`。

## 7. 时序：`_Ready` 里取别的节点安全吗

- 父节点的 `_Ready` **早于**子节点 → 父节点里取自己的子节点没问题。
- 兄弟节点之间：`_Ready` 的调用顺序按树顺序（先上的先跑），**节点本身已经进树了所以引用拿得到**，但对方的 `_Ready` 可能还没跑完，读它初始化的字段会拿到默认值。
- 解决办法：`await ToSignal(node, Node.SignalName.Ready)`，或把初始化改成属性懒加载。

## 8. 什么时候用哪种

| 场景 | 推荐写法 |
| --- | --- |
| 自己的直接子节点 | 相对路径 `GetNode<T>("Name")` |
| 同一场景内、层级可能变 | `%唯一名` |
| 跨场景（如 Player 找 UI 的 Label） | `[Export]` 引用 ＞ 组 ＞ `GetTree().CurrentScene` ＞ `/root/...` |
| 全局单例 / 管理器 | autoload + `/root/名字` |
| 临时调试、一次性脚本 | `/root/...` 绝对路径，后面记得重构掉 |

绝对路径的问题就是硬编码：场景改名字、节点挪位置，编译照样过，只有运行时才炸。

## 9. 出处

- `Player.cs:29` — `GetNode<Label>("/root/Game/ScoreLabel")`（本文起因）
- `Mob.cs:16` — `GetNode<Player>("/root/Game/Player")`（Mob 跨分支找 Player）
- `Player.cs:27-28`、`Mob.cs:15`、`mob/bat/BatModel.cs:11` — 相对路径取子节点
- `Player.cs:15-16` — `[Export]` 引用 + 「改名后场景里旧值失效」的注释
- `game.tscn` — `[node name="Game"]` / `[node name="ScoreLabel" parent="."]` 的树结构
