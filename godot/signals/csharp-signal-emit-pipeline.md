# `[Signal]` 发射一次到底走了多远（源码生成器产物实测 + 性能链路）

出处：Godot 4.8 + C#（`GdQuestBasic3d/GameEvents.cs`），产物由本机 `dotnet build` 实际 dump 出来验证
读者：已经会写 `[Signal]`，想知道「它到底做了什么 / 为什么比 C# event 慢」的人

---

## 0. 一句话结论

`[Signal]` = **C# 多播委托 + 一次进出引擎的往返**。
多出来的代价全在这趟往返上：参数装箱成 `Variant`、跨 C#/C++ 边界、引擎转一圈再拆箱回来。
**如果订阅方全是 C#，这趟往返白跑**——互通能力一点没享受到，只付了开销。

---

## 1. 怎么自己看到生成的代码

源码生成器默认不落盘，加两个 MSBuild 属性就行（不用改 `.csproj`，命令行传参即可）：

```powershell
dotnet build /p:EmitCompilerGeneratedFiles=true /p:CompilerGeneratedFilesOutputPath=obj/_gen
```

产物在 `obj/_gen/Godot.SourceGenerators/Godot.SourceGenerators.ScriptSignalsGenerator/<类名>_ScriptSignals.generated.cs`。
同目录还有 `ScriptProperties`（`[Export]` 的）、`ScriptMethods`、`ScriptSerialization` 等，都是同一套生成器写的。
（看完了记得把 `obj/_gen` 删掉，它只是临时验证用。）

---

## 2. 实测：`[Signal]` 到底生成了什么

手写的只有一行：

```csharp
[Signal] public delegate void PlayerFellEventHandler();
[Signal] public delegate void MobDiedEventHandler(int score);
```

生成器补出来的（`GameEvents_ScriptSignals.generated.cs`，已删注释/整理缩进）：

```csharp
// ① 信号名常量 + 编辑器注册信息
public new class SignalName : global::Godot.Node.SignalName {
    public new static readonly StringName @MobDied = "MobDied";
    public new static readonly StringName @PlayerFell = "PlayerFell";
}
internal new static List<MethodInfo> GetGodotSignalList() {
    // 这两个信号会出现在编辑器信号面板、也能被 GDScript connect
}

// ② C# 事件壳子：+= / -= 只是操作一个普通委托字段
private MobDiedEventHandler backing_MobDied;
public event MobDiedEventHandler @MobDied {
    add => backing_MobDied += value;      // 注意：不是 Connect()
    remove => backing_MobDied -= value;   // 注意：不是 Disconnect()
}

// ③ 发射方法（protected！所以必须包一层 public 对外发射口）
protected void EmitSignalMobDied(int @score) {
    EmitSignal(SignalName.MobDied, [@score]);   // int → Variant 装箱，每次 new 一个数组
}

// ④ 引擎回调 → 回头 Invoke C# 委托
protected override void RaiseGodotClassSignalCallbacks(in godot_string_name signal, NativeVariantPtrArgs args) {
    if (signal == SignalName.@MobDied && args.Count == 1) {
        backing_MobDied?.Invoke(VariantUtils.ConvertTo<int>(args[0]));   // 拆箱
        return;
    }
    base.RaiseGodotClassSignalCallbacks(signal, args);
}
```

### ⚠️ 重要修正

很多资料（包括本目录旧笔记里的说法）写「C# 的 `+=` 底层走 `Connect`/`Disconnect`」——**不对**。
看 ② 就清楚了：C# 侧的 `+=` 只是 `backing_MobDied += value`，**完全不进引擎的连接表**。
后果是：

- 编辑器信号面板**看不到**你在 C# 里写的订阅
- `IsConnected()` 查不到它，`Disconnect()` 也断不掉它
- 断连只能靠自己 `-=`（所以 `_ExitTree` 里退订不是可选项）
- 反过来，GDScript / 编辑器连上的回调**能**正常触发，因为发射（③）走的是引擎

