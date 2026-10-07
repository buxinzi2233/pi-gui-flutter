---
title: "虚拟分区与项目目录树"
version: "1.0.0"
status: "implemented"
type: "feature-and-architecture"
tags: [sidebar, sections, workspace, rpc, migration]
---

# 用分区归类项目

分区是 GUI 中的分组名称，例如“工作”“个人”，没有磁盘路径。分区下的项目对应真实目录；项目根目录只显示一次，不再嵌套一个同名“主目录”。参考 [VS Code 多根工作区](https://code.visualstudio.com/docs/editing/workspaces/multi-root-workspaces) 的目录浏览方式，并保留 Pi 的目录会话和 Git Worktree 功能。

```text
工作
  frontend                     ← 实际项目根目录
    页面调整                   ← 该目录的会话
    src                        ← 真实子目录
      main.dart                ← 点击打开只读文件
      组件重构                 ← 在 src 中创建的会话
个人
  Media
未分区
  旧项目
```

## 操作

- 项目列表标题右侧的“新建分区”按钮创建分区；分区菜单提供重命名、上移、下移、删除。
- 点击分区收起或展开。自建分区的名称、顺序和折叠状态跨重启保存；“未分区”是固定的默认入口，其折叠仅保留在当前窗口。
- 分区右侧文件夹加号添加项目。项目菜单的“移动到分区”改变归类，不移动文件、不停止会话。
- 删除分区前有确认，包含的项目自动移到“未分区”；项目目录、历史和活动会话保留。
- 项目和子目录右侧 `+` 在该真实路径创建会话。文件点击后复用现有只读文档标签。悬停真实目录行显示完整路径。
- 目录只在展开时读取，复用文件浏览器的分页、错误重试和文件预览。历史与目录读取失败可分别重试；刷新按钮刷新已浏览的目录和已展开目录的历史。
- 项目菜单仍可创建 Git Worktree。项目根目录以外的 Worktree 以独立的 `Worktree · 名称` 条目展示，悬停可查看真实路径；位于项目目录内的 Worktree 按真实层级展示并保留 Worktree 标记和移除菜单。不会把主 checkout 再画一遍。
- 搜索匹配分区、项目、Worktree、已加载的文件名和会话标题。只读取已登记 / 已浏览目录的历史摘要，最多两个并行读取；不扫描未展开目录做全盘搜索。

![深色窄侧栏中的虚拟分区和真实目录树](images/project-sections-dark.png)

## 数据与迁移

数据仍由 Node 管理器串行保存到 GUI 自有 `workspaces.json`，不修改 Pi 私有历史文件。位置和覆盖变量见 [工作区与会话](workspaces_sessions.md)。

```json
{
  "catalog": {
    "version": 2,
    "sections": [
      {"id": "section-work", "name": "工作", "collapsed": false}
    ],
    "projects": [
      {"path": "/home/me/frontend", "name": "frontend", "folders": [], "sectionId": "section-work"}
    ],
    "names": {}
  }
}
```

分区使用独立 ID，不以名称或路径作为身份；重名分区不会合并。`sectionId: null` 表示未分区。版本 1 的项目保留原有路径和 Worktree 信息，迁移到未分区；不存在的分区引用同样归入未分区。空项目列表仍保持为空。旧版 GUI 不识别 catalog 版本 2，回退程序前应备份 GUI 自有 `workspaces.json`，避免旧版重新迁移最近目录丢失分区信息。

管理写入复用 `PiRpcClient` 的未确认写请求屏障；超时不自动重发。创建分区的 ID 由前端一次生成，后端同 ID 创建幂等；分区 / 项目元数据写操作串行处理，保存失败由已有 `persistenceWarning` 提示。

## RPC 与目录归属

全部命令经现有 `control` 通道，使用强类型 `PiCatalogService`：

| 命令 | 参数 | 行为 |
| --- | --- | --- |
| `gui_create_section` | `sectionId`, `name` | 创建分区 |
| `gui_update_section` | `sectionId`, `name?`, `collapsed?`, `direction?` | `direction` 为 `up` / `down` |
| `gui_delete_section` | `sectionId` | 项目移到未分区 |
| `gui_move_project` | `path`, `sectionId` | null 表示未分区 |
| `gui_add_project` | `path`, `sectionId?` | 注册并归类，保持 Git 项目去重 |
| `gui_get_catalog` | 无 | 返回分区和项目快照 |

写后沿用 `gui_catalog_changed` 刷新。文件与会话继续复用 `gui_list_files`、`gui_get_file_preview`、`gui_workspace_history`、`gui_open_channel`，没有增加 Agent 命令。

`knownWorkspace()` 先解析真实路径，再用最近的已登记项目 / Worktree 根目录判断归属；普通子目录可以成为会话 cwd。跳出已登记根目录的符号链接不授予目录访问。删除项目撤销子目录归属；子目录会话同样阻止移除其父 Worktree。历史读取后再次检查目录和仓库锁，避免异步期间项目已被移除仍启动会话。重启时能恢复位于子目录的最近会话。

## 关键路径

- `assets/backend/workspace_manager.mjs`：分区迁移与保存、管理 RPC、目录归属、项目与 Worktree 保护。
- `lib/core/rpc/pi_catalog_types.dart`：分区 / 项目强类型模型和管理 API。
- `lib/core/rpc/pi_rpc_client.dart`：分区写请求的确认屏障。
- `lib/ui/features/home/controllers/workbench_controller.dart`：分区操作状态、目录按需加载、子目录历史保留、目录刷新。
- `lib/ui/features/home/widgets/workbench_sidebar.dart`：复用 `AppTreeTile`、`AppMenuButton`、`AppDialog`、`AppTextField`、`AppActionButton`、`AppIconButton` 拼装侧栏；不直接访问文件系统或拼 RPC JSON。
- `lib/ui/features/home/views/home_view.dart`：带分区身份的目录选择入口。
- `lib/l10n/app_zh.arb`、`app_en.arb`：新增界面文案，颜色 / 排版和动效沿用主题与原子组件。

## 验证

核心回归保留在 `test/project_sections_test.dart`、`test/workspace_manager.test.mjs` 和 `test/workbench_browser_test.dart`，覆盖旧数据迁移、分区删除保留内容、排序折叠保存、子目录与符号链接边界、目录缓存生命周期和无 Agent 启动的项目归类。

```bash
flutter analyze --no-pub
flutter test --no-pub
node --test test/workspace_manager.test.mjs test/workspace_backend.test.mjs test/workspace_browser_backend.test.mjs
flutter build linux --release -t lib/main.dart
```

界面使用临时隔离夹具走查创建、移动、重命名、删除、真实子目录展开、深浅色与 280px 窄侧栏；不将样式断言加入永久测试，不重启承载活动会话的 GUI。

本地验证结果：Flutter 静态分析无问题，171 项 Flutter 测试通过，26 项工作区 Node 测试通过；`tool/check_parallel_rpc.mjs` 覆盖真实 Pi 的分区归类 / 删除及子目录 cwd，`tool/check_workspace_rpc.mjs` 验证原单会话协议，两者均未请求模型。Linux Release 构建与项目自带打包脚本执行成功。Windows 未在本机构建。
