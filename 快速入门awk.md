# 概述
`awk`的基本功能是在文件中搜索包含某些模式的行并执行对应的操作。作为数据驱动的工具，`awk`内置了一种解释型编程语言，这种语言的处理流程为：
* 描述要使用的数据的模式（`pattern`）
* 给出对应模式的处理方法（`action`）
* 在输入中寻找指定模式的数据
* 根据事先定义的处理`action`进行操作

这种类似`SQL`的执行方式导致`awk`适合被用来处理数据，命令的使用方式如下：
```shell
awk 'program' input-file1 input-file2 ...
```
如果处理方法（`program`）较长 可以将其放到脚本文件中，然后用`-f`选项加载：
```shell
awk -f program-file input-file1 input-file2 …
```
其中，处理方法的格式按照`pattern { action }`模式，我们看下面这个例子：
```shell
> cat test.awk
#!/usr/bin/awk -f
BEGIN { print "Don't Panic!"}
> chmod +x test.awk
> ./test.awk
Don't Panic!
```
在本例中我们新建了一个文件，将该文件的解释器指定为`awk -f`，然后在文件中写处理脚本，最后给文件赋予可执行权限并执行。其中，`BEGIN`就是`pattern`，他描述了我们要处理的数据；`print "Don't Panic!"`是`action`，他决定了找到数据之后应该执行什么操作；因此这个脚本最终完成的功能是在开始执行脚本时打印一个字符串。

>[warning]警告：`awk`中的引号遵循贪婪匹配原则，因此上述命令直接在命令行中运行会出错，需要使用特殊方法处理引号。

```shell
> awk ‘BEGIN { print "Don't Panic!"}’  # 会报错
> awk -v sq="'" 'BEGIN { print "Don" sq "t panic!" }'  # 通过变量
Don't panic!
> awk 'BEGIN { print "Don\47t panic!" }'  # 使用 ascii 码
Don't panic!
```

# 简单上手
我们先新建两个数据文件用于`awk`语法的学习：
```shell
> cat mail-list
Amelia       555-5553     amelia.zodiacusque@gmail.com    F
Anthony      555-3412     anthony.asserturo@hotmail.com   A
Becky        555-7685     becky.algebrarum@gmail.com      A
Bill         555-1675     bill.drowning@hotmail.com       A
Broderick    555-0542     broderick.aliquotiens@yahoo.com R
Camilla      555-2912     camilla.infusarum@skynet.be     R
Fabius       555-1234     fabius.undevicesimus@ucb.edu    F
Julie        555-6699     julie.perscrutabor@skeeve.com   F
Martin       555-6480     martin.codicibus@hotmail.com    A
Samuel       555-3430     samuel.lanceolis@shu.edu        A
Jean-Paul    555-2127     jeanpaul.campanorum@nyu.edu     R

> cat inventory-shipped
Jan  13  25  15 115
Feb  15  32  24 226
Mar  15  24  34 228
Apr  31  52  63 420
May  16  34  29 208
Jun  31  42  75 492
Jul  24  34  67 436
Aug  15  34  47 316
Sep  13  55  37 277
Oct  29  54  68 525
Nov  20  87  82 577
Dec  17  35  61 401

Jan  21  36  64 620
Feb  26  58  80 652
Mar  24  75  70 495
Apr  21  70  74 514
```
## 单模式
`awk`在处理文本时每次会读取一行，然后根据`pattern`判断当前行是否符合要求，如果符合则执行对应的`action`。如下例，我们打印匹配`/li/`的行的第一列内容：
```shell
> awk '/li/ {print $1}' mail-list
Amelia
Broderick
Julie
Samuel
```
在`pattern`中不仅可以用正则表达式，还可以使用表达式和内置的条件判断关键字：
```shell
# 使用表达式打印长度小于 35 的行
> awk 'length($0) < 35' /etc/passwd
root:x:0:0:root:/root:/bin/bash
sync:x:4:65534:sync:/bin:/bin/sync

# 使用条件判断关键字打印最长行的长度
> awk '{if (max < length($0)) max = length($0)} END {print max}' /etc/passwd
98  # 支持 if、else 关键字
> awk '{max = max < length($0) ? length($0) : max} END {print max}' /etc/passwd
98  # 三目运算符也支持
```
我们也可以配合管道使用：
```shell
# 接收管道输入
# 计算当前目录下所有文件的总大小
> ls -l | grep ^d | awk '{ x+=$5 } END { print "total bytes: " x}'
total bytes: 86016

# 重定向输出到管道
# 获得长度为 3 的用户名并排序
> awk -F: 'length($1)==3 {print $1}' /etc/passwd | sort
bin
irc
man
sys
tss
```

