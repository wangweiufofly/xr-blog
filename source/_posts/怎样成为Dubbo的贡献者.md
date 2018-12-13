---
title: 怎样成为dubbo的贡献者
date: 2018-11-17 21:11:50
tags:  
    - github
    - Dubbo
---

## 一.关系图
![关系图](/assets/2018/github-guanxitu.png)
<!-- more -->
## 二.操作流程
![步骤](/assets/2018/dubbo-naotu.png)

## 三.一些命令
- 以dubbo为例
 dubbo原地址：https://github.com/apache/incubator-dubbo.git
 fork后地址：https://github.com/wangweiufofly/incubator-dubbo.git

#### 1、git clone fork的项目到本地
``` markdown
git clone https://github.com/wangweiufofly/incubator-dubbo.git
```

#### 2、git remote add upstream增加源分支地址
``` markdown

cd incubator-dubbo

//增加远程分支
git remote add upstream https://github.com/apache/incubator-dubbo.git

//fetch源分支的新版本到本地
git fetch upstream

//查看远程源
git remote -v
```

#### 3、git push 提交代码到github

``` git
//本地创建分支并切换
git branch mybranch
git checkout mybranch
git branch -a

//提交
git commit -m "测试的一次提交"

//提交代码到github
git push --set-upstream origin test
```

#### 4、提交给github的new pull request
这块是github界面操作,入下图。
<img src="/assets/2018/dubbo-pull-request.png" width = "800" div align=center />


## 四、科普GitHub功能按钮
github中的watch、star、fork的作用

#### watch
- 观察，点击watch可以看到如下的列表。

默认每一个用户都是处于Not watching的状态，当你选择Watching，表示你以后会关注这个项目的所有动态，以后只要这个项目发生变动，如被别人提交了pull request、被别人发起了issue等等情况，
你都会在自己的个人通知中心，收到一条通知消息，如果你设置了个人邮箱，那么你的邮箱也可能收到相应的邮件。

#### star
- star，理解为`关注`或者`点赞`，当你点击 star,表示你喜欢这个项目或者通俗点，可以把他理解成朋友圈的点赞吧，表示对这个项目的支持。

不过相比朋友圈的点赞，github 里面会有一个列表，专门收集了你所有 start 过的项目，点击 github 个人头像，可以看到 your star的条目，点击就可以查看你 star 过的所有项目了。如下图
<img src="/assets/2018/git-stat.png" width = "800" div align=center />

#### fork
-  fork，相当于你自己有了一份原项目的拷贝，当然这个拷贝只是针对当时的项目文件，如果后续原项目文件发生改变，你必须通过其他的方式去同步。

使用 fork 这个功能，一般不会用，除了有一些项目，可能存在 bug 或者可以继续优化的地方，你想帮助原项目作者去完善这个项目。




