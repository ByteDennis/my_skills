---
id: 6a03caf99b8e6
name: Git Skills
tags: [git]
updated_at: 2026-05-18T16:01:01.730241Z
---

## Git 命令速查

本卡整理自一次真实 session：将上游仓库（`puiching/main`）以 vendor 方式拉取到 `references/puiching/`，以及围绕这一操作的检查与审计命令

| 标题 | 内容 |
|------|------------------|
| 1. Inspection | `status -sb`、`status -- <path>`、`log --oneline`、`log A..B`、`rev-list --left-right --count`、`rev-parse --show-toplevel --git-dir`、`remote -v`、`branch -r`、`ls-tree`、`show <ref>:<path>` |
| 2. Diff | `diff --stat`、跨 ref 的 `diff <A>:<path> <B>:<path>`、负 pathspec `':!<pattern>'` |
| 3. Fetching | `fetch <remote>`、`-C <dir>` 单次切目录、`-c <key>=<val>` 单次覆盖 config (本会话用来禁 LFS) |
| 4. Tree extraction | `git archive <ref> \| tar -x [-C <target>]` + 子树切片 |
| 5. Recovery | `checkout -- / restore`、`.gitignore` 跟已 track 文件的关系、`grep -rl + xargs rm` 清 LFS pointer |
| 6. Composite recipes | 一段贴就能跑的 vendor-pull / 比对 fork / 看子树 commits / **按 remote URL 找回失踪仓库** 的命令组合 |
| 7. Pitfalls | 7 条本会话亲历的坑——`tar -x` 跟 shell cwd 走、`-C` 不传给 tar、`archive` 会跑 smudge、`.gitignore` 不撤已 track、worktree 的 `.git` 不是本地、`fetch` 安全可重跑等 |

---

## 1. 状态检查 — "我在哪？现在是什么状态？"

### `git status --short --branch`

```
$ git status --short --branch
## main...origin/main [ahead 1]
M  src/app.js
?? newfile.txt
```

> 状态(简洁) + 分支/上游行。

`--short` 将每条路径压缩为两列：**左列**为暂存区状态，**右列**为工作区状态（`M` 已修改、`A` 已添加、`D` 已删除、`??` 未追踪、`MM` 两处均有改动）。`--branch`（或 `-b`）在首行输出类似 `## branch...origin/branch [ahead 3, behind 1]` 的追踪信息——一眼判断是否需要 push/pull。`--short` 格式也便于用 `awk` 脚本化处理。

### `git status --short -- <pathspec>`

```
$ git status --short -- src/app.js
M  src/app.js
```

> 将 `git status` 的范围限定到指定路径。

`--` 分隔符告诉 git "之后的是 pathspec，不是选项"（防止歧义文件名）。适用于 monorepo 或只关心某个子树时。pathspec 支持 glob 和 `:!pattern` 排除语法，例如 `git status -- ':!**/__pycache__/'`。这套语法同样适用于 `diff`、`log`、`add`、`restore`，值得专门记忆。

### `git log --oneline -N [-- <path>]`

```
$ git log --oneline -3 -- src/app.js
a1b2c3d Fix login bug
e4f5g6h Add dark mode
7i8j9k0 Initial commit
```

> 紧凑历史（每提交一行），可按路径过滤。

`--oneline` 是 `--pretty=oneline --abbrev-commit` 的缩写——只显示短 SHA + 提交主题。`-N` 限制回溯深度；加路径 pathspec 可追问"这个文件/目录被哪些提交改过？"。进阶用法：`--grep="vendor"` 按提交信息过滤、`-S "函数名"` 启用**pickaxe**（查找某字符串被加入/删除的提交）。

### `git log --oneline A..B`

```
$ git log --oneline main..feature-x
e4f5g6h Add dark mode
a1b2c3d Fix login bug
```

> B 可达而 A 不可达的提交——即"B 相对 A 多了什么"。

两点语法 `A..B` 是"上次同步以来上游做了什么"的标准答案。三点 `A...B` 是对称差集（双方各有但对方没有的提交），配合 `--left-right` 可标注每条提交属于哪侧。

### `git rev-list --left-right --count HEAD...puiching/main`

```
$ git rev-list --left-right --count HEAD...origin/main
20       284
```

> 输出两个数字——三点范围左右各有多少提交。

`--count` 把 SHA 列表折叠为计数，`--left-right` 按左右分列。输出 `20    284` 意为"HEAD 有 20 个上游没有的提交，上游有 284 个 HEAD 没有的提交"。这是 rebase / merge / squash 前的"分叉程度"标准检查。**记住：左 = HEAD，右 = 另一侧**。

### `git rev-parse --show-toplevel --git-dir`

