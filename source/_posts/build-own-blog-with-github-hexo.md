---
title:手把手教你用github+hexo建博客
date: 2017-12-24 22:42:53
tags:
---


#手把手教你用github+hexo建博客
[TOC]

<!-- md common-header.md -->

本文内容包括：
环境准备：git npm nodejs ,hexo
github
hexo branch
本地博客
config hexo环境，然后push到远程
next主题
git submodule
github sync fork 项目
config
随意切换环境写博客
绑定域名

需要了解什么：熟悉git 命令，github基本操作

##创建一个 github.io 仓库管理 blog 分支和 hexo 分支
你可能会问为啥要 blog 分支和 hexo 分支，因为我想仅使用一个 git 仓库就可以完成写博客、发布博客、存储的的需求，至于为什么，且往下看，一步一步你就会明白。
### 创建一个github.io仓库并创建hexo分支
在 github上创建一个名为 teffy.github.io 的仓库，默认生成的 master 分支为我们博客存储的分支，再创建 hexo 分支用做写博客和发布博客环境的存储分支。
```
# 在github 上创建 teffy.github.io，很简单
git clone git@github.com:teffy/teffy.github.io.git
cd teffy.github.io # 进入仓库根目录
git checkout -b hexo # 创建 hexo 分支
git push origin hexo:hexo # push hexo 分支到远程
```
###设置hexo分支为主分支

