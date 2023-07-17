# 1.波浪线拓展

​ 波浪线`~`会自动扩展成当前用户的主目录

​ `~+`会扩展成当前所在的目录，等同于`pwd`命令

# 2.'?'字符拓展

​ `?`字符代表文件路径里面的任意单个字符，不包括空字符。比如，`Data???`匹配所有`Data`后面跟着三个字符的文件名。

```bash
# 存在文件 a.txt 和 b.txt
$ ls ?.txt
a.txt b.txt
```

# 3.'\*'字符拓展

​ `*`字符代表文件路径里面的任意数量的任意字符，包括零个字符。

```bash
# 存在文件 a.txt、b.txt 和 ab.txt
$ ls *.txt
a.txt b.txt ab.txt
```

​ 注意，`*`不会匹配隐藏文件（以`.`开头的文件）

​ 注意，`*`字符扩展属于文件名扩展，只有文件确实存在的前提下才会扩展。如果文件不存在，就会原样输出。

# 4.方括号扩展

​ 方括号扩展的形式是`[...]`，只有文件确实存在的前提下才会扩展。如果文件不存在，就会原样输出

```bash
# 存在文件 a.txt 和 b.txt
$ ls [ab].txt
a.txt b.txt

# 只存在文件 a.txt
$ ls [ab].txt
a.txt
```

​ 方括号扩展还有两种变体：`[^...]`和`[!...]`。它们表示匹配不在方括号里面的字符，这两种写法是等价的。

# 5.[start-end] 扩展

方括号扩展有一个简写形式`[start-end]`，表示匹配一个连续的范围。比如，`[a-c]`等同于`[abc]`，`[0-9]`匹配`[0123456789]`

```bash
# 存在文件 a.txt、b.txt 和 c.txt
$ ls [a-c].txt
a.txt
b.txt
c.txt

# 存在文件 report1.txt、report2.txt 和 report3.txt
$ ls report[0-9].txt
report1.txt
report2.txt
report3.txt
```

# 6.大括号扩展

​ 大括号扩展`{...}`表示分别扩展成大括号里面的所有值，各个值之间使用逗号分隔。比如，`{1,2,3}`扩展成`1 2 3`

```bash
$ echo {1,2,3}
1 2 3

$ echo d{a,e,i,u,o}g
dag deg dig dug dog

$ echo Front-{A,B,C}-Back
Front-A-Back Front-B-Back Front-C-Back
```

​ 注意，大括号扩展不是文件名扩展。它会扩展成所有给定的值，而不管是否有对应的文件存在。大括号内部的逗号前后不能有空格。

# 7.{start..end} 扩展

​ 大括号扩展有一个简写形式`{start..end}`，表示扩展成一个连续序列。比如，`{a..z}`可以扩展成 26 个小写英文字母。支持逆序

```bash
$ echo {a..c}
a b c

$ echo d{a..d}g
dag dbg dcg ddg

$ echo {1..4}
1 2 3 4

$ echo Number_{1..5}
Number_1 Number_2 Number_3 Number_4 Number_5
```

# 8.变量扩展

​ Bash 将美元符号`$`开头的词元视为变量，将其扩展成变量值。变量名除了放在美元符号后面，也可以放在`${}`里面。`${!string*}`或`${!string@}`返回所有匹配给定字符串`string`的变量名。

```bash
$ echo $SHELL
/bin/bash
```

# 9.子命令扩展

​ `$(...)`可以扩展成另一个命令的运行结果，该命令的所有输出都会作为返回值。

```bash
$ echo $(date)
Tue Jan 28 00:01:13 CST 2020
```

​ 还有另一种较老的语法，子命令放在反引号之中，也可以扩展成命令的运行结果。

```bash
$ echo `date`
Tue Jan 28 00:01:13 CST 2020
```

# 10.算术扩展

​ `$((...))`可以扩展成整数运算的结果

```bash
$ echo $((2 + 2))
4
```

# 11.字符类

`[[:class:]]`表示一个字符类，扩展成某一类特定字符之中的一个。常用的字符类如下。

- `[[:alnum:]]`：匹配任意英文字母与数字

- `[[:alpha:]]`：匹配任意英文字母

- `[[:blank:]]`：空格和 Tab 键。

- `[[:cntrl:]]`：ASCII 码 0-31 的不可打印字符。

- `[[:digit:]]`：匹配任意数字 0-9。

- `[[:graph:]]`：A-Z、a-z、0-9 和标点符号。

- `[[:lower:]]`：匹配任意小写字母 a-z。

- `[[:print:]]`：ASCII 码 32-127 的可打印字符。

- `[[:punct:]]`：标点符号（除了 A-Z、a-z、0-9 的可打印字符）。

- `[[:space:]]`：空格、Tab、LF（10）、VT（11）、FF（12）、CR（13）。

- `[[:upper:]]`：匹配任意大写字母 A-Z。

- `[[:xdigit:]]`：16 进制字符（A-F、a-f、0-9）。

# 12.量词语法

​ 量词语法用来控制模式匹配的次数。它只有在 Bash 的`extglob`参数打开的情况下才能使用

    ?(pattern-list)：匹配零个或一个模式。
    *(pattern-list)：匹配零个或多个模式。
    +(pattern-list)：匹配一个或多个模式。
    @(pattern-list)：只匹配一个模式。
    !(pattern-list)：匹配给定模式以外的任何内容。
