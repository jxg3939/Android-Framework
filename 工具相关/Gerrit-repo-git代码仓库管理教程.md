# Gerrit + repo + git 代码仓库管理教程

> 面向 Android 这类**多仓库超大工程**的日常工作流。三者关系一句话概括:
> **git 管一个仓库的版本,repo 管几百个 git 仓库的批量同步,Gerrit 管所有仓库的代码审核与权限。**

## 1 三者是什么关系

```
                    ┌────────────────────────────────────────┐
                    │  Gerrit (服务端, 带 Web UI)             │
                    │  - 托管几百个 git 仓库                  │
                    │  - refs/for/<branch> 收变更, 走审核      │
                    │  - 标签: Code-Review / Verified         │
                    │  - 权限: 谁可读、谁可推、谁可合入          │
                    └───────────────▲────────────────────────┘
                                    │ ssh/git (29418 / 80)
                    ┌───────────────┴────────────────────────┐
                    │  repo (客户端脚本, 一个 python 脚本)      │
                    │  - 读 manifest.xml 决定拉哪些仓库         │
                    │  - 批量 sync / status / forall / upload │
                    └───────────────▲────────────────────────┘
                                    │ 调用
                    ┌───────────────┴────────────────────────┐
                    │  git (底层, 每个仓库一份 .git)           │
                    │  - 本地仓库、分支、暂存区、rebase        │
                    └────────────────────────────────────────┘
                                    │
                    ┌───────────────┴────────────────────────┐
                    │  工作区: frameworks/base, system/core…  │
                    └────────────────────────────────────────┘
```

关键点:

| 层次 | 作用域 | 你什么时候直接用 |
|------|--------|------------------|
| git | 单个仓库 | 改代码、提交、rebase、cherry-pick、查看历史 |
| repo | 整个 workspace(所有仓库) | 初始化、同步、批量状态、批量上传 |
| Gerrit | 服务端 | 代码审核、合入、权限、查 change |

**最常见的误解**:`repo sync` 不是 git 命令,它是把 manifest 里每个仓库各跑一遍 `git fetch` + `git rebase/checkout`。所以出错时最终还是要回到 git 层面排查。

## 2 git:单仓库的基本功

### 2.1 四个区域

```
工作区(working tree) --git add--> 暂存区(index) --git commit--> 本地仓库(.git) --git push--> 远程(Gerrit)
        ▲                                                                              │
        └──────────────────── git checkout/reset ◄────────── git fetch ◄──────────────┘
```

- `git add -p`:逐块挑选改动进暂存区,提交前自检的利器。
- `git restore --staged <file>`:从暂存区退回工作区(旧写法 `git reset HEAD <file>`)。
- `git restore <file>`:丢弃工作区改动(旧写法 `git checkout -- <file>`)。

### 2.2 提交与修改提交(Gerrit 更新 patchset 的唯一方式)

```bash
# 正常提交
git commit -s          # -s 加 Signed-off-by; Android 一般还需要 Bug:/Test: 行
# 修改刚才那次提交(保持 Change-Id 不变 => Gerrit 上是同一个 change 的新 patchset)
git add -p
git commit --amend --no-edit
# 只想改提交信息
git commit --amend
```

**铁律**:一次提交 = 一个 change。更新已有 change **只能** `--amend`,不能再 `git commit` 一次,否则会冒出两个 change。

### 2.3 分支与变基

```bash
git fetch origin                 # 只更新远程跟踪分支, 不动工作区
git rebase origin/master         # 把本地提交挪到最新 master 之上(线性历史)
git rebase -i HEAD~3             # 交互式: squash/fixup/reword/drop/reorder
git rebase --continue            # 解决冲突后继续
git rebase --abort               # 放弃整个 rebase
```

Gerrit 通常**不允许 merge commit**(除非项目开了 `Merge If Necessary`)。如果本地 `git pull` 产生了 merge commit,push 会报:

```
! [remote rejected] ... merge commit ... not permitted
```

正确姿势是 `git pull --rebase` 或先 `git fetch` 再 `git rebase origin/<branch>`。

### 2.4 cherry-pick 与 rebase --onto

Android 同一份补丁常常要同时进 master、各种 release 分支。

```bash
# 把某个提交拿到当前分支(注意此处不是提交到 Gerrit)
git cherry-pick <sha1>
git cherry-pick -x <sha1>       # -x 在提交信息里注明 "(cherry picked from ...)"
git cherry-pick -n <sha1>       # 只应用到工作区, 不自动提交
```

