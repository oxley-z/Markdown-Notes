---
author: 鱼鹰Osprey
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzU2MDgyNzgyMw==&mid=2247490303&idx=1&sn=eb38b157c2b1fe024541c33064b05576&chksm=fda0f61f9bfeb4fdbf7f5464248f5f727faad423da9ffa36cbb43c7676dc7c49a5fc39150db5&mpshare=1&scene=1&srcid=06294xRcbl4nEXMy1vBVkT3T&sharer_shareinfo=5a06cec003c0d3adf092efbdc50c67bd&sharer_shareinfo_first=5a06cec003c0d3adf092efbdc50c67bd#rd
saved: 2026-06-29 09:21:26
tags:
  - 笔记同步助手
id: d6e00782-7593-45e8-b38a-178bf99ec44e
---

公众号名称：鱼鹰谈单片机

作者名称：鱼鹰Osprey

发布时间：2026-06-29 08:30

![[image/e8cf5505c0aef956e42ea39032acc389_MD5.jpg]]

来源：公众号【鱼鹰谈单片机】

作者：鱼鹰Osprey

ID ：emOsprey

> 大家好，我是鱼鹰。本文适合已经会用 `git add / commit / push` 的道友。不会这些也能看，但可能会时不时发出"原来还能这样"的感慨。

---

## 一、`git remote add`：不是只有 `origin`

很多教程只教 `git remote add origin xxx`，于是很多人以为远程仓库只能叫 origin。其实 remote 就是给远程地址起个"昵称"，你可以有多个。

### 实际场景

你 fork 了一个开源项目到自己仓库，但原作者仓库还在更新。你只绑定自己的仓库，拉新代码就很麻烦。

```
# 绑定自己的仓库（常规操作）
git remote add origin https://github.com/你的用户名/项目.git

# 再绑定原作者仓库，通常叫 upstream
git remote add upstream https://github.com/原作者/项目.git

# 查看所有远程仓库
git remote -v
```

之后想同步原作者更新：

```
git fetch upstream
git merge upstream/main
```

### 一个小技巧

如果你公司用了 GitLab，同时自己 GitHub 上也有 fork，名字可以起得直观一点：

```
git remote add company https://gitlab.company.com/项目.git
git remote add github https://github.com/你的账号/项目.git
```

> 结论：`origin` 只是习惯，不是规定。合理命名多个 remote，能让你在多个仓库之间切换得更清楚。

---

## 二、`git pull origin develop --rebase`：让提交历史更干净

`git pull` 默认是 `git fetch + git merge`。如果远程有更新，本地也有提交，Git 会生成一个 merge commit：

```
*   Merge branch 'develop' of github.com:xxx/xxx
|\
| * 远程别人的提交
* | 你的提交
```

这种 merge commit 多了，历史图会变得像一团毛线。

### 实际场景

你和同事都在 `develop` 分支上开发。早上你开始写功能 A，中午同事推送了功能 B。你下午要 push 前，先 pull 一下。

```
git pull origin develop --rebase
```

`--rebase` 的意思是：先把你的提交临时"拿下来"，把远程新提交应用上去，再把你的提交依次放回去。结果历史变成一条直线：

```
* 你的提交 A2
* 你的提交 A1
* 同事的功能 B
* 更早的提交
```

### 注意点

-   如果你的提交已经 push 到远程了，并且其他人也在用这个分支，**不要再 rebase**，否则同事会骂你。
    
-   如果这个分支只有你一个人用，那么可以 rebase，但是这样会改变提交记录，因此需要 **\-f** 强制推送到远程
    
-   rebase 过程中如果有冲突，按提示解决后继续：
    

```
git add .
git rebase --continue
```

> 结论：在本地未 push 的提交上，`pull --rebase` 能让历史更线性、更易读。

---

## 三、`git commit --amend`：改完提交再后悔药

你是不是也经常这样：刚 `git commit -m "fix bug"`，突然发现还有个小文件没加进去，或者提交信息写错了一个字。

