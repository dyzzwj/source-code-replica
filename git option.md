

暂存区

工作区

本地仓库

远程仓库



# 冲突

<<<<<<<到=======是在当前分支合并之前的文件内容 
=======到>>>>>>> psr/psr-02是在其它分支（psr/psr-02）下修改的内容 

需要在这个两个版本中选择一个，然后把标记符号也要一起删除



# 添加

git add .	将工作区所有文件添加到暂存区 

git add file1 file2 ...	将工作区指定文件添加到暂存区

git add [dir]	将工作区中某个文件夹中所有所有文件添加到暂存区

git commit -m 'msg'	将暂存区的文件提交到本地仓库中



git reset . 清空暂存区

git rm file1 file2 ...	删除工作区文件，并且也从暂存区也删除对应文件的记录

**git rm --cached file	从暂存区中删除文件，但是工作区依然还有该文件**



**git revert <commit>**	`revert`命令撤销指定的commit并且新建一个commit，新建commit的内容与指定commit的前一个commit内容保持一致

git revert 原理：根据你要回退的提交所做的改动做相反的改动，然后重新提交代码，使代码达到没有这些旧提交所能达到的状态。回退旧的提交必然会导致当前最新代码发生变化，比如之前某个提交加了一行代码，那么回退就是在相同位置减一行代码。Git不会真的把旧提交抛弃，如果直接抛弃，历史记录就追踪不到了。因此，旧提交其实是没动的，Git只是根据旧提交反着做了一遍，这才导致最新的代码发生了改变。只有把revert命令回退导致的改动重新提交，revert命令才算真的完成并生效，否则效果只相当于修改了工作树的文件而已。

使用git revert可能会产生的问题：

因为git revert是用新提交覆盖旧提交，因此，被覆盖的提交等于不会被采用了。如果两个分支（假设是master和A分支）先合并再用revert回滚，之后又合并（A合并到master），就会发现在master分支上，A分支第一次合并之前的修改大部分不见了。这是因为从时间的发生顺序来看，A分支第一次合并之前的修改发生在revert之前，revert发生在后，而 revert抛弃了A第一合并之前的修改，那么再合并Git就认为你永远抛弃了A第一次之前的修改。

要解决这个问题，需要把revert产生的提交再revert一次。

**git reset <commit>	丢弃掉某个提交之后的所有提交**

`git reset`<commit>的原理是，让最新提交的指针回到以前某个时点，该时点之后的提交都从历史中消失。

git reset	撤销上一次向暂存区添加的所有文件(默认不改变工作区中的该文件)

git reset HEAD file	撤销上一次向暂存区添加的某个指定文件，不影响工作区中的该文件

git reset --soft HEAD~3	将当期分支的指针倒退三个 commit，倒退指针的同时，不改变暂存区 
    soft: 不改变工作区和缓存区，只移动 HEAD 到指定 commit。
	mixed: 只改变缓存区，不改变工作区。这是默认参数，通常用于撤销git add。
	hard：改变工作区和暂存区到指定 commit。该参数等同于重置，可能会引起数据损失。git reset --hard等同于git reset --hard 				HEAD。

 **git checkout --filename**	将指定文件从暂存区或本地仓库恢复到工作区（优先使用暂存区恢复，暂存区没有再从本地仓库恢复）
  	//它的原理是先找暂存区，如果该文件有暂存的版本，则恢复该版本，否则恢复上一次提交的版本（本地仓库）。
  	//git checkout HEAD~ -- <filename> 指定从某个 commit 恢复指定文件.这会同时改变暂存区和工作区



# stash

 **git stash**	暂时保存所有未提交的修改。所有没有commit的代码，都会暂时从工作区移除，回到上次commit时的状态。

 git stash list	列出所有暂时保存的工作

 git stash apply stash@{1}	恢复某个暂时保存的工作

 **git stash pop**	恢复最近一次stash的文件
 git stash drop 丢弃最近一次stash的文件
 git stash clear	删除所有的stash

 	默认情况下，git stash会缓存下列文件：
		添加到暂存区的修改（staged changes）
  	  Git跟踪的但并未添加到暂存区的修改（unstaged changes）
    **不会缓存以下文件：**
  	  在工作目录中新的文件（untracked files）
  	  被忽略的文件（ignored files）



