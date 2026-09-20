# `.editorconfig` 控制 C# 补全生成的代码风格

## 1. 问题：override 补全补出了没用的单行方法

在 Godot 脚本里敲 `override` 选 `_Ready`，结果补出来：

```csharp
public override void _Ready() => base._Ready();   // 单行表达式体，还带一句没用的 base 调用
```

想要的是块体：

```csharp
public override void _Ready() {
    base._Ready();
}
```

**根因**：Roslyn 的 override 补全不是固定模板，它会读 `.editorconfig` 里的代码风格选项决定生成什么形态。关键是这一项：

```editorconfig
csharp_style_expression_bodied_methods = when_on_single_line:silent
```

值可以是 `true` / `false` / `when_on_single_line`。设成 `when_on_single_line` 时，因为 `base._Ready();` 能写成一行，就被折叠成 `=>` 单行体。改成 `false` 就固定生成块体。

## 2. 修复

```editorconfig
[*.cs]
# 方法一律块体（影响 override 补全的生成结果）
csharp_style_expression_bodied_methods = false:suggestion
csharp_style_expression_bodied_constructors = false:silent
csharp_style_expression_bodied_operators = false:silent
csharp_style_expression_bodied_local_functions = false:silent

# 属性/访问器仍保留表达式体，这两项不影响补全，看个人喜好
csharp_style_expression_bodied_properties = true:silent
csharp_style_expression_bodied_accessors = true:silent
```

## 3. 其它会影响「补全/代码生成」的选项

| 选项 | 影响 |
| --- | --- |
| `csharp_style_expression_bodied_methods` | override 补全生成块体还是 `=>` |
| `csharp_style_var_for_built_in_types` / `_when_type_is_apparent` / `_elsewhere` | 生成局部变量时用 `var` 还是显式类型 |
| `dotnet_style_qualification_for_field` / `_for_property` / `_for_method` | 生成代码是否加 `this.` 前缀 |
| `dotnet_style_predefined_type_for_locals_parameters_members` | 用 `int` 还是 `Int32` |
| `csharp_style_namespace_declarations` | 新建文件时用文件作用域命名空间还是块级 |
| `csharp_prefer_braces` | 「添加大括号」重构的默认判断 |
| `dotnet_sort_system_directives_first` | 整理 using 时的排序 |

## 4. 严重级别（冒号后面那一段）

```
选项名 = 值:严重级别
```

- `silent`：只影响 IDE 行为（格式化、代码生成），不给任何提示
- `suggestion`：给小灯泡 / 三个点的弱化提示
- `warning` / `error`：正常诊断级别，会出现在错误列表

**注意**：`suggestion` 会在已有代码上冒出提示，嫌烦改成 `silent`，补全行为完全一样。
严重级别默认**不会**影响 `dotnet build`，只有显式开启 `<EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>` 才会让 IDE 规则参与构建。

## 5. 改完不生效？

`.editorconfig` 在**语言服务器启动时**读取，改完要重启：

- VS Code 命令面板 → `OmniSharp: Restart OmniSharp`（老版 C# 扩展）
- 或 `Developer: Reload Window`（C# Dev Kit / Roslyn LSP）

老版 OmniSharp 还要确认设置里 `omnisharp.enableEditorConfigSupport = true`；C# Dev Kit 默认读取 `.editorconfig`，无需额外配置。

## 6. Godot 相关的补充

- `base._Ready()` 在 Godot 4 里是**空实现**，调用它没有副作用，补全出来后直接删掉那行即可。
- Roslyn 没有「不生成 base 调用」的选项（Rider 有这个设置，VS Code 没有），只能手动删。
- 文件是隐藏属性，写入被拒（EPERM）时先 `attrib -H 文件名`，改完再 `attrib +H` 恢复。

## 7. 为什么打 `public override void` 的途中没有提示？

两层原因叠加：

1. **VS Code 的补全列表按「光标所在单词」做前缀过滤**。
   打 `public` 时当前单词是 `public`，候选里没有以 `public` 开头的方法，列表就是空的；
   打到 `_Ready` 时当前单词变成 `_Ready`，才匹配上。
   这是 VS Code 的通用行为，不是 C# 插件的问题（Rider 是「关键字触发式」，行为不一样）。

2. **Roslyn 的 override 成员列表由 `override` 关键字 + 空格触发**。
   只有光标位于 `override ` 之后，语言服务器才会返回「可重写成员」这一组候选。

**正确用法**：只打 `override` + **一个空格**，成员列表立刻弹出，再敲方法名前缀过滤即可。
不用先写 `public override void`。列表没弹出来时按 `Ctrl+Space` 手动触发。

相关设置（默认通常是开的，被改过就会失灵）：

```jsonc
"editor.quickSuggestions": { "other": true },   // 边打字边弹
"editor.suggestOnTriggerCharacters": true,      // 空格、. 等触发字符能唤醒列表
```

如果列表被 AI 的行内补全挡住，可调 `"editor.inlineSuggest.enabled"` 或把行内补全的显示顺序调低。

## 8. 出处

- 项目：`e:\GodotProject\GdQuestBasic3d\.editorconfig`（第 87 行附近）
- 触发代码：`Player.cs` 的 `public override void _Ready()`
- 官方文档：`dotnet-format` / `EditorConfig` 参考 `https://learn.microsoft.com/dotnet/fundamentals/code-analysis/code-style-rule-options`
