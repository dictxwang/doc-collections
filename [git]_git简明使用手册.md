# git简明使用手册

介绍git的常用命令和常见问题的处理方法

### 一、git的常用命令

#### 1、配置

###### 查看配置
git config -l  /  git config --list
###### 编辑配置
git config -e  # 需要确保当前路径是git目录

git config -e --global  # 编辑全局配置，在任意路径都可以执行

###### 编辑提交时的用户信息
git config user.name "dictxwang"  # 编辑当前git库的信息

git config user.emal "xxx@163.com"

git config --global user.name "wangqiang" # 编辑全局的信息

git config --global user.email "yyy@gmail.com"

###### 关闭命令提示中对中文路径或文件名的转义

git config --global core.quotepath false  # 默认打开转义，将显示类似这样的提示： rename "git\345\256\242\346\210\267.md"...

#### 2、新建代码仓库

###### 在当前路径新建代码库
git init
###### 新建目录，并将其初始化为代码库
git init dictxwang001
###### 下载项目和它整个代码历史
git clone  https://github.com/dictxwang/doc-collections

#### 3、文件增加与删除

###### 添加指定文件到暂存区
git add filex.log
###### 添加指定目录到暂存区，包括子目录
git add dirx.log
###### 添加当面目录的所有文件到暂存区
git add .
###### 添加文件是，需要逐个确认
git add -p
###### 删除工作区文件，并将删除放入暂存区
git rm filex.log
###### 修改文件名，并将修改放入暂存区
git mv filex.log filey.log

#### 4、提交操作

###### 提交暂存区到仓库区
git commit -m "this is commit mesage"
###### 提交暂存区指定文件到仓库区
git commit filex [filey] -m "this is commit message"
###### 提交工作区自上次commit之后的变化，直接到仓库区
git commit -a
###### 提交时显示所有的diff信息
git commit -v
###### 使用一次新的commit，替代上一次提交
###### 如果代码没有任何新变化，则用来改写上一次的commit的信息
git commit --amend -m "this is new commit message"

#### 5、远程同步

###### 下载远程仓库的所有变动
git fetch
###### 下载远程仓库的分支变动
git fetch -p
###### 显示所有远程仓库
git remote -v
###### 显示某个远程仓库的信息
git remote show origin
###### 增加一个新的远程仓库，并命名
git remote add dictxwang_test https://github.com/dictxwang/test001
###### 删除远程仓库
git remote rm dictxwang_test
###### 取回远程仓库的变化，与本地分支合并，包括branch和tag信息的变化
git pull [remote] [branch]
###### 上传本地指定分支到远程仓库
git push [remote] [branch]
###### 强行推送当前分支到远程仓库
git push [remote] --force

#### 6、 分支操作

###### 查看本地分支
git branch
###### 查看远程分支
git branch -r
###### 新建一个分支
git branch dictxwang002
###### 新建一个分支，并切换到新分支
git checkout -b dictxwang002
###### 将本地分支推送到远程仓库
git push origin dictxwang002:dictxwang002  # :dictxwang0208表示没有这个分支就新建
###### 切换到指定分支，并更新工作区
git checkout dictxwang002
###### 删除本地分支
git branch -d localBranchName
###### 删除远程分支
git push origin --delete remoteBranchName

#### 7、TAG操作

###### 列出所有的tag
git tag
###### 在当前分支新建一个tag
git tag tag_dictxwang_001
###### 基于指定分支新建一个tag
git tag tag_dictxwang_002 main
###### 删除本地tag
git tag -d tag_dictxwang_001
###### 推送本地tag到远程仓库
git push origin tag_dictxwang_002
###### 查看tag信息
git show tag_dictxwang_001
###### 新建一个分支，指向某个tag
git checkout -b dictxwang003 tag_dictxwang_001

#### 8、撤销操作

###### 撤销工作区的修改，即未执行add操作的修改
git checkout -- filex  # 撤销指定文件的修改

git checkout .  # 撤销所有的修改

###### 暂时将未提交的修改隐藏，便于直接切换分支
git stash
###### 将隐藏的修改重新载入
git stash pop
###### 重置到指定的版本，中间所有的改动和commit都会被覆盖掉
###### 相当于是直接修改头部版本指针
###### 尽量避免使用reset，因为会覆盖中间的版本信息
git reset --hard HEAD  # 回退到最新版本

git reset --hard CommitId  # 回退到指定的版本

###### 回滚到指定的版本，不会覆盖中间的commit
###### 相当于是将历史版本重新载入作为新的版本，中途可能需要处理冲突
git revert CommitId  # 发生冲突后，修改冲突文件，重新执行 git add filex，在执行 git revert --continue

#### 9、信息查看

###### 状态显示，如查看有变更的文件
git status
###### 显示当前分支的版本历史
git log
###### 显示版本历史，同时显示每次提交发生变更的文件
git log --stat
###### 按关键字查找提交记录
git log --grep dictxwang
###### 查看指定文件的变更记录
git log --follow filex.log

git whatchanged filey.log

###### 显示指定行数的提交记录
git log -10  # 显示最近10次提交

git log -10 --oneline  # 一条提交日志在一行显示

###### 显示所有提交过的用户，并按照提交次数排序
git shortlog -sn
###### 显示指定文件何人在何时修改过
git blame filex
###### 显示暂存区和工作区的差异
git diff
###### 显示当前工作区与当前分支最新提交的差异
git diff HEAD
###### 显示写了多少行代码，统计未提交的代码
git diff --shortstat "@{0 day ago}"
###### 显示某次提交的元数据和内容变化

git show [CommitId]

###### 对比暂存区和特定提交（不带commitId默认和HEAD对比）

git diff --staged [CommitId]

git diff --cached [CommitId]

###### 对比两次commitgit

diff commitId1 commitId2

###### 显示某次提交时，某个文件的内容
git show [CommitId]:filex.log  # 不指定CommitId时，默认显示最新的一次commit
###### 显示当前分支所有的操作记录，包括提交、回退的操作
git reflog

#### 10、其他操作

###### 打包操作
git archive HEAD --format=tar > dictxwang-1.0.0.tar  # 打包最新提交的版本

git archive HEAD --format=tar.gz -o dictxwang-1.0.0.tar.gz

git archive HEAD --format=tar --prefix=wang_ -o dictxwang-1.0.1.tar  # 给根路径下所有的文件和文件夹加上前缀

git archive CommitId --format=tar.gz dictxwang-1.0.2.tar  # 打包指定的版本

###### 合并提交操作
###### 通过rebase可以合并一下零散的提交，比如对同一个文件的多次调试修改
git rebase -i HEAD~3   # 合并最近三次提交，需要编辑合并脚本，其中s是合并

git rebase --edit-todo  # 异常退出vi窗口时，重新进入

git rebase --continue  # 异常退出时，修改完成后继续执行rebase

### 二、git的常见问题

#### 1、冲突的处理

*合并代码或者pull操作都可能产生冲突*

###### 合并代码
git merge main
###### pull操作
git pull -a 
###### 查看冲突的文件（任意一种皆可）
git ls-files -u  | cut -f 2 | sort -u

git ls-files -u  | awk '{print $4}' | sort | uniq

git diff --name-only --diff-filter=U

###### 处理冲突：编辑冲突文件->执行git add->重新commit即可
vim conflictFilex.log

git add .

git commit -m "处理冲突001"
