# 行记录
`awk`使用内置变量`RS`用于分隔每一个记录（`record`），默认`RS=\n`，也就是每一行是一个记录。每个记录的内容使用内置变量`$0`存储，匹配到的分割内容使用`RT`存储，总记录数量使用内置变量`NR`存储。
```shell
> echo "record 1 AAAA record 2 BBBB record 3 " | awk '
        BEGIN {RS="\n|( [[:upper:]]+ )"}
        {printf "Record = %s, RT = [%s]\n", $0, RT }
        END {print NR}'
Record = record 1, RT = [ AAAA ]
Record = record 2, RT = [ BBBB ]
Record = record 3 , RT = [      # 这输出了一个换行符
]
3  # 总记录数
```

## 行分隔符

行分隔符可以被赋予以下类型的值：
* `RS == "\n"`：默认情况，每一行是一个记录。
* `RS == 任意字符`：以这个字符作为记录分隔符。
* `RS == regexp`：已匹配正则表达式的结构作为记录分隔符。
* `RS == ""`：以空行作为记录分割，如下例：
```shell
> cat addr.data 
Jane Doe
123 Main Street
Anywhere, SE 12345-6789

John Smith
456 Tree-lined Avenue
Smallville, MW 98765-4321

> cat addr.awk 
BEGIN { RS = "" ; FS = "\n" }
# 每一行为一个字段，以空行作为记录分隔符
{
      print "Name is:", $1
      print "Address is:", $2
      print "City and State are:", $3
      print ""
}

> awk -f addr.awk addr.data 
Name is: Jane Doe
Address is: 123 Main Street
City and State are: Anywhere, SE 12345-6789

Name is: John Smith
Address is: 456 Tree-lined Avenue
City and State are: Smallville, MW 98765-4321
```

## 多行输入

| 用法 | 会改变的变量 | `awk`/`gawk` |
| --- | --- | --- |
| `getline` | `$0`,`NF`,`FNR`,`NR`, `RT` | `awk` |
| `getline var` | `var`,`FNR`,`NR`, `RT` | `awk` |
| `getline < file` | `$0`,`NF`, `RT` | `awk` |
| `getlinevar<file` | `var`, `RT` | `awk` |
| `command | getline` | `$0`,`NF`, `RT` | `awk` |
| `command | getline var` | `var`, `RT` | `awk` |
| `command |& getline` | `$0`,`NF`, `RT` | `gawk` |
| `command |& getline var` | `var`, `RT` | `gawk` |
`getline`方法用于从输入中读取下一个记录（默认为行），其不带参数的用法如下：
```shell
# 移除文件中 /* */ 之间的注释。注意：可能为多行注释。
{
    while ((i = index($0, "/*")) != 0) {
        out = substr($0, 1, i - 1)  # 注释前的部分
        rest = substr($0, i + 2)    # ... */ ... 注释段
        j = index(rest, "*/")       # 用于判断当前行是否有 */ 注释尾 
        if (j > 0) {
            rest = substr(rest, j + 2)  # 直接删除单行注释
        } else {
            while (j == 0) {
                # 逐行读取
                if (getline <= 0) {
                    print("unexpected EOF or error:", ERRNO) > "/dev/stderr"
                    exit
                }
                # 拼接字符串
                rest = rest $0
                j = index(rest, "*/")   # 用于判断当前行是否有 */ 注释尾 
                if (j != 0) {
                    rest = substr(rest, j + 2)
                    break
                }
            }
        }
        # 拼接字符串
        $0 = out rest
    }
    print $0
}

# var1 = var2 var3 这么写用于拼接变量
> awk '{ if (NR < 2) { $(NF+1) = $2 $3; print} }' inventory-shipped 
Jan 13 25 15 115 1325
```

通过`getline var`可以将下一个记录加载到`var`中：
```shell
> seq 5 | awk '{ if ((getline tmp) >0 ) { print tmp; print $0 } else { print $0 } }'
2  # 将下一个记录赋值给 tmp
1  # 当前记录 $0
4
3
5  # 读取下一个记录失败，进入 else
```

此外，我们可以通过管道将命令的执行结果传递给`getline`。`awk`中的管道概念和`shell`的管道类似，都是使用`|`符号。如果在`awk`程序中打开了管道，必须先关闭该管道才能打开另一个管道，也就是说一次只能打开一个管道。：
```shell
> awk 'BEGIN { "date" | getline current_time; close("date"); print "Report on:", current_time} '
Report on: Tue 10 Mar 2020 01:17:08 PM CST

> echo "@execute who" | awk '{ if ($1 == "@execute") { $2 | getline; print } }' 
remilia  tty1         2020-03-10 10:53 (:0)
```