跨分支搬到 Gerrit 的标准做法其实是本地 cherry-pick 后 push 到另一个分支的 `refs/for/`:

```bash
git fetch origin refs/changes/23/12345/3   # 直接取某个 change 的某个 patchset
git checkout FETCH_HEAD                     # 用它建临时 HEAD
git log -1 --format=%s                      # 看看是不是要的那个
```

### 2.5 出事后救命

```bash
git reflog                    # 所有 HEAD 移动记录, 找到误删的提交
git reset --hard HEAD@{3}     # 回到 reflog 里的那个点
git reset --soft HEAD~1       # 撤提交但保留改动在暂存区
git reset --mixed HEAD~1      # 撤提交, 改动回工作区(默认)
git stash / git stash pop     # 临时打包未提交改动
git fsck --lost-found         # 找游离对象(极端情况)
```

### 2.6 Android commit message 规范

```
frameworks: 修复 SurfaceFlinger 合成后撕裂问题

问题: 在 120Hz 高刷屏上, 部分场景合成帧被提前提交到 display,
导致上半屏是旧帧下半屏是新帧。

改动: 在 commit 前检查 retire fence 的 signal time ...

Bug: 123456789
Test: atest SurfaceFlingerTest
Change-Id: I4d2a1e0c9f3b7a8e5d6c1b2a3f4e5d6c7b8a9f01   <-- Gerrit 靠它关联 change
```

要点:
- 空行前是**标题**(建议 ≤ 50 字符, 不要以句号结尾),空行后是正文。
- `Bug:` / `Test:` / `Signed-off-by:` 等 **footer 放在最后一段**,Gerrit 的 hook 会把 `Change-Id` 追加在最后。
- **Change-Id 是 Gerrit 的身份标识**,格式固定为 `I` + 40 位十六进制。同一个 change 的所有 patchset 共享同一个 Change-Id。

## 3 repo:多仓库的批量管理

repo 本身是一个 python 脚本(不是 git 的子命令),通过 `~/.gitconfig` 里的 `url.<base>.insteadOf` 与 ssh 访问 Gerrit。

### 3.1 初始化 workspace

```bash
# 装 repo(国内建议走清华/中科大镜像)
curl -s https://mirrors.tuna.tsinghua.edu.cn/git/git-repo -o ~/bin/repo
chmod a+x ~/bin/repo

mkdir aosp && cd aosp
# -u 清单仓库地址, -b 清单分支, -m 清单文件名(默认 default.xml)
# --depth=1 / --partial-clone 只拉最新一层和部分 blob, 省磁盘和时间
repo init -u ssh://gerrit.example.com:29418/manifest -b main \
          -m default.xml --no-tags --depth=1 --partial-clone --clone-filter=blob:limit=10M
```

初始化做了什么:

```
.repo/
├── manifests.git/        # manifest 仓库的裸库
├── manifests/            # manifest 仓库的工作区(default.xml 等)
├── manifest.xml -> manifests/default.xml   # 软链接, repo 实际读它
├── projects/             # 每个项目的 git 目录都放这(*.git 目录)
├── project-objects/      # 多个 worktree 共享对象时用
├── repo/                 # repo 自身(git 仓库, 可 repo selfupdate)
└── local_manifest.xml    # 可选: 本地额外/覆盖清单, 不参与提交
```

因此:workspace 里某项目下的 `.git` 其实是**文件**,内容为 `gitdir: .repo/projects/xxx.git`。这也是为什么 `repo sync` 能一边管几百个仓库一边共享对象。

### 3.2 manifest 的结构

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <!-- 远程: name 是 repo 内部的别名, fetch 是真正的 URL, review 是 Gerrit 的 ssh 地址 -->
  <remote name="origin"
          fetch="ssh://gerrit.example.com:29418"
          review="gerrit.example.com:29418" />

  <!-- 默认属性: 没写 origin 的项目都用 remote="origin", revision="main" -->
  <default remote="origin" revision="main" sync-j="8" />

  <!-- 单项目: path 是本地路径, name 是仓库名 -->
  <project path="frameworks/base" name="platform/frameworks/base" groups="pdk" />

  <!-- 引用别的清单, 典型的分层: default.xml include 各子清单 -->
  <include name="pdk.xml" />

  <!-- 把已有项目摘掉 / 复制文件 / 建立链接 -->
  <remove-project name="platform/packages/apps/OldApp" />
  <copyfile src="build/envsetup.sh" dest="build/envsetup.sh" />
  <linkfile src="build/envsetup.sh" dest="envsetup.sh" />

  <!-- 覆盖某个项目的 revision(比如 fix 到特定 tag) -->
  <project path="external/foo" name="platform/external/foo" revision="refs/tags/android-14.0.0_r1" />
