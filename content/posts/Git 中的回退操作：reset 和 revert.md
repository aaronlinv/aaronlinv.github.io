---
date: '2022-07-08T08:36:33+08:00'
title: 'Git 中的回退操作：reset 和 revert'
categories: ["Git"]
---

Git 中回退有 `reset` 和 `revert`，这两个的区别就是是否保留更改记录

假设当前的提交情况是：`A <- B <- C <- D <- HEAD`，如下图：

![](../Git中的回退操作：reset和revert/1929786-20220707135502142-725016820.png)

<br/>

当前是 `D`，希望回退到 `A`，那我们可以使用 `reset` 命令，reset 后再看 git log 就会发现：`B <- C <- D` 宛如没有出现过，这适用于想完全舍弃 `A` 之后的修改

但是如果我们想保留 `B <- C <- D` 的修改记录，可能这三个 commit 的功能只是暂时用不到，以后可能还用到，或者可能当前分支是一个公共分支，`B <- C <- D` 可能已经被同步到了其他小伙伴电脑上，为了尽量避免代码冲突。这些情况就需要使用 `revert` 命令，这样会重新生成新的 commit，其中包含回退的记录（假设 `D` 这个 commit 是添加了一些代码，那么 revert `D` 的 commit 就是删除这些代码）

## reset

使用 `git reset A` ，reset 会修改 head 的指向，这样可以回滚到 `A`，默认使用的参数是 `--mixed`，这个参数决定了 `reset` 时 Git 该如何处理工作区和暂存区


一般地，我们对于代码的修改会体现在 `working tree`（工作区）的变动，通过 `git add` 添加即将要提交的文件到 `index`（暂存区），通过 `git commit` 再提交到 `repository`（本地仓库），如下图：

<br/>

![](../Git中的回退操作：reset和revert/1929786-20220707135908237-1667172116.png)
<br/>

查看帮助：`git help reset`
```bash
 git reset [<mode>] [<commit>]
     This form resets the current branch head to <commit> and possibly updates the index (resetting it to the
     tree of <commit>) and the working tree depending on <mode>. If <mode> is omitted, defaults to --mixed. The
     <mode> must be one of the following:

--soft
    Does not touch the index file or the working tree at all
--mixed
    Resets the index but not the working tree
--hard
    Resets the index and working tree
```

假设我们现在处在 `D`，那么分别会有三种 `reset` 的情况

- 我们执行 `git reset --soft A`，字面意思，轻柔地 reset，只将 `repository` 回滚到了 `A`，而 `working tree`、`index` 维持 `reset` 之前的状态，保持不变，这个时候直接执行 `commit`，这时候会得到一个和 `D` 修改内容相同的 commit `D'`（二者的 commit id 是不相同的），`--soft` 很适合呈现多次 commit 的修改的叠加效果，假设 `B`、`C`、`D` 都是针对某一个功能的修改，其中的 commit 可能修改了同一个文件，想整合这些 commit 都修改了哪些内容，就可以使用 `--soft` 从 `D` reset 到 `A`，那么 `B`、`C`、`D` 的修改都会出现在 `index` 中
<br/>
 ![](../Git中的回退操作：reset和revert/1929786-20220707114554953-2139323703.png)

<br/>

- 我们执行 `git reset --mixed A`，`repository`、`index` 会回滚，`working tree` 维持 `reset` 之前的状态，这个时候直接 commit，将无法提交，因为 `repository` 与 `index` 都被回滚了，二者是相同的，没有变化则无法提交 commit
<br/>
![](../Git中的回退操作：reset和revert/1929786-20220707114620124-972768789.png)

<br/>

- 我们执行 `git reset --hard A`，按照参数 `hard` 的字面意思，reset 的非常强硬，`repository`、`index`、`working tree` 都会回滚到 `A`，因为 `working tree` 工作区也回滚了，所以本地的所有修改也将丢失，`--hard` 相对来说比较危险，需要确保工作区没有需要保留的代码，`--hard` 适合的情况是对于当前的即将要提交的代码失去信心，准备推倒重来
<br/>
![](../Git中的回退操作：reset和revert/1929786-20220707114820141-1037749239.png)


## revert
如果想在回滚的同时保留 commit 记录，就需要使用 revert，revert 就是生成原 commit 逆向修改的 commit，从而实现会滚。当前是 `D`，希望回退到 `A`，就需要按顺序依次 revert `D`、`C`、`B` ：

```bash
git revert D

git revert C

git revert B
```
<br/>

![](../Git中的回退操作：reset和revert/1929786-20220707115137110-1323384794.png)

<br/>

每一次 revert 都会生成新的 commit，需要依次手动输入 commit message，也可以先 revert 最后集中 commit

```bash
git revert --no-commit D
git revert --no-commit C
git revert --no-commit B
git commit -m " Revert D C B"
```


使用 revert 需要注意，如果即将 revert 的 commit 是一个 merge commit，那么会出现错误


## 使用 reset 方式，创建回滚的 commit

如果需要保留回滚记录，但是需要 revert 的 commit 是 merge commit，那就必须手动指定 `mainline`，比较麻烦，可以用 reset 的方式创建回滚 commit，这种方式不受 merge 方式的影响：

```bash
git reset --hard A
git reset --soft D 

git commit -m " Revert D C B"
```

通过这种方式也能回滚回 `A`，并且生成一个新的 commit，其中包括了 `D`、`C`、`B` 的逆向修改：

1. 先 `reset --hard` 到 `A`，这时 `repository`、`index`、`working tree` 都会回滚到 `A`
<br/>
![](../Git中的回退操作：reset和revert/1929786-20220707114820141-1037749239.png)

2. 再 `reset --soft` 到 `D`，这时 `repository` 指向了 `D`，但是 `index` 和 `working tree` 还保持在 `A`，当前即将 commit 的就是 `B`、`C`、`D` 逆向修改的叠加
因为目前 head 已经指向了 `A`，所以通过 `git log` 无法查询到 `D` 对应的 `commit id`，此时可以通过 `git reflog` 查询到历史的提交记录
<br/>
![](../Git中的回退操作：reset和revert/1929786-20220707114929030-964492421.png)

<br/>

## 参考资料
[Pretty Git branch graphs](https://stackoverflow.com/a/1060886/19141665)

[git revert 用法](https://www.cnblogs.com/0616--ataozhijia/p/3709917.html)

[git reset soft,hard,mixed之区别深解](https://www.cnblogs.com/kidsitcn/p/4513297.html)

[What's the difference between git reset --mixed, --soft, and --hard?](https://stackoverflow.com/a/3528483/19141665)

[How can I revert multiple Git commits?](https://stackoverflow.com/a/1470452/19141665)

[Understanding Git — Index](https://medium.com/hackernoon/understanding-git-index-4821a0765cf)

---
相关阅读：[Git 常见操作梳理](https://www.cnblogs.com/aaronlinv/p/13611307.html)