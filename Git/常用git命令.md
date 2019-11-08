### 参数

1. `-` 后面紧跟短字母选项。    

   如`git checkout -b v1` 其中b是选项  v1是参数

2. `-- `后面紧跟长字母选项。  

   如`git rm --cached README.md` 其中cached是选项  README.md是参数 

3. `--` 后面没有紧跟字母，说明选项结束，接下来的是参数。

   如`git checkout -- README.md` 其中README.md是参数，没有选项

### 初始化

```bash
# 生成ssh key, -t 密钥算法  -C 注释
ssh-keygen -t rsa -C "wangtao970503@gmail.com"

# 设置commit时用户名以及邮箱
git config --global user.name "wangtao"
git config --global user.email "wangtao970503@gmail.com"
git config --list # 列出所有配置
```

### 分支

```bash
git branch                                                  # 查看本地分支
git branch -r                                               # 查看远程分支
git branch <name>                          					# 创建分支
git checkout -b <name>                     					# 创建分支并切换到此分支
git checkout -b <name> <exist_branch> | <commit_id>			# 基于一个分支或者提交来创建分支
git branch -d <name>                               			# 删除分支(分支已经被合并)
git brach -D <name>                                         # 强行删除分支(分支未合并)
git merge <name>            								# 合并指定分支到当前分支
```

**切换分支注意事项: **

切换分支前必须保证当前分支的所有文件(被git管理的文件)已经commit，否则切换失败。对于那些从未被commit过的文件，则没有关系。

### 提交

```bash
git add .                           					  # 将工作区内容提交至暂存区
git rm --cached <filepath>                       	      # 删除暂存区指定文件
git rm --cached -r <dir>								  # 删除暂存区指定目录
git rm <file>                                             # 删除工作区文件以及暂存区文件
git reset HEAD -- <filepath>                              # 将暂存区的文件内容恢复成仓库内容
git commit -m "msg"										  # 将暂存区内容提交至版本仓库
git commit -m "msg" --amend                               # 修改最近的提交, 不产生新提交
git checkout -- <file|dir>                                # 将工作区的文件恢复成暂存区文件内容
```

### 撤销

`git rm --cached  ` 与 `git reset HEAD -- <file>`之间的异同:

* 前者会将指定文件从暂存区中删除，意味者提交后会将此文件从本地仓库中删除。

* 后者是用本地仓库的文件来恢复暂存区中的指定文件。

* 两个命令都是对暂存区做修改，如果不做其它操作便不会影响工作空间以及本地仓库，也就是说这两个命令

  可以用来做`git add` 命令的后悔药。

举例:

比方说有一个a.txt，内容是first，把它添加到暂存区，然后提交至本地仓库.

接着修改文件内容为second，把它添加到暂存区后，可是对这一次add不满意，需要撤销便可以使用

`git reset HEAD` 命令来恢复**暂存区**的这个文件.

`git checkout -- <file|dir>` 命令用来恢复工作区文件，工作原理是使用暂存区的文件恢复工作区文件.

1. 想要恢复工作区某个文件的内容，该文件内容修改后**未添加到暂存区中**

   `git checkout -- <file>` 此时暂存区该文件的内容与本地仓库中一致

2. 想要恢复工作区某个文件的内容，该文件内容修改后**已添加到暂存区中**，分两步走
   * `git reset HEAD file` 利用本地仓库内容恢复暂存区该文件内容
   * `git checkout -- file` 使用暂存区内容恢复工作区内容

### 历史

```bash
git log -n													# 查看最近n条提交历史
git log -p -n												# 查看提交, 并展开差异
git log -n --graph                                  		# 图形表示的分支合并历史
git log <remote_branch_name>                                # 查看远程分支历史
```

### 远程仓库

```bash
# 添加远程仓库地址, 其中origin是给远程仓库起的名字，一般使用"origin"这个名字
git remote add origin <remote_url>
# 查看给远程仓库起的名字
git remote
# 查看远程仓库起的名字以及地址
git remote -v
# 删除远程仓库, remove接自己起的远程仓库名字
git remote remove origin
# 推送当前分支到远程仓库master分支, 第一次推送需要使用-u选项, 进行关联
git push -u orgin master
# 关联之后直接使用git push推送代码
git push
# 推送本地指定分支到远程指定分支
git push -u origin <local_branch>:<remote_branch>

```

理解：工作区和暂存区是一个公开的工作台，任何分支都会用到，并能看到工作台上最新的内容，只要在工作区、暂存区的改动未能够提交到某一个版本库（分支）中，那么在任何一个分支下都可以看得到这个工作区、暂存区的最新实时改动。
使用git stash就可以将暂存区的修改藏匿起来，使整个工作台看起来都是干净的。所以要清理整个工作台，那么前提是必须先将工作区的内容都add到暂存区中去。之后在干净的工作台上可以做另外一件紧急事件与藏匿起来的内容是完全独立的

