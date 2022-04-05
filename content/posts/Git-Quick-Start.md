---
title: Git Quick Start
author: kiosk
tags: ["git"]
categories: ["git"]
date: 2020-12-19 19:11:00
---
只是git入门的简单指南。没什么大不了的 :)

<!--more-->
Git 是一个开源的分布式版本控制系统，用于敏捷高效地处理任何或小或大的项目。Git 与常用的版本控制工具 CVS, Subversion 等不同，它采用了分布式版本库的方式，不必服务器端软件支持。
<img src='https://img1.kiosk007.top/static/images/git/git.png' style='height:340px' />


参考: https://git-scm.com/book/zh/v2



# 安装 git 

[安装 Git Linux 版](https://git-scm.com/download/linux)
``` bash
$ apt-get install git
$ apt-get install gitk
```
gitk 是可以可视化的git客户端工具，更多工具参考 [git-gui](https://git-scm.com/download/gui/linux) ，如 [Gitkraken (小章鱼)](https://juejin.cn/post/6844903904451231757)、 [magit](https://magit.vc/manual/magit/Installing-from-the-Git-Repository.html#Installing-from-the-Git-Repository) 等等


# 配置
- **配置 user 信息**

``` bash
$ git config --global user.name  "your_name"
$ git config --global user.email "your_name@domain.com"

# --local 只对某个分支有效
# --global 对当前用户所有仓库有效
# --system 对系统所有用户有效

$ git config --global core.editor "vim"    # git 交互改为vim
 
```

- **查看config的配置**

``` bash
$ git config --list
```

# 使用

区域划分

- **工作区(Working Area)**：当前工作目录
- **暂存区(Staging Area)** ：下次会被提交的部分
- **本地仓库(Local Repository)**：本地代码仓库，维护本地已提交的部分
- **远程仓库本地镜像(Remote/Origin Repository)**：远程仓库的一份本地副本，用于跟踪远程
- **远程仓库(Remote Repository)**：远程代码仓库


<img src="https://img1.kiosk007.top/static/images/git/git_workfull.png">

改动操作使得文件处于不同状态，git status可以查看改动的文件所处于的状态
- **未跟踪状态(Untracked)**:  尚未被仓库管理的文件，即仓库里没有的文件
- **已修改状态(Modified)**：仓库里有，本地修改
- **已暂存状态(Staged)**:  仓库里有，本地修改并加入暂存，改动会被提交
- **未修改状态(Unmodified)**: 文件提交后由仓库管理，本地无修改


## 入门操作

- <font color='#EEB422'> 新建仓库 </font>

远程项目不存在时，本地初始化后上传

``` bash
cd <项目目录>
git init  //初始化，生成.git子目录
git remote add origin <项目git地址>
git add .  //当前目录下所有内容加入暂存区
git commit -m "Initial commit"
git push -u origin master
```

远程项目已存在时，直接从远程下载

``` bash
git clone <项目git地址>
cd <项目目录>
touch README.md
git add README.md
git commit -m "add README"  //参数m表示提交信息
git push -u origin master   //参数u表示关联远程分支
```

## 基本工作流

<img src="https://img1.kiosk007.top/static/images/git/git_base_workflow.png">

- <font color='#EEB422'> 添加和推送 </font>

``` bash
1. 使用如下命令：git add <filename> 或者 git add *
2. 使用如下命令以实际提交改动：git commit -m "代码提交信息"
3. 现在，你的改动已经提交到了 HEAD，但是还没到你的远端仓库。
```

- <font color='#EEB422'> 添加和推送 </font>

``` bash
1. 通过git fetch把远程改动拉取到本地
2. 将远程修改与本地融合merge/rebase
- 基于merge: 合并远程和本地改动
  - git pull等价于git fetch+git merge
- 基于rebase: 以远程为基准，将本地修改操作重放
  - git pull --rebase等价于git fetch+git rebase
```


## git 分支

分支是用来将特性开发绝缘开来的。在你创建仓库的时候，master 是“默认的”分支。在其他分支上进行开发，完成后再将它们合并到主分支上。

<img src='https://img1.kiosk007.top/static/images/git/branches.png' >

``` bash
- 创建一个叫做“feature_x”的分支，并切换过去： git checkout -b feature_x
- 切换回主分支： git checkout master
- 删除分支： git branch -d feature_x
- 将自己的分支推送到远端： git push origin <branch>
- 删除远程分支：git push origin --delete <分支名> 
- 列出本地分支：git branch
- 列出远程分支：git branch --remotes

```

## git 命令

- <font color='#EEB422'> 删除文件 </font>

``` bash
- 本地删除文件，删除动作不加入暂存区，此时提交库内文件不会被删除：rm <文件>
- 本地删除文件，同时把删除动作加入暂存区: git rm，相当于rm <文件>+ git add <文件>
- 希望删除仓库文件，但是本地文件依然保留，成为未跟踪状态：git rm --cached <文件>
```

- <font color='#FF8247'>**基本命令**</font>

``` bash
# 重命名文件
$ git mv old_filename new_filename

# 删除文件
$ git rm filename

# 查看工作区的状态
$ git status

# 为软件发布创建标签, 如v1.0.0
$ git tag v1.0.0 1b2e1d63ff


##---------- 查看git提交后日志
$ git log
$ git log --oneline     # 一行显示
$ git log -n 4          # 查看最近的几次
$ git log --all --graph # 以图形化方式显示所有的提交日志
$ git log --graph --oneline --decorate --all  # 通过 ASCII 艺术的树形结构来展示所有的分支, 每个分支都标示了他的名字和标签 
$ git log --author=bob  # 只看某一个人的提交记录

##---------- 查看 diff
$ git diff              # 对比工作区和暂存区的差异
$ git diff --cached  	# 对比暂存区和HEAD所含文件(commit)的差异
$ git diff -- style.css # 只查看对某个文件的 diff 差异 (工作区和暂存区)
$ git diff tmp master -- <file> # 比较 tmp 和 master 分支的文件差异(也可以将branch改成commit)

##---------- 取消提交
$ git reset HEAD        # 取消暂存 （取消 git add）
$ git restore --staged <file>  # 功能同上, 取消暂存 （可以 git status 查看当前的暂存状态）

$ git checkout -- <file>      # 在工作区的修改撤销到最近一次git add 或 git commit时的内容
$ git reset --hard c04b46549  # 恢复到历史上的某个 commit (工作区和暂存区都会清空)

```

- <font color='#FF8247'> 版本管理 </font>

``` bash
# 查看本地分支
$ git branch -av

# 基于某个历史版本创建分支(基于 ac886ae 创建一个tmp分支)
$ git checkout -b tmp ac886ae 

# 删除某个分支
$ git checkout -d ac886ae  # 没有merge的分支不能通过 -d 删除
$ git checkout -D ac886ae


###################### commit 相关 ###########################

# 修改最近一次提交的 commit 信息
$ git commit --amend

# 修改前几次提交的 commit 信息
$ git rebase -i ac224ct   # 交互式操作，ac224ct 是要修改的commit的父commit
将pick改为r  :wq退出，
变更内容     :wq退出

```
## git stash

`git stash` 命令用于暂时保存没有提交的工作。运行该命令后，所有没有commit的代码，都会暂时从工作区移除，回到上次commit时的状态。

它处于 `git reset --hard`（完全放弃还修改了一半的代码）与 `git commit`（提交代码）命令之间，很类似于“暂停”按钮。

``` bash
# 暂时保存没有提交的工作
$ git stash
Saved working directory and index state WIP on workbranch: 56cd5d4 Revert "update old files"
HEAD is now at 56cd5d4 Revert "update old files"
# 列出所有暂时保存的工作
$ git stash list
stash@{0}: WIP on workbranch: 56cd5d4 Revert "update old files"
stash@{1}: WIP on project1: 1dd87ea commit "fix typos and grammar"
# 恢复某个暂时保存的工作
$ git stash apply stash@{1}
# 恢复最近一次stash的文件
$ git stash pop
# 丢弃最近一次stash的文件
$ git stash drop
# 删除所有的stash
$ git stash clear
```



# git flow
参考：[git-flow 的工作流程](https://www.git-tower.com/learn/git/ebook/cn/command-line/advanced-topics/git-flow/)

git-flow 模式会预设两个主分支在仓库中：

**master** 只能用来包括产品代码。你不能直接工作在这个 master 分支上，而是在其他指定的，独立的特性分支中（这方面我们会马上谈到）。不直接提交改动到 master 分支上也是很多工作流程的一个共同的规则。

**develop** 是你进行任何新的开发的基础分支。当你开始一个新的功能分支时，它将是_开发_的基础。另外，该分支也汇集所有已经完成的功能，并等待被整合到 master 分支中。


## 开始新功能

创建一个新的feature 分支
``` bash
$ git flow feature start rss-feed
Switched to a new branch 'feature/rss-feed'

Summary of actions:
- A new branch 'feature/rss-feed' was created, based on 'develop'
- You are now on branch 'feature/rss-feed'
```

## 完成一个功能

我们的新功能终于完成。使用 “feature finish” 命令会把我们的工作整合到主 “develop” 分支中去。在这里它需要等待：

一个在更广泛的 “开发” 背景下的全面测试。
稍后和所有积攒在 “develop” 分支中的其它功能一起进行发布。
之后，git-flow 也会进行清理操作。它会删除这个当下已经完成的功能分支，并且换到 “develop” 分支。

``` bash
$ git flow feature finish rss-feed
Switched to branch 'develop'
Updating 6bcf266..41748ad
Fast-forward
    feed.xml | 0
    1 file changed, 0 insertions(+), 0 deletions(-)
    create mode 100644 feed.xml
Deleted branch feature/rss-feed (was 41748ad).
```
## 常见的git flow 开发流程

1. 创建develop分支
``` bash
git branch develop
git push -u origin develop
```

2. 开始新Feature开发

``` bash
git checkout -b some-feature develop
# Optionally, push branch to origin:
git push -u origin some-feature    

# 做一些改动    
git status
git add some-file
git commit
```

3. 完成Feature
``` bash
git pull origin develop
git checkout develop
git merge --no-ff some-feature
git push origin develop

git branch -d some-feature

# If you pushed branch to origin:
git push origin --delete some-feature    
```

4. 开始Relase

``` bash
git checkout -b release-0.1.0 develop

# Optional: Bump version number, commit
# Prepare release, commit
e. 完成Release

git checkout master
git merge --no-ff release-0.1.0
git push

git checkout develop
git merge --no-ff release-0.1.0
git push

git branch -d release-0.1.0

# If you pushed branch to origin:
git push origin --delete release-0.1.0   


git tag -a v0.1.0 master
git push --tags
```

5. 开始Hotfix

``` bash
git checkout -b hotfix-0.1.1 master    
g. 完成Hotfix

git checkout master
git merge --no-ff hotfix-0.1.1
git push


git checkout develop
git merge --no-ff hotfix-0.1.1
git push

git branch -d hotfix-0.1.1

git tag -a v0.1.1 master
git push --tags
```




# git常见使用

## <font color='#FF4500'>**分离头指针**</font>

通常，我们工作在某一个分支上，比如 master 分支。这个时候 master 指针和 HEAD 指针是一起前进的，每做一次提交，这两个指针就会一起向前挪一步。但是在某种情况下（例如 checkout 了某个具体的 commit），master 指针 和 HEAD 指针这种「绑定」的状态就被打破了，变成了分离头指针状态。

``` bash
➜ git checkout fb7d808
注意：正在切换到 'fb7d808'。

您正处于分离头指针状态。您可以查看、做试验性的修改及提交，并且您可以在切换
回一个分支时，丢弃在此状态下所做的提交而不对分支造成影响。

如果您想要通过创建分支来保留在此状态下所做的提交，您可以通过在 switch 命令
中添加参数 -c 来实现（现在或稍后）。例如：

  git switch -c <新分支名>

或者撤销此操作：

  git switch -

通过将配置变量 advice.detachedHead 设置为 false 来关闭此建议

HEAD 目前位于 fb7d808 Learn CSS demo

```

git 在对于这种没有 branch 的变更会被清除掉。所以如果想要变更最好跟着分支进行变更。

## git stash 命令实用指南

为什么 git stash 很重要，假设Git没有暂存变更命令，当你在有2个分支（A和B）的仓库上工作时，假设这两个分支已经分叉很长时间，并且有不同的头，当你处理A的时候，团队要求修复B分支的一个错误，你迅速将你的修改保存到A分支(但没有提交 commit),并且尝试用 `git checkout B` 来切到B分支，git 会立即终止这个操作，并报错。“你对以下文件的本地修改会被签出覆盖... ...请在切换分之前提交你的修改或者将他们暂存起来”

在这种情况下有几种方法来分支切换。
- 在分支A中创建一个提交（git switch -c），提交并推送你的修改，以修复 B 中的错误。然后签出A，并运行 `git reset HEAD`, 来恢复修改。
- 手动保留不被Git 追踪文件中的改动。

第二种就不说了，一点也不极客。第一种方法虽然看起来很传统，但是不灵活，因为保存未完成工作的修改会被当做一个检查点，而不是一个仍在进行中的补丁。这就是 `git stash` 的场景。

`git stash` 将未提交的改动保存在本地，让你可以进行修改，切换分支及其他的操作。然后当你需要的时候，你可以重新应用这些存储的改动。暂存是本地范围的，不会被`git push` 推送的远端。

以下是一次 `git stash` 的操作顺序
1. 将修改保存到分支A
2. 运行 `git stash`
3. 签出分支B
4. 修正B分支的错误
5. 提交并推送到远程
6. 查看分支A
7. 运行 `git stash pop` 来取回暂存的改动。

``` bash
# 现将工作区的内容保存到暂存区
$ git add *
# 将暂存区的内容 暂时保存
$ git stash   
$ git stash save "message"   # 功能同上

# 切到历史版本修复bug ... 

$ git stash apply  # 将临时保存区的内容恢复，但不会删除记录，pop会删除。
$ git stash pop 

```



## <font color='#FF4500'> cherry pick </font>

假设你在一个分支上已经做了很多次提交，但你意识到这个分支是错误的，该怎么办。
要么切换到正确的分支重复所有的变更。然后重新提交。要么呢就要用到 `cherry pick` 这个工具。

`git cherry-pick` 可以将相同的commit提交复制到另一个分支上。就没有必要在不同的分支上做相同的操作。
> 注意：`cherry-pick` 出来的提交会在另一个分支中创建带有新hash的提交，

- 它是如何工作的

假设有2个分支，`toC` 和 `toB`, 现在有个bug在2个版本上都存在。我们在`toC` 分支上已经修复了这个bug。在 `toC`分支上运行`git log`命令，获取这次提交的 hash 值, 简单起见复制 25560 即可
```bash
$ git log
commit 255604842840febb7e11bbb443013fa584e76219 (HEAD -> master, tag: v1.0.1, origin/master, origin/HEAD)
Author: igolaizola <11333576+igolaizola@users.noreply.github.com>
Date:   Thu Sep 26 09:31:17 2019 +0200
```
然后切换到`toB`分支上，将刚刚的bugfix提交合入。
```bash
$ git checkout toB
$ git cherry-pick 25560
```

- 如果遇到了 “nothing to commit,working tree clean The previous cherry-pick is now empty,possibly due to conflict resolution” 不要惊慌，按照建议运行 `git commit --allow-empty` 即可。这个将打开你的编辑器，编辑提交信息即可。

- 如果遇到了合并重提，解决冲突后，输入 `git cherry-pick --continue` 恢复。

参考: https://mp.weixin.qq.com/s/J7sVxIoIVClEirClBQoUtA
参考：https://zhuanlan.zhihu.com/p/156726632


## <font color='#FF4500'>**指定文件不需要git管理**</font>

在项目根目录下创建 **`.gitignore`** ,写入不需管理的文件名即可

## <font color='#FF4500'>**`Git创建本地分支并关联远程分支`**</font>

当我想从远程仓库里拉取一条本地不存在的分支时：

``` bash
$ git checkout -b 本地分支名 origin/远程分支名
```
如果出现以下报错
```
fatal: Cannot update paths and switch to branch 'dev2' at the same time.
Did you intend to checkout 'origin/dev2' which can not be resolved as commit?
```
表示拉取不成功。我们需要先执行
``` bash
$ git fetch
```

再执行

``` bash
$ git checkout -b 本地分支名 origin/远程分支名
```

修改之后再推送的话，可以执行
```bash
$ git push --set-upstream origin 远程分支名
```


# git 备份

<img src="https://img1.kiosk007.top/static/images/git/git_transport.webp" style="height:400px" >

前两个协议是本地协议，后两个是远端托管平台。
哑协议传输进度不可见，智能协议传输可见。

``` bash
# 将一个文件仓库 clone
# 哑协议
$ git clone --bare /home/kiosk/Project/Git/kiosk_demo/.git

# 智能协议
$ git clone --bare file:///home/kiosk/Project/Git/kiosk_demo/.git
```


- <font color='#FF8C00'>**创建远程**</font>

可以使用 git remote add 命令将远程 URL 与名称匹配，例如 
**`git remote add origin  <REMOTE_URL> `** 
这会将名称 origin 与 REMOTE_URL 关联。

``` bash
$ git remote add origin https://github.com/user/repo.git
# Set a new remote

$ git remote -v
# Verify new remote
> origin  https://github.com/user/repo.git (fetch)
> origin  https://github.com/user/repo.git (push)
```
远程 name 已存在,此错误消息表示您尝试添加的远程与本地仓库中的远程名称相同。可以尝试修改远程名。

``` bash
$ git remote -v
# 查看现有远程
> origin  https://github.com/OWNER/REPOSITORY.git (fetch)
> origin  https://github.com/OWNER/REPOSITORY.git (push)

$ git remote rename origin destination
# 将远程名称从 'origin' 更改为 'destination'

$ git remote -v
# 验证远程的新名称
> destination  https://github.com/OWNER/REPOSITORY.git (fetch)
> destination  https://github.com/OWNER/REPOSITORY.git (push)
```

若远程的url发生了变化，可以通过 
**`git remote set-url origin https://github.com/USERNAME/REPOSITORY.git `** 
修改
