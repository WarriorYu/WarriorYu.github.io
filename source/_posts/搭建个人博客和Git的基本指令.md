---
title: 搭建个人博客和Git的基本指令
date: 2018-09-17 18:57:10
categories: 
- 博客搭建
tags: 
- 个人博客/Git
---



### Create a new post

```
$ hexo new "My New Post"
```

### 清除缓存

$ hexo clean 

### Generate static files

```
$ hexo generate
```

More info: [Generating](https://hexo.io/docs/generating.html)

### Deploy to remote sites

```
$ hexo deploy
```

More info: [Deployment](https://hexo.io/docs/deployment.html)



### Git 的使用

### Git 文件状态介绍

- **已修改（modified）** ———— 表示修改了文件，但还没保存到数据库中
- **已暂存（staged）** ———— 表示对一个已修改文件的当前版本做了追踪，使之包含在下次提交的快照中
- **已提交（committed）**———— 表示数据已经安全的保存在本地数据库中

### Git 常用命令速查表

**创建版本库**

```
$ git clone <url>                  #克隆远程版本库
$ git init                         #初始化本地版本库
```

**修改和提交**

```
$ git status                       #查看状态
$ git diff                         #查看变更内容
$ git add .                        #跟踪所有改动过的文件
$ git add <file>                   #跟踪指定的文件
$ git mv <old><new>                #文件改名
$ git rm<file>                     #删除文件
$ git rm --cached<file>            #停止跟踪文件但不删除
$ git commit -m "commit messages"  #提交所有更新过的文件
$ git commit --amend               #修改最后一次改动
```

**查看提交历史**

```
$ git log                    #查看提交历史
$ git log -p <file>          #查看指定文件的提交历史
$ git blame <file>           #以列表方式查看指定文件的提交历史
```

**撤销**

```
$ git reset --hard HEAD      #撤销工作目录中所有未提交文件的修改内容
$ git checkout HEAD <file>   #撤销指定的未提交文件的修改内容
$ git revert <commit>        #撤销指定的提交
$ git log --before="1 days"  #退回到之前1天的版本
```

**分支与标签**

```
$ git branch                   #显示所有本地分支
$ git checkout <branch/tag>    #切换到指定分支和标签
$ git branch <new-branch>      #创建新分支
$ git branch -d <branch>       #删除本地分支
$ git tag                      #列出所有本地标签
$ git tag <tagname>            #基于最新提交创建标签
$ git tag -d <tagname>         #删除标签
```

**合并与衍合**

```
$ git merge <branch>        #合并指定分支到当前分支
$ git rebase <branch>       #衍合指定分支到当前分支
```

**远程操作**

```
$ git remote -v                   #查看远程版本库信息
$ git remote show <remote>        #查看指定远程版本库信息
$ git remote add <remote> <url>   #添加远程版本库
$ git fetch <remote>              #从远程库获取代码
$ git pull <remote> <branch>      #下载代码及快速合并
$ git push <remote> <branch>      #上传代码及快速合并
$ git push <remote> :<branch/tag-name>  #删除远程分支或标签
$ git push --tags                       #上传所有标签
```

推送代码到远程仓库

在终端运行命令

```
git push
```

，将文件推送到远程仓库：

```
$ git push origin master
Counting objects: 8, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (4/4), done.
Writing objects: 100% (8/8), 626 bytes | 626.00 KiB/s, done.
Total 8 (delta 0), reused 0 (delta 0)
To https://git.coding.net/Yangconghou/learn_git.git
 * [new branch]      master -> master
```

`git push`是推送命令，实际上是把本地的`master`分支推送到了远程仓库，相当于在远程有了一个代码仓库的备份。

> 使用 Git 管理文件时，每次结束工作前请依次执行`git add`、`git commit`和`git push`命令将文件推送到 远程仓库。



1、查看本地分支

```
$ git branch
```

```
结果：
➜  WarriorYu.github.io git:(code) ✗ git branch
* code
  master
```

2、  list both remote-tracking and local branches (查看所有分支)

```
$ git branch -a 
```

```
结果：
➜  WarriorYu.github.io git:(code) ✗ git branch -a 
* code
  master
  remotes/origin/code
  remotes/origin/master
```

3、我想看一下暂存区（staging area）里的内容

```
$ git ls-files
```

结果：

```
➜  _posts git:(code) ✗ git ls-files 
The Start.md
test.md
```

```
$ git ls-files --stage/-s   注：show staged contents' object name in the output
```

结果：

```
➜  _posts git:(code) ✗ git ls-files --stage
100644 a06e35bb0bbcc29593483ce13b37f9b4d28cd585 0	The Start.md
100644 a06e35bb0bbcc29593483ce13b37f9b4d28cd585 0	test.md
```

4、在我的_post文件夹里_新建一个test.md文件，并推到远程仓库

```
$ git add "test.md"
```

先拉取最新的代码

```
git pull origin code 
```

结果；

```
➜  _posts git:(code) ✗ git pull origin code 
From https://github.com/WarriorYu/WarriorYu.github.io
 * branch            code       -> FETCH_HEAD
Already up-to-date.
```



提交：-m后面添加提交的备注

```
git commit -m "add test.md"    
```



```
➜  _posts git:(code) ✗ git commit -m "add test.md"
[code 9145a31] add test.md
 1 file changed, 10 insertions(+)
 create mode 100644 source/_posts/test.md
```

推送到远程仓库：

```
$ git push origin code
```
结果：
```
➜  _posts git:(code) ✗ git push origin code
Everything up-to-date
```

### 参考文章

[Android ABI的浅析](https://www.jianshu.com/p/d2119b3880d8)