```
$ git rev-parse --show-toplevel --git-dir
~/competition/TAAC2026-xiang
~/competition/TAAC2026/.git/worktrees/TAAC2026-xiang
```

> 输出工作树根目录和 `.git` 元数据目录位置。

"我到底在哪？"的终极命令。当你怀疑自己处于 worktree、submodule，或某个看似独立却属于父仓库的目录时，用它来确认。`--git-dir` 在 worktree 场景下会返回 `<main>/.git/worktrees/<name>`，而非本地的 `.git`。

### `git remote -v`

```
$ git remote -v
origin  git@github-work:001eander/TAAC2026.git (fetch)
origin  git@github-work:001eander/TAAC2026.git (push)
puiching        https://github.com/001eander/TAAC_2026.git (fetch)
puiching        https://github.com/001eander/TAAC_2026.git (push)
```

> 列出所有 remote 及其 fetch/push URL。

每个 remote 显示两行——fetch URL 和 push URL（二者可以不同，用 `git remote set-url --push` 单独设置）。命名惯例：`origin` 指向自己的 fork，`upstream`/`puiching` 等指向协作方。

### `git branch -r`

```
$ git branch -r
  origin/main
  origin/feature-x
  origin/bugfix-login
```

> 列出远程追踪分支（`refs/remotes/<remote>/<branch>`）。

`git fetch` 之后，远程分支以 `<remote>/<branch>` 形式出现在本地——这是只读的本地指针，记录上次 fetch 时的远端状态。`git branch -a` 同时显示本地和远程追踪分支；`| grep puiching/` 快速过滤。

### `git ls-tree [-r] [--name-only] <tree-ish>[:<path>]`

```
$ git ls-tree -r --name-only main:src
src/app.js
src/utils.js
src/index.js
```

> 列出树对象内容，无需检出。

`git ls-tree puiching/main` 即"上游 main 根目录有什么"。`:<path>` 后缀深入子树；`-r` 递归到叶节点；`--name-only` 去掉 mode/type/SHA，只留干净的文件列表。在不写入工作区的情况下对比两棵树，这是不二之选。

### `git show <ref>[:<path>]`

```
$ git show main:src/app.js
import express from 'express';
const app = express();
...
```

> 输出某 ref 的内容（默认：提交信息 + diff；加 `:<path>` 输出文件原始内容）。

两种主要用法：`git show abc123` 显示提交信息 + unified diff；`git show abc123:README.md` 输出该提交时的 README，无需检出。`git show --stat <ref>` 只看文件级变更统计，适合快速浏览大提交的波及范围。

### `git show <ref>:<path> > /dev/null 2>&1`（存在性探针）

> 通过退出码判断某路径在指定 ref 中是否存在。

```bash
if git show "puiching/main:$name" > /dev/null 2>&1; then
    echo "双方均有"
else
    echo "仅本地有"
fi
```

比 `git ls-tree ... -- $name` 更简洁的是/否判断。

---

## 2. 差异对比 — "改了什么？"

### `git diff [--stat] [A] [B] [-- <path>]`

```
$ git diff --stat main feature-x -- src/
 src/app.js   | 10 +++++++---
 src/utils.js |  2 +-
 2 files changed, 9 insertions(+), 3 deletions(-)
```

> 对比工作区、暂存区或任意两个 ref，可选汇总模式。

| 用法 | 含义 |
|------|-------------|
| `git diff` | 未暂存变更（工作区 vs 暂存区） |
| `git diff --cached` | 已暂存变更（暂存区 vs HEAD） |
| `git diff HEAD` | 全部变更（工作区 vs HEAD） |
| `git diff A B` | 任意两个 ref |
| `git diff --stat ...` | 换成文件级统计摘要 |

配合 pathspec（`-- src/ ':!src/vendor/'`）可精确控制范围。

### `git diff <ref-A>:<path> <ref-B>:<path>`

```
$ git diff main:src/app.js feature-x:src/app.js
-import express from 'express';
+import fastify from 'fastify';
 const app = express();
```

> 直接跨分支对比单个文件，不用checkout任何一边

只要 SHA 在对象库中可达（包括孤立提交），都可以这样对比。

### `git diff A B -- ':!<pattern>'`

```
$ git diff main feature-x -- ':!*.test.js'
-import express from 'express';
+import fastify from 'fastify';
```

> 带负 pathspec 的差异对比，过滤掉噪音路径。

`:!`（或 `:(exclude)`）前缀表示排除。可叠加多个排除 pathspec，例如 `-- src/ ':!src/vendor/'`（包含 src/，但排除 src/vendor/）。在 monorepo 中尤为有用。

---

## 3. 拉取 — 从远端同步状态，不改动本地分支

### `git fetch <remote>`

