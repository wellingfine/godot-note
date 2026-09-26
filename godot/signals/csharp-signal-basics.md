# Godot C# 信号入门：`[Signal]` 从零到用

出处：Godot 4.8 + C# 项目实践（`GdQuestBasic3d/GameEvents.cs` 全局事件总线）
读者：刚接触 Godot 的 C# 初学者

---

## 0. 先说人话：信号就是「广播」

想象一个广播站：

- **发广播的人**（比如怪物死了）只管喊一句「我死了！」，**不关心谁在听**；
- **听广播的人**（比如玩家）自己调到这个频道，听到就加分；
- 两边**互相不认识**，谁改名字、谁消失都不影响对方。

这就是 Godot 的信号（Signal）。C# 里写它要用 `[Signal]` 这个特性。

> 类比 C#：信号长得像 `event`（`+=` 订阅、`-=` 退订），但它**不是 .NET 原生的 event**，
> 底层是 Godot 引擎自己的一套机制（见 §3）。

---

## 1. 为什么需要它：一个真实场景

需求：**打死一只怪，玩家加分，UI 更新。**

最直觉的写法（❌ 不推荐）：

```csharp
// Mob.cs：怪物直接去抓玩家，然后改它的分数
_player = GetNode<Player>("/root/Game/Player");
_player.AddScore(1);
```

问题：

- 怪物**必须知道**玩家挂在 `/root/Game/Player`，节点一改名就崩
- 以后用刷怪器动态生成的怪，每只都要再找一次玩家
- 「分数」的逻辑散在两个类里，改需求要两头翻

用信号的写法（✅）：

```csharp
// Mob.cs：我只广播，谁爱听谁听
GameEvents.Instance.EmitMobDied(ScoreValue);
```

一句话区别：**直接调用 = 我去找你；信号 = 我喊一声，谁关心谁响应。**

---

## 2. 最小可用示例（先照抄跑通）

```csharp
using Godot;

public partial class GameEvents : Node {

  // ① 声明一个信号：委托名必须以 EventHandler 结尾
  [Signal] public delegate void MobDiedEventHandler(int score);

  public static GameEvents Instance { get; private set; }
  public override void _EnterTree() => Instance = this;

  // ② 对外提供的「发射口」（为什么必须包一层见 §4）
  public void EmitMobDied(int score) => EmitSignalMobDied(score);
}
```

```csharp
// ③ 订阅：谁关心就在自己的 _Ready 里 +=
public override void _Ready() {
  GameEvents.Instance.MobDied += OnMobDied;
}

// ④ 退订：节点要消失时必须 -=，否则内存不释放
public override void _ExitTree() {
  GameEvents.Instance.MobDied -= OnMobDied;
}

private void OnMobDied(int score) {
  GD.Print($"加了 {score} 分");
}
```

只有三件事：**声明 → 发射 → 订阅（+退订）**。

---

## 3. 背后发生了什么（想知道再看）

`[Signal]` **在运行时什么都不做**，它只是一个**编译期的标记**。
`dotnet build` 时，Godot 自带的**源码生成器**（`Godot.SourceGenerators`，随 `Godot.NET.Sdk` 装进来）
会扫描这些标记，自动帮你**写出剩下的代码**。

你写的：

```csharp
[Signal] public delegate void MobDiedEventHandler(int score);
```

Godot 帮你生成的（源码里看不到，编译产物里有）：

```csharp
public static partial class SignalName {
  public static readonly StringName MobDied = "MobDied";   // 信号名常量
}
public event MobDiedEventHandler MobDied;        // 让你能写 += / -=（底层是 Connect/Disconnect）
protected void EmitSignalMobDied(int score)     // 强类型发射方法
  => EmitSignal(SignalName.MobDied, score);
```

所以这些东西**不是你写的，也不是 C# 自带的**，全是生成的：

