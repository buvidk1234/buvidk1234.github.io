+++
date = '2026-07-22T20:54:40+08:00'
draft = false
title = 'Git Submodule'
tags = ['Git', 'Git Submodule', '工程化']
+++

# Git Submodule：原理与协作流程

`git submodule` 将一个独立 Git 仓库挂载到另一个仓库的工作树中。被挂载的仓库称为 submodule，外层仓库称为 superproject。父仓库记录的不是子模块文件副本或分支名，而是子仓库的**某个提交 ID**。

本文以 Git 官方文档 [gitsubmodules](https://git-scm.com/docs/gitsubmodules) 为准，使用一个最小的 `app` 与 `libfoo` 仓库示例说明其工作原理、日常操作和适用边界。

**工作树（working tree）** 是当前 commit 被检出到文件系统中的文件集合，也就是用户实际查看和修改的目录。它与暂存区和 Git 元数据目录不同：

```text
工作树       -- git add -->  暂存区（.git/index）
暂存区       -- git commit -> 提交对象（.git/objects/）
```

例如，`third_party/libfoo` 是 submodule 的工作树；该目录中的文件可以被编辑，但它们的提交历史、索引和对象保存在对应的 Git 目录中。

## Submodule 结构

设主仓库为 `app`，其中有一个子模块 `third_party/libfoo`。它们是两个相互独立的仓库：

```text
app (superproject，父仓库)
├── .gitmodules                 # 受版本控制的子模块声明
├── src/
└── third_party/libfoo          # submodule 工作树，也是独立 Git 仓库
    └── ...
```

父仓库的一个提交中，`third_party/libfoo` 对应的树项模式是 `160000`，Git 称它为 **gitlink**。其对象 ID 是子仓库的 commit ID，例如：

```text
160000 a1b2c3d4e5f6789012345678901234567890abcd 0  third_party/libfoo
```

这行并不保存 `libfoo` 的源文件，也不表示“跟随 `stable` 分支最新代码”。它只表示：检出这个父仓库提交时，`third_party/libfoo` 应当位于 `a1b2c3d...`。因此同一个父仓库提交在任何机器、任何时间都能还原同一组源码版本，这是 submodule 最重要的价值。

可以用下列命令亲自观察 gitlink：

```bash
git ls-files -s -- third_party/libfoo
git ls-tree HEAD third_party/libfoo
```

普通文件的模式通常是 `100644` 或 `100755`，子模块则是 `160000`。父仓库的 `git diff` 因而只会显示子模块提交从哪个 SHA 变为哪个 SHA。

## 两份配置与一个实际工作树

理解 submodule 的关键是分清 `.gitmodules` 与本地配置。

| 位置 | 是否提交 | 作用 |
| --- | --- | --- |
| `.gitmodules` | 是 | 声明子模块名称、路径、克隆 URL，以及可选的跟踪分支、浅克隆等共享设置。 |
| `.git/config` | 否 | 本地可用的 `submodule.<name>.url`、`active` 等配置；初始化时由 `.gitmodules` 复制而来。 |
| `.git/modules/<path>/` | 否 | 子模块实际的 Git 目录，保存对象、引用和配置。 |
| `<path>/.git` | 否 | 现代 Git 中是一个很小的文本文件，指向上面的 Git 目录。 |

例如 `.gitmodules` 可包含：

```ini
[submodule "third_party/libfoo"]
    path = third_party/libfoo
    url = https://github.com/example/libfoo.git
    branch = stable
    shallow = true
```

其中 `path` 决定工作树位置，`url` 决定首次克隆来源。`branch` 是执行 `git submodule update --remote` 时要查询的远端分支；它**不会改变**父仓库 gitlink 仍然锁定具体提交这一事实。`shallow = true` 让初始化或更新可以使用浅克隆，减少下载量，但也意味着需要旧历史时可能必须补取对象。

`.gitmodules` 是随 superproject 提交的共享配置，`.git/config` 是当前 clone 的本地配置。首次执行：

```bash
git submodule update --init third_party/libfoo
```

Git 按以下顺序处理：

1. 从 `.gitmodules` 读取 `submodule.third_party/libfoo.url`；
2. 如果 `.git/config` 中还没有该 URL，则将它写入本地配置；
3. 使用本地配置中的 URL 克隆或获取 `libfoo`；
4. 检出 superproject gitlink 指定的 commit。

初始化后，可以只在当前 clone 中改用镜像地址：

```bash
git config submodule.third_party/libfoo.url https://mirror.example.com/libfoo.git
```

该修改只写入 `.git/config`，不会修改或提交 `.gitmodules`。如果仓库维护者后来修改了 `.gitmodules` 中的 URL，`git pull` 只会更新 `.gitmodules`，不会自动覆盖已有本地 URL。需要采用新的共享 URL 时，执行：

```bash
git submodule sync --recursive
git submodule update --init --recursive
```

`git submodule sync` 会把 `.gitmodules` 中的 URL 同步到 `.git/config`；如果本地故意配置了镜像地址，执行该命令会覆盖这个本地 URL。

Git 官方文档给出的 submodule 配置优先级从高到低为：

1. 命令行参数，例如 `--recurse-submodules`；
2. submodule 自身的 `$GIT_DIR/config`、`.gitattributes` 和 `.gitignore` 等配置；
3. superproject 的 `.git/config`；
4. superproject 中受版本控制的 `.gitmodules`。

Git 只自动递归处理 active submodule。active 状态依次按以下规则判定，前面的规则优先：

1. `submodule.<name>.active` 是否显式设置为 `true` 或 `false`；
2. submodule 路径是否匹配 `submodule.active` pathspec；
3. 是否设置 `submodule.<name>.url`。

例如，只激活 `third_party/` 下的模块：

```bash
git config submodule.active 'third_party/*'
git submodule update --init
```

## 为什么检出后经常是 detached HEAD

父仓库需要精确检出 gitlink 指向的提交，默认的 `git submodule update` 使用 `--checkout` 策略。因此子模块常处于 detached HEAD：HEAD 直接指向一个提交，不附着在本地分支上。这是可复现构建所需的正常状态，不是错误。

```text
父仓库 HEAD
  └── gitlink: a1b2c3d...
        └── 子模块 HEAD: a1b2c3d... (detached)
```

在子模块里开发时，应先创建或切换到分支，再提交和推送：

```bash
cd third_party/libfoo
git switch -c fix/route-timeout
# 编辑、测试
git add .
git commit -m "fix: handle route timeout"
git push -u origin fix/route-timeout
```

然后回到父仓库提交新的 gitlink。只提交子模块内部的代码，并不会自动让父仓库指向新版本。

## 一个完整的最小例子

创建两个仓库：`libfoo` 提供库代码，`app` 使用该库。

先在 `libfoo` 中提交一个版本：

```bash
mkdir libfoo
cd libfoo
git init
printf 'v1\n' > version.txt
git add version.txt
git commit -m "release: libfoo v1"
git branch -M stable
git remote add origin https://github.com/example/libfoo.git
git push -u origin stable
```

再在 `app` 中添加 submodule：

```bash
mkdir app
cd app
git init
git submodule add -b stable https://github.com/example/libfoo.git third_party/libfoo
git add .gitmodules third_party/libfoo
git commit -m "build: add libfoo submodule"
```

此时 `app` 中包含两个可独立操作的 Git 仓库。现代 Git 的 submodule 工作树只包含一个指向父仓库 Git 目录的 `.git` 文本文件：

```text
app/
├── .git/
├── .gitmodules
└── third_party/libfoo/
    ├── .git              # 指向 app/.git/modules/third_party/libfoo
    └── version.txt
```

`app` 的提交只保存 `third_party/libfoo` 的 gitlink。其他开发者获取 `app` 时，必须初始化 submodule：

```bash
git clone https://github.com/example/app.git
cd app
git submodule update --init --recursive
```

或直接递归克隆：

```bash
git clone --recurse-submodules https://github.com/example/app.git
```

## 日常操作：从接入到升级

### 新增一个子模块

在父仓库根目录执行：

```bash
git submodule add -b stable https://github.com/example/libfoo.git third_party/libfoo
git commit -m "build: add libfoo submodule"
```

这会完成四件事：克隆子仓库、写入 `.gitmodules`、在索引中加入一个 `160000` gitlink、在本地配置记录 URL。`-b stable` 用于记录远端跟踪分支，初始 gitlink 依然是当时检出的具体提交。

不要手工 `git clone` 到一个普通目录后期待 Git 自动把它当子模块。应使用 `git submodule add`，或在确实需要迁移已有目录时按照官方的迁移流程处理索引和 `.gitmodules`。

### 将本地工作区恢复到父仓库锁定版本

拉取父仓库新提交后，gitlink 可能已变化。执行：

```bash
git pull --recurse-submodules
git submodule update --init --recursive
```

第一条可在拉取时同时递归获取子模块；第二条把工作树检出到当前父仓库提交记录的 SHA。也可以配置默认行为：

```bash
git config --global submodule.recurse true
```

全局开启前应确认团队习惯，因为此设置会影响许多 Git 命令的递归范围。

### 审阅状态与差异

```bash
git submodule status --recursive
git status
git diff --submodule=log
```

`git submodule status` 每行开头的符号很有用：

- 空格：子模块位于父仓库期望的提交。
- `-`：子模块尚未初始化。
- `+`：工作树的子模块提交与父仓库记录不一致。
- `U`：存在合并冲突。

`git diff --submodule=log` 比只看两个 SHA 更适合代码审查，它会列出两个子模块提交之间的提交日志。发布前还应进入子模块检查脏文件，因为子模块内部未提交的修改同样会影响构建：

```bash
git -C third_party/libfoo status --short
git submodule foreach --recursive 'git status --short'
```

### 有意升级到一个确定版本

最稳妥的升级流程是先选定并验证子仓库提交，再更新父仓库 gitlink：

```bash
cd third_party/libfoo
git fetch origin tag v1.2.3
git switch --detach v1.2.3
# 运行 app 的构建和测试

cd ../..
git diff --submodule=log -- third_party/libfoo
git add third_party/libfoo
git commit -m "build: bump libfoo submodule"
```

`detached HEAD` 时，`HEAD` 直接指向某个 commit。切换到其他分支后，`HEAD` 不再引用它；该 commit 不会立即删除，但如果没有 branch 或 tag 引用，可能在 reflog 过期后被 GC 清理。

若维护策略是“取某个分支当前最新提交”，可使用：

```bash
git submodule update --remote --recursive
git add third_party/libfoo
git commit -m "build: update upstream submodules"
```

`--remote` 会从 `.gitmodules` 的 `branch` 配置（或默认远端 HEAD）获取目标提交。它只改变本地子模块 HEAD；只有将 gitlink 提交到父仓库，其他人才能获得同一升级结果。对包含多个相互依赖 submodule 的工程，不应无验证地批量执行该命令；应按兼容组合升级、测试，然后一次提交相关 gitlink。

### 在子模块中修复并回流

修改依赖源码一般需要两个提交或两个 PR：一个属于子仓库，一个属于父仓库。

```bash
# 1. 在子仓库的分支完成修复并推送
cd third_party/libfoo
git switch -c fix/example
# edit, test, commit, push

# 2. 在父仓库记录已验证的子仓库提交
cd ../..
git add third_party/libfoo
git commit -m "build: use libfoo fix/example"
git push
```

这两个仓库独立审查、独立权限控制。父仓库只通过 SHA 选择“采用哪一次子仓库提交”。如果子模块提交尚未推送到协作者可访问的远端，协作者执行 `submodule update` 会失败，因此应先确保子仓库提交可获取，再合并父仓库的 gitlink 更新。

推送 superproject 前可要求 Git 检查 submodule commit 是否已发布：

```bash
git push --recurse-submodules=check
```

也可以让 Git 尝试按需推送 superproject 引用但远程尚不可用的 submodule commit：

```bash
git push --recurse-submodules=on-demand
```

## `update` 的几种策略

默认 `git submodule update` 使用 checkout 策略：检出父仓库记录的提交，通常导致 detached HEAD。Git 也支持在有本地分支时采用合并或变基策略：

```bash
git submodule update --merge path/to/module
git submodule update --rebase path/to/module
```

前者把父仓库期望的提交合并到当前子模块分支，后者把当前本地提交变基到该提交之上。它们适合在子模块里同时维护本地工作时同步父仓库，但会引入普通 Git 合并/变基的冲突处理成本。团队的自动化构建和 CI 应优先使用默认 checkout，以保证干净、确定的输入。

可以把策略写到配置中：

```ini
[submodule "third_party/libfoo"]
    update = checkout
```

官方文档还支持 `none`，用于明确禁止某个模块被普通 update 操作自动检出。无论选择哪种策略，父仓库提交中的 gitlink 都是最终版本来源。

## 生命周期：停用与删除

本文只讨论现代 Git 的三种状态：

| 形态 | gitlink | `.gitmodules` 配置 | 工作树 | Git 目录 |
| --- | --- | --- | --- | --- |
| 基本形态 | 有 | 有 | 有 | 通常在 `.git/modules/` |
| deinitialized | 有 | 有 | 空 | 通常保留 |
| deleted | 无 | 无对应配置 | 无 | 可能保留 |

### 临时停用 submodule

先检查工作树：

```bash
git -C third_party/libfoo status --short
```

没有输出表示没有待处理的修改。此时使用非强制形式：

```bash
git submodule deinit third_party/libfoo
```

`deinit` 会清空 submodule 工作树，并删除当前 clone 的 `.git/config` 中对应的本地配置；它不会修改 superproject 的 gitlink、`.gitmodules` 或提交历史。`.git/modules/third_party/libfoo` 通常会保留，因此之后仍可恢复。

如果工作树有未提交修改或未跟踪文件，非强制命令会拒绝执行。需要保留修改时，先在 submodule 中提交，或保存到 stash：

```bash
git -C third_party/libfoo stash push --include-untracked -m "before deinit"
git submodule deinit third_party/libfoo
```

`--include-untracked` 不会保存被 `.gitignore` 忽略的文件；确实需要保存这类文件时使用 `git stash push --all`。

需要丢弃工作树内容时，才使用 `-f`：

```bash
git submodule deinit -f third_party/libfoo
```

`-f` 会强制删除工作树，未提交修改和未跟踪文件无法通过该工作树恢复。它不会删除 `.git/modules/third_party/libfoo` 中的 Git 对象；恢复工作树：

```bash
git submodule update --init third_party/libfoo
```

如果之前使用了 stash，恢复后再执行：

```bash
git -C third_party/libfoo stash pop
```

从项目历史中删除模块：

```bash
git rm third_party/libfoo
git commit -m "build: remove libfoo submodule"
```

`git rm` 会删除工作树、暂存区中的 gitlink 和 `.gitmodules` 中的对应配置，并将这些删除操作放入 superproject 的暂存区；提交后才会从项目历史中删除该模块。若 submodule 有本地修改，`git rm` 默认会拒绝；确认不要这些修改时可以使用 `git rm -f third_party/libfoo`，同样会造成数据丢失。`.git/modules/third_party/libfoo` 可能继续保留，使 Git 在检出旧的 superproject commit 时可以复用对象。

## Git 元数据目录结构

普通 Git 仓库的 `.git` 是仓库元数据目录，不是项目源代码目录。不同 Git 版本和操作状态下，目录内容会有差异；以下是常见项目及其作用：

| 文件或目录 | 作用 |
| --- | --- |
| `HEAD` | 当前检出位置。通常内容为 `ref: refs/heads/main`；detached HEAD 时直接保存 commit ID。 |
| `config` | 仓库级配置，包括 remote、branch、用户行为和 submodule 本地配置。 |
| `index` | 二进制暂存区，记录下一次 commit 的候选 tree；`git add` 修改它，`git commit` 读取它。它不是提交历史。 |
| `objects/` | Git 对象数据库，保存 blob、tree、commit、tag 对象；对象可能以 loose 或 pack 形式存在。 |
| `refs/heads/` | 本地分支引用，例如 `refs/heads/main`。引用文件中的值是 commit ID。 |
| `refs/remotes/` | 远程跟踪分支，例如 `refs/remotes/origin/main`。 |
| `refs/tags/` | tag 引用；大量引用可能被压缩到 `packed-refs`。 |
| `packed-refs` | 压缩存储的 refs，Git 查找引用时与 `refs/` 一起使用。 |
| `logs/` | reflog，记录 HEAD 和引用的移动历史，例如 `logs/HEAD`、`logs/refs/heads/main`。 |
| `hooks/` | 客户端和服务端 hook。目录中的 `.sample` 文件只是示例，去掉后缀并赋予执行权限后才会运行。 |
| `info/exclude` | 仅对当前仓库生效的忽略规则，不纳入版本控制。 |
| `description` | GitWeb 等旧式工具使用的仓库描述文件，普通 Git 操作通常不依赖它。 |

以下文件或目录只在特定操作后出现：

| 文件或目录 | 出现条件和作用 |
| --- | --- |
| `FETCH_HEAD` | 最近一次 `fetch` 获取到的远程引用及其提交。 |
| `ORIG_HEAD` | merge、rebase、reset 等操作前保存的原始 HEAD，便于回退。 |
| `MERGE_HEAD` | merge 尚未完成时，记录待合并的提交。 |
| `CHERRY_PICK_HEAD` | cherry-pick 发生冲突或等待提交时，记录当前提交。 |
| `rebase-merge/`、`rebase-apply/` | rebase 进行中的状态文件。 |
| `shallow` | 浅克隆的边界 commit 列表。 |

可以使用 Git 自身命令获取当前仓库的元数据路径，不要假设 `.git` 一定是目录：

```bash
git rev-parse --git-dir
git rev-parse --git-common-dir
```

对于 submodule，工作树中的 `<path>/.git` 通常是一个文本指针文件：

```text
gitdir: ../../.git/modules/third_party/libfoo
```

因此，`third_party/libfoo/.git` 不是完整 Git 目录；该 submodule 的 `HEAD`、`index`、`objects/`、`refs/` 等内容位于：

```text
.git/modules/third_party/libfoo/
```

这也是为什么 submodule 可以删除工作树而保留 Git 对象，之后再通过 `git submodule update --init` 恢复工作树。

## 合并冲突与常见故障

### 两个分支都改了同一子模块指针

这不是源代码逐行冲突，而是两个 gitlink 指向不同提交。先检查两个候选提交的关系：

```bash
git -C third_party/libfoo fetch origin
git -C third_party/libfoo merge-base --is-ancestor <ours> <theirs>
git diff --submodule=log
```

如果其中一个提交是另一个的祖先，通常选择较新的提交即可。若两者分叉，需要在子仓库中先合并或选择一个包含所需改动的提交，再在父仓库 `git add third_party/libfoo` 并完成合并。不要只凭 SHA 大小或提交日期作决定。

### 子模块 URL 已变更或无法下载

父仓库更新 `.gitmodules` 后，本地 `.git/config` 不会自动覆盖已有 URL。同步后再初始化：

```bash
git submodule sync --recursive
git submodule update --init --recursive
```

对于私有 fork，还要确认 SSH key、HTTPS 凭据和权限能同时访问父仓库及每个子模块仓库。

若出现 `fatal: remote error: upload-pack: not our ref`，说明 superproject 的 gitlink 指向的 commit 无法从当前 submodule 远程获取。常见原因是 commit 尚未推送、远程历史被改写、`.gitmodules` 指向错误 fork，或服务端禁止按对象 ID 获取不可达对象。应发布或恢复该 commit、修正 URL，或者让 superproject 改为引用远程可获取的 commit；重复执行 `git submodule update` 不会修复错误的版本契约。

### 只 clone 了父仓库，目录为空

这通常不是文件丢失，而是尚未初始化子模块：

```bash
git submodule update --init --recursive
```

今后使用 `git clone --recurse-submodules`，或为仓库配置 `submodule.recurse`，可减少这一类问题。

### 浅克隆后找不到旧提交

带 `shallow = true` 的模块下载历史较少。若需要查看旧提交、变基或解决较深的合并关系，可在相应子模块中补全历史：

```bash
git -C third_party/libfoo fetch --unshallow
```

仓库原本不是浅克隆时，该命令会失败；此时可使用 `git -C third_party/libfoo fetch --tags origin`。不要在没有确认磁盘和网络成本前，对大型依赖仓库盲目完整拉取。

## 何时适合使用 submodule

submodule 适合以下场景：

- 依赖本身是独立项目，有独立提交历史、维护者和发布节奏。
- 父项目需要锁定并审计一组精确上游提交，构建可复现性高于“总是最新”。
- 需要直接修改、调试或构建依赖源码，例如应用需要维护自己的 `libfoo` fork。
- 依赖体积大，不希望把完整历史和文件复制进父仓库。

它不一定适合以下场景：

- 只是语言包依赖。Go module、Maven、npm、Cargo 等包管理器通常能更好地处理版本解析、缓存和发布产物。
- 团队无法保证每个开发者和 CI 都能访问所有子仓库。
- 依赖不再独立演进，或必须以单一原子提交修改父子两侧并统一审查。这种情况下 vendoring、monorepo 或 subtree 可能更直接。

submodule 不是依赖解析器，也不会替你解决多个仓库之间的 API 兼容性；它只严格保存“父仓库选择了哪些子仓库提交”。在本例中，`app` 通过 gitlink 固定 `libfoo` 的源代码版本，构建过程每次都能恢复同一版本集。

## 团队约定清单

1. 克隆使用 `--recurse-submodules`；构建前执行 `git submodule update --init --recursive`。
2. 评审 gitlink 改动时使用 `git diff --submodule=log`，不要只审一个 SHA 的变化。
3. 升级先验证子模块提交，再提交父仓库指针；关联模块按兼容组合一起升级。
4. 修改子模块要在子模块分支提交并推送，随后单独提交父仓库 gitlink。
5. 不把子模块的未提交修改当作可交付状态；CI 应从干净工作树检出父仓库锁定的提交。
6. URL 变更后运行 `git submodule sync --recursive`；需要嵌套模块时始终使用 `--recursive`。

## 参考资料

- Git 官方文档：[gitsubmodules](https://git-scm.com/docs/gitsubmodules)
- Git 官方命令参考：[git-submodule](https://git-scm.com/docs/git-submodule)
