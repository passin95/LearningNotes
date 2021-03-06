
<!-- TOC -->

- [一、基本概念](#%E4%B8%80%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5)
- [二、Git 命令](#%E4%BA%8Cgit-%E5%91%BD%E4%BB%A4)
  - [2.1 全局配置](#21-%E5%85%A8%E5%B1%80%E9%85%8D%E7%BD%AE)
  - [2.2 查看信息](#22-%E6%9F%A5%E7%9C%8B%E4%BF%A1%E6%81%AF)
    - [2.2.1 git status](#221-git-status)
    - [2.2.2 git log](#222-git-log)
    - [2.2.3 git diff](#223-git-diff)
  - [2.3 远程操作](#23-%E8%BF%9C%E7%A8%8B%E6%93%8D%E4%BD%9C)
    - [2.3.1 git remote](#231-git-remote)
    - [2.3.2 git clone](#232-git-clone)
    - [2.3.3 git push](#233-git-push)
    - [2.3.4 git fetch](#234-git-fetch)
    - [2.3.5 git pull](#235-git-pull)
  - [2.4 管理变动](#24-%E7%AE%A1%E7%90%86%E5%8F%98%E5%8A%A8)
    - [2.4.1 git add](#241-git-add)
    - [2.4.2 git checkout (撤销文件修改)](#242-git-checkout-%E6%92%A4%E9%94%80%E6%96%87%E4%BB%B6%E4%BF%AE%E6%94%B9)
    - [2.4.3 git rm](#243-git-rm)
    - [2.4.4 git commit](#244-git-commit)
    - [2.4.5 git revert](#245-git-revert)
  - [2.5 版本穿梭](#25-%E7%89%88%E6%9C%AC%E7%A9%BF%E6%A2%AD)
    - [2.5.1 git reset](#251-git-reset)
    - [2.5.2 git checkout](#252-git-checkout)
    - [2.5.3 git stash](#253-git-stash)
  - [2.6 分支操作](#26-%E5%88%86%E6%94%AF%E6%93%8D%E4%BD%9C)
    - [2.6.1 git branch](#261-git-branch)
    - [2.6.2 git merge](#262-git-merge)
    - [2.6.3 git rebase](#263-git-rebase)
  - [2.7 标签管理](#27-%E6%A0%87%E7%AD%BE%E7%AE%A1%E7%90%86)
  - [2.8 高级命令](#28-%E9%AB%98%E7%BA%A7%E5%91%BD%E4%BB%A4)
    - [2.8.1 git cherry-pick](#281-git-cherry-pick)
    - [2.8.2 git describe](#282-git-describe)
    - [2.8.3 交互式 rebase](#283-%E4%BA%A4%E4%BA%92%E5%BC%8F-rebase)
- [三、本质解析](#%E4%B8%89%E6%9C%AC%E8%B4%A8%E8%A7%A3%E6%9E%90)
  - [3.1 merge](#31-merge)
  - [3.2 rebase](#32-rebase)
  - [3.3 reset](#33-reset)
  - [3.4 checkout](#34-checkout)
- [四、Git 练习题](#%E5%9B%9Bgit-%E7%BB%83%E4%B9%A0%E9%A2%98)
- [五、SSH 传输设置](#%E4%BA%94ssh-%E4%BC%A0%E8%BE%93%E8%AE%BE%E7%BD%AE)
- [六、.gitignore 文件](#%E5%85%ADgitignore-%E6%96%87%E4%BB%B6)
- [参考资料](#%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99)

<!-- /TOC -->


# 一、基本概念

commit id：对提交的信息数据进行一个 SHA1 算法所得，该 id 几乎不会重复。并且每个 commit 真正意义上是不可修改的，下文的修改 commit 都是指生成一个新的 commit 代替了旧的 commit。

工作区：日常情况下，工作使用的目录和文件。

版本库：工作区的隐藏目录.git。版本库有一个重要的区域叫 stage(暂存区),当我们使用 git add 文件时，会将文件的变动添加到暂存区。

staged changes：所有 add 进暂存区的修改。

unstaged changes：Git 跟踪的（在版本库中存在的）但并未添加到暂存区的修改。

untracked files：在工作目录中新添加的文件。

HEAD：它永远指向当下的 commit。

master：本地仓库默认分支，一般作为主分支使用。

origin/master：远程仓库 origin 的 master 分支镜像。

origin/HEAD：远程仓库 origin 的 HEAD 镜像，它永远指向分支 origin/master。

HEAD，master，origin/master，origin/HEAD 等概念都类似于内存中的指针，区别在于它们指向的是 commit。

<div align="center">
<img src="../pictures//a1198642-9159-4d88-8aec-c3b04e7a2563.jpg" />
</div>

# 二、Git 命令

## 2.1 全局配置

```
git config --global user.name "your name"

告诉 git 你的名字。

git config --global user.email "email@example.com"

告诉 git 你的邮箱。

git config -l

查看 git 所有配置信息。

git config --system --unset credential.helper

清空 git 账号密码的缓存。
```

## 2.2 查看信息

### 2.2.1 git status

```
git status 

查看当前所有文件的状态。

git status 文件

查看指定文件名的状态。
```

### 2.2.2 git log

```
git log

查看所有的 commit 日志。

git log 文件

查看指定文件的 commit 日志。

git log -p

查看所有 commit 的日志详情。

git log --graph

查看以 ASCII 图形表示的分支合并历史。

git log -n

显示最近 n 条 commit 日志。

git log --no-merges 

过滤 merge commit。

git log --oneline

输出所有 commit id 的前几位值以及 commit 描述。

git reflog

查看 HEAD 引用的移动过程。

git reflog <name>

查看分支 name 的移动过程。
```

### 2.2.3 git diff 

```
git diff 文件

查看工作区与暂存区的区别。

git diff --staged

查看暂存区与当前 commit 的区别。

git diff HEAD

查看工作目录和当前 commit 的区别。
```

## 2.3 远程操作

### 2.3.1 git remote 

```
git remote -v

显示本地仓库所关联远程仓库地址名和对应的别名。

git remote add <远程仓库别名 origin> <地址 url>

将本地仓库和该地址的远程仓库地址 url 关联，并设置别名 origin。
```

### 2.3.2 git clone 

```
git clone url 

将远程代码仓库克隆到本地。
```

### 2.3.3 git push

```
git push <仓库地址别名 origin> <分支名 feature>

将本地代码仓库的 feature 分支的 commit 推送至远程仓库 origin。
```

### 2.3.4 git fetch
```
git fetch <仓库别名 origin>

拉取远程仓库 origin 相对于本地新的 commit，并将 origin/master 指向最新的 commit。
```

### 2.3.5 git pull

```
git pull <远程仓库别名 origin> <分支名 feature>

将远程仓库 origin 的 feature 分支的 commit 合并到当前分支。

git pull 等价于（git fetch + git merge feature）
```

## 2.4 管理变动

### 2.4.1 git add 

```
git add .

将该工作区所有文件的修改添加到暂存区。

git add 文件

将该文件的修改添加到暂存区。
```

### 2.4.2 git checkout (撤销文件修改)

```
git checkout 文件

把文件在工作区的修改全部撤销，这里有两种情况：

一种是 readme.txt 自修改后还没有被放到暂存区，现在，撤销修改就回到和版本库一模一样的状态；

一种是 readme.txt 已经添加到暂存区后，又作了修改，现在，撤销修改就回到添加到暂存区后的状态。
```

### 2.4.3 git rm

```
git rm 文件

从版本库中删除该文件。
```

### 2.4.4 git commit 

```
git commit -m "描述"

将暂存区的更改进行提交，生成一个新的 commit，HEAD 指向该 commit。

git commit --amend

修改当前的 commit 内容。
```

### 2.4.5 git revert

```
git revert HEAD 

将 HEAD 指向的 commit 的操作生成一个相反操作的的 commit（用于对已经不能修改的内容进行修改 例如修改 master 分支上的旧 commit）。

git revert <commit id>

将该 commit 的操作生成一个相反操作的的 commit。
```

## 2.5 版本穿梭

### 2.5.1 git reset

git reset 命令都会带着 HEAD 所指向的分支一起移动，例如 HEAD 此时指向 master，执行 git reset HEAD^ 命令，则此时 master 指向该 commit，HEAD 指向 master。

```
git reset HEAD^(HEAD~1)

将 HEAD 指向上一个 commit，若工作区和暂存区存在文件修改需先处理。reset 成功后，会保留工作区内容，并清空暂存区。

git reset <commid id>

将 HEAD 指向指定 commit。

git reset --soft HEAD^

将 HEAD 指向上一个 commit，保留工作目录，并把所有的文件差异放置暂存区。

git reset --mixed HEAD^

将 HEAD 指向上一个 commit，保留工作目录，并把所有的文件差异放置工作区。

git reset --hard HEAD^

将 HEAD 指向上一个 commit，并清空工作区和暂存区的修改。

```

### 2.5.2 git checkout

以下命令 HEAD 不会携带分支引用一起移动。

```
git checkout <name>

切换到分支 name。

git checkout <commit id>

切换到具体 commit。

git checkout -b <name>

创建新分支 name 并切换到新分支 name（实质为 HEAD 指向新分支 name）。

git checkout HEAD^

将 HEAD 指向上一个 commit，若工作区和暂存区存在修改需先处理。

git checkout <commid id>

将 HEAD 指向指定 commit。

git checkout --detach

把 HEAD 和 当前所指向的分支脱离，直接指向当前 commit。
```

### 2.5.3 git stash

```
git stash

临时存放当前分支 staged changes 和 unstaged changes 的变动。

git stash -u

临时存放当前分支 staged changes、 unstaged changes、untracked files 的变动。

git stash pop

恢复当前分支临时存放的变动。

git stash list 

查看 stash 列表。

git stash pop stash@{id}

pop 指定的 stash。

git stash clear

清空所有 stash。
```

## 2.6 分支操作

### 2.6.1 git branch 

```
git branch

查看本地分支。

git branch -r

查看远程仓库镜像分支。

git branch <name>

创建一个名为 name 的分支。

git branch -d <name>

删除名为 name 的分支（只能删除已合并的分支，若要强制删除使用-D）。
```

### 2.6.2 git merge 

```
git merge <name>

合并分支 name 到当前 HEAD 所指向的分支。

git merge --abort

用于处理合并冲突，取消该次 merge。

git merge --continue 

用于处理 merge 冲突，已修改完冲突文件，继续 merge。
```

### 2.6.3 git rebase 

```
git rebase <name>

将 HEAD 所指向的分支上的所有新的 commit（两分支共同的 commit 分叉之后的 commit）“拷贝”一份新的 commit,并接上分支 name 所指向的 commit。

git rebase -i HEAD^^

将HEAD所指向commit在内的最近两个2条commit重新进行“编辑”，若存在修改，则生成2条新的commit代替原来的两个commit。

git rebase --abort

特殊情况下，用于取消该次 rebase。

git rebase --continue 

特殊情况下，处理完之后，继续执行 rebase。例如 rebase 合并冲突，已修改完冲突文件，继续 rebase。
```

## 2.7 标签管理

```
git tag

查看所有 tag。

git tag <tagname>

在当前分支的当前 commit 打一个名为 name 的 tag。

git tag <tagname> <commit id>

对 commit id 所对应的 commit 打一个名为 name 的 tag。

git tag -a <tagname> -m <描述> <commit id>

对 commit id 所对应的 commit 打一个名为 name 且有描述的的 tag。

git tag -d <tagname>

删除本地 tag。

git show <tagname>

查看名为 name 的标签信息。
```

## 2.8 高级命令

### 2.8.1 git cherry-pick

```
git cherry-pick <commit id> <commit id> ……

将指定的多个 commit 复制后追加到 HEAD 指向的 commit(branch) 之后。
```

### 2.8.2 git describe 

``` 
git describe <ref>

<ref> 可以是任何能被 Git 识别成提交记录的引用，如果你没有指定的话，Git 会以你目前所检出的位置（HEAD）。

它会输出一段信息：<tag>_<numCommits>_g<hash>

tag 表示的是离 ref 最近的标签， numCommits 是表示这个 ref 与 tag 相差有多少个提交记录， hash 表示的是你所给定的 ref 所表示的提交记录哈希值的前几位。
```

### 2.8.3 交互式 rebase 

```
git rebase -i <commid id> or <branch name> or <HEAD^> or <HEAD~1>

指定的 commit 到当前 commit 进行重新调整（增删改 commit）。
```

输入命令后会进入一个操作界面，并可指定不同 commit 的执行操作，操作的方式如下：

- pick：什么都不修改；
- reword：修改 commit 的提交信息；
- edit：修改 commit 中的文件内容变动；
- squash: 将指定 commit 和它前一个 commit 合成一个 commit，并支持修改 commit 的提交信息；
- fixup：类似 squash，区别在于指定 commit 会丢失提交信息，直接使用上一个 commit 的提交信息。

保存操作界面的修改后，若有 commit 涉及 edit 操作则会进入一个中间状态，之后操作都是基于该 commit 开始，在编辑完成后，输入 git rebase --continue 完成命令；反之直接修改成功。

# 三、本质解析

## 3.1 merge

<img src="../pictures//15fddc2aad5a0279.gif" width="400"/>

此图含义：当前 HEAD 指向 master 分支，使用命令如下:

```
git merge branch1
```

本质：branch1 的路径上的所有 commit 的内容一并应用到当前分支 master，然后自动生成一个新的 commit，HEAD 和 HEAD 所指向的分支 master 指向新的 commit。

## 3.2 rebase

<img src="../pictures//1600abd620a8e28c.gif" width="400"/>

此图含义：当前 HEAD 指向 master 分支，使用命令如下:

```
git checkout branch1

git rebase master
```

本质：将当前分支 commit 序列重新设置基础点, branch1 的路径上至原基础点（图中为 2）的所有 commit 一并复制到指定分支 master，HEAD 和 HEAD 所指向的分支 指向最新的 commit。

## 3.3 reset

<img src="../pictures//15fe19c8a3235853.gif" width="400"/>

此图含义：当前 HEAD 指向 branch1 分支，使用命令如下:

```
git reset 第三个 commit id
```

<img src="../pictures//15fe333cb605b0de.gif" width="400"/>

此图含义：当前 HEAD 指向 branch1 分支，使用命令如下:

```
git reset branch2
```

本质：将 HEAD 以及它所指向的 branch 指向具体的某个 commit（若不指向任何 branch 则只有 HEAD 移动）。

## 3.4 checkout

<img src="../pictures//160089d53b4f65a5.gif" width="400"/>

此图含义：当前 HEAD 指向 branch1 分支，使用命令如下:

```
git chekout branch2
```

本质：将 HEAD 指向某个 commit 或者 branch（不携带所指向的分支一起移动）。
git chekout 文件的本质是：回到所指向 commit 的该文件内容（视觉上便是清空了工作区的变动，因此暂存区内容是不清理的）。

# 四、Git 练习题

[https://github.com/pcottle/learnGitBranching](https://github.com/pcottle/learnGitBranching)

# 五、SSH 传输设置

Git 仓库和 Github 中心仓库之间的传输是通过 SSH 加密。

在用户主目录下，看看有没有.ssh 目录，或者该目录下有没有 id_rsa 和 id_rsa.pub 这两个文件，如果没有，可以通过以下命令来创建 SSH Key：

$ ssh-keygen -t rsa -C "youremail@example.com"

你需要把邮件地址换成你自己的邮件地址，然后一路回车，使用默认值即可，由于这个 Key 也不是用于军事目的，所以也无需设置密码。

然后把公钥 id_rsa.pub 的内容复制到 Github "Account settings" 的 SSH Keys 中。

# 六、.gitignore 文件

使用 git add 命令时会忽略 .gitignore 文件中的文件或目录。

这个文件的规则对已经追踪的文件是没有效果的，此时需要使用命令 git rm --cached file 删除该文件以及对该文件的追踪。

不需要全部自己编写，可以到 [https://github.com/github/gitignore](https://github.com/github/gitignore) 中进行查询。

# 参考资料

- [Git 教程 - 廖雪峰](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)
- [Git 原理详解及实用指南 - 扔物线](https://juejin.im/book/5a124b29f265da431d3c472e)