</manifest>
```

`groups` 是很有用的一层控制:`repo init -g default,-pdk` 之类可以只拉一部分仓库。

### 3.3 同步与批量操作

```bash
repo sync -c -j8 --no-tags --prune          # 日常同步(最常用)
#   -c/--current-branch 只同步当前分支, 大幅省带宽
#   -j8                 8 并发
#   --no-tags           不拉 tag(Android 仓库 tag 极多)
#   --prune             删掉远程已删除的分支对应的本地跟踪分支
#   -f/--force-sync     本地被改乱了, 强推覆盖
#   -d/--detach         同步后把 HEAD 脱离到 manifest 指定 revision
#   -l/--local-only     只用本地已有对象, 不联网(适合网络差时断点续传)
#   -s/--smart-sync     配合 smart manifest, 按上次成功编译的清单对齐

repo status                       # 等价于所有仓库的 git status, 只列有变化的
repo diff                         # 所有仓库的 diff
repo branch                       # 看每个仓库当前分支
repo info .                       # 看当前项目: 路径/远程/分支/清单 revision

# 对所有仓库执行同一命令(repo 的杀手锏)
repo forall -c 'git log --oneline -1'
repo forall -c 'git clean -xdf'           # 危险: 清掉所有未跟踪文件
repo forall external/  -p -c 'git describe --tags'   # -p 打印项目名, -e 只对有改动的项目
# 可用环境变量: $REPO_PROJECT $REPO_PATH $REPO_REMOTE $REPO_RREV $REPO_LREV $REPO_I_MANIFEST
```

### 3.4 本地分支与上传

```bash
# 建一个本地工作分支(名字即 topic 名), 只对当前项目
repo start mytopic .
# 对所有有改动的项目建同名分支
repo start mytopic --all

repo upload .                       # 上传当前项目当前分支上的提交
repo upload --cbr -t mytopic        # --cbr 用当前分支, -t 打 topic
repo upload --re reviewer@example.com --cc teammate@example.com
repo upload -d                      # 传成 draft(仅自己可见)
repo upload --no-verify             # 跳过本地 hook

repo prune mytopic                  # 分支已合并进 Gerrit 后, 删掉本地分支
repo abandon mytopic                # 强制丢弃本地分支
repo download platform/frameworks/base 12345 3   # 直接把别人的 change+patchset 拉到本地
```

`repo upload` 的判定逻辑:**只上传"本地分支有、远程分支没有"且提交信息里带 Change-Id 的提交**。所以:

- 忘了 commit → 没东西可传;
- 提交信息里没有 Change-Id → 报 `no Change-Id` 或上传后被 Gerrit 拒;
- 忘了 `repo start` → `upload` 报 `no branches ready for upload`。

### 3.5 固定版本的清单(发版/回滚用)

```bash
# 把当前所有仓库的实际 revision 凝固成一份清单, 用于可复现的编译
repo manifest -r -o pinned-2026-10-12.xml
# 之后别人用这份清单初始化, 拿到的就是完全一致的代码
repo init -u ssh://gerrit.example.com:29418/manifest -b main -m pinned-2026-10-12.xml
```

注意 `-r`(revision-as-HEAD)会把每个仓库的当前 HEAD SHA1 写死。**改了 manifest 也要走 Gerrit 提交**,因为 manifest 仓库自己就是一个 Gerrit 项目。

### 3.6 常用 .repo 目录下的手工排查

```bash
cd .repo/manifests && git log --oneline -5      # 清单仓库的最近改动
cat .repo/manifest.xml | grep -n frameworks     # 确认某项目来自哪份清单的哪一行
repo forall -c 'git remote -v'                  # 确认每个仓库的 fetch 地址对不对
```

## 4 Gerrit:代码审核与合入

### 4.1 核心概念

| 概念 | 说明 |
|------|------|
| Change | 一次待审核的变更,编号全局唯一(如 `12345`)。一个 commit 对应一个 change |
| Patch Set | change 的某一版。`git commit --amend` 后 push,生成 Patch Set 2、3… |
| Review Label | `Code-Review -2..+2`、`Verified -1..+1`(CI 打分) |
| Submit | 满足所有 label 条件后合入目标分支 |
| Topic | 给一组相关 change 打的标签,便于批量查看/提交(一次跨仓库改动常用) |
| `refs/for/<branch>` | **推变更用的虚拟 ref**,不是真实分支,只走审核 |
| `refs/changes/xx/yyyy/z` | 真实存放 patchset 的 ref,`xx` 是 change 号后两位 |

推送到 `refs/heads/<branch>` 是直接改分支(通常被禁),推 `refs/for/<branch>` 才是"提一个待审 change"。

### 4.2 一次性配置

```bash
# 1) 生成 ssh key 并注册到 Gerrit 的 Settings -> SSH Keys
ssh-keygen -t rsa -b 4096 -C "you@example.com"

