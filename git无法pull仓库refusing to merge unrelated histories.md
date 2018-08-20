# git无法pull仓库refusing to merge unrelated histories

如果合并了两个不同的开始提交的仓库，在新的 git 会发现这两个仓库可能不是同一个，为了防止开发者上传错误，于是就给下面的提示
> fatal: refusing to merge unrelated histories

于是使用下面的命令:
``` git pull origin master --allow-unrelated-histories```