# diff

git diff	查看工作区与暂存区的差异

**git diff file.txt**	查看某个文件的工作区与暂存区的差异` `

**git diff --cached** 查看暂存区与当前commit的差异

git diff <commitBefore> <commitAfter>	查看两个commit的差异

git diff HEAD	查看工作区与上一次commit之间的差异

git diff <commit>	查看工作区与某个commit的差异

git diff topic master	查看topic分支与master分支最新提交之间的差异`



# 分支

**git branch <branch-name>**	基于当前分支创建分支

**git checkout <branch-name>**	从当前分支切换到其他分支

**git checkout -b <branch-name>**	新建并切换到新建分支 

git branch -d <branch-name>	删除分支

**git merge <branch-name>**	将当前分支与指定分支合并（指定分支合并（覆盖）当前分支）例：在master分支执行git merge dev 把dev合并到master

git merge *--abort #如果Git版本 >= 1.7.4*   取消某次合并

git cherry-pick <commit-id>	 将指定的提交应用于当前分支

git branch	显示本地仓库的所有分支

git branch -v	查看各个分支最后一个提交对象的信息

git branch --merged	查看哪些分支已经合并到当前分支

git merge <remote-name>/<branch-name>	把远程分支合并到当前分支 如`git merge origin/serverfix`

如果是单线的历史分支不存在任何需要解决的分歧，只是简单的将HEAD指针前移，所以这种合并过程可以称为快进（Fast forward），而如果是历史分支是分叉的，会以当前分叉的两个分支作为两个祖先，创建新的提交对象；如果在合并分支时，遇到合并冲突需要人工解决后，才能提交。

git checkout -b <branch-name>  <remote-name>/<branch-name>	在远程分支的基础上创建本地分支

git checkout -b <branch-name> dev	在本地分支dev的基础上创建本地分支



# 如何开启一个功能分支

创建功能分支处必须在基于master创建，git checkout master，在master分支上执行

gitflow feature start zwj-LIVE-439   会分别在本地和远程创建feature/zwj-LIVE-439分并关联本地分支和远程分支

原理：

1、git checkout master

2、git pull

3、git checkout -b feature/zwj-LIVE-439 master   基于master分支创建zwj-LIVE-439

4、git push --set-upstream origin feature/zwj-LIVE-439  创建远程分支并关联本地分支



# 如何合并功能分支到测试环境

在功能分支feature/zwj-LIVE-439上执行 gf f mergeto develop

1、git checkout feature/zwj-LIVE-439

2、git pull

3、git checkout develop

4、git pull

5、git checkout develop

6、git merge --no-ff feature/zwj-LIVE-439   //不使用fast-forward方式合并，保留分支的commit历史

7、git push

8、git checkout feature/zwj-LIVE-439



# 远程仓库、远程分支

**git remote add [remote-name] [url]**	添加远程仓库并给远程仓库取个别名

**git push**	将本地分支的更新，推送到远程主机

git push <remote-name> <branch-name>	将本地仓库指定分支推送到远程仓库。例子：git push origin master	将本地的master分支推送到origin主机的master分支

git push origin --delete dev	**删除远程分支dev**

git push <remote-name> <local-branch>:<remote-branch>	将本地指定分支推送到远程仓库指定分支（两分支不同名） 

git push -u origin master 如果当前分支与多个主机存在追踪关系，则可以使用`-u`选项指定一个默认主机，这样后面就可以不加任何参数使用`git push`。

git pull	**将远程仓库的更改合并到当前分支中**，默认模式下，git pull = git fetch + git merge HEAD        **可能会产生冲突**

git pull <远程主机名> <远程分支名>:<本地分支名>	将远程主机某个分支的更新，与本地的指定分支合并

git fetch <remote-name>	从远程仓库中抓取本地仓库中没有的更新（不会自动合并到当前工作分支），配合git merge进行合并

git push --set-upstream origin dev	当前分支关联远程仓库origin的dev远程分支







# 查看

git log	查看提交历史

git reflog	显示当前分支的最近几次提交

git status	查询当前工作区中所有文件的状态

