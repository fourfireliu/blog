---
title: GIT的一些小tip
date: 2016-07-15 10:12:00
tags:
 - git
categories:
 - Java
---



​       随着开源潮流的滚滚而来，github应该是当今互联网开发最流行的开源代码库和版本控制系统了，没有之一。几乎所有的互联网公司也都搭建了公司内部的git服务器。很多人喜欢用eclipse自带的git插件，但是中间隔了一层有时候并不是那么好用，也有可能命令执行失败而插件没有报出来，所以我还是更相信Git Bash:) 暂且在这里记录几条平时用的较少但很好用的git命令吧。

<!-- more -->

1. 有时候在版本a上开发，紧急情况下需要把一个在版本a上的commit弄到用于发布的版本b上去，这时候git cherry-pick 就很有用，它的作用就是"Apply the changes introduced by some existing commits"

    首先 切换当前本地版本到版本a，git log看一下要传递的commit id，比如

   commit 8dd3e797b973de002cf6203cbc1a065e8dc1dfef

   复制下来，然后，切换本地版本到b，git pull拉一下最新代码，执行命令

   git cherry-pick 8dd3e797b973de002cf6203cbc1a065e8dc1dfef 

   这个commit就会在你本地处于一个commit的状态，你接下来只需要执行git push即可，当然，如果代码有冲突的话，就需要解决冲突后再提交了。

2. 有时候刚push了几个commit到版本a上去，尼玛这时候产品一个邮件说因为xxx原因，这个需求这期先不上了或者就废弃了，需要回退代码。这时候相当于需要把这几个已经push到git服务器的commit取消。这时候你就需要git revert，它的作用如其名 反转"Revert some existing commits"，效果就是push了一个新commit，内容和你原先的commit完全相反。删一行对应着加一行...

   首先，还是git log看一下要revert的commit id，比如

   commit 8dd3e797b973de002cf6203cbc1a065e8dc1dfef

   复制下来，然后执行

   git revert 8dd3e797b973de002cf6203cbc1a065e8dc1dfef

   这时你就会处于要提交一个新commit的状态 commit注释就是revert xxx，接下来只需要执行git push即可。如果代码是比较早的，revert导致了冲突 你也要先解决冲突才行。另外，如果要revert几个commit，请注意他们的先后顺序，先revert后提交的，就像一个stack一样，避免不必要的代码冲突。

3. 这个情况可能出现的比较少 有时候git add git commit 过后，忽然由于什么原因git push不了了 或者想起了什么需要在这个commit上重新修改，总之就是想要从git commit状态回退到一切都未提交状态，修改的代码当然还得在，这个时候可以使用

   git reset --soft HEAD^ 

   HEAD^就表示上一次提交​，这命令表示reset到上一次提交状态 但是用--soft来保证这次的修改还在。

