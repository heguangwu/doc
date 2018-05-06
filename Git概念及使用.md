# Git概念及使用

------

学习优达学城上《如何使用 Git 和 GitHub》的一些笔记。

基本概念及相关操作：

> * 仓库（repository，已经纳入版本管理）：clone（克隆远端仓库到本地），init（初始化本地目录为一个仓库）
> * 暂存区（staging area）：add（将工作区加入到暂存区），commit（将暂存区提交到仓库,查看commit历史信息：git log）
> * 工作区（working directory）：git status(查看工作区和暂存区修改)

## 常见命令：

### 0. 更清晰查看版本信息
```
git log --graph --oneline
```
### 1. 比较工作区和暂存区文件差异
```
git diff
```
### 2. 比较暂存区和仓库的文件差异
```
git diff --staged
```
### 3. 切换到特定版本或分支
```
git checkout 9ce2558cc087fe128ccfb7f0e7c29ab8297e35ab
git checkout master
```
### 4. 查看和父节点的差异
```
git show 9ce2558cc087fe128ccfb7f0e7c29ab8297e35ab
```
### 5. 分支操作
```
#查看分支
git branch

#创建dev分支
git branch dev

#合并分支(将master分支合并到dev)
git merge master dev

#删除分支
git branch -d dev
```
分支合并会存在冲突，当存在冲突时会提示，如下（合并master到dev2）提示world.txt存在冲突：
```
$ git merge master dev2
Auto-merging world.txt
CONFLICT (content): Merge conflict in world.txt
Automatic merge failed; fix conflicts and then commit the result.
```
实际上文件冲突当地方会合并同时出现在world.txt中，其中：>>>>HEAD 表示dev2的修改，====master表示master分支部分。需要手动修改这个文件。

## github相关操作
1. 在github上创建项目，记录项目地址，如git@gitee.com:hnjme/cloud-remote.git
2. 使用git init在本地创建git项目，并commit到本地
3. 查看将项目添加到远端git仓库(实际上只是建立关联，并不会提交代码)：
```
git remote add origin  git@gitee.com:hnjme/cloud-remote.git
```
其中origin上远端仓库标识。
4. 将本机的仓库(master)提交到远端(orgin)
```
git push origin  master
```
大多数情况下，我们可以推分支，即现在本地建立分支将分支推送到github
```
git branch dev
git checkout dev
git commit 
git push origin dev
```
5. 查看远端仓库情况
```
git remote  -v
输出如下：
origin	git@gitee.com:hnjme/cloud-remote.git (fetch)
origin	git@gitee.com:hnjme/cloud-remote.git (push)
```
5. 拉取远端修改到本地
```
git pull origin master
```
6. 和远端仓库的合并
```
git fetch origin master
git merge origin/master master
#远端是origin/master
```
fetch和pull的区别：pull相当于fetch && merge
