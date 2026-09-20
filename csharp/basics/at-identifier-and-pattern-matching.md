# `@` 转义标识符与 `is` 类型模式匹配

## 1. `@` 转义标识符

当变量名与 C# 关键字冲突时，在前面加 `@` 转义，告诉编译器「这是标识符，不是关键字」。

```csharp
// event 是 C# 关键字，必须用 @ 转义才能当变量名
public override void _UnhandledInput(InputEvent @event) { }
```

- `@event` 与正常变量名**完全等价**，编译后无任何区别。
- 也可以直接改名避开关键字，可读性更好，连 `@` 都不需要：
  ```csharp
  public override void _UnhandledInput(InputEvent inputEvent) { }  // 推荐
  public override void _UnhandledInput(InputEvent e) { }            // 简写
  ```
- 常见冲突关键字：`event`、`class`、`object`、`string` 等。Godot 生成的默认 C# 脚本常用 `@event`，因为要和行动一致。

## 2. `is` 类型模式匹配（type pattern）

```csharp
if (@event is InputEventMouseMotion motion) {
    // 匹配成功：motion 已是 InputEventMouseMotion 类型，可直接用
    RotateY(-motion.Relative.X * MouseSensitivity);
}
```

一次完成三件事：**判断类型 + 安全转换 + 绑定变量**。

- 等价于旧写法，但更简洁：
  ```csharp
  var motion = @event as InputEventMouseMotion;
  if (motion != null) { /* 用 motion */ }
  ```
- 只有匹配成功才会绑定 `motion`，块外不可用。

## 3. 其它写法

- `switch` 模式匹配（多种输入类型时更清晰）：
  ```csharp
  switch (@event) {
    case InputEventMouseMotion motion:
      // ...
      break;
    case InputEventKey key when key.Pressed:
      // ...
      break;
  }
  ```
- 带守卫：`if (@event is InputEventMouseMotion motion && motion.Relative.X != 0) { }`

## 4. Godot 实战出处

Godot 的 `_UnhandledInput(InputEvent @event)` 把键盘、鼠标、手柄等所有输入都塞进基类 `InputEvent`，运行时具体类型多变。用 `is` 模式匹配挑出 `InputEventMouseMotion` 后，才能访问其特有的 `Relative`（鼠标相对位移）属性。