**不要重新 commit，用 amend。**

### 实际场景 1：补充漏掉的文件

```
# 刚提交完，发现漏了一个文件
git add 漏掉的文件.js
git commit --amend --no-edit
git commit --amend # 记简单点，即使弹出编辑窗口，直接关闭就行
```

`--no-edit` 表示不修改提交信息，只是把新文件合并进上一次提交。

### 实际场景 2：修改提交信息

```
# 把上一次的提交信息改掉
git commit --amend -m "修复登录页样式错位问题"
```

### 注意点

-   amend 会改变 commit 的 hash 值。
    
-   如果上一次提交已经 push 到远程，amend 后需要强制推送：
    

```
git push origin 分支名 --force-with-lease
```

`--force-with-lease` 比 `--force` 安全，会检查远程有没有你本地不知道的更新。

> 结论：`commit --amend` 是本地提交最好的"后悔药"，但 push 过的提交要慎用。

---

## 四、`git reset --soft` 和 `--hard`：两种回退，两种人生

`git reset` 是回退到某个 commit，但 `--soft` 和 `--hard` 差别巨大。

### `git reset --soft HEAD～1`

回退到上一个 commit，但**保留改动在暂存区**。

#### 实际场景

你连续做了三个提交，发现第三个提交其实应该和第二个合并：

```
# 回退到上一个 commit，改动保留在暂存区
git reset --soft HEAD～1

# 然后重新提交，和前面的合并
git commit --amend
```

或者你想把最近的提交拆成多个小提交：

```
git reset --soft HEAD～1
git add 文件1
git commit -m "第一部分改动"
git add 文件2
git commit -m "第二部分改动"
```

### `git reset --hard HEAD～1`

回退到上一个 commit，**所有改动全部丢弃**。

#### 实际场景

你写了一段实验性代码，发现完全走偏了，想全部重来：

```
git reset --hard HEAD～1  # 可以是任意节点值
```

> ⚠️ 危险操作！丢弃的改动找不回来，除非之前有 commit 或 stash。还有可以通过 `git reflog` 查看操作记录

### 对比表

| 命令 | 提交历史 | 工作区改动 | 暂存区改动 |
| --- | --- | --- | --- |
| `git reset --soft HEAD～1` | 删除最近 commit | 保留 | 保留 |
| `git reset --mixed HEAD～1` | 删除最近 commit | 保留 | 清空 |
| `git reset --hard HEAD～1` | 删除最近 commit | 删除 | 删除 |

> 结论：`--soft` 是"我想重新整理提交"，`--hard` 是"我不要了，全删"。

---

## 五、`git rebase -i`：提交历史的整形手术

`-i` 是 interactive（交互式）。这是 Git 里最能提升历史可读性的命令。

### 实际场景

你开发一个功能，过程中产生了 5 个提交：

```
* 修改了 typo
* 调试打印 log
* 登录接口联调
* 登录页面 UI
* 初始化登录模块
```

这些提交要合并到 `main` 分支。你希望历史清爽一点，把 typo 和调试打印合并掉。

```
git rebase -i HEAD～5
```

会弹出一个编辑器：

```
pick 初始化登录模块
pick 登录页面 UI
pick 登录接口联调
pick 调试打印 log
pick 修改了 typo
```

把后面两个改成 `squash`（或简写 `s`）：

```
pick 初始化登录模块
pick 登录页面 UI
pick 登录接口联调
squash 调试打印 log
squash 修改了 typo
```

保存退出后，会让你写新的合并提交信息。最终历史变成：

```
* 登录接口联调（包含 typo 修复和删除调试 log）
* 登录页面 UI
* 初始化登录模块
```

### 常用命令

| 命令 | 作用 |
| --- | --- |
| `pick` | 保留这个提交 |
| `reword` | 保留提交，但修改提交信息 |
| `squash` | 合并到上一个提交 |
| `fixup` | 合并到上一个提交，并丢弃提交信息 |
| `drop` | 删除这个提交 |
| `edit` | 暂停在这里，允许你修改 |

