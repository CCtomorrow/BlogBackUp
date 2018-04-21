---
title: 'Git 用法全'
date: 2017-07-23 21:35:40
tags: [Git]
categories: [Git]
---
#### 先推荐两本书:[ProGit](https://git-scm.com/book/zh)，[Git Community Book 中文版](http://gitbook.liuhui998.com/index.html)
#### 1.记录每次更新
工作目录下的文件都只有两种状态:已跟踪或者未跟踪。
已跟踪的文件指那些已经纳入了版本控制，在上次快照中有他们的记录，工作一段时间后，他们的状态可能处于已修改，未修改或者已放入暂存区。
除此之外的文件就是未跟踪的，他们既不存在快照中，也没放入暂存区。
##### 1.1.检查文件的状态:git status
![检查文件的状态](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/52586039.png)

##### 1.2.跟踪新文件:git add <file>
![跟踪新文件](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/53526870.png)
可以看到文件已被跟踪，并处于暂存状态。
git add 使用文件或者目录作为参数，如果是目录的话，会递归跟踪目录下的所有文件。

<!-- more -->

##### 1.3.暂存已修改文件:git add <file>
现在修改sample.txt这个文件。然后查看状态。
![暂存已修改文件](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/55163184.png)
可以看到文件已经被修改了，上面的new file那里，是上次add的记录。
执行add命令，再次查看状态:
![](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/55266326.png)
可以看到文件已经被暂存了，这里add命令其实不止一个作用的，是一个多功能命令:跟踪新文件；把已跟踪文件放入暂存区；还能用于合并时把有冲突的文件标记为已解决状态。
##### git add:可以把这个命令理解为:添加内容到下一次提交中。
例如:如果你对一个文件做了两次修改，第一次修改完成了，然后执行了add命令，然后继续修改了，然而并没有执行add命令，这个时候如果你提交的话，提交上去的就是第一次修改的版本，即最后一次执行add命令时候的版本。

##### 1.4.忽略文件:.gitignore
创建.gitignore文件。格式规范如下:
- 所有空行或者以 `#` 开头的行都会被 Git 忽略。
- 可以使用标准的 glob 模式匹配。
- 匹配模式可以以`/`开头防止递归。
- 匹配模式可以以`/`结尾指定目录。
- 要忽略指定模式以外的文件或目录，可以在模式前加上惊叹号`!`取反。
例如:
```
# no .a or .o files
*.[ao]

# but do track lib.a, even though you're ignoring .a files above
!lib.a

# only ignore the TODO file in the current directory, not subdir/TODO
/TODO

# ignore all files in the build/ directory
build/

# ignore doc/notes.txt, but not doc/server/arch.txt
doc/*.txt

# ignore all .pdf files in the doc/ directory
doc/**/*.pdf
```

##### 1.5.查看已暂存和未暂存的修复:git diff
在sample.txt换行添加`dd`的文字
![查看已暂存和未暂存的修复](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/56761893.png)
可以看到，我们在fg所在行添加了一个换行，然后添加了一个空行和一行文字，文字为`dd`。
##### git diff命令比较的是当前文件和暂存区域快照之间的差异。也就是修改了还没暂存起来的变化内容。
若要查看已经暂存的将要添加到下次提交的内容，可以使用`git diff --cached`,1.6.1及高版本可使用`git diff --staged`。
![git diff](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/57163609.png)

##### 1.6.提交更新:git commit
确定了需要提交的暂存了之后就可以提交了，使用`git commit -m "msg"`，可以提交啦，这种方式是要先使用git add命令暂存的，就是提交的是暂存了的内容，没有暂存的内容是提交不了的。
还可以跳过使用暂存区，使用`git commit -a -m "msg"`，使用这个命令其实就是git会把已经跟踪过的文件暂存起来一起提交。

##### 1.7.删除文件:git rm
如果我们需要删除从git版本库里面删除一个文件的话，需要做如下操作。
![删除文件](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/58175897.png)
先执行`rm sample.txt`文件把文件从本地删除，然后执行`git rm sample.txt`把文件从已跟踪文件中移除。然后提交。如果删除之前已经修改过并且存入了暂存区的话，就需要`git rm -f sample.txt`，如果想要保存本地文件，但是并不想让git继续跟踪这个文件了，可以`git rm --cached sample.txt`。


