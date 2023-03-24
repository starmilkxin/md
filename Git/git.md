- [概念](#概念)
  - [四个工作区域](#四个工作区域)
  - [工作流程](#工作流程)
  - [文件的四种状态](#文件的四种状态)
- [常用命令](#常用命令)
  - [新建代码库](#新建代码库)
  - [远程仓库操作](#远程仓库操作)
  - [设置签名](#设置签名)
  - [查看文件状态](#查看文件状态)
  - [查看文件不同](#查看文件不同)
  - [提交与修改](#提交与修改)
  - [查看提交历史](#查看提交历史)
  - [拉取与推送](#拉取与推送)
  - [分支管理](#分支管理)

# 概念
## 四个工作区域
Git本地有四个工作区域：工作目录（Working Directory）、暂存区(Stage/Index)、资源库(Repository或Git Directory)、git仓库(Remote Directory)。文件在这四个区域之间的转换关系如下：
+ Workspace： 工作区，就是你平时存放项目代码的地方
+ Index / Stage： 暂存区，用于临时存放你的改动，事实上它只是一个文件，保存即将提交到文件列表信息
+ Repository： 仓库区（或版本库），就是安全存放数据的位置，这里面有你提交到所有版本的数据。其中HEAD指向最新放入仓库的版本
+ Remote： 远程仓库，托管代码的服务器，可以简单的认为是你项目组中的一台电脑用于远程数据交换

![6c0c8c086c0fe78c0ad70d011d1d6cc8.png](en-resource://database/1803:1)

## 工作流程
git的工作流程一般是这样的：
１. 在工作目录中添加、修改文件；
２. 将需要进行版本管理的文件放入暂存区域；
３. 将暂存区域的文件提交到git仓库。

因此，git管理的文件有三种状态：已修改（modified）,已暂存（staged）,已提交(committed)
![2ea174b5fd2a047aa7d125cc79df6c56.png](en-resource://database/1805:1)

## 文件的四种状态
版本控制就是对文件的版本控制，要对文件进行修改、提交等操作，首先要知道文件当前在什么状态，不然可能会提交了现在还不想提交的文件，或者要提交的文件没提交上。

GIT不关心文件两个版本之间的具体差别，而是关心文件的整体是否有改变，若文件被改变，在添加提交时就生成文件新版本的快照，而判断文件整体是否改变的方法就是用SHA-1算法计算文件的校验和。
![2df55c9f07da476e025d66c1934d154c.png](en-resource://database/1807:1)

+ Untracked:   未跟踪, 此文件在文件夹中, 但并没有加入到git库, 不参与版本控制. 通过git add 状态变为Staged.
+ Unmodify:   文件已经入库, 未修改, 即版本库中的文件快照内容与文件夹中完全一致. 这种类型的文件有两种去处, 如果它被修改, 而变为Modified. —— 如果使用git rm移出版本库, 则成为Untracked文件
+ Modified: 文件已修改, 仅仅是修改, 并没有进行其他的操作. 这个文件也有两个去处, 通过git add可进入暂存staged状态.——使用git checkout 则丢弃修改过,返回到unmodify状态, 这个git checkout即从库中取出文件, 覆盖当前修改
+ Staged: 暂存状态. 执行git commit则将修改同步到库中, 这时库中的文件和本地文件又变为一致, 文件为Unmodify状态. ——执行git reset HEAD filename取消暂存,文件状态为Modified

# 常用命令
## 新建代码库

```bash
# 在当前目录新建一个Git代码库 
git init

# 新建一个目录，将其初始化为Git代码库
git init [project-name]

# 下载一个项目和它的整个代码历史
git clone [url]
```

## 远程仓库操作

```bash
#显示所有远程仓库
git remote -v

#显示某个远程仓库的信息
git remote show [remote] 

#添加远程版本库 
git remote add [shortname] [url]

# 删除远程仓库
git remote rm name

# 修改仓库名
git remote rename old_name new_name  
```

## 设置签名

```bash
# 项目级别/仓库级别：仅在当前本地库范围内有效
#信息保存位置：./.git/config 文件
git config user.name tom_pro
git config user.email goodMorning_pro@atguigu.com

#系统用户级别：登录当前操作系统的用户范围
# 信息保存位置：~/.gitconfig 文件
git config --global user.name tom_glb
git config --global goodMorning_pro@atguigu.com

#二者都有时采用项目级别 的签名
```

## 查看文件状态

```bash
#查看指定文件状态
git status [filename]

#查看所有文件状态
git status
```

## 查看文件不同

```bash
#尚未缓存的改动：
git diff

#查看已缓存的改动：
git diff --cached

#查看已缓存的与未缓存的所有改动：
git diff HEAD

#显示摘要而非整个 
git diff --stat
```

## 提交与修改

```bash
# 添加指定文件到暂存区
git add [file1] [file2] ...
# 添加指定目录到暂存区，包括子目录
git add [dir]
# 添加当前目录的所有文件到暂存区
git add .

#提交暂存区到本地仓库中
git commit -m [message]
git commit [file1] [file2] ... -m [message]
#-a 参数设置修改文件后不需要执行 git add 命令，直接来提交
git commit -a

#回退版本
git reset [--soft | --mixed | --hard] [HEAD] #--mixed 为默认
#一个^表示后退一步，n 个表示后退 n 步
#~n 表示回退n步

#删除文件
#将文件从暂存区和工作区中删除
git rm <file>
#强行从暂存区和工作区中删除修改后的文件
git rm -f <file>
#从暂存区域移除，但仍然希望保留在当前工作目录中
git rm --cached <file>
```

## 查看提交历史

```bash
git log #空格向下翻页 b 向上翻页 q 退出

#简洁版本
git log --oneline

#查看历史中什么时候出现了分支、合并。以下为相同的命令，开启了拓扑图选项
--graph

#逆向显示所有日志
--reverse

#查找指定用户的提交日志
git log --author=Linus

#查看指定文件的修改记录
git blame <file>
```

## 拉取与推送

```bash
git pull <远程主机名> <远程分支名>:<本地分支名>

git push <远程主机名> <本地分支名>:<远程分支名>

#如果本地版本与远程版本有差异，但又要强制推送可以使用 --force 参数
git push --force origin master
```

## 分支管理

```bash
#列出分支基本命令
git branch

#手动创建一个分支
git branch (branchname)

#切换分支
git checkout (branch)

#删除分支
git branch -d (branchname)

#分支合并
git merge (branchname)
```