> 结论：`git rebase -i` 是整理本地提交历史的核武器，适合在合并到主分支前让历史"变薄"。

---

## 六、`git stash`：临时存一下，待会再来

你正在改功能 A，突然老板说"先修个线上 bug"。功能 A 还没改完，不能 commit，怎么办？

```
# 把当前改动存起来
git stash push -m "功能 A 做到一半"

# 切换分支修 bug
git checkout hotfix
# ... 修完 bug 提交 ...

# 切回来，恢复之前的改动
git checkout 原来的分支
git stash pop
```

### 常用操作

```
# 查看 stash 列表
git stash list

# 恢复最近一个 stash，并从列表删除
git stash pop

# 恢复但不删除
git stash apply

# 删除最近一个 stash
git stash drop

# 清空所有 stash
git stash clear
```

### 实际场景

有时候你想 pull 最新代码，但本地有未提交的改动：

```
git stash
git pull origin develop --rebase
git stash pop
```

> 结论：`stash` 就是你的临时抽屉，适合被打断时保存当前进度。

---

## 七、`git checkout`：切换、新建、提取文件

`git checkout` 是 Git 里最常用的命令之一，但很多新手只把它当成"切换分支"用。其实它能做的事远不止于此。

### 一个前提：干净的工作空间

Git 切换分支时，如果工作区有未提交的改动，通常会报错：

```
error: Your local changes to the following files would be overwritten by checkout:
```

但有一个例外：**如果目标分支和当前分支指向同一个 commit 节点**，即使工作区不干净，也能切换成功。这个特性在"代码写到一半发现分支名起错了"时特别有用。

### `git checkout branch-name-xx`：切换分支

```
# 切换到已有的本地分支
git checkout develop

# 如果本地没有 xx-demo，但远程有 origin/xx-demo
# Git 会自动基于远程分支创建一个同名的本地分支并切换过去
git checkout xx-demo
```

> 注意：这种自动创建要求工作空间是干净的。

### `git checkout hash-value-or-tag`：临时查看某个提交

线上出问题了，你想看上周三发布版本 `v1.2.0` 时的代码长什么样：

```
# 切换到某个 tag
git checkout v1.2.0

# 或者切换到某个 commit hash
git checkout a3f7d2e
```

这时候你处于"分离 HEAD"状态，可以查看代码，但最好**不要直接修改提交**。如果想基于这个点做修改，应该先新建分支：

```
git checkout -b hotfix-from-v1.2.0
```

### `git checkout -b`：新建并切换分支

```
# 基于当前分支，新建一个分支并切换
git checkout -b feature/pay

# 基于远程分支新建本地分支
git checkout -b feature/pay origin/feature/pay
```

### `git checkout hash-or-branch -- file`：提取指定版本的文件

这是 checkout 最容易被忽略的神技。

假设你不小心把 `src/config.js` 改坏了，想恢复成 `main` 分支上的版本：

```
git checkout main -- src/config.js
```

或者你想看某个历史 commit 里的 `README.md`：

```
git checkout a3f7d2e -- README.md
```

提取出来的文件（或者**文件夹**）会直接进入 **Staged（暂存区）** 状态，你可以打开查看、编辑，确认没问题后再 commit。

> 结论：`git checkout` 不只是"切分支"，它是切换、查看历史、恢复文件的多面手。

---

## 八、`git diff`：不只是看改了什么

`git diff` 大家都不陌生，但加上一些参数后，它会变成排查问题的利器。

### `git diff hash1 hash2 --stat`：只看改动了哪些文件

假设你想知道从 `v1.2.0` 到 `v1.3.0` 到底动了哪些文件：

```
git diff v1.2.0 v1.3.0 --stat
```

输出大概长这样：