## 多模式
`awk`支持多个模式配合多个操作，同时支持多文件聚合，我们看下面这个例子：
```shell
# 查看多个文件中匹配正则 1 或正则 2 的行
> awk '/12/ {print $0} /21/ {print $0}' mail-list inventory-shipped 
Anthony      555-3412     anthony.asserturo@hotmail.com   A
Camilla      555-2912     camilla.infusarum@skynet.be     R
Fabius       555-1234     fabius.undevicesimus@ucb.edu    F
Jean-Paul    555-2127     jeanpaul.campanorum@nyu.edu     R
Jean-Paul    555-2127     jeanpaul.campanorum@nyu.edu     R
Jan  21  36  64 620
Apr  21  70  74 514

# 计算 11 月份的交易总量
> awk '/Nov/ {sum+=$5} END {print sum}' inventory-shipped 
577
```
>[warning]**警告**：每个`pattern`对应紧跟着的第一个`{ action }`，如果不写`pattern`则对所有行都执行紧跟着的`{ action }`。例如：`pattern { action1 } { action2 }`表示对满足`pattern`的行执行`action1`，并且对所有行执行`action2`。

# 高级用法
## 命令行选项
`awk`中最常用的选项为`-F`、`-f`和`-v`，其他选项可以参考[命令行选项](https://www.gnu.org/software/gawk/manual/gawk.html#Options)：
`-F`：用于指定分隔符，默认分隔符为空格。
```shell
> awk -F: '/^systemd/ {print $1}' /etc/passwd
systemd-timesync
systemd-network
systemd-resolve
systemd-coredump
```
`-f`：用于指定输入脚本。
`-v`：用于自定义变量。
```shell
> seq 5 | awk -v sum=-16 '{sum+=$0} END {print sum}'
-1
```

我们可以通过`-`或`/dev/stdin`显式指定标准输入作为要处理的文件，这种方式通常配合管道来使用：
```shell
> echo "ack" | awk "{print}" t1.awk
# 只能处理 t1.awk

> echo "ack" | awk "{print}" - t1.awk 
# 能共同处理管道传递来的数据和制定的文件

> cat t2.awk | awk "{print}" t1.awk /dev/stdin t3.awk
# 还可以控制文件和管道输入的处理顺序
```

## 参数
除了通过`-v`选项显式指定变量外，还可以直接在命令行中使用`var=value`的方式隐式传递变量给`awk`脚本，但是这两种方式在使用上有一些差异，如下例：
```shell
> cat showargs.awk
BEGIN {
    printf "A=%d, B=%d\n", A, B
    for ( i = 0; i < ARGC; i++ )
        printf "\tARGV[%d] = %s\n", i, ARGV[i]
}
END { printf "A=%d, B=%d\n", A, B }

> awk -f showargs.awk -v A=1 B=2 /dev/null
A=1, B=0
        ARGV[0] = awk
        ARGV[1] = B=2
        ARGV[2] = /dev/null
A=1, B=2
```
区别在于：在脚本的`BEGIN`阶段，通过`-v`显式指定的的变量会被赋值，而通过命令行隐式传递的参数不会被赋值，但在这个阶段我们可以完整地访问所有传递的文件名和隐式参数。隐式传递参数主要被用于针对不同的输入（文件）设置不同的分隔符格式的`RS`、`FS`、`OFS`或`ORS`变量：
```shell
> awk '/^[Dd]/{print $1,$2}' FS=: OFS=- /etc/passwd FS=" " OFS=% inventory-shipped 
daemon-x  # 输入分隔为冒号，输出分隔为减号
dnsmasq-x
Dec%17  # 输入分隔为空格，输出分隔为百分号
```

如果需要拦截选项作为参数，可以通过`--`停用所有默认命令行选项，这样就可以接收所有的选项：
```shell
cat myprog.awk 
BEGIN {
    for (i = 1; i < ARGC; i++) {
        if (ARGV[i] == "-v" || ARGV[i] == "-q")
            print "recived", ARGV[i]
        else if (ARGV[i] ~ /^-./) {
            e = sprintf("%s: unrecognized option -- %c",
                    ARGV[0], substr(ARGV[i], 2, 1))
            print e > "/dev/stderr"
        } else
            break
        delete ARGV[i]
        # 不删除的参数会被当做文件名
        # 例如： -q 被当做文件名会导致找不到文件报错
    }
}

> awk -f myprog.awk -v -q -p /dev/null
# 会报错 因为不符合 -v 的语法

> awk -f myprog.awk -- -v -q -p /dev/null
# 不会报错 因为使用了 -- 屏蔽默认选项
recived -v
recived -q
awk: unrecognized option -- p

> awk -f myprog.awk -q -v -p /dev/null
# 不会报错 因为 -q 不存在
# 因此 -q 之后的都被当做参数读取
recived -q
recived -v
awk: unrecognized option -- p
```

## 环境变量

* `AWKPATH`：`awk`可执行文件的位置。
* `AWKLIBPATH`：用于寻找`awk`预编译库。
* **其他环境变量**：参考[环境变量表](https://www.gnu.org/software/gawk/manual/gawk.html#Environment-Variables)。

## 退出状态码
* **正常退出**：退出状态码等于`C`语言中的`EXIT_SUCCESS`常量，通常为`0`。
* **发生错误**：发生语法错误则退出状态码等于`C`语言中的`EXIT_FAILURE`常量，通常为`1`。
* **严重错误**：找不到文件或其他`fatal`错误则退出状态码为`2`。
* **自定义状态码**：使用`exit 状态码`语句可以实现退出状态，为了保证对所有系统都兼容，建议取值范围为`[0, 126]`。
```shell
> echo "" | awk '{ exit 126 }'; echo $?
126  # 自定义退出状态码
```

## 模块
在`gawk`中实现了脚本的模块化，我们可以在脚本文件中通过`@include "文件名"`的方式引用其他脚本文件，支持嵌套引用：
```shell
> cat t[1-3].awk
BEGIN {
    print "This is script test1."
}
@include "t1.awk"
BEGIN {
    print "This is script test2."
}
@include "t2.awk"
BEGIN {
    print "This is script test3."
}

> gawk -f t3.awk 
This is script test1.
This is script test2.
This is script test3.
```
`gawk`同时支持自定义扩展库，可以通过`@load "库名"`或在命令行中`-l库名`的方式加载扩展库（参考：[扩展库的编写方法](https://www.gnu.org/software/gawk/manual/gawk.html#Dynamic-Extensions)）：
```shell
> gawk '@load "ordchr"; BEGIN {print chr(65) }'
A
> gawk -lordchr 'BEGIN { print chr(65) }'
A
```

# 配合正式则表达式
## 针对单列的正则
下面的例子展示了如何对文本的第`1`列进行正则匹配，选择所有第一列包含`J`的行：
```shell
> awk '$1 ~ /J/' inventory-shipped
Jan  13  25  15 115
Jun  31  42  75 492
Jul  24  34  67 436
Jan  21  36  64 620
```
上述写法等价于使用条件判断关键字：
```shell
> awk '{ if ($1 ~/J/) print }' inventory-shipped
```
使用`!~`可以对匹配取反：
```shell
> awk '$1 !~ /[DAJSM]/' inventory-shipped 
Feb  15  32  24 226
Oct  29  54  68 525
Nov  20  87  82 577

Feb  26  58  80 652
```

## 动态正则
我们可以通过定义正则变量实现动态正则：
```shell
> awk 'BEGIN {digits_regexp = "^[[:digit:]]+"} $0 ~ digits_regexp {print}' /etc/hosts
127.0.0.1       localhost
127.0.1.1       CT7GK

> seq 5 | awk 'BEGIN { reg = @/3/ } $0~reg {print}'
3
```

## 正则替换函数
`awk`中提供了两种正则替换函数，分别用于单个匹配替换和全局匹配替换：
```shell
> echo "hi! hi yourself" | awk '{ sub(/hi/, "howdy", $0); print}'
howdy! hi yourself

> echo "hi! hi yourself" | awk '{ gsub(/hi/, "howdy", $0); print}'
howdy! howdy yourself
```

# 模式与变量

## 条件控制
`awk`支持类似`C`语言的条件控制语法。
* 完全相同的：`if/else`、`while`、`do-while`、`for`、`switch`、`break`、`continue`。
* `exit`：退出并返回状态码作为`awk`的退出状态。
```shell
BEGIN {
    if (("date" | getline date_now) <= 0) {
        print "Can't get system date" > "/dev/stderr"
        exit 1
    }
    print "current date is", date_now
    close("date")
}
```
* `next`：结束当前记录的操作。
```shell
NF != 4 {
    printf("%s:%d: skipped: NF != 4\n", FILENAME, FNR) > "/dev/stderr"
    next
}
```
* `nextfile`：结束对当前文件的操作。
## 从 shell 中读取
`awk`可以读取在`shell`中定义的变量：
```shell
printf "Enter search pattern: "
read pattern
awk "/$pattern/ "'{ nmatches++ }
     END { print nmatches, "found" }' /path/to/data
```

## 特殊模式关键字
* `BEGIN`：针对单个输入文件，在所有输入被读取之前触发的模式，一般用于做初始化、输出头部信息等工作。
* `END`：针对单个输入文件，在所有输入被处理之后触发的模式，要注意`$0`和`NF`在这里不可用。
* `BEGINFILE`和`ENDFILE`：`gawk`扩展的模式关键字，区别于对每个文件都起作用的`BEGIN`和`END`，这两个扩展的关键字是对所有文件读取输入之前和处理输入之后触发的模式。
* 空模式：能够匹配所有模式。

## 变量的真假
* `真`：`true`、非零数、非空字符串。
* `假`：`false`、0、”“。

## 变量的种类
我们可以通过`typeof(x)`函数获得一个变量的种类：
* 数组：显示为`array`，还可以通过`isarray(x)`函数判断。
* 正正则表达式：显示为`regexp`。
* 未定义的变量：
```shell
> awk 'BEGIN { print (a==0 && a=="" ? "a is untyped" : "a has atype!"); print typeof(a)}'
a is untyped
unassigned
```
* 未赋值的变量：
```shell
BEGIN {
    # 创建了 a[1] 但没有给它赋值
    a[1]
    print typeof(a[1])  # unassigned
}
```
* 字符串变量：
```shell
> awk 'BEGIN { a="a string"; print typeof(a); b=a; print typeof(b)}'
string
string
```
* 数字变量：
```shell
> awk 'BEGIN { a=42.23; print typeof(a); b=a; print typeof(b)}'
number
number
```

## 变量的转换方式
`awk`会根据当前代码的上下文将变量的格式进行转换，例如，如果表达式`foo + bar`中的`foo`或`bar`的值恰好是字符串，则在执行加法运算之前，会将其转换为数字；如果数字值出现在字符串连接中，则它们将转换为字符串。
```shell
> awk 'BEGIN {two=2;three=3;print (two three) + 4}'
27
# 先进行字符串拼接 (two three) = "23"
# 然后计算加法 23+4=27
```
我们可以通过设置`CONVFMT`变量对转换为字符串的**浮点数**做出格式约束，要注意的是如果要转换的数字是整数则不起作用：
```shell
> awk 'BEGIN {CONVFMT = "%2.2f"; a = 12.1; b=1.0; print (a ""),(b "")}'
12.10 1
```
变量转换的特性在某些情况下会导致意料之外的结果：
```shell
> awk 'BEGIN {print -12 " " (-24); print -12 " " -24}'
-12 -24
-12-24 
# 在不加括号的情况下 -12 " " -24 等价于 -12 (" “ -24)
⇒ -12 (0 - 24)
⇒ -12 (-24)
⇒ -12-24

# 字符串强转数字为 0
> awk 'BEGIN {foo="a string"; foo=foo+5; print foo}'
5

# 比较字符串与数字
# 把数字强转为字符串然后根据字典顺序比较
> echo hello | awk '{ printf("%s %s < 42\n", $1,
>                            ($1 < 42 ? "is" : "is not")) }'
hello is not < 42
```

# 附录
[内置变量](https://www.gnu.org/software/gawk/manual/gawk.html#User_002dmodified)