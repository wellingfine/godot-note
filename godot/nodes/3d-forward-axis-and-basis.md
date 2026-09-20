# Godot 3D 的方向约定：-Z 是前，以及 Basis / GlobalBasis 的区别

> 出处：`GdQuestBasic3d/Bullet.cs`（子弹沿自身前方飞行）
> ```csharp
> // Godot 3D 里 -Z 才是正前方，Basis.Z 指向背后，所以要取负号
> // 用 GlobalBasis（世界旋转）配合 GlobalPosition（世界坐标），避免父节点旋转叠加影响
> var dir = -GlobalBasis.Z.Normalized() * 2 * (float)delta;
> GlobalPosition += dir;
> ```

## 1. 为什么是 -Z

Godot 用**右手坐标系**，`Camera3D` 默认看向自己的 **-Z**。引擎沿用了 OpenGL 的约定：摄像机朝 `-Z` 看，`+Y` 是上，`+X` 是右。

于是整个引擎的"前"都统一成 `-Z`：

- 建模/导出时模型的正面朝 `-Z`；
- `LookAt()` 让物体的 `-Z` 指向目标；
- GDScript 常量 `Vector3.FORWARD == (0, 0, -1)`（C# 里是 `Vector3.Forward`）。

所以取"自身前方"永远是 **`-Basis.Z`**，不是 `+Basis.Z`。凭直觉写 `+Z` 是新手最常见的方向 bug。

## 2. Basis 是什么

`Basis` 是一个 3×3 矩阵，**只含旋转和缩放，不含位置**。三根轴就是它的三个列向量：

```csharp
Basis.X   // 局部 +X 轴在父空间中的指向
Basis.Y   // 局部 +Y 轴
Basis.Z   // 局部 +Z 轴（注意：指向"后"，不是前）
```

完整变换由 `Transform3D` 承载：`Transform3D = Basis（旋转+缩放） + Origin（位移）`。

读 `.tscn` 里的 `transform = Transform3D(9 个数, x, y, z)`：前 9 个是 basis 的**行主序**，**取列才是轴向量**。

## 3. Basis vs GlobalBasis

| 局部 | 世界 | 含义 |
| --- | --- | --- |
| `Basis` | `GlobalBasis` | 旋转+缩放 |
| `Position` | `GlobalPosition` | 位移 |
| `Transform` | `GlobalTransform` | 上面两者合起来 |

- `Basis` 是**相对父节点**的，父节点一转它就跟着变含义；
- `GlobalBasis` 一路把所有祖先的旋转都乘上去了，是**世界空间**下的真实朝向。

**规则：跨父子读写一律用 `Global*`；只有"相对父节点"的运动才用局部属性。**

典型场景就是子弹：它在场景根下，但出生点由枪口的 `Marker3D` 决定，中间隔着玩家、相机、glb 好几层旋转，任何一层翻转都会让方向错。用 `GlobalBasis` + `GlobalPosition` 可以完全绕开这些层级。

## 4. 常用写法

```csharp
// 朝自己的正前方移动（世界空间，最稳）
GlobalPosition += -GlobalBasis.Z.Normalized() * Speed * (float)delta;

// 取世界前方向量（比如做射线检测）
Vector3 forward = -GlobalTransform.Basis.Z;

// 让节点面向目标：之后 -Basis.Z 就指向 target
LookAt(target.GlobalPosition, Vector3.Up);
```

要点：

- **`Normalized()` 不能省。** 父链上只要有缩放，`GlobalBasis.Z` 的长度就不是 1，直接乘会把速度也缩放掉。
- **赋值要配对。** 用 `GlobalPosition` 加的位移必须是世界向量；用 `Position` 就得是父空间向量。混用会静默出错。
- `LookAt()` 的第二个参数是 up 向量，默认 `Vector3.Up`；目标与自身位置重合时会退化，别传同一个点。

## 5. 排查清单

1. 方向反了 → 先看是不是忘了负号（`-Basis.Z` 才是前）。
2. 方向"歪"或跟着某个父节点变 → 换 `GlobalBasis`。
3. 速度忽快忽慢 → 父链有缩放，加 `Normalized()`。
4. 位置对但朝向错 → 多半是 glb/glTF 导入自带的补偿旋转，见 `3d-local-space-and-parent-rotation.md`。
5. 编辑器里按 **T** 开 Use Local Space，直接看每根轴指向哪。
