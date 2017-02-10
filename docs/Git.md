# Git的学习笔记咯

> 参考 **沉浸式学 Git** http://igit.linuxtoy.org/

----

# 立即使用

在网页上先创建了仓库，设置好.gitignore

```bash
git clone  github提供的地址(用ssh的)
# 现在创建了你的仓库文件夹，将需要上传的文件放进去
cd 你的仓库名称
git add .
git commit -a -m "这次改了些啥？"
git push
```

----

# git push免密码

参照http://blog.csdn.net/chfe007/article/details/43388041

首先生成自己的ssh密钥

    ssh-keygen -t rsa -b 4096

然后把id_rsa.pub的内容设置到github中，**网页端操作**；建议顺带启用**两步验证**

新手还告诉git自己是谁：

    git config --global user.email "你的邮箱"
    git config --global user.name "你的用户名"

如果当前仓库是https的，改为git方式：

    git remote set-url origin git@github.com:用户名/仓库名称.git
    
----

# bash别名设置

通过修改~/.bashrc来设置别名，让git的日常使用更简单：

```
func_g(){
  git add .
  git commit -a -m "$1"
  git push
}
alias g=func_g
alias gs='git status '
alias ga='git add '
alias gb='git branch '
alias gc='git commit'
alias gd='git diff'
alias go='git checkout '
alias gp='git push'
alias gl="git log --all --pretty=format:'%h %ad | %s%d [%an]' --graph --date=short"
```

![gl的效果](https://raw.githubusercontent.com/zjuchenyuan/notebook/master/download/img/gl.jpg)

完成一次提交，现在只需要`g "提交信息"`

要立即生效，可以执行`source ~/.bashrc`

----

# Git也要翻墙

代码参见[code/ssgit.txt](code/ssgit.txt)

----

# 好玩的命令们

## git status

查看状态咯~

## git reset

已经`git add`了，想取消这一步就用`git reset`

## git checkout

啊。。。代码搞坏了我要回滚到上次commit，用`git checkout -- 文件名`

## git reset --soft <commit_id>

撤销到某次commit，但不删除新增文件

其中commit_id可以从`git log`获得

## 恢复git reset --hard删除的文件

git的历史是不能用命令修改的，丢失的commit用reflog可以找回，除非git已经把它当成垃圾删除（30天）

```
git stash save
git reflog # 查看丢失的那个commit的id
git checkout 那个commitid
git branch recover # 创建recover分支
git checkout master # 回到master
git merge recover # 合并recover到master
git branch -d recover # 合并完成后就可以删了
```

----

# 哲学

* 为啥要**git add**呢?

因为有些时候两个文件可能是不相关的修改，应该分别提交两次

> 通过分开暂存和提交，你能够更加容易地调优每一个提交。

* 为啥不改.profile而是改.bashrc呢

因为win10中只要有一个bash窗口没关掉，启动bash就不是登录，而是相当于再开了个`docker exec -i -t bashonwin10 /bin/bash`

此时是不会执行登录脚本.profile的，但是.bashrc还是会执行的
