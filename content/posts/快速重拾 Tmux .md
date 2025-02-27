---
date: '2023-04-13T08:43:33+08:00'
title: '快速重拾 Tmux'
categories: ["重拾系列"]
---


`Tmux` 是一个 Linux （Mac OS也支持）下的终端复用器，相较于 `Screen` 更为强大，但快捷键和操作逻辑也更复杂，一段时间不用，就很容易忘记相关的命令和快捷键。本文旨在通过一个简单的场景，快速重拾 Tmux

`Tmux` 通常用来保持会话（session），如果我们通过 ssh 连接服务器处理打包等的耗时操作，那么网络波动可能会导致连接断开，使得操作失败，使用 `Tmux` 会话会被保持，任务依然会继续，我们可以随时恢复会话

`Tmux` 另一个常用的功能是分屏，快速地创建 `Window` 和 `Pane`，方便地在不同的任务间穿梭

## 修改配置

```sh
vim ~/.tmux.conf
```

```sh
# 将默认修饰键（prefix） ctrl + b 修改：ctrl + a
set -g prefix C-a
unbind C-b
bind C-a send-prefix

# 激活鼠标模式
set-option -g -q mouse on

# 修改分屏快捷键
# 左右分屏
bind h split-window -h
# 上下分屏
bind v split-window -v

# 可以取消默认的分屏快捷键映射
# unbind '"'
# unbind %

# 将 tmux 的复制模式键绑定设置为 vi 模式
setw -g mode-keys vi

# windows 和 panes 的序号从 1 开始 
set -g base-index 1
setw -g pane-base-index 1
```

重新加载 Tmux 配置文件

```sh
tmux source-file ~/.tmux.conf
```

## 命令 & 快捷键

### 命令

这些命令大多是用于 tmux Session 的增删改查，一些命令进入 tmux 后将无法使用

```sh
# 创建新的 session
tmux new -s <session-name>
# 删除 seesion
tmux kill-session -t 0
# 重命名 seesion
tmux rename-session -t 0 <new-name>
# 查看 所有 session
tmux ls

# 进入最近使用的 session
tmux attach
# 进入编号为 1 的 session 
tmux attach -t 1
```

可以定义一些 alias 简化输入

```sh
# 添加到 shell 初始化脚本中
# Bash Shell 是 ~/.bashrc
# Zsh Shell  是 ~/.zshrc

alias tnew='tmux new -s'
alias tatt='tmux attach'
alias tkill='tmux kill-session -t'
alias tkillall='tmux kill-session -a'
alias tname='tmux rename-session -t'
alias tls='tmux ls'
```

### 快捷键

在使用下面的快捷键之前，都需要先按 tmux 的修饰键（prefix），修改后的修饰键为：Ctrl + a；具体做法是：先按住 Ctrl 再按一下 a，这时可以松开 Ctrl 和 a，这时 prefix 已经生效了，我们可以加上下面的任意按键以实现对应的功能

tmux 有 `Session`、`Window`、`Pane` 这三个比较重要的概念

#### 会话 Seesion

- d：分离会话（detach）
- $：修改当前 Session 名称
- s：显示 Session 列表（session）

#### 窗口 Window 

- c：创建一个新的 Window (create)
- p：切换到上一个 Window（previous）
- n：切换到下一个 Window（next）
- w：显示 Window 列表（window）
- ,：修改当前 Window 的名称
- 数字键：切换到对应编号的 Window，比如 prfix + 0 就是切换到编号为 0 的 Window

#### 窗格 Pane

- %：创建一个 Pane（水平排布），使用前面的配置后，可以使用 h（horizontal）
- "：创建一个 Pane（垂直排布），使用前面的配置后，可以使用 v（vertical）
- 空格：Pane 的垂直排布和水平排布之间相互转换
- x：移除当前 Pane，会出现提示是否需要 kill-pane，输入 y 确认，也可以使用 ctrl + d（无需按 prefix）直接终止 pane
- z：全屏当前 Pane
- ;：将光标移动到上次使用的 Pane
- o：将光标移动到下一个 Pane（顺时针）
- Ctrl + o：旋转当前窗口的pane，下一个 Pane 会代替上一 Pane 的位置，光标会保持在原 Pane
- Alt + 方向键：以 5 个单元格为单位移动边缘以调整当前面板大小

