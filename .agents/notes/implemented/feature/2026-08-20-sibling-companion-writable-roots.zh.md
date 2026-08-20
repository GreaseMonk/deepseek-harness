# Agent Note：伴随可写根目录——在 `workspace-write` 下授予工作区的同级 worktree 树

Status: implemented

[English](2026-08-20-sibling-companion-writable-roots.md) | 中文

## Problem

`workspace-write` 只授予一个根目录：会话不可变的 cwd。若某仓库的 worktree 位于同级目录（工作树 `/repos/app`，worktree 位于 `/repos/app.worktrees/<branch>`），其工作集就有一半落在边界之外。每一次 worktree 的创建、检出与编辑都会落到被拒绝的路径上，而可用的两个手段都不对：把整个会话放宽到 `danger-full-access`，等于彻底放弃约束；或把会话根目录设为 `/repos`，则父目录下的每个同级仓库都变为可写。

这个缺口是结构性的，而非逐工具的。`writableRoots` 是 `workspace-write` 唯一被赋予含义的地方，而该含义此前只由一个字段推导，因此任何消费方都无法在不改变模式的前提下表达"工作区加上这一个伴随目录"。

## Decision

由策略所有者解析伴随根目录；模式词汇表保持不变。

### `SandboxExecutionPolicy.extraWritableRoots`

一个与 `workspaceRoot` 并列携带的可选 `readonly string[]`。默认缺省，因此未做配置的部署所解析出的策略与此前逐字节一致。`writableRoots` 将其并入同一份规范化、去重后的允许列表，于是 Seatbelt profile 与进程内 fs 围栏无需学习新概念即可获得该授权。

### `dsh-sandbox-policy.siblingWritableSuffixes`

一个经校验的 `Config` 字段：追加到已解析工作区根目录自身路径的后缀，每个后缀命名一个伴随目录。`['.worktrees']` 会把 `/repos/app` 解析为 `/repos/app.worktrees`。

采用后缀而非自由路径，是因为这样授权就不可能命名工作区同级目录以外的任何位置：派生路径由会话自身的根目录计算而来，绝不逐调用传入。空后缀或含路径分隔符的条目在加载时即被拒绝。后缀作用于工作区在执行环境中的原始拼写，`writableRoots` 会将工作区及其同级根目录一并规范化，因此各执行层比较的是同一标识。

### 各 runner 的授权写法

Seatbelt 按路径字符串匹配，因此伴随根目录无论是否存在都会被授予——于是 agent 可以自行创建该 worktree 树，而这通常正是第一步。

bwrap 的 `--bind` 与 Landlock 启动器在遇到无法打开的路径时都会 fail closed（`add_rule` 对无法打开的授权根返回 `EXIT_LAUNCHER_FAILURE`），因此在这两个 runner 上，伴随根目录只有在存在之后才会加入授权。后续调用会重新推导 profile 并纳入它。过滤逻辑放在各方言的构建器中而非 `writableRoots` 内，因为这是 runner 的物化约束，而不是对模式承诺的改变。

### Model experience

`sandbox:policy` 会在 `workspace-write` 上下文中列出伴随根目录，使模型知道自己可以在那里写入。该句子仅在实际解析出根目录时出现，从而保持默认渲染逐字节稳定。

## Alternatives considered

**在 `writableRoots` 中硬编码 `.worktrees`。** 改动最小，但是错的：这会把某一个团队的目录约定变成 sandbox 词汇表的属性，而使用其他目录名的部署将毫无手段可用。随部署而变的选择应当是经校验的 `Config` 字段。

**接受绝对伴随路径而非后缀。** 更通用，但在此处严格更差：绝对路径与会话根目录无关，一个配置错误的条目会静默授予一个与该工作区毫无关系的目录。后缀使授权可从工作区推导，而这正是值得保留的性质。

**授予工作区的父目录。** 无需新配置即可覆盖 worktree 树，同时也授予了其下每一个同级仓库——边界将名存实亡。

**新增第四个 `SandboxMode`。** 模式是关于文件效果的承诺；"工作区加伴随目录"是同一个承诺作用于不同的根集合，因此它属于根目录推导，而不属于每个后端都要 switch 的词汇表。

## Consequences

- 部署按 composition 逐一选择加入；随附的 bundle 未配置任何后缀，因此默认约束保持不变。
- 伴随授权的范围就是被命名的那个目录。`.worktrees` 整体可写，这符合意图——它是 agent 自己的暂存树——但并未收窄到某一个分支的 worktree。
- 对于尚未创建的伴随根目录，bwrap 与 Landlock 的行为不同于 Seatbelt。此差异由测试固定，而非被抹平，这与两种方言在 `/tmp` 上已有的差异处理方式一致。