| 写法 | 哪来的 | 干什么 |
| --- | --- | --- |
| `[Signal]` | Godot 提供的特性 | 编译期标记，告诉生成器「这是信号」 |
| `SignalName.MobDied` | 生成 | 信号名常量，避免手写字符串 |
| `MobDied += / -=` | 生成 | 订阅/退订，底层走引擎的 Connect/Disconnect |
| `EmitSignalMobDied(score)` | 生成 | 强类型发射（**protected**，见 §4） |
| `EmitSignal(StringName, params Variant[])` | Godot 引擎 API | 真正把信号发出去的底层方法 |

---

## 4. 名字是怎么定的（推导规则）

名字是**拼出来的，不是随便起的**，只有两步：

```
委托名 MobDiedEventHandler
   ↓ 去掉结尾的 EventHandler
信号名 MobDied
   ↓ 前面加固定前缀 EmitSignal
发射方法 EmitSignalMobDied
```

同一个信号名派生出一整族名字：

| 产物 | 名字 | 规则 |
| --- | --- | --- |
| 信号名 | `MobDied` | 委托名去掉 `EventHandler` |
| 订阅用 event | `MobDied` | 与信号名同名 |
| 名字常量 | `SignalName.MobDied` | 与信号名同名 |
| 发射方法 | `EmitSignalMobDied` | `EmitSignal` + 信号名 |
| 参数 | `(int score)` | 照抄委托参数 |

换一个例子：`[Signal] delegate void HealthChangedEventHandler(int cur, int max)`
→ `EmitSignalHealthChanged(cur, max)`、`SignalName.HealthChanged`。

### 一个重要细节：发射方法是 `protected`

实测（故意在别的类里调用）：

```
error CS0122: 'GameEvents.EmitSignalMobDied(int)' is inaccessible due to its protection level
```

也就是说：**只有信号所在的类（及其子类）能发射它**。
所以 §2 里那个 `EmitMobDied()` 包装不是多余的 —— 它是**唯一的对外发射口**，
广播权归事件总线自己所有，别人只能听不能喊。这其实是好事，能防止乱发信号。

---

## 5. 必须记住的四条规则（违反就编译报错）

Godot 附带了 Roslyn 分析器，写错时 **IDE 直接出波浪线**，不用等运行：

| 诊断码 | 含义 | 正确写法 |
| --- | --- | --- |
| `GD0201` | 委托名必须以 `EventHandler` 结尾 | `MobDiedEventHandler` ✅ / `MobDied` ❌ |
| `GD0202` | 参数类型不受支持 | 用 `int`、`string`、`Vector3` 等引擎能识别的类型 |
| `GD0203` | 必须返回 `void` | `delegate void ...` |
| `GD0001` | 类必须加 `partial` | `public partial class Xxx : Node` |

另外隐式要求：**类必须继承自 `Node`**（信号是节点的机制）。

---

## 6. 与 GDScript 对照

```gdscript
signal mob_died(score)      # [Signal] delegate void MobDiedEventHandler(int score)
mob_died.emit(score)        # EmitSignalMobDied(score)
mob_died.connect(callable)  # MobDied += Handler
```

信号本身始终是引擎那套东西，**GDScript 也能连 C# 的信号**，这是它相对 C# 原生 `event` 的优势。

---

## 7. 进阶：全局事件总线（Autoload）

当一个节点**动态生成**（比如刷怪器刷出来的怪），或者两边是**兄弟节点**时，
把信号放在一个全局单例上最省事。

做法：

1. 写一个继承 `Node` 的 `GameEvents`，里面只放信号（就是 §2 的代码）
2. 在 Godot 编辑器里 `项目设置 → 全局 → 自动加载` 添加 `res://GameEvents.cs`
   （等价于 `project.godot` 里出现：）

```ini
[autoload]
GameEvents="*res://GameEvents.cs"
```

3. 谁都能通过 `GameEvents.Instance` 订阅/发射，不需要任何节点路径

完整链路（本项目实例）：