#### 复制文本

- [：进入复制模式，因为我们配置了 `setw -g mode-keys vi` 所以我们可以直接用 vim 的快捷键跳转单词或者行

我们可以通过 `空格键` 开始选中，这时移动光标可以扩大选取，按 `回车` 完成文本复制

- ]：粘贴复制的文本

进入复制模式后，可以通过 `q` 退出复制模式

## 场景

tmux 就像 vim 一样，如果不经常使用，就很容易忘记快捷键，可以通过一个场景把这些零碎的知识串起来，同时场景也方便重复练习和举一反三

我们可以在 tmux 里，编译运行一个 c 的 hello world，`prefix` 默认为 `Ctrl + b`，配置里我们修改为 `Ctrl + a`

1. 使用 tmux 创建新的 Session，并指定名称为：run-c

```sh
# 使用 alias 的话可以用 tnew run-c
tmux new -s run-c
```

2. 我们可以使用 `prefix + ,` 将 Windows 名称修改为 `hello-world`

3. 使用 vim 编辑 hello.c

```
vim hello.c
```
按 `i` 进入 vim 的编辑模式，输入：

```c
#include<stdio.h>

int main(){
    printf("hello world\n");
}
```

按`ESC` 退出编辑模式，键入 `:w` 保存输入

4. 使用 `prefix + %` （修改了配置则可以使用 prefix + h）在右侧添加一个新的 Pane 用于编译

5. 新增的 Pane 将屏幕一分为二，但是编译不需要这么大，我们可以通过 `prefix + Alt + 右方向键` 缩小 Pane 宽度，按完 prefix 后，可以多次按 `Alt + 右方向键` 持续缩小 Pane 宽度

6. 在右侧 Pane 我们可以使用 `gcc hello.c` 编译 hello.c

7. 使用 `./a.out` 运行 hello world 程序

8. 使用 `prefix + ;`，将光标切换回左侧 Pane，如果觉想暂时收起右侧的 `Pane`，可以用 `prefix + z`，最大化或取消最大化当前 `Pane`

9.  我们可以继续编辑文件，输入 `i` 进入 vim 编辑模式，将 `world`，修改为 `tmux`，按`ESC` 退出编辑模式，键入 `:w` 保存输入

10. 使用 `prefix + ;`，将光标切换回右侧 Pane，完成编译和运行

```sh
gcc hello.c
./a.out
```

10. 使用 `prefix + x`，关闭右侧 Pane，按 `y` 确认关闭

11. 使用 `prefix + d`（tmux detach），将当前会话与窗口分离，回到我们自己的 Shell

12. 使用 `tmux attach`（修改了配置则可以使用 `tatt`），回到我们刚出 detach 的 Session

## 参考资料

[tmux: some considerations, some best practices](https://github.com/datamade/how-to/blob/main/shell/tmux-best-practices.md)
[How to Boost 10X Productivity with Tmux](https://towardsdatascience.com/how-to-boost-10x-productivity-with-tmux-ead3d3d452f9)
[Tmux 使用教程 - 阮一峰](https://www.ruanyifeng.com/blog/2019/10/tmux.html)
[手把手教你使用终端复用神器 tmux](https://www.bilibili.com/video/BV1KW411Z7W3/)
[Tmux + Vim 工作流! 同时操作多个项目, 追求极致的丝滑流畅!](https://www.bilibili.com/video/BV1fK4y1Y7eG/)
[「TMUX」十分钟掌握 tmux -- 高效的终端复用工具 : )](https://www.bilibili.com/video/BV1ab411J7xT)
[十分钟掌握 TMUX](https://ovirgo.com/posts/tmux/)
[Y分钟速成X，其中 X=tmux](https://learnxinyminutes.com/docs/zh-cn/tmux-cn/)
[Tmux的快捷键,包括调整窗口大小](https://www.cnblogs.com/lovesKey/p/6741141.html)