---

## 3. 发射一次的完整链路

以 `Mob` 死亡 → `Player` 加分为例：

```
Mob.TakeDamage()
  → GameEvents.Instance.EmitMobDied(score)      ← 自己包的 public 发射口
      → EmitSignalMobDied(score)                ← 生成的方法，protected
          → int 装箱成 Variant，new Variant[] 分配
          → EmitSignal(SignalName.MobDied, …)   ← 跨 C#/C++ 边界
          → 引擎遍历连接列表、做参数转换
          → 回调 RaiseGodotClassSignalCallbacks ← 又跨回 C#
          → VariantUtils.ConvertTo<int>          ← 拆箱
          → backing_MobDied?.Invoke(score)       ← 普通多播委托
              → Player.OnMobDied(score) → AddScore(score)
```

对比纯 C# event 只有两步：`EmitMobDied()` → `MobDied?.Invoke(score)`。

### 每一步的代价

| 环节 | 开销 | 纯 C# event 有吗 |
| --- | --- | --- |
| `int` → `Variant` 装箱 | 有 | 无 |
| `new Variant[]` 分配 | 有（每次发射一次，进 GC） | 无 |
| C# → C++ 跨边界 | 有（两次：去 + 回） | 无 |
| 引擎遍历连接表 | 有 | 无 |
| `ConvertTo<int>` 拆箱 | 有 | 无 |
| 委托 Invoke | 有 | 有（唯一相同项） |

**量级**：单次比纯委托贵约一个数量级，但绝对时间仍在微秒以下。
判断标准：**一秒几次 → 随便用；一帧几十次以上 → 换纯 C# event 或直接调用。**
信号「性能差」的锅，基本都出在把高频逻辑（每帧移动、每颗子弹）塞进了信号。

---

## 4. 那什么时候值得付这趟往返

只有引擎真的需要知道这件事时：

1. **GDScript 要订阅**（混合语言项目）
2. **要在编辑器信号面板可视化连线**，连线存进 `.tscn`
3. **要被 `AnimationPlayer` 触发**，或需要 `Connect(..., SignalFlags.Deferred)` / `OneShot`
4. **订阅的是引擎内置信号**（`BodyEntered`、`Timeout`、`Pressed`）——这个没得选，不用信号就得 `_PhysicsProcess` 里手动轮询

全 C# 且以上都不沾 → 用 `event Action<...>`，省掉整条链路：

```csharp
public event Action<int> MobDied;
public event Action PlayerFell;

public void EmitMobDied(int score) => MobDied?.Invoke(score);
public void EmitPlayerFell() => PlayerFell?.Invoke();
```

迁移成本极低：订阅语法 `+=` / `-=` 两边完全一样，**订阅方一行都不用改**，只动 `GameEvents` 内部。

---

## 5. 顺带厘清：三种「事件」不是一回事

| 写法 | 谁在实现 | 引擎知道吗 | 典型用途 |
| --- | --- | --- | --- |
| `BodyEntered += H` | 引擎内置信号 | 知道（必须） | 碰撞、Timer、按钮 |
| `[Signal] delegate …EventHandler` | Godot 源码生成器 | 知道 | 跨语言 / 编辑器可见的自定义事件 |
| `event Action<…>` | .NET 原生委托 | 不知道 | 纯 C# 内部解耦，性能最好 |

引擎内置信号的 `+=` 看着和自定义信号一样，其实它走的是引擎那套（Godot 给内置信号生成的访问器是真的 `Connect`）；
只有**你自己声明的** `[Signal]`，C# 侧 `+=` 才是纯委托（见 §2 修正）。

---

## 相关笔记

- `signals/csharp-signal-basics.md` — `[Signal]` 从零到用（声明/发射/订阅/退订、命名推导、Autoload 总线）
- `signals/csharp-event-vs-godot-signal.md` — 该用 `[Signal]` 还是 C# 原生 `event`（选型结论）
