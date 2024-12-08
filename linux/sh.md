shell 有交互和非交互两种模式。

#### 交互模式

交互模式：用命令行

简单来说，你可以将 shell 的交互模式理解为执行命令行。



#### 非交互模式

执行脚本

可以使用下面的命令让 shell 以非交互模式运行：

```bash
sh /path/to/script.sh
bash /path/to/script.sh
source /path/to/script.sh
./path/to/script.sh
```



#### 解释器

`#!` 决定了脚本可以像一个独立的可执行文件一样执行，而不用在终端之前输入`sh`, `bash`, `python`, `php`等。

```bash
# 以下两种方式都可以指定 shell 解释器为 bash，第二种方式更好
#!/bin/bash
#!/usr/bin/env bash
```





#### 声明变量 与 访问变量

变量名外面的花括号是可选的，加不加都行，加花括号是为了帮助解释器识别变量的边界，所以推荐加花括号。

```bash
word="hello"
echo ${word}
# Output: hello
```



#### 变量类型

- **局部变量** - 局部变量是仅在某个脚本内部有效的变量。它们不能被其他的程序和脚本访问。（生命周期伴随脚本）
- **环境变量** - 环境变量是对当前 shell 会话内所有的程序或脚本都可见的变量。创建它们跟创建局部变量类似，但使用的是 `export` 关键字，shell 脚本也可以定义环境变量。（生命周期伴随shell会话）



#### 单引号和双引号

shell 字符串可以用单引号 `''`，也可以用双引号 `“”`，也可以不用引号。

- 单引号的特点
  - 单引号里不识别变量
  - 单引号里不能出现单独的单引号（使用转义符也不行），但可成对出现，作为字符串拼接使用。
- 双引号的特点
  - 双引号里识别变量
  - 双引号里可以出现转义字符

综上，推荐使用双引号。





### 7.1. 条件语句

跟其它程序设计语言一样，Bash 中的条件语句让我们可以决定一个操作是否被执行。结果取决于一个包在`[[ ]]`里的表达式。

由`[[ ]]`（`sh`中是`[ ]`）包起来的表达式被称作 **检测命令** 或 **基元**。这些表达式帮助我们检测一个条件的结果。这里可以找到有关[bash 中单双中括号区别](http://serverfault.com/a/52050)的答案。

共有两个不同的条件表达式：`if`和`case`。



#### if

（1）`if` 语句

`if`在使用上跟其它语言相同。如果中括号里的表达式为真，那么`then`和`fi`之间的代码会被执行。`fi`标志着条件代码块的结束。

```bash
# 写成一行
if [[ 1 -eq 1 ]]; then echo "1 -eq 1 result is: true"; fi
# Output: 1 -eq 1 result is: true

# 写成多行
if [[ "abc" -eq "abc" ]]
then
  echo ""abc" -eq "abc" result is: true"
fi
# Output: abc -eq abc result is: true
```

（2）`if else` 语句

同样，我们可以使用`if..else`语句，例如：

```bash
if [[ 2 -ne 1 ]]; then
  echo "true"
else
  echo "false"
fi
# Output: true
```

（3）`if elif else` 语句

有些时候，`if..else`不能满足我们的要求。别忘了`if..elif..else`，使用起来也很方便。

```bash
x=10
y=20
if [[ ${x} > ${y} ]]; then
   echo "${x} > ${y}"
elif [[ ${x} < ${y} ]]; then
   echo "${x} < ${y}"
else
   echo "${x} = ${y}"
fi
# Output: 10 < 20
```





### 7.2. 循环语句

循环其实不足为奇。跟其它程序设计语言一样，bash 中的循环也是只要控制条件为真就一直迭代执行的代码块。

Bash 中有四种循环：`for`，`while`，`until`和`select`。

#### `for`循环

`for`与它在 C 语言中的姊妹非常像。看起来是这样：

```bash
for arg in elem1 elem2 ... elemN
do
  ### 语句
done
```