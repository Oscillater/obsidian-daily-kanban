# Obsidian Daily + Kanban

一个轻依赖的 Obsidian 工作流：一个看板、一个每日笔记模板、一组 GitHub 风格的热力图。
全部是纯 Markdown，不依赖任何插件的私有格式，换插件、换机器都不会把数据锁死。

## Features

- **每日笔记由模板生成** —— 固定事务清单会自动复制进当天的笔记，每天一份独立副本，勾选其中一个习惯不会影响别的日子。
- **看板是唯一数据源** —— 所有任务都放在 `KanBan.md` 里，用大标题分组。任务完成时会自动在该行写上 `done:: 日期`，无论你是在当日笔记里勾的，还是直接在看板里勾的。
- **点一下就排进今天** —— “Add to today” 按钮会把任务复制成当日笔记里的一行真实清单，并带上按大标题着色的标签，不需要手填任何日期。
- **热力图** —— `Stats.md` 会为每一个习惯、每一个看板大标题各画一张贡献图。

## Install

1. 克隆本仓库，用 Obsidian 打开这个文件夹（Open folder as vault）。
2. 安装 **Dataview** 社区插件（设置 → 社区插件 → 浏览 → Dataview）。仓库里已经带上了插件设置文件 `.obsidian/plugins/dataview/data.json`，装完 JavaScript 查询就是开启状态，不用再手动点开关。
3. 命令面板 → `Daily notes: Open today's note`，固定事务清单会自动填进去。

只需要这一个插件，用途只有索引和脚本；需要 Obsidian 1.x。

## How it works

| 操作 | 实际写入的内容 |
|---|---|
| 打开新的一天 | `templates/daily.md` 被复制成 `daily/YYYY-MM-DD.md`，其中的固定事务脚本把自己替换成一份普通清单 |
| 点击 *Add to today* | 在当日笔记的 `%%task-begin%%` / `%%task-end%%` 标记之间追加一行 `- [ ] 任务 <span …>标题</span>` |
| 勾选复选框 | 笔记里的脚本察觉到状态**变化**，在看板对应行写上 `[x]` 和 `done:: 今天` |
| 取消勾选 | 把看板那一行的 `done::` 再删掉 |
| 直接在 `KanBan.md` 里勾选 | 看板顶部的同步块做同样的事 |
| 点击 *Remove* | 从当日笔记里删掉那一行 |

热力图的数据来自两处：习惯完成情况读取每一份日记的 `## Routine` 段（按文件缓存，只有改动过的笔记会重读），完成次数读取看板里所有的 `done::`。

## Layout

```
.
├── KanBan.md           看板：大标题 = 类目，done:: = 完成日期
├── Routine.md          固定事务清单（会被复制进每一份日记）
├── daily/              每天一份笔记，由 Daily Notes 插件创建
├── templates/daily.md  模板：固定事务复制器 + 任务控制器
├── Stats.md            热力图
└── .obsidian/          日记、模板与 Dataview 配置
```

## Customise

- **颜色** —— `templates/daily.md` 和 `Stats.md` 顶部的 `PALETTE`（两处必须保持一致）。颜色按索引分配，新增看板类目或新习惯会自动取下一个颜色，超过 8 个开始循环。
- **文件名** —— 脚本里的 `KANBAN`、`ROUTINE` / `SRC` 常量。如果你重命名了 `Routine.md` 或 `KanBan.md`，记得同步改这里。
- **热力图跨度** —— `Stats.md` 里的 `WEEKS`。
- **固定事务** —— 直接编辑 `Routine.md`，第二天新建的笔记就会生效。

## Notes

- 脚本是内嵌在 Markdown 文件里的（DataviewJS 不支持加载外部脚本），所以 `KanBan.md` 顶部才会有那个同步块。
- 不要在不同操作系统之间用不同大小写重命名同名文件：Windows 和 macOS 的文件系统不区分大小写，Git 会把它当成删除加新增，容易丢文件。
