title: Work Flow
date: 2015-12-02 22:24:50
tags:
- git
---

# Work Flow


## Git Flow

![](https://about.gitlab.com/images/git_flow/gitdashflow.png)

### 主要分支

核心库应该有两个无限生命周期的分支：

- master
- develop

![main-branches](http://nvie.com/img/main-branches@2x.png)

我们把`origin/master`分支作为production主分支，它的HEAD版本的代码总是处于随时可以release的状态。  

我们把`origin/develop`分支作为develop主分支，它的HEAD版本的代码总是处于最新的交付的开发状态。有时候也称之为**集成分支**。这也分支也是构建每日的晚间快照版本的来源。

当`develop`分支的源码达到一个稳定的状态并且准备release的时候，所有的修改应该合并回`master`分支，然后做一个release序号的标记(tag)。

所以，当每次修改合并回`master`的时候，就是一个新的production release版本。所以理论上，我们可以使用一个Git钩子脚本来进行自动构建然后发布到我们的生产环境。

<!--more-->

### 支持分支

并列于`master`和`develop`分支，我们的开发模型使用一些不同的支持分支来达到，团队成员间并行开发、简化功能追踪、准备production release、快速修复生产环境问题等目的。不同于主分支，这些分支的生命周期都是有限制的，所以最终它们会被删除。

我们会使用的分支类型：

- Feature分支
- Release分支
- Hoxfix分支

这些分支都有特殊的目的，并且有严格的规则规定每个分支的原始分支、最终合并回哪个分支。

#### Feature分支

分支来源：develop  
分支合并：develop  
命名规则：除了master, develop, release-\*, or hotfix-\*的任何命名

`Feature分支`用来为即将到来或者遥远未来的release开发新功能。当开始开发一个新功能时，这些功能将来会被纳入哪个release版本还不太清楚。
一个`Feature分支`的生存周期为，开始新功能的开发直到分支被合并回主干或者功能被抛弃。

![Feature branch](http://nvie.com/img/fb@2x.png)

`Feature分支`同时应该开发人员本地的repos里，而不是origin。

##### 合并回develop分支

```
$ git checkout develop
Switched to branch 'develop'
$ git merge --no-ff myfeature
Updating ea1b82a..05e9557
(Summary of changes)
$ git branch -d myfeature
Deleted branch myfeature (was 05e9557).
$ git push origin develop
```

当要合并的分支的所有节点是当前分支顶端节点的祖先节点时，Git默认使用fast-forward模式进行合并。

这里添加no-ff选项，有如下好处

- 版本演变更为清晰，可以清楚的看到哪些代码改动是和新功能相关的
- 去除新功能更为方便

![git merge --on-ff](http://nvie.com/img/merge-without-ff@2x.png)

#### Release分支

分支来源：develop  
分支合并：develop、master  
命名规则：release-*  

`release分支`的目的是为release做准备。它允许我们在最后时刻做一些细小的修改。他们允许小bugs的修改和准备发布元数据（版本号，开发时间等等）。通过这个分支来进行release版本的创建，可以让我们在完成当前准备release的开发工作后，立马着手下一个release版本的开发。

正是在`release分支`被创建的时刻就是给release指定版本号的最好时机。`develop分支`反应的时下一次release的代码，但是在`release分支`创建之前，我们并不能确定最终release的版本是0.3还是1.0。

##### 创建Release分支

```
$ git checkout -b release-1.2 develop
Switched to a new branch "release-1.2"
$ ./bump-version.sh 1.2
Files modified successfully, version bumped to 1.2.
$ git commit -a -m "Bumped version number to 1.2"
[release-1.2 74d9424] Bumped version number to 1.2
1 files changed, 1 insertions(+), 1 deletions(-)
```
当创建完分支后，我们切换到该分支，给他分配版本号。这个分支直到最后的release产品的发布是一直存在的。在这其间，对于这次release的bug都是在这个分支中进行修改，但是新功能的添加是不允许的。

##### 结束Release分支
当`release分支`最后可以进行releae产品时候，我们必须

- `release分支`合并回`master分支`
- 为了将来历史版本的管理，`master分支`的提交必须设置tag
- `release分支`作的改动需要合并回`develop分支`

```
$ git checkout master
Switched to branch 'master'
$ git merge --no-ff release-1.2
Merge made by recursive.
(Summary of changes)
$ git tag -a 1.2
```

```
$ git checkout develop
Switched to branch 'develop'
$ git merge --no-ff release-1.2
Merge made by recursive.
(Summary of changes)
```

#### Hotfix分支
分支来源：master  
分支合并：develop、master  
命名规则：hotfix-*  

![hof-fix](http://nvie.com/img/hotfix-branches@2x.png)

`hotfix`分支产生是为了解决production版本的预料之外的状态。比如说在production版本中有一个致命的bug在release阶段是没有发现的。

本质上是为了让团队的成员可以继续他们的开发工作的同时，另外的人可以单独地修复这个bug。

##### 创建Hotfix分支

```
$ git checkout -b hotfix-1.2.1 master
Switched to a new branch "hotfix-1.2.1"
$ ./bump-version.sh 1.2.1
Files modified successfully, version bumped to 1.2.1.
$ git commit -a -m "Bumped version number to 1.2.1"
[hotfix-1.2.1 41e61bb] Bumped version number to 1.2.1
1 files changed, 1 insertions(+), 1 deletions(-)
```
同时也需要设置新的版本号。

##### 结束Hotfix分支
当bug修复完之后，需要将分支代码合并回`master`和`develop`分支。

```
$ git checkout master
Switched to branch 'master'
$ git merge --no-ff hotfix-1.2.1
Merge made by recursive.
(Summary of changes)
$ git tag -a 1.2.1
```

```
$ git checkout develop
Switched to branch 'develop'
$ git merge --no-ff hotfix-1.2.1
Merge made by recursive.
(Summary of changes)
```

这里有一个例外就是，当合并`hotfix分支`的时候，如果有一个`release分支`存在，那么`hotfix分支`应该合并到这个分支，因为`release分支`最终会合并会`develop分支`。


Git flow提倡一个master分支和一个develop分支，同时也包括了feature分支、release分支、hotfixe分支。开发工作在develop分支进行，然后提交到release分支，最后合并到master分支。 有个问题就是现在很多团队都在尝试持续交付。持续交付意味着你的默认分支分支是在任何时候都是可以部署的。 这同时意味着hotfixed和release分支以及它们引进的一些礼节是可以避免的。 这些礼节的一个例子就是release分支的合并回develop分支。虽然一些工具可以解决帮助解决这类问题，但是开发人员还是只合并到了master分支而忘记合并会develop分支。根本的原因还是git flow有点过于复杂了。

## GitHub Flow

![](https://about.gitlab.com/images/git_flow/github_flow.png)

什么是GitHub Flow

- 任何在master分支上的东西都是可部署的
- 做任何新工作，从master分支创建一个描述性命名的分支(例如new-oauth2-scopes)
- 提交代码到本地的那个分支并且定期地push到服务器上的同一分支
- 当需要反馈或者帮助，或者分支已经准备好要合并，开启一个pull/merge request
- 当有人review过分支后，合并回master
- 一旦分支合并回master后，应该立即部署

### 任何在master分支上的东西都是可部署的

`master`分支应该是一直稳定的，部署或者从他创建分支都是安全的。如果你push了一些没有经过测试的代码并且破坏了构建，你就打破了整个开发的团队的准则，对此应该感到非常的糟糕。每个我们push的分支都应该测试过并被报告到讨论组，如果你没有在本地测试过，你可以push到一个topic分支（哪怕只有一次提交），然后让Jenkins来测试它。

你可以有一个`deploy`分支，每当部署时就把部署的代码push到这个分支，以记录部署的代码。当然也可以把部署的代码SHA值直接写入到webapp中。

### 从master分支创建一个描述性命名的分支

每当需要做任何工作的时候，你从稳定的`master`分支创建一个个描述性命名的分支，举几个例子，`user-content-cache-key`, `submodules-init-task` or `redis2-transition`。这么做有几个好处, 一是每天你fetch的时候，都可以看到每个人正在着手的主题，避免重复工作，二十放置一个分支一段时候重新着手工作，能够非常容易的回忆起来。

![](https://cloud.githubusercontent.com/assets/70/6769774/7988902c-d0a8-11e4-94c9-dc132461ffe4.png)

### 不断地push到命名分支

从部署的角度，唯一需要担心的是`master`分支。所有不是`master`的分支都是正在进行的工作，所以推送的服务器的命名分支不会破坏任何东西。

同时它确保了我们的工作的备份。更重要的是，让所有人处于一个持续的交流状态。一个简单的‘git fetch’，就能看到现在所有的TODO list的工作。

```
$ git fetch
remote: Counting objects: 3032, done.
remote: Compressing objects: 100% (947/947), done.
remote: Total 2672 (delta 1993), reused 2328 (delta 1689)
Receiving objects: 100% (2672/2672), 16.45 MiB | 1.04 MiB/s, done.
Resolving deltas: 100% (1993/1993), completed with 213 local objects.
From github.com:github/github
 * [new branch]      charlock-linguist -> origin/charlock-linguist
 * [new branch]      enterprise-non-config -> origin/enterprise-non-config
 * [new branch]      fi-signup  -> origin/fi-signup
   2647a42..4d6d2c2  git-http-server -> origin/git-http-server
 * [new branch]      knyle-style-commits -> origin/knyle-style-commits
   157d2b0..d33e00d  master     -> origin/master
 * [new branch]      menu-behavior-act-i -> origin/menu-behavior-act-i
   ea1c5e2..dfd315a  no-inline-js-config -> origin/no-inline-js-config
 * [new branch]      svg-tests  -> origin/svg-tests
   87bb870..9da23f3  view-modes -> origin/view-modes
 * [new branch]      wild-renaming -> origin/wild-renaming
```

### 在任何时候开启一个pull/merge request

GitHub有一个pull Request功能，通常被用来进行开源工作 - fork一个项目，修改，然后发送一个pull request给项目的维护者。 然后，它同时是一个非常好的内部reviwe系统。

![](https://cloud.githubusercontent.com/assets/70/6769767/5054b4ba-d0a8-11e4-8d38-548ecf157018.png)

如果在开发功能的过程中卡住了，又或者你是一个开发者并且需要设计者来review你的代码等等。你可以开启一个pull request，然后通过@username来cc别人寻求帮助。 

因为Pull Request功能可以在代码的任何一行进行评论，这点非常棒。同时非常的高效，甚至比你们在同一个开发间，可以直接到对方的电脑上看代码更高效，因为这样不需要同时占用两个人的时间，并且有一个非常好的记录方式。

如果一个分支打开太长的时间，你感觉和`master`有点不同步，你可以合并`master`分支的代码到topic分支来进行代码同步。

![](https://cloud.githubusercontent.com/assets/70/6769754/2162f69e-d0a8-11e4-8c98-d2bb581f7152.png)

### 当review过pull request后合并代码

当review过pull request后，并且分支通过了CI和测试，我们就可以把它合并回master分支进行部署。

### review后立即部署

最后，你的工作合并回`master`分支。这意味这如果你现在不部署，别人会基于它开始新的工作，并且下一次部署很有可能在下几个小时就会发生。 所以由于你也不希望别人发布任何你写的有问题的代码，请尽快部署。同时我们也倾向于确保合并的代码是稳定的，并且倾向于只发布自己的代码。


合并任何代码到`master`分支，并且经常性的部署，减少了"在库"代码的数量，合并符合continuous delivery的实践。

## 和Github Flow相似的工作流

Atlassian推荐的[a similar strategy][a-similar-strategy]会rebase功能分支。

![](http://atlassian.wpengine.netdna-cdn.com/wp-content/uploads/rebase-on-feature.gif)

为了避免合并回`master`分支可能产生冲突而导致的意外问题。我们值需要同步`master`分支，以让功能分支的版本领先于`master`分支，这样最终的合并不会产生任务问题。通过合并`master`分支来同步会产品很多无关性的提交。通过rebase可以让提交树更为清晰

```
git fetch origin
git rebase origin/master
```

或者有人同时在一个功能分支上工作，则需要同步它们的代码

```
git rebase origin/PRJ-123-awesome-feature
```

## Gitlab Flow

### Production分支

![](https://about.gitlab.com/images/git_flow/production_branch.png)

Github Flow假定每次你合并功能分支时都可以部署到生产环境。但是实际情况会有你无法控制准确的release时机，比如说iOS应用需要通过AppStore的验证。在这种情况下，可以创建一个`production`分支来反映部署的代码。你可以通过合并`master`分支的代码到`production`分支来部署一个新版本。版本控制系统的合并时间就是大致的部署时间。如果需要更精确的时间，可以通过脚本在每次部署时创建tag。

### Environment分支

![](https://about.gitlab.com/images/git_flow/environment_branches.png)

每一个环境自动更新`master`分支的代码是个好主意。只要这个情况，环境的名称可以和分支的名称不同。假设你有一个staging环境，pre-production环境和production环境。当有人想要部署pre-production环境时，他们只要创建一个从`master`分支到`pro-production`分支的merge request。线上代码的同步则通过合并`pre-production`分支到`production`分支。 这样提交往downstream分支流动的流程确保一切在所有环境上都测试过。

如果需要cherry-pick一个hotfix的提交，通常是在功能分支上进行开发然后合并到master，则先不删除功能分支。如果`master`分支没问题，然后合并到其他的分支。如果需要更多的手工测试，可以通过从feature分支发送merge request到downstream分支。一个`极端`的Environment分支就是为每个功能分支创建环境就像[Teatro][teatro]。

### Release分支

![](https://about.gitlab.com/images/git_flow/release_branches.png)

只有需要向外部世界release软件的情况下，你才需要release分支。在这种情况下，每个分支包含一个minor版本号(2-3-stable, 2-4-stable)。stable分支使用`master`分支作为起始点，并且尽可能晚地被创建（下一个minor版本创建之前）。通过尽量晚地创建分支，减少了应用bugfix到多个分支的次数。 如果可能，这些bugfix首先被合并回`master`分支，然后cherry-pick到release分支。这样你就不会忘记合并回`master`分支，然后在以后的版本再次遇到同样的bug了。这叫做 ‘upstream first’策略，同样被google和red hat尝试。

每当一个bugfix被包括在一个`release`分支，通过创建一个新的tag来创建补丁版本。一些工程还有一个stable分支指向最新的`release`分支。

在这个流程中，通常不会有一个`production`分支。

### 不要使用rebase重排提交

使用git你可以rebase`feature`分支来重排`master`分支上的提交。这可以避免合并`master`分支到`feature`分支的一次merge提交，创建一个漂亮的提交历史。然后当push到远程服务器后你就不应该再rebase提交了。当用rebase来更新分支时，你需要一遍又一遍的解决同样的冲突。有时，你可以通过reuse recorded resolutions (rerere) 来解决，但是不使用rebase的话，你只需要解决一次冲突。

有很多策略可以避免多次的merge提交。需要合并`master`分支有三个主要的原因：使用别人的代码，解决冲突，长期分支。对于需要使用别人代码的情况，可以通过cherry-pick提交来解决。对于解决冲突，则可以通过一些策略（例如修改CHANGELOG时，不要再文件尾部添加内容），尽量来避免冲突。对于长期分支应该则应该尽量分解任务来避免。


[a-similar-strategy]: http://blogs.atlassian.com/2014/01/simple-git-workflow-simple/
[teatro]: http://teatro.io/