#### 2.撤销操作
##### 2.1.重新提交:git commit --amend
有时候我们提交完了发现有些文件忘记加入暂存区了，比如
![重新提交](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/59382322.png)
我们上次提交完了发现forget.txt的最后一次修改没有假如暂存区，怎么办呢，这个时候就可以先执行add命令把文件添加到暂存区，然后执行`git commit --amend`。这样就不会造成需要提交两次了。
注意：如果已经推送到服务器了就不要使用这个了。

##### 2.2.取消暂存的文件:git reset HEAD <file>
![取消暂存的文件](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/59772952.png)

##### 2.3.撤销对文件的修改:git checkout -- <file>
![撤销对文件的修改](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/60323992.png)
可以看到修改没有了，但是这种撤销是不可恢复的因为是用另外一个文件覆盖当前的文件。


#### 3.远程仓库
##### 3.1.查看远程仓库:git remote -v
![查看远程仓库](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/60667489.png)
指定选项 -v，会显示需要读写远程仓库使用的 Git 保存的简写与其对应的 URL。

##### 3.2.添加远程仓库:git remote add [short-name] [url]
![添加远程仓库](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/60782219.png)
这里添加的是一样的url，只是演示怎么使用。

##### 3.3.从远程仓库拉取:git fetch [remote-name]/[branch-name]
![从远程仓库拉取](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/61098754.png)

##### 3.4.推送到远程仓库:git push [remote-name] [branch-name]
![推送到远程仓库](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/61192074.png)
你也可以运行:git push origin master:master，它会做同样的事 - 相当于它说，“推送本地的 master 分支，将其作为远程仓库的 master分支” 可以通过这种格式来推送本地分支到一个命名不相同的远程分支。 如果并不想让远程仓库上的分支叫做 master，可以运行 git push origin master:mm 来将本地的 master 分支推送到远程仓库上的 mm分支。

##### 3.5.查看远程仓库的详细信息:git remote show [remote-name]
![查看远程仓库的详细信息](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/61277198.png)

##### 3.6.远程仓库的移除与重命名:git remote rename [from-name] [to-name]
移除:git remote rm [remote-name]

##### 3.7.查看远程分支的列表:git ls-remote
![查看远程分支的列表](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/85818777.png)
远程分支总以(remote)/(branch-name)，git clone默认分配的命令就是origin，所以平时看到很多remote的名字就是origin。

##### 3.8.跟踪远程分支:git branch -u (remote)/(branch-name)
设置已有的本地分支跟踪一个刚刚拉取下来的远程分支，或者想要修改正在跟踪的上游分支，你可以在任意时间使用 -u 或 --set-upstream-to 选项运行 git branch 来显式地设置。
![跟踪远程分支](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/498587.png)

##### 3.9.删除远程分支:git push [remote] --delete [branch-name]
例如:git push origin--delete develop