`getline`还可以从外部文件中读取记录，并且同样可以将记录加载到变量中：
```shell
> seq 5 | awk '{ if ($1 == 2) { getline < "/etc/hosts"; print $0 } else { print } }'
1
127.0.0.1       localhost  # 只读了一行
3
4
5

# 通过变量循环加载并打印
> seq 5 | awk '{ if ($1 == 2) { while ((getline line < "/etc/hosts") >0 ) print line } else { print } }'
1
127.0.0.1       localhost
127.0.1.1       CT7GK
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
3
4
5

# 当然也可以处理带有特殊标记的记录
# 下面这个例子用于打印使用 @include 导入的文件
{
     if (NF == 2 && $1 == "@include") {
          while ((getline line < $2) > 0)
               print line
          close($2)  # 不关闭的话多次打开同一个文件会无效
     } else
          print
}

# 这种会打印 5 次 hosts 文件的全部内容
> seq 5 | awk '{ while ((getline line < "/etc/hosts") >0 ) print line; close("/etc/hosts")}'

# 这种就只会打印 1 次，因为没有关闭，重复加载不被记录
> seq 5 | awk '{ while ((getline line < "/etc/hosts") >0 ) print line }' 
```

`getline`同样可以配合数组使用：
```shell
> awk 'BEGIN { system("echo 1 > f"); while ((getline a[++c] < "f") >0 ) {}; print c}'
2
# 第一次成功读取，因此 ++c 导致 c 的值为 1
# 第二次读取失败，但是 ++c 仍然被执行，导致 c 的值为 2
```

# 列字段
`awk`使用内置变量`FS`用于分隔每一个列字段（`field`），默认`FS=" "`或`FS=[[:space:]]+`，也就是连续的空白字符，但要注意的是，每个记录前后的空白字符会被忽略。每个记录中的每个字段标号的取值范围为`[1, NF]`，我们可以通过`$1`或`$NF`或`$(2*1)`的方式访问对应的字段，其中`NF`记录了当前记录中字段的总数。
```shell
> awk 'BEGIN {FS="[[:space:]]+"} /^A/ {print $1, $(2*1), $NF}' mail-list 
Amelia 555-5553 F
Anthony 555-3412 A
```
## 修改列字段内容
我们可以通过`var=$N`将字段的值**拷贝**给变量，赋值过程会在内存中开辟一段空间将`$N`的值完整写入，然后让`var`指向这段空间，我们看下面这个例子：
```shell
> awk ' { nboxes = $3; $3-=10 } {print nboxes, $3} ' inventory-shipped
25 15
32 22
24 14
...
```
可以看出，对`$3`的修改不会影响已经被赋值的`nboxes`，但是这并不代表每次修改都是独立的，我们看下面这个例子：
```shell
> awk '/J/ { $(NF+1)=$2; $2+=5; print $0 }' inventory-shipped 
Jan 18 25 15 115 13
Jun 36 42 75 492 31
Jul 29 34 67 436 24
Jan 26 36 64 620 21
```
我们新建了一个原本不存在的列`$(NF+1)`并将`$2`的值拷贝给它，然后我们对`$2`进行修改，最后通过`$0`访问整个记录。从结果来看，我们对任何列的修改最终会体现在当前记录上，我们再看下面的例子：
```shell
> awk '/J/ { $(NF+1)=($2+$3+$4); $2=0; print $0}' inventory-shipped 
Jan 0 25 15 115 53
Jun 0 42 75 492 148
Jul 0 34 67 436 125
Jan 0 36 64 620 121
```
可以看出，输出中的`$2+$3+$4`和`$6`并不相同，这也说明给变量赋值后修改列字段的值并不会影响原变量。但是修改列字段的个数会反馈到`NF`变量中：
```shell
awk '/^S/ { print NF; $(NF+1) = 0; print NF; $(NF+1) = 0; print NF}' mail-list 
4  # 原列数
5  # 增加一列后的 NF
6  # 又增加一列后的 NF
```
修改直接修改`NF`的值也会反馈到`$0`：
```shell
> echo a b c d e | awk '{OFS = ":"; NF=7 ; $NF="new"; print $0}'
a:b:c:d:e::new  # 自动补充中间的空字段

> echo a b c d e | awk '{OFS = ":"; NF=3 ; print $0}'
a:b:c  # 自动删除多余的字段
```
上述行为都会触发`$0`的`rebuild`机制，一般我们修改列字段的内容都会触发这个机制，但是尝试仅通过设置`FS`和`OFS`来更改记录中的字段分隔符，这种不会触发`rebuild`，但是我们可以通过`$1=$1`来强制触发：
```shell
> echo a b c d e | awk '{OFS = ":"; print $0}'
a b c d e  # 不修改列字段就不会触发 rebuild
> echo a b c d e | awk '{OFS = ":"; $1=$1 ;print $0}'
a:b:c:d:e  # 强制触发 rebuild
```

