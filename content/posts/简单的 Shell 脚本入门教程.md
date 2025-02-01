---
date: '2022-01-05T09:08:33+08:00'
title: '简单的 Shell 脚本入门教程'
categories: ["简单入门"]
---

Shell脚本 运作方式与解释型语言相当，如果有语言基础，学起 Shell 脚本就非常容易，但是 Shell 与常见的语言不同，一些常见的函数在 Shell 中需要组合一些命令得以实现

# 工具推荐
Shell 似乎没有定制的 IDE，这里推荐 VS Code 搭配对应的插件：

1. shellman 智能提示和自动补全，在插件页面有介绍常用代码片段的触发关键词，作者在 [Shellman reborn](https://medium.com/@remisa.yousefvand/shellman-reborn-f2cc948ce3fc) 中写到了 Shellman 诞生的故事，挺有趣的
2. shellcheck 语法静态检查工具，插件安装后需要本地安装 shellcheck，参考 [shellcheck Installing](https://github.com/koalaman/shellcheck#installing)，Mac OS 可以使用 `brew install shellcheck`，这样在写 Shell 的时候，语法有误的地方就会以波浪线的方式提示
3. shell-format 代码整理，Win 快捷键：Alt + Shift + F，Mac OS 快捷键：option + shift + F
4. Code Runner 脚本运行，右键 `Run Code`，Win 快捷键：Ctrl + Alt + N，Mac OS 快捷键：control + option + N


# 运行 shell 脚本

新建脚本：`test.sh`
```shell
#!/usr/bin/env bash

# 使用echo 打印字符串或者变量
echo 'hello world'
```
可以用 Code Runner 运行，就会输出：`hello world`

在 Shell脚本 的第一行一般会写 `#!/bin/bash` 这个是 [Shebang](https://zh.wikipedia.org/wiki/Shebang)，`#!` 后面是解释器的绝对路径，脚本将用该解释器执行。还有一种写法是：`#!/usr/bin/env bash`，`/usr/bin/env` 是 env 命令的绝对路径，而 env 命令用于显示系统中已存在的环境变量，其中包含了 `$PATH` ，会在 `$PATH` 包含的目录依次找 `bash`，常见的命令行解释器有：sh ,bash ,zsh(Mac OS 默认解释器)

如果在 Linux 或 类Unix 下运行，有这么几种方式：
1. 先给脚本添加执行权限：`chmod +x test.sh`，然后运行脚本：`./test.sh`，这种方式执行会读取 Shebang，用指定的解释器执行脚本
2. `sh test.sh`，使用 sh 这个解释器执行脚本，当然也可以用其他执行，比如：`bash test.sh`。与第一种方式相同，当前的 shell 是父进程，生成一个子 shell 进程（子进程会继承父进程的环境变量），在子 shell 中执行脚本，脚本执行完毕，退出子 shell 回到当前 shell
3. source 点命令方式：`source test.sh` 等效于 `. test.sh`。source 让脚本在当前 shell 执行，不生成新的子进程。使用 source 执行脚本，脚本中对于环境变量的修改会作用于当前 shell，这就是为什么我们在修改了一些配置如：`~/.bashrc`，执行 `source ~/.bashrc` 后配置就生效了
4. exec 方式：有需要先给脚本添加执行权限：`chmod +x test.sh`，执行 `exec ./test.sh`，也是让脚本在同一个进程上执行不生成新的子进程，与 source 的区别就是，在脚本执行完成后进程会被结束


# 基础命令
可以按照 [[Bash Shell] Shell学习笔记](https://www.cnblogs.com/maybe2030/p/5022595.html) 学习，这篇文章讲的非常详细，本篇博客也是在学习这篇文章后写下的

## 获取输入
使用 `read` 命令，从标准输入流 (stdin) 获取输入
```shell
#!/usr/bin/env bash
read var
echo "${var}"
```

运行脚本，输入任意字符，回车确认，输入的值会赋值给变量 `var`，并打印出该变量
## 输出
```shell
#!/usr/bin/env bash
var=1
# 输出变量
echo ${var}
# 输出字符串 显示部分字符需要转义
echo "\"hello world\"" # "hello world"

# 换行使用 -e 参数：使转义字符生效
# 使用 \n 换行
echo -e "newline\n"
```

也可以让 shell 输出不同颜色的字符，可以参考：[shell脚本中echo显示内容带颜色](https://www.cnblogs.com/lr-ting/archive/2013/02/28/2936792.html)

```shell
#!/usr/bin/env bash
echo -e "\033[30m 黑色字 \033[0m" 
echo -e "\033[31m 红色字 \033[0m" 
echo -e "\033[32m 绿色字 \033[0m" 
echo -e "\033[33m 黄色字 \033[0m" 
echo -e "\033[34m 蓝色字 \033[0m" 
echo -e "\033[35m 紫色字 \033[0m" 
echo -e "\033[36m 天蓝字 \033[0m" 
echo -e "\033[37m 白色字 \033[0m" 
```

## 变量使用
```shell
# = 两边不能有空格
var="hello world"
num=100


# 在引用变量时，这种方式可以，但是推荐下面一种
echo $var
# 推荐在使用字符串变量时，在两侧加上双引号，否则如果变量字符串中存在空格，则字符串会被切分
echo "$var"
# 如果涉及字符串拼接，可以在变量名两侧加上花括号
echo "变量为: ${var}."

# 将变量设置为只读，再次修改会报错
readonly var
# var="wolrd"

# 删除变量，不能删除 readonly 修饰的变量
unset num
```

变量赋值时，变量名命名规则和其他语言类似，**注意变量赋值时 `=` 两边不能有空格**

使用时在变量名前加上 `$`，推荐所有的变量都使用 `${}` 的方式使用变量
## 运算
算术运算：Bash 原生不支持数学运算，可以使用 `awk` 和 `expr`

注意乘号需要加上转义：`\*`，而且**运算符两侧必须空格**
```shell
a=10
b=3
val=`expr $a + $b`
echo "a + b : $val"
val=`expr $a - $b`
echo "a - b : $val"
val=`expr $a \* $b`
echo "a * b : $val"
val=`expr $b / $a`
echo "b / a : $val"
val=`expr $b % $a`
echo "b % a : $val"
```
## 执行命令
$()与 ``（反引号）都可以用于执行命令，并会将执行的结果返回，shellcheck 推荐使用第一种 $() 的方式

```shell
#!/usr/bin/env bash
result=`date "+%Y-%m-%d"`
echo "${result}"

result=$(date "+%Y-%m-%d")
echo "${result}"
```

## 运算符
关系运算符只支持数字，如果字符串为数字也可以，关系运算符包括：

|  运算符   | 含义  |
|  ----  | ----  |
| -eq  | 等于 |
| -ne  | 不等于 |
| -gt  | 大于 |
| -lt  | 小于 |
| -ge  | 大等于 |
| -le  | 小等于 |

条件表达式必须放在 `[]` 中，并且 `[` 的右侧，和 `]` 的左侧必须留有空格

布尔运算符列表：
|  运算符   | 含义  |
|  ----  | ----  |
| !  | 非 |
| -o  | 或 (or) |
| -a  | 与 (and) |

```shell
#!/usr/bin/env bash

a="10"
b="3"
c=1

if [ ${a} -ne ${b} ]
then
    echo "相同"
else
    echo "不相同"
fi

if [ ${a} -gt ${b} -a ${b} -gt ${c} ]
then
    echo "a > b & b > c"
fi
```
其他常用判断：
1. 直接在 `[  ]` 中放字符串变量 如 `[ ${str} ]` 则就是判断 `str` 这个字符串是否非空
2. -f 判断是否为普通文件，如：`[ -f $file ] `
3. -d 判断是否为文件夹，如：`[ -d $file ] `

## 字符串截取
字符截取的格式：`${string: start :length}` 
索引从 0 开始，可以省略 `:length` 这样就截取到最后，注意空格要空在 `:` 后，否则可能提示：bad substitution
```shell
#!/usr/bin/env bash
string="hello world"
echo ${string: 1 : 3} # ell
# 截取到最后
echo ${string:1} # ello world
```

## 数组
```shell
#!/usr/bin/env bash
# 1. 定义数组：使用括号声明，用“空格”分隔开，也可以换行隔开
arr=(1 2 3)
strArr=(
"first"
"second"
)

# 2. 读取数组：通过下标读取，下标从 0 开始计算
echo "${arr[0]}"

# 使用 * 或者 @ 读取所有元素
echo ${arr[*]}
echo ${arr[@]}

# 读取数组长度 读取全部元素前面加上 #
echo ${#arr[*]}
echo ${#arr[@]}

# 遍历下标
for(( i=0;i<${#strArr[@]};i++)) 
do
echo ${strArr[i]};
done;

# for in 遍历元素
for element in ${strArr[*]}
do
echo $element
done

# 3. 修改数组元素
strArr[0]="modify"
echo ${strArr[0]}

# 4. 删除元素
unset arr[1]
echo ${#arr[*]}
echo ${arr[*]} # 1 3
# ！使用 unset 要注意，这其实并不是真正删除了该元素，而只是将该元素置空，所以使用下标遍历会出问题，如下
echo "数组遍历："
for(( i=0;i<${#arr[@]};i++)) 
do
echo "index ${i} -> ${arr[i]}";
done;
# index 0 -> 1
# index 1 -> 

# 解决 unset 无法真正删除的方法：重新赋值给新的数组
echo "数组遍历："
arr=( "${arr[@]}" )
for(( i=0;i<${#arr[@]};i++)) 
do
echo "index ${i} -> ${arr[i]}";
done;
# index 0 -> 1
# index 1 -> 3
```

## 判断语句
使用 `if` 和 `fi` 定义判断的边界，使用 `then` , `elif` , `else` 定义条件
```shell
#!/usr/bin/env bash
#!/usr/bin/env bash

a=10
b=20
if [ $a == $b ]
then
    echo "相等"
else
    echo "不相等"
fi

if [ $a == $b ]
then
    echo "相等"
elif [ $a -lt $b ]
then
    echo "a 小于 b"
else
    echo "其他情况"
fi
```

## 函数
调用函数时，我们可以传入参数，可以通过 `$n` 来获取参数，这里的 `n` 表示 需要取的参数的索引，当n>=10时，需要使用${n}来获取参数

`$#` 传递给函数的参数个数，`$*` 和 `$@` 显示所有传递给函数的参数，`$?` 表示函数的返回值，也可以用于获取上一个命令的退出状态，执行成功会返回 0，失败返回 1

```shell
# 定义函数
#!/usr/bin/env bash
funWithParam(){
    echo "参数个数：$#"  # 参数个数：11
    echo "传递给函数的所有参数：$*" # 传递给函数的所有参数：1 2 3 4 5 6 7 8 9 34 73
    echo "$1" # 1

    # 超过 9 的参数需要用 ${} 接收参数，否则直接显示数值
    echo "$10" # 10
    echo "${11}" # 73  
}

# 调用函数：函数名后面直接跟上参数
funWithParam 1 2 3 4 5 6 7 8 9 34 73
echo "$?" # 0
```

## 输入输出重定向
使用 `>` 将应该输出到终端上的数据重定向输出到文件，`>` 默认为覆盖文件，使用 `>>` 追加写入文件
使用 `<` 将默认从键盘输入的数据，定向为从文件输入
```shell
# who 命令用于显示系统中有哪些使用者正在上面
# 将结果输入 who.txt
who > who.txt

# wc -l 作用是计算文本行数
wc -l < who.txt
```

一般情况下，每个 Unix/Linux 命令运行时都会打开三个文件：
1. 标准输入 (stdin)：stdin 的文件描述符为 0，Unix 程序默认从 stdin 读取数据
2. 标准输出 (stdout)：stdout 的文件描述符为 1，Unix 程序默认向 stdout 输出数据
3. 标准错误输出 (stderr)：stderr 的文件描述符为 2，Unix 程序会向 stderr 流中写入错误信息

所以一般我们后台启动应用并且输出日志文件都使用：
```shell
nohup java -jar xxx.jar >> nohup.log  2>&1 & 
```
`nohup`：(no hang up) 保证在**退出帐户**或者**关闭终端**之后继续运行相应的进程
`>> nohup.log`：将 `java -jar xxx.jar` 的输出追加到 `nohup.log` 文件
`2>&1`：将 `java -jar xxx.jar` 的 标准错误输出 也重定向到 标准输入
`&`：让进程在后台运行



默认情况下，command > file 将 stdout 重定向到 file，command < file 将stdin 重定向到 file。
如果希望 stderr 重定向到 file，可以这样写：

## 坑梳理
1. 变量赋值时，变量名命名规则和其他语言类似，**注意变量赋值时 `=` 两边不能有空格**
2. 数组 unset 元素，并不是真正的移除元素
3. 获取参数时，当 n>=10 时，需要使用${n}来获取参数

## 常见的特殊 Shell 环境变量
- `$$` 表示当前Shell进程的ID，即pid
- `$0` 表示当前脚本的绝对路径
- `$#` 传递给脚本或函数的参数个数
- `$n` 传递给脚本或函数的参数
- `$?` 上个命令的退出状态
- `$*` 和 `$@` 传递给脚本或函数的所有参数
- `$n` n 代表 1~9 其中任意一个数字，传递给脚本或函数该位置的参数

`$*` 和 `$@` 区别：

```shell
#!/usr/bin/env bash
function asterisk () {
    echo "\"\$*\""
    for var in "$*"
    do
        echo "$var"
    done
}

function mail () {
    echo "\"\$@\""
    for var in "$@"
    do
    echo "$var"
    done
}
asterisk a b c 
mail a b c
```
输出
```shell
"$*"
a b c
"$@"
a
b
c
```
当 `$*` 和 `$@` 直接使用效果相同，都是接收一份数据如上所示的例子，接收到的就是：`a b c`，一份数据，以空格隔开。加了双引号后 `"$@"` 会将每个参数都当成一份独立的数据



# 参考资料
[VS code 打造 shell脚本 IDE](https://zhuanlan.zhihu.com/p/199187317)
[#!/bin/bash 和 #!/usr/bin/env bash 的区别](https://blog.csdn.net/qq_37164975/article/details/106181500)
[Shell脚本 - wiki](https://zh.wikipedia.org/wiki/%E5%A4%96%E5%A3%B3%E8%84%9A%E6%9C%AC)
[Linux跑脚本用sh和./有什么区别？](https://www.zhihu.com/question/41441630)
[执行shell脚本三种方法的区别：（sh、exec、source）](https://blog.csdn.net/kouryoushine/article/details/91361718)
[Shell特殊变量：Shell $0, $#, $*, $@, $?, $$和命令行参数](http://c.biancheng.net/cpp/view/2739.html)
[exec 跟 source 差在哪？](https://wiki.jikexueyuan.com/project/13-questions-of-shell/exec-source.html)
[bash - 如何删除数组中的元素，然后在 Shell 脚本中移动数组？](https://www.coder.work/article/2568469)
[nohup /dev/null 2>&1 含义详解](https://blog.csdn.net/u010889390/article/details/50575345)
[Linux—shell中$(( ))、$( )、``与${ }的区别](https://www.cnblogs.com/chengd/p/7803664.html)
[Shell $*和$@的区别](http://c.biancheng.net/view/807.html)