![](http://p1cycpoti.bkt.clouddn.com/blog/imgs/change_default_branch.png) 

##本地搭建博客
### hexo 初始化
进入 git 仓库 teffy.github.io 根目录
```
hexo init teffy.github.io
```
这样会生成 teffy.github.io 这样一个目录，和我们的git仓库名
![](http://p1cycpoti.bkt.clouddn.com/blog/imgs/repeat_dir.png) 
这样就重复了，我们需要把 hexo 生成的环境移动到仓库根目录
``` 
mv teffy.github.io/* ./ # 把hexo生成的 teffy.github.io 下所有文件移动到 git 仓库根目录
rmdir teffy.github.io # 删除 hexo init 生成的 teffy.github.io 目录
``` 
那么你可能会问，为啥不在 teffy.github.io 目录的上一级执行 hexo init teffy.github.io ，这样不就省去了这一步吗？那是因为 hexo init 要求后面的目录是一个空目录，而 teffy.github.io 是一个 git 仓库，里面就算没有其他文件也会有一个.git隐藏文件，所以无法这么操作。
###测试本地博客
在teffy.github.io目录下，注意切换到 hexo 分支，查看分支 git branch ，切换分支 git checkout hexo 。
```
hexo s # 完整命令 hexo server
```
在浏览器里打开 http://localhost:4000/ 就可以看到本地博客的效果。
###测试一下发布 blog
先在 git 仓库 根目录安装 hexo-deployer-git
```
npm install hexo-deployer-git --save
```
我们先来测试一下发布，打开_config.yml，修改最下面的Deployment
```
# Deployment ，我使用的是git， repository就是teffy.github.io的仓库地址，这里用https，注意还要要指定branch为master，因为master是我们的blog静态文件所存储的分支
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repository: https://github.com/teffy/teffy.github.io.git
  branch: master
```
发布一下
```
hexo clean && hexo g && hexo d
```
然后访问一下 [http://teffy.github.io](http://teffy.github.io) 看看是否发布成功
###提交 hexo 环境到 github
hexo 环境已经OK了，我们就把这套 hexo 环境提交到github上，在这之前先配置一下  [.gitignore](https://github.com/teffy/teffy.github.io/blob/hexo/.gitignore) ，要不然会把一些不必要的文件也提交上去。
```
.DS_Store
Thumbs.db
#db.json 这里我用了markdown include 插件，如果忽略的话，会报错找不到这个文件，所以我就不忽略了
*.log
node_modules/
public/
.deploy*/
```
提交到 github 。
```
git add . 
git commit -m "hexo env"
git push orign hexo # 这里注意是  hexo 分支
```
###换一个工作空间继续写博客
到此为止，我们整个 blog 和 hexo 环境都已经全部创建完成，而且我们可以做到随意换一个工作空间就可以继续写博客了。** Waht ??? **这么做就可以了？是的，那么，先来测试一把看看。这里我就在电脑换个目录操作来模拟换一个工作空间继续写博客，当然真实情况一般是换了电脑，那就还需要将各种环境工具先安装好，然后 clone 下来我们的仓库之后，在 hexo 分支上，先安装一下 hexo ，然后就可以像刚才一样愉快的玩耍了。
```
➜  blog: mkdir test
➜  blog: cd test 
➜  test: git clone git@github.com:teffy/teffy.github.io.git
➜  test: cd teffy.github.io 
➜  teffy.github.io git:(hexo): git status
位于分支 hexo
➜  teffy.github.io git:(hexo): hexo s #这里先本地测试一下，会提示这个仓库没有安装hexo，按照提示安装一下
ERROR Local hexo not found in ~/Workspace/blog/test/teffy.github.io
ERROR Try running: 'npm install hexo --save' #提示在这
➜  teffy.github.io git:(hexo) npm install hexo --save  # 安装hexo
+ hexo@3.4.4
added 329 packages in 7.519s
➜  teffy.github.io git:(hexo) ✗ hexo s #再来一次
INFO  Start processing
INFO  Hexo is running at http://localhost:4000/. Press Ctrl+C to stop.
^CINFO  Catch you later
➜  teffy.github.io git:(hexo) ✗ vim source/_posts/hello-world.md #修改一下文章测试一下呢
➜  teffy.github.io git:(hexo) ✗ hexo s  # 换个姿势，再来一次
INFO  Start processing
INFO  Hexo is running at http://localhost:4000/. Press Ctrl+C to stop.
^CINFO  Good bye
# 接下来测试能不能发布
➜  teffy.github.io git:(hexo) ✗ hexo clean && hexo g && hexo d
INFO  Deleted database.
INFO …… # 省略部分log
INFO  Deploy done: git 
```
再去访问一下 [http://teffy.github.io](http://teffy.github.io) 看看我们的修改是否发布成功。OK，我们在新的环境中做的修改就可以提交上去了。
```
git status
git diff
git add .
git cm -m "modify test"
git push origin hexo
```
那我们再切回老的目录的时候（相当于切换回原来的工作空间）只需要 git pull 更新一下就可以了。
## 使用git submodule 管理主题
虽然已经把blog和hexo环境都搞好了，而且还只用一个git仓库就管理了blog和hexo，更重要的是，还有可以随便换电脑都能写博客这种骚操作，还有更骚的吗？答案是有的，接下来我们要用 git submodule 来管理自己喜欢的主题，在这之前，你应该知道的是一般主题的开发者也都是使用github托管。你可能会问，主题还用管理吗，直接放在仓库里不就OK了，当然那样是可以的，但是后续如果主题的开发者有什么更新，我们想更新就会非常麻烦，基本是石器时代的感觉，而我们使用git submoudle就是单车变摩托的感觉了。这样我们博客和主题的仓库就分开进行管理了，主题仓库作为博客仓库的一个子模块。
### 添加 git submodule
我使用的是 [Next](https://github.com/iissnan/hexo-theme-next) 主题，我们先来 fork 一份，以便后续我们可以同步主仓库的更新。之后在我们的 git 仓库中的 themes 目录下把刚才 fork 的主题 git 仓库添加到我们的博客的仓库中的 git submodule 。
```
cd themes
git submodule add git@github.com:teffy/hexo-theme-next.git
git commit -m "add next theme submodule"
git push origin hexo
```
先来测试一下 Next 主题，修改 _config.yml 中主题配置
```
# theme: landscape
theme: hexo-theme-next
```
本地测试看一下效果，然后提交到 github 上去。
```
hexo s
git add .
git commit -m "use next theme"
git push origin hexo
```
### 管理主题仓库
来看看怎么管理主题呢，做一个简单的主题配置修改
```
# cd 进入 themes/hexo-theme-next 目录，修改 _config.yml
git add .
git commit -m "update config"
git push origin master
# 然后回到 teffy.github.io 仓库根目录
cd ../..
git status #可以看到        修改：     themes/hexo-theme-next (新提交)
git add .
git cm -m "theme update"
git push origin hexo
```
### 换工作空间之后管理主题
那么问题来了，就是我们切换工作空间了之后主题仓库怎么办？除了之前说的需要安装几个工具和 git 操作之外，还需要再处理一下主题，接着在刚才的新的工作空间里操作一下。
```
git pull # 先更新
Fast-forward
 .gitmodules            |  3 +++
 _config.yml            | 17 ++++++++++-------
 themes/hexo-theme-next |  1 +
# 这里通过git log 就可以看到有主题的更新，但是这里只是更新了 submodule ，会生成一个 submodule 仓库的空目录，而真正主题的仓库文件还没有更新到这个新的工作空间中
teffy.github.io git:(hexo) ll themes/hexo-theme-next 
总用量 0    # 这里可以看出是个空目录
git submodule init
git submodule update  #这样就可以将主题仓库中的文件更新下来
```
更新完主题之后可以 hexo s 本地测试一下，妥妥的。
### 同步主题主仓库的更新
那么问题又来了，如果主题的开发者有了更新，我fork的主题仓库怎么同步主仓库的更新呢，我们可以使用 git fetch 命令来进行
```
git remote add upstream https://github.com/iissnan/hexo-theme-next.git # 先指定上游主仓库
git fetch upstream # 更新主仓库内容
git status # 查看是否有更新
git checkout master # 切换回 master 分支
git merge upstream/master # 将上游 master 分支和自己的 master 分支合并
# 这里有可能会有冲突，那么解决一下冲突然后，需要 git add . 和 git commit 一下
git push origin master # 最后将这些主仓库 master 分支的改动提交到我们自己的仓库中去
```
## 写博客
说了这么多，还没说怎么写博客呢。
```
hexo new "first_blog"
```
然后使用 markdown 编辑器写好文章之后，先本地看下效果之后，就可以部署到 github上了，当然还要把这些新的文章提交到 hexo 环境中去，以便换了工作空间还可以对已经发布的文章修改。
```
hexo clean && hexo g && hexo d
git add . 
git commit -m "new blog"
git push orign hexo # 这里注意是  hexo 分支
```
### markdown include 插件
这里我推荐一款插件 [markdown include](https://github.com/tea3/hexo-include-markdown)
有点问题
## 绑定域名
上面的步骤做完之后已经可以愉快的写博客了，那么再加上一个个性化的域名是不是更加完美？恩，你说啥就是啥！
###买个域名
买域名就不详细说了，各个域名服务商的操作都很简单，准备好 money 就行了，只说一点，国外域名服务商不需要备案，国内的需要备案。
### 配置 github 仓库
在我们的 仓库的 hexo 分支下，source目录新建个CNAME文件，里面写上买好的域名，然后重新部署一下就 OK 了。
### 配置 DNS
在域名服务商的管理控制页面添加两条A记录和一个CNAME记录。

| Host(主机记录) | 记录类型 | Points To(记录值) |
| :----------------- | :---------- | :-------------------- |
| @                          | A                | 192.30.252.153      |
| @                          | A                | 192.30.252.154      |
| www                   | CNAME   | teffy.github.io       |

### 配置 dnspod
如果是在国外域名服务商买的域名，由于墙太高的原因，最好使用国内域名服务器，我使用的是 [dnspod](https://www.dnspod.cn)，注册登录之后和刚才一样添加两条A记录和一个CNAME记录。然后回到国外域名服务商的管理控制页面，修改 DNS 服务器。
```
#域名服务器
f1g1ns1.dnspod.net
f1g1ns2.dnspod.net
```
最后喝杯茶等一会就生效了，直接访问域名 [http://teffy.me](http://teffy.me) 试试。
## 配置博客 & 配置主题









<!-- md common-footer.md -->