# 2) 简化 ssh, 免去每次 -p 29418
cat >> ~/.ssh/config <<'EOF'
Host gerrit.example.com
    Port 29418
    User your_gerrit_username
    IdentityFile ~/.ssh/id_rsa
EOF

# 3) 装 commit-msg hook(每个仓库都要, repo 会按 manifest 的 review 属性自动装)
curl -Lo .git/hooks/commit-msg \
  http://gerrit.example.com/tools/hooks/commit-msg && chmod u+x .git/hooks/commit-msg
# 或者 scp 版本
scp -p -P 29418 user@gerrit.example.com:hooks/commit-msg .git/hooks/

# 4) 验证连通
ssh gerrit.example.com gerrit version
ssh gerrit.example.com gerrit ls-projects | head
```

**commit-msg hook 的唯一职责**:在提交信息末尾自动补一个 Change-Id。没有它,Gerrit 会拒绝 push(除非服务端允许缺省)。

### 4.3 一条完整的提单流程

```bash
# ---- 1. 同步到最新 ----
repo sync -c -j8

# ---- 2. 建工作分支 ----
repo start fix-tearing frameworks/native

# ---- 3. 改代码 ----
cd frameworks/native
# ... edit ...

# ---- 4. 提交 ----
git add -p
git commit      # 编辑器里写好标题/正文/Bug:/Test:, 保存时 hook 自动补 Change-Id
git log -1      # 检查一遍

# ---- 5. 更新到最新基线, 解决冲突 ----
git fetch origin && git rebase origin/main

# ---- 6. 上传 ----
repo upload --cbr -t fix-tearing
# 终端会列出每个 change 的目标分支, 并要求 y/n 确认

# ---- 7. 之后处理审核意见: 继续改 + amend, 同一个 Change-Id => 新 patchset ----
git add -p && git commit --amend --no-edit
repo upload --cbr -t fix-tearing

# ---- 8. 合入后清理 ----
repo sync -c            # 已合入的提交会被同步下来
repo prune fix-tearing  # 删本地工作分支
```

### 4.4 直接用 git push(不经过 repo upload)

离开 repo 或只想传一个仓库时:

```bash
git push origin HEAD:refs/for/main
# push option 可在 % 后追加, 逗号分隔
git push origin HEAD:refs/for/main%topic=fix-tearing,r=reviewer@example.com,cc=team@example.com,wip
#   topic=xxx        打 topic
#   r=/cc=           加 reviewer / cc
#   wip / ready      标记 work-in-progress / 转正式
#   private          仅自己与 reviewer 可见
#   publish-comments 同时发布草稿评论
#   force            允许非快进(权限足够时)
```

更新已有 change:先 `amend` 再 push **同一个分支**,Gerrit 靠 Change-Id 认出来是新 patchset。

### 4.5 Gerrit 服务端操作(ssh 命令行)

```bash
G="ssh gerrit.example.com gerrit"