## 列分隔符
我们可以通过以下方式设置列分隔符：
```shell
# 使用 -F 选项
> awk -F: '{ print $1 }' /etc/passwd
> awk -F'\n' '{ print $1 }' /etc/hosts

# 使用 FS 变量
> sed 1q /etc/passwd | awk -v FS=":" '{ print $1 }'
> sed 1q /etc/passwd | awk '{ print $1 }' FS=":"
> sed 1q /etc/passwd | awk 'BEGIN {FS=":"} { print $1 }'
```
列分隔符`FS`支持以下类型：
* `FS == " "`：默认值，用于匹配连续的空白字符。

* `FS == 任意字符`：表示按照这个字符去拆分字段。

* `FS == 正则表达式`：按照匹配正则表达式的字符串拆分字段，特殊字符需要转义。

* `FS == ""`：一个字符一拆，每个字符都当做独立的字段。

## 处理定宽字段
通过内置变量`FIELDWIDTHS`可以控制每个字段的长度，具体用法如下：
```shell
# 多余的字段可以用 * 收集
> echo 1234abcdefghi | awk 'BEGIN {FIELDWIDTHS = "2 2 *"} { print NF, $1, $2, $3}'
3 12 34 abcdefghi

# 如果设定的总宽度小于记录的长度，多余部分会被丢弃
> echo 1234abcdefghi | awk 'BEGIN {FIELDWIDTHS = "2 2 3"} { print NF, $1, $2, $3, $4}'
3 12 34 abc

# 如果设定的总宽度大于记录的长度，能收集多少就收集多少
> echo 1234abcdefghi | awk 'BEGIN {FIELDWIDTHS = "2 20"} { print NF, $1, $2}'
2 12 34abcdefghi
```
我们还可以通过切片的方式指定每个字段的起始位置和长度：
```shell
> w | awk 'BEGIN {FIELDWIDTHS = "7 2:4"} NR>2 { print $1,$2 }'
remilia tty1
```

# gawk扩展
## 基于内容的字段分割
如果我们要处理以逗号分隔的`csv`文件，我们可以使用`FS`来指定一个复杂的正则表达式用于分割字段：
```shell
> echo -e "   \"abc\", \"jkl\" , 132   " | awk -v FS='("* *, *"*)|(^ *")|(" *$)' '{print NF}'
4
```
这样做的问题在于很难处理边界的空格和分隔符周围的复杂情况，而且如果某个字段中包含分隔符（如：`\"abc, ced\"`），会产生意想不到的结果，因此我们需要更灵活的处理方法。在`gawk`中引入了基于内容的字段分割方法，我们可以通过给内置变量`FPAT`赋予正则表达式来指定我们想要的字段的格式：
```shell
FPAT = "([^,]*)|(\"[^\"]*\")"
```
这个正则表达式描述的是我们想要的字段的格式：
1. 不包含任何逗号的可以为空的字段
2. 被一对双引号包裹的字段

通过下面的例子可以进一步理解这个用法：
```shell
> cat fpat.data 
Robbins, Arnold,"1234 A Pretty Street, NE" , MyTown,,12345-6789,USA

> cat fpat.awk
BEGIN {
    FPAT = "([[:space:]]*[^,]*[[:space:]]*)|([[:space:]]*\"[^\"]*\"[[:space:]]*)"
}

{
    print "NF = ", NF
    for (i = 1; i <= NF; i++) {
        gsub(/(^[[:space:]]*)|([[:space:]]*$)/, "", $i)
        printf("$%d = <%s>\n", i, $i)
    }
}

> awk -f fpat.awk fpat.data 
NF =  7
$1 = <Robbins>
$2 = <Arnold>
$3 = <"1234 A Pretty Street, NE">
$4 = <MyTown>
$5 = <>
$6 = <12345-6789>
$7 = <USA>
```

## 查看当前的列分割方式
通过`PROCINFO["FS"]`可以查看当前使用的分割方法：
* `PROCINFO["FS"] == "FS"`：用正则表达式分割。
* `PROCINFO["FS"] == "FIELDWIDTHS"`：用指定长度分割。
* `PROCINFO["FS"] == "FPAT"`：根据内容分割字段。
* 其它：通过`API`导入的自定义分割方式。
## 输入超时

```shell
# 单位 milliseconds
PROCINFO["input_name", "READ_TIMEOUT"] = 100
```
## 错误重试
```shell
PROCINFO["input_name", "RETRY"] = 1
```