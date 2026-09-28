# VS Code 里让 Godot 项目「看着不乱」

## 1. Godot 到底生成了哪些噪音文件

| 文件 / 目录 | 谁生成的 | 要不要进 git | 日常要不要手改 |
| --- | --- | --- | --- |
| `*.import` | 每个被导入的资源旁（`.png` `.svg` `.glb` `.obj` `.wav` …） | **要** | 不要（在 Godot 的「导入」面板改） |
| `*.uid` | Godot 4.4+ 给脚本/着色器等文本资源生成的 UID 文件（本仓库是 4.8） | **要** | 不要 |
| `.godot/` | 编辑器缓存 + 真正的导入产物（`.godot/imported/`） | 不要（已忽略） | — |
| `.mono/`、`bin/`、`obj/` | C# 构建产物 | 不要（已忽略） | — |

关键区分：

- `.import` 存的是**导入参数**（压缩模式、是否生成碰撞、动画裁剪等），换台机器没有它，Godot 会用默认值重新导入，设置就丢了。
- 真正占地方的导入产物在 `.godot/imported/`，那才是被 `.gitignore` 忽略的部分。
- `.uid` 存的是资源的 UID，`.tscn` / `.tres` / `project.godot` 里的 `uid://bgrsguprdu0fl` 靠它反查成路径。没有它，场景里的 `uid://` 引用会解析失败。

本仓库实测（`git ls-files`）：`icon.svg.import`、`Player.cs.uid`、`mob/bat/bat_model.glb.import` 等全部在版本库里 —— 也就是说**它们不能进 `.gitignore`**，只能在 VS Code 里"眼不见为净"。

## 2. 几种过滤机制别搞混

| 机制 | 作用范围 | 影响 git 吗 |
| --- | --- | --- |
| `.gitignore` | 只管版本控制（git 状态、提交） | 是 |
| `files.exclude` | 只管 VS Code 的**资源管理器**显示 | 否 |
| `search.exclude` | 只管搜索结果（默认继承 `files.exclude`） | 否 |
| `files.watcherExclude` | 只管文件监听，不显示也不隐藏，纯性能 | 否 |
| `explorer.excludeGitIgnore` | 让资源管理器**按 `.gitignore` 隐藏**文件 | 否（但读取 gitignore） |

一句话：`.gitignore` 不会让 VS Code 少显示一个文件，`files.exclude` 也不会让 git 少跟踪一个文件。两者各管一摊。

## 3. 推荐配置（本仓库 `.vscode/settings.json`）

```jsonc
{
  // Godot 给每个资源生成的伴随文件：.import 存导入参数、.uid 存 UID。
  // 它们必须提交进 git，但日常从不用手改 → 在资源管理器里隐藏。
  "files.exclude": {
    "**/*.import": true,
    "**/*.uid": true,
    "**/.godot": true,
    "**/.mono": true,
    "**/bin": true,
    "**/obj": true
  },
  // 纯性能优化：.godot/ 和构建输出变动极频繁，不让 VS Code 监听它们
  "files.watcherExclude": {
    "**/.godot/**": true,
    "**/bin/**": true,
    "**/obj/**": true
  }
}
```

搜索默认跟随 `files.exclude`，一般不用再写 `search.exclude`；万一搜得到，再单独配一份同内容的 `search.exclude`。

## 4. 想"折叠"而不是"消失" → 文件嵌套

`files.exclude` 是彻底隐藏。如果想保留随时点开的能力，改用 `explorer.fileNesting`，把伴随文件折叠到主文件下面：

```jsonc
{
  "explorer.fileNesting.enabled": true,
  "explorer.fileNesting.expand": false,   // 默认折叠
  "explorer.fileNesting.patterns": {
    "*.cs": "$(capture).cs.uid",
    "*.gdshader": "$(capture).gdshader.uid",
    "*.png": "$(capture).png.import",
    "*.svg": "$(capture).svg.import",
    "*.wav": "$(capture).wav.import",
    "*.glb": "$(capture).glb.import",
    "*.obj": "$(capture).obj.import"
  }
}
```

`$(capture)` 就是 `*` 匹配到的主文件名（`Player.cs` → `Player.cs.uid`）。

**两者二选一**：同一批文件既被 `files.exclude` 排除又被 nesting 匹配时，`files.exclude` 赢，嵌套根本不会显示。

## 5. 踩过的坑

1. **排除之后 Ctrl+P 也找不到了**。`files.exclude` 同时作用于快速打开。真要改 `.import` 的内容，别在 VS Code 里翻，去 Godot 编辑器：选中资源 →「导入」面板 → 改参数 → 重新导入，Godot 会自己把 `.import` 写回去。这才是正道。
2. **`explorer.excludeGitIgnore = true` 会把 `.vscode` 自己藏起来**。本仓库 `.gitignore` 忽略了 `.vscode/`，一旦开启这个开关，`.vscode/settings.json` 就从资源管理器消失了，之后想改都点不到。要用的话先确认 `.vscode` 没被忽略，或者接受"靠命令面板/文件菜单打开"。
3. **`files.exclude` 不影响已经打开的标签页**，也不影响 Godot 编辑器自己的 FileSystem 面板 —— 它只是 VS Code 的显示层。

## 6. `.vscode` 不计入 git，配置怎么同步

本仓库 `.gitignore` 里有 `.vscode/`，所以这份配置只在本机生效，换机器/重装就回到原始状态。想让队友也生效，两条路：

**A. 把 `settings.json` 提交进仓库**

```gitignore
.vscode/*
!.vscode/settings.json
```

注意不能写成：

```gitignore
.vscode/
!.vscode/settings.json   # ❌ 无效
```

Git 的规则是「父目录被整体忽略时，不再往下遍历，里面的 `!` 反选不生效」。必须让目录本身不被完全忽略（`.vscode/*` 忽略内容但目录还在），反选才会起作用。

**B. 不动 git，用 VS Code 自己的同步**：Settings Sync（`设置同步` 打开即可，走 GitHub/Microsoft 账号）或 Profiles，把这类编辑器偏好跟着账号走。

## 7. 出处

- 项目：`e:\GodotProject\GdQuestBasic3d\.vscode\settings.json`（原内容只有 `files.exclude` 的 `.import` / `.uid` 两条）
- 项目：`e:\GodotProject\GdQuestBasic3d\.gitignore`（`.godot/`、`.mono/`、`.vscode/` 等）
- 验证命令：`git ls-files | findstr /I ".uid .import"` → 两者均已被跟踪，确认不能进 `.gitignore`
- 引擎版本：`project.godot` 的 `config/features=PackedStringArray("4.8", "C#", "Forward Plus")`