$G query --format=JSON --current-patch-set status:open owner:self | head
$G query change:12345 --format=JSON | python3 -m json.tool
$G ls-projects --type=code --prefix=platform/frameworks
$G set-reviewers --add reviewer@example.com 12345
$G review 12345,3 --code-review +2 --message '"LGTM"'
$G stream-events                       # 实时看事件流(合入/评论)
```

网页端常用动作:

| 动作 | 说明 |
|------|------|
| Rebase | 把 change 变基到目标分支最新提交(比本地 rebase 省事) |
| Cherry Pick | 把 change 复制到另一个分支,直接生成新 change(跨分支同步补丁首选) |
| Abandon / Restore | 放弃 / 恢复一个 change(状态可逆,不删数据) |
| Add to Topic | 加 topic,便于 `topic:xxx` 一次查全 |
| Edit / Delete | 删掉某个 patchset(仅自己,且未提交前) |
| Submit | 满足 label 后合入。失败时看 "Submit Requirements" 里缺哪一项 |

### 4.6 提交策略与权限(管仓库的人需要懂)

- **Submit Type**(在项目 `refs/meta/config` 的 `project.config` 里配):
  - `Fast Forward Only`:只接受线性前进,最严格,Android 主仓常用;
  - `Rebase If Necessary`:不满足时自动 rebase(会改写 committer);
  - `Merge If Necessary`:允许 merge commit;
  - `Cherry Pick`:God 模式,每次提交都在目标分支重新生成一个 commit(适合 release 分支)。
- **常用权限组**:`Read`、`Push`(到 `refs/for/*`)、`Label Code-Review`、`Label Verified`、`Submit`、`Forge Author/Committer`、`Push merge commit`、`Push` 到具体 ref。
- **配置入口**:项目的 `refs/meta/config` 分支,里面是 `project.config` 和 `groups`。改它本身也是一次 Gerrit 提交:

```bash
git fetch origin refs/meta/config && git checkout FETCH_HEAD
# 编辑 project.config / rules.pl
git commit -a -m "调整 Code-Review 权限" && git push origin HEAD:refs/meta/config
```

## 5 常见问题排查表

| 现象 | 原因 | 处理 |
|------|------|------|
| `remote rejected ... missing Change-Id` | 缺 commit-msg hook,或重写提交信息时把 Change-Id 删了 | 装 hook;`git commit --amend` 手工补一行 `Change-Id: I...` |
| `no new changes` | 本地改动没 commit,或改了但没 amend(内容与 patchset 完全相同) | `git status` 检查;`git commit` 或 `--amend` |
| `merge commit ... not permitted` | 用了 `git pull` 而非 `--rebase` | `git reset --hard` 回退后改成 `git fetch && git rebase origin/<branch>` |
| `prohibited by Gerrit: not permitted: create` | 推到了 `refs/heads/xxx` 而不是 `refs/for/xxx` | 改推送目标 |
| `no branches ready for upload` | 没 `repo start`,当前项目不在工作分支上 | `repo start <topic> .` |
| `repo sync` 报 `duplicate path` | 两份 manifest 声明了同一路径 | 检查 `local_manifest.xml` 与 include 的清单 |
| `repo sync` 卡在某个仓库 | 单仓库太大或网络抖动 | `repo sync <path>` 单独重试;或 `-l` 只用本地对象 |
| 某仓库被改乱,`sync` 报错 | 工作区有本地改动 / 分支被改 | `repo sync -f <path>` 强推;或先 `git stash` |
| `insufficient permission for adding an object` | ssh 用户或仓库路径不对 | `git remote -v` 检查地址;`ssh gerrit.example.com gerrit ls-projects \| grep xxx` |
| `error: object file ... is empty` | 上次同步被中断,对象库损坏 | 删掉该仓库目录重新 `repo sync`(或 `.repo/projects/xxx.git` 后重拉) |
| `Permission denied (publickey)` | key 没注册或端口不对 | `ssh -v -p 29418 user@host` 看握手细节 |
| `Cannot rebase: You have unstaged changes` | 工作区脏 | `git stash` → rebase → `git stash pop` |
| push 之后看不到 change | 上传到了错误分支,或被 `-d` 传成 draft | `gerrit query owner:self` 全部列出 |

## 6 速查表

```bash
# ================= git =================
git add -p                         # 逐块暂存
git commit --amend --no-edit       # 更新 patchset 的正确姿势
git fetch origin && git rebase origin/main
git log --oneline --graph --decorate -20
git show <sha1>
git cherry-pick -x <sha1>
git reset --soft HEAD~1            # 撤提交留改动
git reflog                         # 救命
git fetch origin refs/changes/45/12345/2 && git checkout FETCH_HEAD

# ================= repo =================
repo init -u <url> -b <branch> -m default.xml --depth=1 --partial-clone
repo sync -c -j8 --no-tags --prune
repo start <topic> .
repo status / repo diff / repo branch / repo info .
repo forall -p -c '<cmd>'
repo upload --cbr -t <topic>
repo download <project> <change> [patchset]
repo prune <topic> / repo abandon <topic>
repo manifest -r -o pinned.xml
repo selfupdate

# ================= gerrit =================
scp -p -P 29418 u@gerrit:hooks/commit-msg .git/hooks/     # 装 hook
git push origin HEAD:refs/for/main%topic=t,r=reviewer@x.com
ssh gerrit gerrit query --format=JSON status:open owner:self
ssh gerrit gerrit ls-projects
ssh gerrit gerrit review 12345,3 --code-review +2
```
