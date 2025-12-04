---
title: BUAA-OS Pre
catalog: true
date: 2025-02-23 22:28:29
subtitle: OS预习Linux&&Lab0总结
sticky: 3
lang: en
header-img: /img/header_img/newhome_bg.jpg
tags:
- OS
categories:
- OS
---

## 写在前面

> 道阻且长，行则将至。
>
> 2025年开学快乐！

​	正式开学前，OSpre已经不自觉地进入我的脑子里（对...对吗），pre中的篇幅较大，涉及方面较多，特为2025OSpre写一份复习总结，主要体现一些指导书的难点与自己的思考。:D

## 初识Linux系统

​	以前我对Linux系统的了解仅仅止步于会简单地操作与使用，本次预习的内容让我在Linux更加从容自在地学习、运用。下面来看一下我们学到了什么新要点吧~

### Linux基础操作	

#### 文件操作

​	在 Linux 的哲学下**一切皆文件**，包括目录、设备等等全部是以文件方式存在于操作系统中的。因此 ***文件操作*** 的对象为所有一版文件（包括目录）。

| 命令  | 用法                        | 作用                                                         |
| ----- | --------------------------- | ------------------------------------------------------------ |
| touch | *touch [选项] 文件名*       | 当文件存在时更新文件的时间戳，**当文件不存在时创建新文件。** |
| rm    | *rm [选项] 文件*            | **删除文件。**                                               |
| cp    | *cp [选项] 源文件 目标路径* | 将源文件（也可以是目录）**复制**为目标路径对应的文件（如果目标路径是文件）或复制到目标路径（如果目标路径是目录）。 |
| mv    | *mv [选项] 源文件 目标路径* | 将源文件（也可以是目录）**移动**为目标路径对应的文件（如果目标路径是文件）或移动到目标路径（如果目标路径是目录）。 |
| diff  | *diff [选项] 文件1 文件2*   | 对两个文件进行**比较**操作。                                 |

​	其中选项作为一个非必需的指令部分却也有着很重要的用处，可以提供丰富的功能同时，也会带来一定的风险。如`rm -r `为递归删除目录及其内容，`rm -f `为强制删除目录，`cp -r `为递归复制目录中的所有内容。值得一提的是，`mv`指令无需附加选项`-r`即可直接完成对目录下所有内容的移动。

​	对于编辑文件内部，我们使用的工具是**Vim**这款开源文本编辑器，教程说这是一款编辑效率极高的编辑器，给人一种功能强大的感受。篇幅有限，放上官方的Vim教程推荐供参考：https://coolshell.cn/articles/5426.html

#### 查找操作

​	查找操作中涉及两个相似但不相同的指令：`find` 和 `grep`。下面给出指令对比：

| 命令 | 用法                       | 作用                                                         |
| ---- | -------------------------- | ------------------------------------------------------------ |
| find | *find [路径] <选项>*       | 在给定路径下递归地查找文件，输出符合要求的文件的路径。如果没有给定路径，则在当前目录下查找。 |
| grep | *grep [选项] PATTERN FILE* | 输出匹配PATTERN的文件和相关的行。                            |

### 实用工具介绍

#### GCC（GNU Compiler Collection，GNU 编译器套件）

​	虽然GCC的用法相当简单，但其中的步骤相当复杂，即从源代码到执行文件，步骤图示如下：

![compile](c_compilation_process.jpg)

​	上图中展示了从源代码到执行文件的四个步骤：**预处理→编译→汇编→链接**，而现实中GCC为我们将这些程序封装起来降低了我们操作的复杂度。

#### make & Makefile

​	make是最常用的C语言项目构建工具之一，可将文件编译链接成可执行文件，基本格式如下：

```makefile
target: dependencies
    command 1
    command 2
    ...
    command n
```

​	其中的`dependencies`文件是必填的，如果不写却在下面的某`command`中使用到，可能会导致奇怪的编译错误。运行时，下面的`command`会在`make target`后显式地在shell中执行相应的命令，换句话说，所有的`command`都是`shell`语句，大胆地写就OK，唯一需要注意的是每个`command`前面必须有一个`Tab`，否则也会导致出错。

​	值得一提的是，下面是教程举的栗子：

```makefile
all: hello

hello: hello.c
    gcc -o hello hello.c
```

​	实际上，这里的`all`也只是一个*target*的名称，它的效果其实是执行后续的其他*target*，并没有其他的特殊意义，甚至可以用其它字符串替换，而将它放置在`hello`前，使得在这种情况下，以下三种命令是等价的：

```shell
$ make
$ make all
$ make hello
```

[^p.s.]: 笔者在完成Lab0时发现了一些Makefile的很好用的高级用法，希望以后能更新叭

#### tmux（Terminal multiplexer）

​	终端复用神器。

​	使用时需要用`Ctrl + B`做快捷键***引子***（表格以*代称），常用操作如表格所示：

| 快捷键                | 效果     |
| --------------------- | -------- |
| * + %                 | 左右分屏 |
| * + "                 | 上下分屏 |
| *＋Up/Down/Left/Right | 切换窗格 |
| * + Space             | 变换布局 |
| * + x                 | 关闭窗格 |
| * + d                 | 分离对话 |

​	当我们分离离开tmux后，可以使用下述代码返回：

```shell
tmux ls # 查询tmux的Num
tmux a -t <Num>
```

​	此外，笔者在tmux中遇到**无法滚动页面**的问题，实际上需要使用`Ctrl + B +[`即可进入滚动页面。