```csharp
// Mob.cs —— 只广播，不认识 Player / UI
public void takeDamage() {
  if (health <= 0) return;       // 用 <= 0 而不是 == 0，防止同一只怪重复加分
  _batModel?.PlayOneShotAnimation();
  health -= 1;
  if (health == 0) {
    GameEvents.Instance.EmitMobDied(ScoreValue);
    QueueFree();
  }
}
```

```csharp
// Player.cs —— 分数只有自己能改，对外只暴露 AddScore
public override void _Ready() {
  GameEvents.Instance.MobDied += OnMobDied;
}
public override void _ExitTree() {
  GameEvents.Instance.MobDied -= OnMobDied;
}
private void OnMobDied(int score) => AddScore(score);
public void AddScore(int amount) {
  score += amount;
  _scoreLabel.Text = $"Score: {score}";
}
```

好处：

- 换场景、改节点名、Player 暂时不存在都不影响
- 新刷出来的怪自动生效，不用逐个连线
- 分数逻辑只有一处，好改

---

## 8. 踩坑清单

- **忘记 `-=`**：订阅者被总线一直引用，`QueueFree()` 后也不释放 → 内存泄漏
- **重复订阅**：编辑器连一次 + 代码 `+=` 一次 = 回调执行两遍
- **Autoload 没注册**：`Instance` 是 null，`_Ready` 里一访问就空引用
- **改了信号参数**：编辑器里已有的连线不会自动更新，运行时才报不匹配
- **IDE 里标红「找不到 `EmitSignalMobDied`」但 `dotnet build` 成功**：
  因为生成的代码 IDE 还没刷新。先 `dotnet build` 一次，
  VS Code 再执行 `C#: Restart Language Server`（或 `Developer: Reload Window`）
- **重命名**：IDE 改委托名会同步更新生成的三个名字，但**不会**更新裸字符串
  `EmitSignal("MobDied", ...)`、GDScript 的 `connect("mob_died", ...)` 和编辑器连线
  → 所以**永远用 `SignalName.Xxx` / `EmitSignalXxx()`，别写字符串**
- **编辑器面板**：内置节点信号（`BodyEntered`、`Timeout`、`Pressed`）可以在节点面板
  的 Signals 标签可视化连线；**自定义 C# 信号别依赖面板**，用代码 `+=` 最稳
  （4.x 各版本表现不一，有相关 issue 报不显示）

---

## 9. 自己动手验证（强烈建议）

1. 把委托名改成 `MobDied`（去掉 `EventHandler`）→ 看 IDE 是否报 `GD0201`
2. 在别的类里写 `GameEvents.Instance.EmitSignalMobDied(1);` → 看是否报 `CS0122`
3. 想看生成的代码长什么样，在 `.csproj` 里加一行再 build：

```xml
<PropertyGroup>
  <EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
</PropertyGroup>
```

生成的文件会落到 `obj/Debug/net8.0/generated/` 下，打开看看就全明白了。

---

## 10. 术语小抄

| 词 | 意思 |
| --- | --- |
| 信号 Signal | Godot 的「广播」机制，节点之间用它通信 |
| 委托 delegate | C# 里描述「方法签名」的类型，这里用来描述信号长什么样 |
| 特性 Attribute | `[Signal]` 这种方括号标记，给编译器/工具看的 |
| Source Generator 源码生成器 | 编译期间自动帮你写代码的程序 |
| Autoload 自动加载 | 游戏一启动就存在、全局唯一的节点（单例） |
| StringName | Godot 优化过的字符串，用于名字比较，比 `string` 快 |
| Variant | Godot 的「万能类型」，`EmitSignal` 底层靠它传任意参数 |

---

## 相关笔记

- `signals/csharp-event-vs-godot-signal.md` — 该用 `[Signal]` 还是 C# 原生 `event`（选型）
- `nodes/node-path-and-get-node.md` — 为什么 `/root/Game/Player` 这种硬路径不可靠