#### 4.分支基础
git的分支，本质上是指向[提交对象](https://git-scm.com/book/zh/v2/Git-%E5%88%86%E6%94%AF-%E5%88%86%E6%94%AF%E7%AE%80%E4%BB%8B)的可变指针，
##### 4.1.创建分支:git branch [branch name]
![创建分支](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/81139668.png)
这会在当前的提交对象上面创建一个指针。那么现在git怎么知道现在是指向哪个分支呢，git有个特殊的HEAD指针，指向当前所在的本地分支(将HEAD想象成当前分支的别名)，当前现在我们仍然在master分支，`git branch`仅仅是创建一个分支，并不会自动切换到新创建的分支。
我们可以使用`git log --oneline --decorate`查看各个分支当前所指的对象。
![git log](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/81791224.png)

##### 4.2.切换分支:git check [branch-name]
![切换分支](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/81944847.png)
然后我们修改其中的一些文件，再次提交。然后再看一下当前的HEAD指向情况。
![当前的HEAD指向情况](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/82139536.png)
可以看到HEAD指向已经向前移动了和master分支不在一起了。这个时候切换回mater分支，可以发现这次的文件改动并没有影响到mater分支的。
所以`git checkout [branch-name]`这条命名做了两件事，将工作目录恢复成切到的分支所指向的快照内容；将HEAD指针指向所切换的分支。
我们可以使用如下命令，创建并切换到创建的分支:git checkout -b [branch-name]

##### 4.3.合并分支:git merge [branch-name]
意思是把想要合并的分支也就把名字是[branch-name]的分支合并到当前分支。
![合并分支](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/84704005.png)
我们平时开发都是这样子的，开发完成了会把稳定代码合并到主分支，然后继续在别的分支开发别的任务，但是有的情况是在开发别的分支的过程中，线上版本出了问题，我们需要紧急去修复，需要做的工作就是把现在开发的内推提交，然后切换回到主分支，然后切一个hotfix的分支，把问题修复了之后再把代码合并到master分支。
![合并](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/85082051.png)
然后现在的指向情况如下:
![情况如tu](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/85125425.png)
图示如下:
![图示](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/85208574.png)
*在合并的时候，你应该注意到了"快进（fast-forward）"这个词。 由于当前 master 分支所指向的提交是你当前提交（有关 hotfix 的提交）的直接上游，所以 Git 只是简单的将指针向前移动。 换句话说，当你试图合并两个分支时，如果顺着一个分支走下去能够到达另一个分支，那么 Git 在合并两者的时候，只会简单的将指针向前推进（指针右移），因为这种情况下的合并操作没有需要解决的分歧——这就叫做 “快进（fast-forward）”。*
上面的话，可以在[分支的新建与合并](http://dd089a5b.wiz03.com/share/s/3t29Fr0cZkNg2Hu15w0bv6Nw3WHrTN1A2Aa32m7HhG37FnNy)看到。

然后一般我们会删除hotfix分支，然后返回开发分支开发，删除分支的命令是:
`git branch -d [branch-name]`。
然后我们开发完成之后会把开发分支合并到mater。合并之前是这样的:
![合并之前](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/85760753.png)
这个时候的情况和之前合并hotfix分支的情况并不相同的，master所在的提交并不是testing所在提交的直接祖先了。这个时候git会使用两个分支末端所指的快照(即C4和C5)，以及这两个分支的工作祖先C2做一个简单的第三方合并。
 和之前将分支指针向前推进所不同的是，Git 将此次三方合并的结果做了一个新的快照并且自动创建一个新的提交指向它。 这个被称作一次合并提交，它的特别之处在于他有不止一个父提交。
然后合并，合并之后是这样的:
![合并之后](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/86022981.png)
对应的git命令是这样的:
![命令](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/86306720.png)
合并的时候可能会有冲突，对应到想应的冲突文件里面大概是这样:
![](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/86136329.png)
<<<<< HEAD行以及=====之间指的是你当前分支的内容，=====以及>>>>> testing是你要合并的分支的内容。根据你需要的内容进行取舍即可，解决完冲突，执行`add`操作，然后提交即可。
![](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/173225.png)

##### 4.4.分支管理
查看分支列表:git branch
![分支管理](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/84764960.png)
分支前面有星的表示当前在哪个分支，也就是HEAD指针所指的分支。
如果还想查看最后一次的提交:git branch -v
![](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/84877979.png)
看到分支列表里面哪些分支已经合并到*当前分支*，或者未合并的:git branch --merged
![](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/85069564.png)
删除分支:git branch -d [branch-name]

##### 4.5.变基:git rebase [branch-name]
开发任务分叉到两个不同分支，又各自提交了更新。
![变基](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/875758.png)
之前介绍过，整合分支最容易的方法是 merge 命令。 它会把两个分支的最新快照（C3 和 C4）以及二者最近的共同祖先（C2）进行三方合并，合并的结果是生成一个新的快照（并提交）。

如果我们现在要把master的代码弄到testing分支上，除了merge还可以使用变基的方式，变基的意思是:可以把在从C2之后testing分支上面引入的补丁和修改，在C4的基础上应用一次。
*在 Git 中，这种操作就叫做 变基。 你可以使用 rebase 命令将提交到某一分支上的所有修改都移至另一分支上，就好像“重新播放”一样。*
![](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/1447202.png)
可以看到，我们在testing上面有两次提交的吧，刚好应用了两次。

它的原理是首先找到这两个分支（即当前分支 testing、变基操作的目标基底分支 master）的最近共同祖先 C2，然后对比当前分支相对于该祖先的历次提交，提取相应的修改并存为临时文件，然后将当前分支指向目标基底 C4, 最后以此将之前另存为临时文件的修改依序应用。
![](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/2162188.png)

如上图所示的。然后我们可以回到master分支，把testing的代码合并到master分支啦。
![](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/44859060.png)

此时，C5' 指向的快照就和上面使用 merge 命令的例子中 C6 指向的快照一模一样了。 这两种整合方法的最终结果没有任何区别，但是变基使得提交历史更加整洁。 你在查看一个经过变基的分支的历史记录时会发现，尽管实际的开发工作是并行的，但它们看上去就像是串行的一样，提交历史是一条直线没有分叉。
![](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/45179622.png)
上面的图红色圈起来的是rebase的，蓝色圈起来的是merge的，可以看到merge是把三个提交合并然后生成了一个新的提交的，rebase就不会了，会把从公共祖先mater分支以后的两次改版(develop的改动)再次在master的最后一次提交上面回放。

请注意，无论是通过变基，还是通过三方合并，整合的最终结果所指向的快照始终是一样的，只不过提交历史不同罢了。 变基是将一系列提交按照原有次序依次应用到另一分支上，而合并是把最终结果合在一起。

下面看一个更有趣的变基的例子:
你创建了一个特性分支 server，为服务端添加了一些功能，提交了 C3 和 C4。 然后从 C3 上创建了特性分支 client，为客户端添加了一些功能，提交了 C8 和 C9。 最后，你回到 server 分支，又提交了 C10。
![](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/82332712.png)
类似这样，假设你希望将 client 中的修改合并到主分支并发布，但暂时并不想合并 server 中的修改，因为它们还需要经过更全面的测试。 这时，你就可以使用 git rebase 命令的 --onto 选项，选中在 client 分支里但不在 server 分支里的修改（即 C8 和 C9），将它们在 master 分支上重放。
`git rebase --onto master server client`
以上命令的意思是：“取出 client 分支，找出处于 client 分支和 server 分支的共同祖先之后的修改，然后把它们在 master 分支上重放一遍”。
![](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/82996634.png)
假如在那些已经被推送至共用仓库的提交上执行变基命令，并因此丢弃了一些别人的开发所基于的提交，那你就有大麻烦了。


#### 5.Git其他知识
##### 5.1.储藏stash
当你在一个分支上面已经做了一些事情，这个时候你想要切换到另外一个分支上去做一点事情，问题是，你不想因为这仅仅做了一半的事情而去做一个提交，这个时候你可以使用`git stash`或者`git stash save`命令。
![](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/65779532.png)
在这时，你能够轻易地切换分支并在其他地方工作；你的修改被存储在栈上。 要查看储藏的东西，可以使用`git stash list`。
![](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/66136029.png)
可以通过原来 stash 命令的帮助提示中的命令将你刚刚储藏的工作重新应用:`git stash apply`。 如果想要应用其中一个更旧的储藏，可以通过名字指定它，像这样:`git stash apply stash@{0}`。 如果不指定一个储藏，Git 认为指定的是最近的储藏
![](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/66595036.png)
可以看到 Git 重新修改了当你保存储藏时撤消的文件。 在本例中，当尝试应用储藏时有一个干净的工作目录，并且尝试将它应用在保存它时所在的分支；但是有一个干净的工作目录与应用在同一分支并不是成功应用储藏的充分必要条件。 可以在一个分支上保存一个储藏，切换到另一个分支，然后尝试重新应用这些修改。 当应用储藏时工作目录中也可以有修改与未提交的文件 - 如果有任何东西不能干净地应用，Git 会产生合并冲突。
应用选项只会尝试应用暂存的工作 - 在堆栈上还有它。 可以运行 git stash drop 加上将要移除的储藏的名字来移除它。
![](http://dd089a5b.wiz03.com/share/resources/faadbdf1-640a-4283-961e-b46ac7a57c62/index_files/67055143.png)
也可以运行 git stash pop 来应用储藏然后立即从栈上扔掉它。
我们还可以从存储创建一个新的分支`git stash branch [branch-name]`。

##### 5.2git blame
如果你要查看文件的每个部分是谁修改的, 那么 git blame 就是不二选择. 只要运行'git blame [filename]', 你就会得到整个文件的每一行的详细修改信息:包括SHA串,日期和作者。

##### 5.3git cherry-pick
如果我在某个分支有一个提交，提交修改了几个文件，我想要把这一个提交放到别的分支去，那么应该怎么办呢，这个时候可以使用`git cherry-pick [commit id]`这个命令了。这个一般可以用来把一个分支的某个功能放到另外一个分支的。
注意：当执行完 cherry-pick 以后，将会生成一个新的提交；这个新的提交的哈希值和原来的不同，但标识名一样。