### 关于Shell脚本

#### Shell脚本执行

```shell
$ touch hello.sh # 创建新文件hello.sh
$ vim hello.sh # 进行编辑
$ chmod +X hello.sh # 添加执行权限
$ ./hello.sh # 执行hello.sh
```

​	其中值得注意的是Linux系统中文件并没有后缀一说，因此`.sh`作为文件类型的标识符也是文件名的一部分，与前面不可分割。在编辑脚本的过程中，我们需要首先指定脚本的解释器，如*Bash*。

```shell
#!/bin/bash
```

#### Bash Shell 语法基础

##### 1.变量

​	Shell本身是弱类型语言，定义普通变量时无需声明变量类型，此外还有脚本参数这种特殊的变量，**用于从脚本外部传递参数**，如`$1` 、`$2`，以及`$#`与`$*`两种特殊变量，分别代表**传递参数个数**以及**传递的全部参数**，下面笔者用部分代码体现参数的特性：

```shell
# shel1.sh
var_name=value # 定义变量<var_name>中间不能有空格
echo ${var_name} # ${var_name}即变量值，{}大多数情况可省去

# shel2.sh
#!/bin/bash
str="Hello, $1 and $2!" # 定义两个脚本参数，注意这里双引号，单引号将无法传递参数
echo $str

# 命令行
$ ./shel2.sh world OS # print: Hello, world and OS!
```

##### 2.条件与循环语句

​	`if` 语句块格式如下：

```shell
if condition1
then
    command11
    command12
    ......
elif condition2
then
    command21
    command22
    ......
else
    command31
    command32
    ......
fi # if的倒写
```

​	`while`语句块格式如下：

```shell
while condition
do
    command1
    command2
    ...
done # do语句的结束符
```

##### 3.函数的定义与调用

​	函数的定义如下：

```shell
function fun_name() { # function与()可选一省略
    body...
    return int_value;
}
```

### 内容编辑和选择输出的自动化

#### `sed`与`awk`的使用

​	对于`sed`指令，我理解为可以在shell中直接操作的“Vim”，提供了添加、删除、替换等功能，即可解放双手！实现编辑与输出的自动化！

​	下面具体举一些笔者实战用的比较多的用法：

```shell
# 用法：sed [选项] '命令' 输入文本
# 其中选项包括[-n]以及[-i],前者是只输出sed处理的内容，而后者是直接修改读取文档，当没有选项时，sed会将整个文件输出出来，并且不会直接修改文件。
# 常用指令如下：
touch tmp.txt # 内含有许多文本
sed -n '2p' tmp.txt # print row 2
sed -n '3d' tmp.txt # delete row 3
sed 's/str1/str2/g' tmp.txt # swap str2 by str1
```

​	这里值得注意的是，如果这句话在shell中包含形参出现需要打**单引号**保护参数，即：

```shell
# modify.sh
#!/bin/bash
sed 's/'$2'/'$3'/g' $1 > temp # 单引号保护参数
mv temp $1
```

​	此外令笔者印象深刻的是，在`Lab0`实验的`hello_os.sh`的题目中，要求将某文件的某几行输出到另一个文件中，题目的要求是*8，32，128，512，1024*行，笔者这里 ~~***想要偷懒***~~ 写成一行代码，却老是导致错误，最后唯一的正确写法如下：

```shell
sed -n -e '8p' -e '32p' -e '128p' -e '512p' -e '1024p' $1 > $2 
# -e表示多个命令一起执行，且不能只写一个
```

​	另一方面，便是在教程中略带神秘感的`awk`指令，感兴趣的读者可以前往[官方教程](https://www.runoob.com/linux/linux-comm-awk.html)，这里笔者只拿Lab0的题目举例，展现其分割的作用：

```shell
#!/bin/bash
touch "$3"
#First you can use grep (-n) to find the number of lines of string.
grep -n "$2" $1 > $3
#Then you can use awk to separate the answer.
awk -F: '{print $1}' $3 > temp && mv temp $3 #用':'分割，并取出第一个':'前面的字符串
```

#### 重定向与管道

​	其实在上文的不少代码中都有***重定向***的身影，我认为`sed/awk+重定向`简直是文件之间的直通车，可以在彼此间交换各种信息，非常的方便快捷，对于自动化大有帮助。对于重定向而言，需要注意两点：

- `>`其实是`1>`的简写，意为标准输出，如果想输出标准错误请使用`2>`
- `>`在实际操作中会先把重定向的目标清空，所以无法在重定向符号前后自引用，~~会发现什么都没有了~~，此外，如果想在目标文件中追加输出，请使用***追加重定向***，即`>>`。

​	而对于***管道***，它的用法更加直接，但是应用面较窄，形式为：

```shell
command1 | command2 | command3 | ...
```

​	其中，`|`即为管道，意思是将前一个`command`的输出，输入到后一个`command`的标准输入中。

​	给人的感觉是重定向的高阶用法，实际上也正是如此，所有管道都可以用重定向表示，对于上述的命令，我们可以用下述重定向平替：

```shell
command3 < <(command2 < <(command1))
# <(command1)：将 command1 的输出作为文件传递给 command2
# <(command2 < <(command1))：将 command2 的输出作为文件传递给 command3
```

## 后记

> **不经一番寒彻骨，怎得梅花扑鼻香。** 愿大家都能趁着自己一股心劲儿，在年轻的时候想做什么就做什么，谋事在人，成事在天，谋且试之，久练则成！