```
src/login.js        |  45 +++++
 src/pay.js          |  12 +-
 src/config.js       |   2 +-
 tests/login.test.js |  23 +++
 4 files changed, 67 insertions(+), 15 deletions(-)
```

**优点**：快速定位影响范围，不用被大量代码 diff 淹没。适合在发版前做变更审查，或者排查"这次上线到底改了什么"。

### 其他常用 diff 姿势

```
# 工作区和暂存区的差异
git diff

# 暂存区和最新 commit 的差异
git diff --cached

# 当前分支和 main 分支的差异
git diff main

# 只看某个文件在两个提交之间的差异
git diff v1.2.0 v1.3.0 -- src/login.js

# 只看新增/删除/重命名了哪些文件
git diff v1.2.0 v1.3.0 --stat
```

### 实际场景

线上出问题了，你怀疑是昨晚的发布引入的。先用 `--stat` 看一眼改动范围：

```
git diff 昨晚的commit 现在的commit --stat
```

发现 `payment/` 目录下有三个文件被改了，再聚焦看具体改动：

```
git diff 昨晚的commit 现在的commit -- payment/
```

> 结论：`git diff --stat` 是快速定位变更范围的"雷达图"，先锁定文件，再深入细节。

---

## 九、组合技：一个完整的工作流

假设你在 `feature/login` 分支开发登录功能：

```
# 1. 基于最新 develop 切分支
git checkout develop
git pull origin develop --rebase
git checkout -b feature/login

# 2. 开发过程中提交（允许历史乱一点）
git add .
git commit -m "初始化登录模块"
# ... 写 UI ...
git add .
git commit -m "登录页面 UI"
# ... 联调接口 ...
git add .
git commit -m "登录接口联调"
# ... 发现 typo ...
git add .
git commit -m "fix typo"

# 3. 开发完，整理提交历史
git rebase -i HEAD～4
# 把 typo 那行改成 squash

# 4. 同步远程 develop 的更新
git checkout develop
git pull origin develop --rebase
git checkout feature/login
git rebase develop

# 5. 推送到远程
git push origin feature/login
```

这样你的 PR/MR 历史干净、线性、好 review。

---

## 总结

| 命令 | 什么时候用 |
| --- | --- |
| `git remote add` | 需要同时关联多个远程仓库 |
| `git pull --rebase` | 想拉取远程更新，同时保持本地提交线性 |
| `git commit --amend` | 刚提交完发现漏了文件或写错提交信息 |
| `git reset --soft` | 回退 commit，但保留改动重新整理 |
| `git reset --hard` | 彻底丢弃最近的改动（危险） |
| `git rebase -i` | 合并、修改、删除本地提交，整理历史 |
| `git stash` | 临时保存当前进度，切换做别的事 |
| `git checkout` | 切换分支、基于远程新建分支、查看历史提交、恢复指定文件 |
| `git diff --stat` | 快速查看两次提交之间改动了哪些文件 |

Git 的学习曲线确实有点陡，但这些命令用熟了，你会发现：同事 review 你的代码更舒服了，你自己找 bug 也更方便了，最重要的是——**少了很多不必要的 merge conflict 和强制推送事故**。

---

**你在 Git 里踩过最大的坑是什么？欢迎在评论区分享。**

_本文示例基于 Git 2.30+，部分命令在老版本上可能有差异。_

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/22901d26_1782696084715?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzU2MDgyNzgyMw%3D%3D%26mid%3D2247490303%26idx%3D1%26sn%3Deb38b157c2b1fe024541c33064b05576%26chksm%3Dfda0f61f9bfeb4fdbf7f5464248f5f727faad423da9ffa36cbb43c7676dc7c49a5fc39150db5%26mpshare%3D1%26scene%3D1%26srcid%3D06294xRcbl4nEXMy1vBVkT3T%26sharer_shareinfo%3D5a06cec003c0d3adf092efbdc50c67bd%26sharer_shareinfo_first%3D5a06cec003c0d3adf092efbdc50c67bd%23rd&s=obsidian)