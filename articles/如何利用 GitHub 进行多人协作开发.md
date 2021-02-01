
## 分支设置
在我的项目中，存在两个核心分支`dev`和`master`，可能有的项目还存在`release`。每个迭代流程中，都会将`dev`的代码合入到`master`。

日常开发时，开发流程基本如下。

1. 开发者从`dev`分支切出新分支作为开发分支。

```
git checkout dev
// 保证dev分支为最新
git pull
// 切出新的开发分支
git checkout -b feat-my-branch
```

2. 开发完成后，提交代码到远端，提交`Pull Request`，填写这个pr的相关信息（如提交了什么内容，为什么这样做等），准备合入到`dev`分支（一个良好的习惯是提交pr前，先自己review一遍自己的代码，减少他人review的成本）。
```
git add .
// 提交到远端前，减少分支上无意义的commit，commit msg尽量规范，符合团队要求
git commit -m "feat: my branch" 
git push --set-upstream origin feat-my-branch
```
![image](https://github.com/TGuoW/blog/blob/master/image/pr.jpg)

3. 经过`Code Review`后（CR过程中，如果有任何的建议或者讨论点，最好都在当前pr下讨论），代码合并到`dev`分支（为保证远端分支比较干净，合入后建议将当前分支删除）。

4. 之后，仓库主要维护者会按照正常迭代节奏将你的代码逐步合入到`master`分支。


多人开发时，可能在A开发过程中，B往`dev`合入了新的代码，那么A想要将B的代码拉取到自己的开发分支，通常有两种操作`merge`和`rebase`，两者各有优缺点。

## merge vs rebase

### merge

```
// 合并代码
git fetch
git merge origin/dev

// 可能会出现冲突，解决掉后
git commit
```
看一个完全是用merge进行代码合并的项目，分支线非常复杂

![image](https://github.com/TGuoW/blog/blob/master/image/git1.jpg)

优点：代码的历史记录不会被改变

缺点：分支线几乎不可读

### rebase 
```
// 合并代码
git fetch
git rebase origin/dev

// 可能会出现冲突，解决掉后
git rebase --continue

// 当然也可以不解决冲突，放弃此次rebase
git rebase --abort

```
而使用rebase进行代码合并的项目，整体的分支线非常的清晰明了。

![image](https://github.com/TGuoW/blog/blob/master/image/git2.png)

rebase 之后，可能会导致代码无法push上去，出现报错。此时就需要`git push -f`，但是这个要慎用，保证这个分支上只有你自己在开发，或者，你本地的代码是Ok的，可以覆盖远端的代码。

优点：分支线可读

缺点：会丢失commit原始数据，某些代码可能找人背锅的时候会找不到:D。此外，rebase时可能会出现多次冲突，此时需要一个一个解决。
