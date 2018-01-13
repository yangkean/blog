# Git归纳

*   [Git归纳](#git%E5%BD%92%E7%BA%B3)
    *   [前言](#%E5%89%8D%E8%A8%80)
    *   [安装后的配置](#%E5%AE%89%E8%A3%85%E5%90%8E%E7%9A%84%E9%85%8D%E7%BD%AE)
    *   [添加仓库](#%E6%B7%BB%E5%8A%A0%E4%BB%93%E5%BA%93)
    *   [记录每次更新到仓库中并推送到远端仓库](#%E8%AE%B0%E5%BD%95%E6%AF%8F%E6%AC%A1%E6%9B%B4%E6%96%B0%E5%88%B0%E4%BB%93%E5%BA%93%E4%B8%AD%E5%B9%B6%E6%8E%A8%E9%80%81%E5%88%B0%E8%BF%9C%E7%AB%AF%E4%BB%93%E5%BA%93)
    *   [分支](#%E5%88%86%E6%94%AF)
    *   [打标签](#%E6%89%93%E6%A0%87%E7%AD%BE)
    *   [减少麻烦](#%E5%87%8F%E5%B0%91%E9%BA%BB%E7%83%A6)
    *   [总结一下](#%E6%80%BB%E7%BB%93%E4%B8%80%E4%B8%8B)
    *   [Reference](#reference)

## 前言

同上篇一样，归纳一下常用的便于使用。

## 安装后的配置

初次安装时，通过[这个](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)安装好 **Git** 后，要配置你个人的用户名称和邮件地址，**Git**每次 commit 都会使用这些信息：

    git config --global user.name "[username]"
    git config --global user.email "[email address]"

使用`--global`是全局设置用户名和密码，以后便不用再输入这两条命令，之后会默认使用这里设置的信息。可用`git config --list`查看所有配置信息。

## 添加仓库

除非是使用 **GitHub** 提供的API，否则无法直接通过 **Git** 远程创建一个 **GitHub** 上的仓库 (因为 **GitHub** 不允许这样做)，所以在添加仓库前先通过 **GitHub** 的网页图形界面先创建一个仓库。

如果你很想用带 **GitHub API** 的命令行装逼的话，可以用下面这条命令远程创建仓库：

    curl -u 'USER' https://api.github.com/user/repos -d '{"name":"REPO"}'

`USER`换成你 **GitHub** 账号的用户名，`REPO` 则是你要创建的仓库名。

然后创建一个目录并进入该目录，执行以下命令新建本地仓库：

    git init [project-name]

或者可以克隆远端项目到本地：

    git clone [remote url]

## 记录每次更新到仓库中并推送到远端仓库

本地仓库是由三部分组成的，即工作目录 (*working directory*)，暂存区 (也叫index)和指向上一次 *commit* 的 *head*。在工作目录繁多的文件中，有一些是已经完成了修改的，我们可以先将已经完成的修改放进暂存区中，用：

    git add [file/folder] # add目录时会递归处理其下文件

在 *add* 之前，可以用下面这条命令查看未暂存文件的改变：

    git diff

接着将文件此刻的状态永久地保存到版本历史中，即 *commit* 所有已 *add* 的文件，用：

    git commit -m "[descriptive message]"

而其他未完成的修改可以不 *add* 和 *commit* 先。可以理解为，*add* 是对文件修改状态的选择，*commit* 是对文件修改状态的保存。

可用以下命令查询文件状态：

    git status

然后就可以将本地的所有 *commit* 提交到远端仓库中，用：

    git push [-u] [remote repo] [branch] # `-u`用来设置本地仓库分支将要跟踪的远端 (也叫上流) 仓库的分支 (不存在就会被创建)

但是直接用远端仓库名和分支 *push* 是会报错的，因为你还没设置 *push* 的远端仓库地址，要设置push的目的地仓库及其简称：

    git remote add [shortname] [remote url]

`remote url`是远端仓库的地址，`shortname`是可以用来引用该地址的简写。

想要查看项目提交历史的话，可以用：

    git log

## 分支

除了主分支，我们还可以建立其他分支用于并行开发，这些分支也可以合并进主分支。先罗列下本地的所有分支看看，注意当前分支会带星号并高亮：

    git branch

我们来创建一个新本地分支：

    git branch [branch name]

要让别人看到你的新分支，你必须通过`git push [remote repo] [branch] `把分支推送到远端仓库才行。如果你不再爱一个分支并想抛弃之，可以用：

    git branch -d [branch name]

不过这只能删除本地分支，要想删除远端仓库，可以用：

    git push -d [remote repo] [branch]

如果想把其他分支合并进当前分支，可以用：

    git merge [branch name]

之后就可以把已被合并的那个分支删掉了：）

想在不同的分支间来去自如？想法不错，用用这个吧：

    git checkout [branch name]

切换分支的同时也会将工作目录 (*working directory*)转换至相应分支的目录。

另外，既然我们可以向远端仓库 *push* 东西，也可以“拉”东西到本地，即：

    git pull

这会下载远端仓库的所有东西并与本地仓库进行合并，相当于`git fetch [remote url]` 加 `git merge [branch name]`，因为`git fetch [remote url]`只下载远端仓库的所有东西但不进行合并。

## 打标签

为软件的发布版本设置标签是我们常用的手法，也是值得推荐的，这样方便我们了解软件的迭代过程。

首先列出已有标签：

    git tag

然后创建标签：

    git tag <tagname>

要把标签传送到远端仓库中，需执行：

    git push [remote repo] <tagname>

删除本地标签：

    git tag -d <tagname>

删除远端标签：

    git push -d [remote repo] <tagname>

## 减少麻烦

为了省去每次 *push* 时都要输入 *GitHub* 账号和密码的麻烦，我们可以用 *SSH* 的方式连接 *GitHub*。首先创建 *SSH key*：

    ssh-keygen -t rsa -C "[email address]"

一路默认后打开`id_rsa.pub`文件：

    vim ~/.ssh/id_rsa.pub

将该文件具体内容粘贴至 *Setting*->*SSH and GPG keys*->*New SSH key*处，克隆远端项目到本地则采用 *SSH* 协议，其他用到远端仓库 *https* 地址的地方也都采用 *SSH* 地址。详情可见 [Connecting to GitHub with SSH](https://help.github.com/articles/connecting-to-github-with-ssh/)，此处不再赘述。

## 总结一下

对于极常用的一些命令，结合实例总结如下：

* git init git-practice  新建本地仓库
* git add .  将当前目录的修改放到暂存区中
* git commit -m "first commit"  将文件此刻状态永久保存到版本历史中
* git remote add origin https://github.com/yangkean/git-practice.git  设置 *push* 的远端仓库地址及其简称
* git push -u origin master  将所有 *commit* 提交到远端仓库中

最后，如果不想用命令行，除了 *git* 自带的 *GUI*，市面上还有很多好用的 *GUI*，比如 [SourceTree](https://www.sourcetreeapp.com/)。

## Reference

* [git - the simple guide](http://rogerdudler.github.io/git-guide/index.html)
* [Pro Git - 2nd Edition](https://git-scm.com/book/en/v2)
* [GitHub Help](https://help.github.com/)
* [搬进 Github](http://www.mengma.com/class/lessonread/15/2)
* [GitHub Git Cheat Sheet](https://services.github.com/on-demand/downloads/github-git-cheat-sheet/)
