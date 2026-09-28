# C# 学习笔记

C# 语言与 .NET 的学习记录，按主题分子目录存放。新增笔记请同步更新本文件的「笔记索引」，以及根目录 `README.md` 的总索引。

## 目录结构（现状）

```
csharp/
├── basics/     语法基础（1 篇）
├── tooling/    工具链与 IDE（1 篇）
└── README.md   本文件
```

预留类目（有内容时再建）：`oop/`（类、继承、接口、多态）、`linq/`、`async/`（async/await、Task）、`patterns/`（设计模式与最佳实践）、`samples/`（可运行示例）。

## 笔记索引

### basics/ — 语法基础

| 笔记 | 内容 |
| --- | --- |
| [at-identifier-and-pattern-matching.md](./basics/at-identifier-and-pattern-matching.md) | `@` 转义标识符（`_UnhandledInput(InputEvent @event)` 里 `event` 是关键字，`@event` 与正常变量名完全等价）与 `is` 类型模式匹配：一次完成「判断类型 + 安全转换 + 绑定变量」，附 `switch` 模式匹配与 `when` 守卫写法；出处是 Godot 的 `_UnhandledInput` |

### tooling/ — 工具链与 IDE

| 笔记 | 内容 |
| --- | --- |
| [editorconfig-completion.md](./tooling/editorconfig-completion.md) | `.editorconfig` 控制 C# 补全生成的代码风格：override 补出 `=> base._Ready();` 的根因是 `csharp_style_expression_bodied_methods = when_on_single_line`、改成 `false` 的写法与其它影响补全的选项、`:silent/suggestion` 严重级别、改完要重启语言服务器；以及「打 `public override void` 途中没提示」的两层原因（VS Code 按当前单词前缀过滤 + Roslyn 的 override 列表由 `override` + 空格触发，只打 `override ` 再敲方法名） |

## 按问题速查

| 我想… | 看哪篇 |
| --- | --- |
| 变量名撞了 C# 关键字怎么办 | basics/at-identifier-and-pattern-matching.md |
| 判断类型并安全转换、绑定变量 | basics/at-identifier-and-pattern-matching.md |
| 补全生成的代码风格不对（块体 / `var` / `this.`） | tooling/editorconfig-completion.md |
| `override` 补全列表弹不出来 | tooling/editorconfig-completion.md |

## 维护约定

- 新笔记放进对应子目录；没有合适类目就新建目录，并在本文件登记。
- 每篇加进「笔记索引」对应表格，用一句话说清「解决什么问题」，关键词（报错码、API 名、配置项）尽量写进去，方便全文搜索。
- 同步更新根目录 `README.md` 的「全部文章目录」。完整约定见 [`AGENT.md`](../AGENT.md)。
