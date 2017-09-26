+++
title = "git order"
date = "2017-04-03T17:06:19+08:00"
draft = false
categories = [
  "tool",
]
tags = [
  "git",
]
keywords = [
  "git",
]
#thumbnailImagePosition: left
#thumbnailImage: git_order.png

+++

# 常用命令
| git orde name | meaning  |    tip  |
| :-----------:| :-------: | :---: |
|`git push origin branch_name`|push本地分支到远程仓库||
|`git stash apply`|pull 完毕后出栈||
|`git stash`|入栈，进入暂存区||
|`git pull origin master --allow-unrelated-histories`|强制pull||
|`git rm -r --cached .DS_Store`|如果在刚开始没加gitignore文件，后来加本地会有缓存，要删除||
|`git checkout --track origin/serverfix`|是下面命令的简写||
|`git checkout -b serverfix origin/serverfix`|从远程分支检出一份本地分支，并切换到该分支||
<!--more-->
|`git fetch branch_name`|拉一个远程分支||
|`git fetch orgin`|同步远程仓库内容到本地||
|`git branch branch_name`|创建本地分支||
|`git chekout -b branch_name`|创建本地分支并切换到该分支||
|`git checkout -b branch_name origin/branch_name`|拉远程分支到本地||
|`git push origin TagName`|上传tag||
|`git tag -d TadName`|删除本地tag||
|`git push origin --delete tag TagName`|删除远程tag||
|`git push origin :branch_name`|删除远程分支|“:”前面的空格不能少，原理是push一个空分支到server，相当于删除|
|`git branch -D branch_name`|删除本地分支||
|`vim ~/.gitconfig`|查看git账户信息||
|`git log`|查看`log`|翻页`space`退出`q`|
| `git reset --hard`| 回到前一次`commint`之前的状态 | 一定要记得先`commit` |
| `git checkout -b develop master`|创建develop分支||
|`git checkout master`|切换分支到`master`分支||
| `git merge --no-ff develop`|当前在master分之上，将develop分支上的内容合并到master分支||
|`git add -all`|表示保存所有变化（包括新建、修改和删除）|也可以用git add .来代替|
|`git status`|查看变动的文件||
|`git rebase -i origin/master`|合并多个commit||
|`git push --force origin myfeature`|合并到远程仓库|如果`rebase`了一定要用force参数。因为分支历史改变了，跟远程分支不一定兼容，有可能要强行推送|
|`git checkout -- xxxxx`|忽略本地未加入暂存队列的文件的修改(该文件相当于被discard)|如果已加入到了暂存列，先执行`git reset HEAD -- xxxxx`，恢复所有`git checkout .`|
|`git reset HEAD -- xxxxx`|取出已存入暂存队列的文件||
|`git reset --hard xxxx`|恢复到指定版本|`git log`找到要恢复的版本|
|进入文件目录，`git log xxx(文件名)`,`git  reset xxxx`(版本号) `xxxx`(文件名),`git checkout xxxx`(文件名)|恢复单个文件的历史版本||
|`git diff`|查看文件冲突|查看指定文件`git diff -w xxxx`|
|`git status -s`|查看文件冲突状态||
|`git show xxxx(版本号)`|查看已经commit的内容 ||
|`git revert HEAD`| 撤销最近一次的提交 |撤销上上次的提交`git revert HEAD^`|

{{< alert warning >}}
--no-ff: 不采用git默认的快进式合并，而是用正常合并
{{< /alert >}}
<!-- <pre>--no-ff: *不采用git默认的快进式合并，而是用正常合并*</pre> -->


# 临时分支
| git orde name | meaning  |    tip  |
| :-----------:| :-------: | :---: |
|`feature`分支|**功能分支**| 为了开发某种功能，从`develop`上分出来。开发完成后，再并入`develop`|
|`git checkout -b feature-`x `develop`|创建功能分支||
|<p>`git checkout develop` </p> `git merge --no-ff feature-`x|先切换到`develop`分支，再把`feature`分支合并到`develop`|x是参数|
|`git branch -d feature-`x|删除feature分支|x是参数|
|`release`分支|**预发布分支**|预发布分支，它是指发布正式版本之前（即合并到`Master`分支之前），我们可能需要有一个预发布的版本进行测试|
|`git checkout -b release-`1.2 `develop`|创建预发布分支|1.2是参数|
|<p>`git checkout master`</p> `git merge --no-ff release-`1.2|合并到master分支|1.2是参数|
|<p>`git checkout develop`</p> `git merge --no-ff release-`1.2|再合并到develop分支|1.2是参数|
|`git branch -d release-`1.2|删除预发布分支|1.2是参数|
|`bug`分支|**bug分支**|软件正式发布以后，难免会出现bug。这时就需要创建一个分支，进行bug修补。**修补bug分支是从Master分支上面分出来的。修补结束以后，再合并进Master和Develop分支。它的命名，可以采用fixbug-*的形式**|
|`git checkout -b fixbug-`0.1 `master`|创建bug分支|0.1是参数|
|<p>`git checkout master` </p> `git merge --no-ff fixbug-`0.1|合并到`master`分支|<font color=red>0.1是参数</font>|
|<p>`git checkout develop`</p> `git merge --no-ff fixbug-`0.1|合并到develop分支|0.1是参数|
|`git branch -d fixbug-`0.1|删除bug分支|0.1是参数|
|`ssh -T git@121.40.53.85`|打印出当前用户|121.40.53.85是仓库地址|
# 上传项目到Git远端仓库
> - 首先创建一个远程仓库，得到远程仓库地址
> - 进入本地目录 `git init`初始化git
> - `git add .`添加当前目录下的所有文件和文件夹
> - 添加提交信息`git commit -m "first commit"`，这时候并未真正的提交到服务器
> - `git remote add origin https://github.com/xxxx/xxxx.git`添加远端仓库，`origin`是默认仓库的名称。
> - 提交到远端仓库`master`分支`git push -u origin master`，`-u`表示以后提交都是默认到`origin`的`master`分支上
> - 之后提交顺序为<br> 1 `git add .`<br> 2 `git commit -m 'xxxxx'` <br> 3 `git push`<br>


# 配置SSH
> - 查看是否存在公钥：`ls -al ~/.ssh`
> - 生成密钥：`ssh-keygen -t rsa -C "your_email@example.com"`，一路回车键
> - `eval "$(ssh-agent -s)"` `ssh-add ~/.ssh/id_rsa`
> - 复制公钥：`pbcopy < ~/.ssh/id_rsa.pub`
> 参考 [github](https://help.github.com/articles/generating-ssh-keys/#platform-mac)

# 清楚git信息
>find . -name ".git" | xargs rm -Rf
#git flow
>
>

_____
参考:
- [官方文档](http://git-scm.com/book/zh/v1/%E8%B5%B7%E6%AD%A5)
- [进阶](http://www.techug.com/10-tips-git-next-level)
- [简书](http://www.jianshu.com/p/63f9ecc3540c?utm_campaign=hugo&utm_medium=reader_share&utm_content=note&utm_source=weibo)
- [Git使用规范流程](http://www.ruanyifeng.com/blog/2015/08/git-use-process.html)
- [理解暂存区域](http://www.worldhello.net/2010/11/30/2166.html)
- [git reset 和 git revert](http://gitbook.liuhui998.com/4_9.html)
- [get reset、git checkout、git revert](https://segmentfault.com/a/1190000003102737)








 








 

