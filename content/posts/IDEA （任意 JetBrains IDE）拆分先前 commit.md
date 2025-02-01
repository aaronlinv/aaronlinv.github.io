---
date: '2024-03-27T08:50:33+08:00'
title: 'IDEA （任意 JetBrains IDE）拆分先前 commit'
categories: ["Git"]
---

最近在合并上游代码，遇到了一个问题：某个 commit 杂糅了几个不同的特性修改，这可能会导致 rebase 上游代码时需要再对该 commit 进行额外的代码冲突处理

解决方法：合并上游分支前，拆分杂糅的 commit，并将其中不同的特性修改合并（Squash）回相关的 commit。可以直接通过命令行进行操作，可以参考：[Break a previous commit into multiple commits](https://stackoverflow.com/a/6217314)。也可以通过 JetBrains 家内置的 Git 进行操作，下面会介绍 IDEA 图形化操作的方法

## 非先前 commit 的拆分

对于刚提交的 commit，要拆分多个 commit 是非常容易的，因为我们只要 `soft reset` commit，将 commit 内容撤销回至 `暂存区`，就可以随意提交 commit

如果对于 `soft reset` 不太了解，可以参考我之前的博客：[Git 中的回退操作：reset 和 revert ](https://www.cnblogs.com/aaronlinv/p/16454183.html)

## 先前 commit 的拆分

先前 commit 指的是：在目标 commit 后已经有了若干个 commit。它无法直接通过 `soft reset` 进行拆分，因为这样会丢失后续的 commit，如下图，我们需要拆分 `B` commit，我们就无法直接使用 `soft reset` ，因为这样会丢失 `C` 和 `D` commit 的修改

所以我们需要使用 rebase，具体步骤：
1. 在 交互式 (interactive) rebase 中将 `B` 标记为 `edit`，这时 `B` 后面的 commit 会被暂时隐藏起来
2. 使用 `soft reset` 将 `B` 撤销回 `暂存区`
3. 将 `B` 的修改内容分多个 commit 提交 `B1` 和 `B2`
4. 使用 rebase 的 `continue` 将刚才隐藏的 `C` 和 `D` 恢复回来，需要注意的是：因为之前的 commit 记录已经改变了，所以这时的 `C` 和 `D` 已经与原来的 commit 记录不相同，故标记为 `C'` 和 `D'`
 
![](../IDEA（任意JetBrainsIDE）拆分先前commit/1929786-20240406133647807-70803179.png)



### 实例准备
演示使用 IDEA，其实 JetBrains 家的使用逻辑差不多，示例仓库使用：[Learn Go with Tests](https://github.com/quii/learn-go-with-tests) 

```shell
git clone git@github.com:quii/learn-go-with-tests.git
```

需要用到 Rebase，IDEA 默认保护主分支，改写 commit 记录的功能会被禁用

![](../IDEA（任意JetBrainsIDE）拆分先前commit/1929786-20240326173139514-293363336.png)

需要先取消分支保护，移除 `Protected branches` 中的 `master;main`

![](../IDEA（任意JetBrainsIDE）拆分先前commit/1929786-20240326173325200-470557609.png)

假设我们需要将下面 commit 拆分为两个：
![](../IDEA（任意JetBrainsIDE）拆分先前commit/1929786-20240326173549906-1525247286.png)

### 1. 启动 Rebase 并标记目标 commit 为 edit

点击 `Interactively Rebase from Here...`
![](../IDEA（任意JetBrainsIDE）拆分先前commit/1929786-20240326173657561-355272434.png)

选择需要拆分的 commit，右键选择 `Stop to Edit`，然后再点击 `Start Rebasing`
![](../IDEA（任意JetBrainsIDE）拆分先前commit/1929786-20240326173955830-407995748.png)

这时有右下角会提示您正在处于 `Rebase` 状态
选择框可以选择 `Continue` 即继续 Rebase，`Abort` 则会退出 Rebase
![](../IDEA（任意JetBrainsIDE）拆分先前commit/1929786-20240326174502325-994025394.png)

commit 列表也会显示感叹号
![](../IDEA（任意JetBrainsIDE）拆分先前commit/1929786-20240326180325399-260917660.png)

### 2. 使用 soft reset 将 commit 撤销回 暂存区

我们需要先 soft reset 到目标 commit 的上一个 commit
选择上一个 commit，右键选择：`Reset Current Branch to Here...`
![](../IDEA（任意JetBrainsIDE）拆分先前commit/1929786-20240326181826922-371872963.png)

选择 `Soft`，这样目标 commit 的修改就会退回到暂存区
![](../IDEA（任意JetBrainsIDE）拆分先前commit/1929786-20240326181955939-1262699884.png)
![](../IDEA（任意JetBrainsIDE）拆分先前commit/1929786-20240326200723734-869876115.png)

### 3. 将暂存区改动分为若干个 commit 提交

这个时候我们就可以继续分开提交两次 commit
![](../IDEA（任意JetBrainsIDE）拆分先前commit/1929786-20240326200837139-649603743.png)
![](../IDEA（任意JetBrainsIDE）拆分先前commit/1929786-20240326200840847-260504442.png)

查看 log，两条 commit 被成功提交
![](../IDEA（任意JetBrainsIDE）拆分先前commit/1929786-20240326201317357-1477966212.png)

### 4. 使用 rebase 的 continue 恢复剩下的 commit

这时候我们需要继续 Rebase，将剩下的 commit 还原回去：点击右下角分分支按钮，选择 `Continue Rebase`
![](../IDEA（任意JetBrainsIDE）拆分先前commit/1929786-20240326204930715-1014809023.png)

完成 Rebase 后，剩余的 commit 就会被追加到我们新提交的 commit 后面，至此我们就完成了先前 commit 的拆分
![](../IDEA（任意JetBrainsIDE）拆分先前commit/1929786-20240326204843891-646919102.png)

## Rebase edit 的拓展

因为这里的案例只是拆分 commit，没有对 commit 进行修改，如果是修改的话，修改完成后需要使用 `git add` 将文件标记为已修改，才能使用 rebase 的 `continue`，这样该 commit 就会被修改。后续的 commit 有可能与本次的改动产生冲突，需要手动处理冲突

## 参考资料

[Break a previous commit into multiple commits](https://stackoverflow.com/a/6217314)