> 下载新对象并更新远程追踪 refs，不触碰工作区或本地分支。

`fetch` 是无破坏性的。输出形如 `06b03d1..12d0305 main -> puiching/main`，表示远程追踪指针的移动。`git pull = fetch + merge`（或 rebase），但多数老手偏好分两步执行，让合并决策更明确。`git fetch --prune` 同时清理已被上游删除的远程追踪分支。

### `git -C <dir> <command>`

```
$ git -C /home/user/projects/myrepo status --short
M  src/app.js
?? newfile.txt
```

> 以 `<dir>` 为工作目录执行 git 命令，比 `cd && git ...` 更干净。

三个使用理由：(1) 脚本中不想改变 shell 的 cwd；(2) 单次 session 中操作多个仓库；(3) 复制粘贴时含义无歧义。本 session 曾险些因 `cd` 后接 `tar -x` 导致解压到错误目录——用 `-C` 可彻底规避此类 bug。

### `git -c <key>=<value> <command>`

```
$ git -c core.autocrlf=false diff main feature-x
-import express from 'express';
+import fastify from 'fastify';
```

> 为单次调用注入配置覆盖，不写入 `git config`。

经典用例——绕过 LFS 执行 archive：

```bash
git -c filter.lfs.smudge=cat \
    -c filter.lfs.process= \
    -c filter.lfs.required=false \
    archive puiching/main | tar -x
```

三个设置合力屏蔽 LFS：用 `cat`（空操作）替换 smudge filter、清空长驻 filter-process、将 LFS 标记为可选。不用 `-c` 而用 `git config` 写入会持久化设置，通常不是你想要的。

---

## 4. 树提取 — 快照式 vendor

### `git archive <ref> | tar -x [-C <target>]`

```
$ git archive main | tar -x -C /tmp/myrepo-export
$ ls /tmp/myrepo-export
src/  package.json  README.md
```

> 将某 ref 的树序列化为 tar 流并解压到目标目录。

`git archive` 输出的只有文件，没有 `.git` 元数据——vendor 上游子目录的标准方案。用 `tar -C <target>` 显式指定解压位置，永远不要依赖 shell 的当前目录。

> **致命陷阱**：`tar -x` 解压到 shell 的 cwd，而非 git 的工作目录。安全写法：
> ```bash
> git archive <ref> | tar -x -C <explicit-target>
> ```

### `git archive <ref>:<subpath> | tar -x -C <target>`

```
$ git archive main:src | tar -x -C /tmp/src-export
$ ls /tmp/src-export
app.js  utils.js  index.js
```

> 只提取源 ref 的某个子树。

`git archive puiching/main:src` 只打包 `src/` 子树，且路径以 `src/` 为根（`src/foo.py` 在 archive 中变为 `foo.py`）。上游仓库很大但只需要其中一个角落时，用这个。

---

## 5. 恢复 — 撤销或隔离问题

### `git restore <path>`

```
$ git restore src/app.js
```

> 从暂存区恢复文件，丢弃未暂存的工作区变更。<br>
> 工作区 → (git add) → 暂存区 → (git commit) → 提交历史

你修改文件后，变更只在工作区。git add 把快照复制到暂存区，git commit 才把暂存区内容写入历史。这样你可以精确控制每次提交包含哪些变更。

### `.gitignore` 与 `git status` 的交互

> `.gitignore` 只隐藏**未追踪**文件；已追踪的文件即使匹配规则，仍会继续被追踪。<br>
> 已追踪（tracked）= 曾经被 git add + git commit 过的文件，Git 已记录其历史。

这是面试中最常被忽视的细节。一旦追踪，.gitignore 对它无效，需用 git rm --cached <file> 解除追踪。

```bash
git rm --cached <path>   # 从暂存区移除，不删除工作区文件
# 然后在 .gitignore 中补上对应规则
```

### `grep -rl "<sentinel>" <path> | xargs rm`

```
$ grep -rl "TODO" ./src | xargs rm
$ ls ./src
index.js  utils.js
```

> 找出含特定标记的文件并删除（用于清理 LFS 指针文件）。

LFS 指针文件以 `version https://git-lfs.github.com/spec/v1` 开头。本 session 用此方式删除了 45 个损坏的指针文件。含空格文件名时，用 `xargs -d '\n' rm` 或 `find ... -exec rm {} +` 更安全。

---

## 6. 实战 Recipe

### Vendor 拉取上游到子目录（无 submodule）

