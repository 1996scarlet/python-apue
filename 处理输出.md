# 输出分隔符
* `OFS`：字段分隔符，在使用时需要注意是否触发`rebuild`。
```shell
> echo "a b c" | awk -v OFS="<->" '{$1=$1; print}'
a<->b<->c

> echo "a b c" | awk -v OFS="<->" '{print}'
a b c  # 没有触发 rebuild 因此无效
```
* `ORS`：记录分隔符。
```shell
> seq 3 | awk -v ORS="\n---\n" '{print}'
1
---
2
---
3
---
```
* `OFMT`：用于设置输出格式。
```shell
> awk 'BEGIN {OFMT = "%.0f"; print 17.23,68.54}'
17 69
```

# print 关键字
在`awk`中，`print`关键字被用于输出，该关键字在实际使用时类似于函数调用，并且支持重定向：
```shell
print item1, item2, …

# 重定向输出
> awk '{ print $1 $2 > "/dev/stderr"}' inventory-shipped

# 打印多行
> awk 'BEGIN { print "line one\nline two\nline three" }'
line one
line two
line three

# 使用逗号分隔则自动添加 OFS
> awk -v OFS="--" '{ print $1,$2 }' inventory-shipped 
Jan--13
Feb--15
Mar--15

> awk -v OFS="     " 'BEGIN { print "Month Crates"; print "----- ------" } { print $1,$2}' inventory-shipped
```

# printf 关键字
`printf`的用法类似于`C`语言中的输出函数，支持格式化：
```shell
printf format, item1, item2, …
```
值得注意的是，`OFS`和`ORS`对`printf`都不起作用，`printf`只按照设定的格式进行输出：
```shell
> awk 'BEGIN { ORS="\n---\n"; OFS="+"; msg="Don\47t Panic!"; printf "%s\n", msg}'
Don't Panic!

> awk 'BEGIN { printf "%s %s\n", "Month", "Crates"; print "----- ------" } { printf "%-5s %6d\n", $1, $2}' inventory-shipped 
```
## 常用格式符
* `%f`：设定小数格式。
```shell
# 格式 [右对齐总宽度].[小数点后位数]
> awk 'BEGIN {printf "%5.1f<->%-5.1f<->%.2f\n", 1, 1, 1}'
  1.0<->1.0  <->1.00

# 宽度和位数可以为变量
> awk 'BEGIN {w=5;p=2;s="abc";printf "|%*.*s|\n", w, p, s}'
|   ab|
```
* `%e %E`：科学计数法。
```shell
> awk 'BEGIN {printf "%10.1e<->%-10.1e<->%.2e\n", 1, 1, 1}'
   1.0e+00<->1.0e+00   <->1.00e+00
```
* `-`：由于左对齐指定长度，长度不足则补充空格：
```shell
> awk 'BEGIN {printf "%-3s<->%-3s<->%-3s\n", "ab", "a", 1235}'
ab <->a  <->1235
```
* `\47`：也就是单引号，表示千分计数法。
```shell
> awk 'BEGIN {printf "%\47d\n", 12345678}'
12,345,678
```
* `N$`：指定顺序。
```shell
> awk 'BEGIN { printf "%3$s %1$s %2$s\n", "a", "b", "c" }'
c a b
```
我们可以通过定义`format`变量将其重复使用：
```shell
> awk 'BEGIN { format = "%-10s %s\n"
              printf format, "Name", "Number"
              printf format, "----", "------" }
             { printf format, $1, $2 }' mail-list
Name       Number
----       ------
Amelia     555-5553
Anthony    555-3412
Becky      555-7685
Bill       555-1675
Broderick  555-0542
Camilla    555-2912
Fabius     555-1234
Julie      555-6699
Martin     555-6480
Samuel     555-3430
Jean-Paul  555-2127
```
# 输出重定向
与在`shell`中的重定向方法类似，我们可以使用`> >> |`对`print`和`printf`的输出进行重定向。
```shell
> awk 'BEGIN {print "line1" > "guide.txt"; print "line2" >> "guide.txt"}'
> cat guide.txt 
line1
line2
```
我们还可以使用管道对输出进行重定向，使用管道的另一个好处是可以将拼凑的命令传递给管道后的命令去执行：
```shell
> awk 'BEGIN { print "ps -a" | "bash" }'
> ll | grep '^[^d]' | awk '$NF ~ /^[[:alpha:]]/ { print "ls -l",$NF | "bash"}'
```

## 关闭重定向
`close`方法的用法如下：
```shell
close(file_name)
close(command)
```
关闭文件的功能我们之前说过，`awk`对已经打开的文件不会通过`getline`重定向或`@include`重新加载，因此建议在处理完当前与文件有关逻辑之后关闭文件。而对于管道也有相似的约束，`awk`在同一时间只能维护一个打开的管道，因此大部分情况下需要通过`close(command)`关闭管道某一侧的命令从而关闭整个管道：
```shell
"sort -r names" | getline foo
close("sort -r names")  # 关闭一侧，其实本质是关闭了文件描述符

sortcom = "sort -r names"
sortcom | getline foo
close(sortcom)  # 也可以通过变量关闭
```

## close 的返回值
我们可以通过`retval = close(command)`的方式获得关闭方法的返回值，返回值定义如下：
| 退出状态 | 返回值 |
| --- | --- |
| 正常退出 | 命令的退出状态码 |
| 通过信号退出 | 256 + 退出信号的值 |
| 由于信号导致程序运行错误 | 512 + 退出信号的值 |
| 内部错误 | -1 |
