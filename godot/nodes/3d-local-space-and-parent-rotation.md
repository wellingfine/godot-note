# 3D 局部坐标系与父子翻转：子弹朝向反了

> 出处：`GdQuestBasic3d`（`player.tscn` / `bullet.tscn` / `Player.cs`）
> 现象：bullet 头朝 `-Z`，`Marker3D` 看着也朝 `-Z`，挂上去后子弹却朝后，必须把 bullet 改成朝 `+Z` 才正常。

## 结论

不是 bullet 错了，是 **Marker3D 的 `-Z` 并不指向枪口前方**。它的父级 `gun_model`（glb 实例）被旋转了约 180°，子节点继承这个翻转坐标系，于是 `Marker3D.-Z` 实际指向相机 `+Z`（玩家身后）。

前提：Godot 约定 **`-Z` 是前**（`Camera3D` 看向自己 `-Z`）；节点的 `transform` 是相对父级的，判断某轴指向哪必须一路乘到根。

## 读 .tscn 的 Transform3D

前 9 个数是 basis 的**行主序**（row0/1/2），**取列才是轴向量**（局部 X/Y/Z 在父级空间中指向哪）。

```tscn
transform = Transform3D(-0.3044, 0, -0.01864, 0, 0.305, 0, 0.01864, 0, -0.3044, 0.484, -0.289, -0.645)
```

```
Z 轴（取第 3 列）= (-0.01864, 0, -0.3044) → 归一化 ≈ (-0.061, 0, -0.998) ≈ 相机 -Z
```

绕 Y 旋转 θ 的矩阵为 `[[cosθ,0,sinθ],[0,1,0],[-sinθ,0,cosθ]]`，代入得 θ ≈ -176.5°（另含 0.305 缩放）。
即 **`gun_model` 的 +Z → 相机 -Z（前），故 -Z → 相机 +Z（后）**。glTF/Blender 导出常带这种补偿旋转：肉眼看"枪是正的"，坐标系已翻。

`Marker3D` 自身旋转是单位矩阵、只有位移 `(…, …, 0.667)`：

- 位置对：沿 `gun_model` 的 +Z 前移 → 相机前方偏下，正是枪口。
- 朝向错：继承父级翻转后 `-Z` 朝后。bullet 头朝 `-Z`，于是朝后。

## 修复：改 Marker3D，别改 bullet

1. 选中 `Camera3D/gun_model/Marker3D` → **Rotation Degrees Y = 180**。绕自身原点转，位置不变，`-Z` 变成真前方。bullet 保持头朝 `-Z`。
2. 更彻底：把 `Marker3D` 拖出 `gun_model`，直接挂到 `Camera3D` 下，摆脱 glb 翻转，重新摆一次位置即可。

## Use Local Space（快捷键 T）

决定 gizmo 画全局轴还是节点自身轴（含父级旋转），**只影响编辑器显示/操作，不改变实际变换**。
排查用法：选中 `Marker3D` 按 `T`，若蓝色 `-Z` 箭头指向玩家，说明这个挂点的"前"确实朝后。

## 射击写法

子弹加到**场景根**，不要挂在枪下，否则跟着玩家走。

```csharp
var bullet = BulletScene.Instantiate<Area3D>();
GetTree().CurrentScene.AddChild(bullet);
bullet.GlobalTransform = _muzzle.GlobalTransform;   // 跨父子一律用 Global*
```

```csharp
// 子弹自身：Godot 的"前"是 -Z
GlobalPosition += -GlobalBasis.Z * Speed * (float)delta;
```

要点：跨父子赋值用 `GlobalTransform/GlobalPosition/GlobalBasis`，用 `Transform/Position` 写的是父级局部坐标；方向统一 `-Basis.Z`，别凭感觉写 `+Z`。

## 排查清单

1. 开 **Use Local Space (T)**，看链上每个节点的局部三轴指向。
2. 逐层读 `transform`：取列当轴向量，一路乘到根。
3. 重点怀疑 glb/glTF 导入节点自带的补偿旋转（常见 180° 绕 Y、-90° 绕 X）。
4. 修"挂点"（Marker3D），别改被挂载的资源（bullet）。