```bash
# 1. 配置 remote（一次性）
git remote add puiching https://github.com/001eander/TAAC_2026.git

# 2. 拉取上游 refs（无破坏性）
git fetch puiching

# 3. 拉取前检查分叉状态
git rev-list --left-right --count HEAD...puiching/main
git log --oneline HEAD..puiching/main          # 上游新增了什么
git diff --stat HEAD puiching/main -- references/puiching/

# 4. 将上游树解压到 vendor 子目录（绕过 LFS）
git -C /path/to/repo \
    -c filter.lfs.smudge=cat \
    -c filter.lfs.process= \
    -c filter.lfs.required=false \
    archive puiching/main | tar -x -C references/puiching/

# 5. 清理损坏的 LFS 指针（如有）
grep -rl "version https://git-lfs.github.com" references/puiching/ | xargs rm

# 6. 审查并提交
git status --short -- references/puiching/
git add references/puiching/ .gitignore
git commit -m "chore(references/puiching): pull puiching/main @<sha>"
```

### 查清上游/fork 关系

```bash
git remote -v                           # 有哪些 remote？
git branch -r | grep <remote>/          # 那里有哪些分支？
git log --oneline -1 <remote>/main      # 上游 tip 是什么？
git rev-list --left-right --count HEAD...<remote>/main  # 分叉多远？
```

### 查看某子树在 N 个提交内的变化

```bash
git log --oneline -N -- <path>          # 触碰该路径的提交
git log --oneline -p -- <path>          # 提交 + patch
git diff <ref-A>..<ref-B> -- <path>     # 两个 ref 之间的净差异
git diff --stat <ref-A>..<ref-B> -- <path>  # 只看文件级统计
```

### 按 remote URL 找回忘了放哪的仓库

> 知道仓库的 remote slug，但忘了克隆到本地哪个目录。

```bash
PATTERN='OWNER/REPO'
find / -type d -name .git 2>/dev/null | while read g; do
    d=$(dirname "$g")
    u=$(git -C "$d" remote -v 2>/dev/null)
    [[ "$u" == *"$PATTERN"* ]] && printf '%s\n%s\n\n' "$d" "$u"
done
```

把 `OWNER/REPO` 换成你记得的任意 URL 片段（仓库名、用户名、域名均可）。

**各部分说明：**

| 片段 | 作用 |
|------|------|
| `find / -type d -name .git 2>/dev/null` | 遍历整个文件系统，找出所有 `.git` 目录；`2>/dev/null` 压掉 `/proc`、`/root` 等权限报错 |
| `dirname "$g"` | 去掉 `/.git` 后缀，得到仓库根目录 |
| `git -C "$d" remote -v` | 列出该仓库所有 remote（不只是 `origin`）——这是早期尝试失败的关键修复点 |
| `[[ "$u" == *"$PATTERN"* ]]` | glob 匹配 remote URL 中的仓库 slug |
| `printf '%s\n%s\n\n'` | 打印路径 + 匹配的 remote 行 |

> [!NOTE]
> **为什么 `git remote get-url origin` 会失败？** 如果你的 remote 不叫 `origin`（例如叫 `my_work`、`upstream`、`work` 等），`get-url origin` 返回空，什么也匹配不到。`remote -v` 会列出**所有** remote，才是可靠的做法。

---

## 7. 踩过（或险些踩到）的坑

1. **`tar -x` 跟着 shell 的 cwd 走，不跟 git 走。** `cd` 之后接 `git archive | tar -x`，解压落点就是 shell 当前目录——可能恰好是某个意外位置。永远用 `tar -x -C <explicit-dir>`。

2. **`git -C <dir>` 不影响 tar。** `-C` 只对 git 有效。`git -C foo archive ... | tar -x` 中，tar 仍在 shell 的 cwd 解压。

3. **`git archive` 会触发 smudge filter。** LFS 追踪的二进制文件需要网络访问才能从指针展开；若 LFS 服务器返回 404 或离线，archive 会中途中断，导致部分解压。用上文的 `-c filter.lfs.*` 三连来屏蔽。

4. **`git status --short` 对 vendor 子树中"从未本地追踪"的新文件只显示 `??`。** 若上游*删除*了某文件，你本地的旧副本不会显示为 `D`——git 从未追踪它，无从比较。需显式 `diff` 与 `<remote>/<branch>` 对比才能发现上游的删除。

5. **`.gitignore` 不能取消追踪已追踪的文件。** 解法：`git rm --cached` + 补充 ignore 规则。

6. **在 worktree 中工作？** `git rev-parse --git-dir` 返回 `<main-repo>/.git/worktrees/<name>` 而非本地 `.git`。看起来像独立仓库的子目录（如 `references/foo/`）实际上可能归属父 worktree——用 `rev-parse --show-toplevel` 确认。

7. **`git fetch` 可以放心重复执行。** 它无破坏性（不合并、不检出）。不确定时，先 fetch，再决定如何处理新状态。

TAGS: git, vendor, lfs, monorepo, interview
