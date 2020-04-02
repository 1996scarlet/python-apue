`[addr]`用于控制`sed`处理文件中的哪些行，我们看下面的例子，对`input.txt`的第`144`行执行替换命令，如果不指定`[addr]`则会对文件中的所有行执行命令：
~~~
# 指定行
sed '144s/hello/world/' input.txt > output.txt

# 指定最后一行
seq 6 | sed '$!s/./x/'

# 所有行
sed 's/hello/world/' input.txt > output.txt
~~~

`[addr]`同样支持正则表达式：
~~~
# 当前行包含 apple 则进行替换
sed '/apple/s/hello/world/' input.txt > output.txt

# 当前行以 st 开头则进行替换
sed '/^st/s/hello/world/' input.txt > output.txt

# 后接 I 可以忽略大小写
> printf "%s\n" a b c | sed '/B/Id'
a
c

# 拓展正则模式
> echo 'a+b=c
aab
' | sed -n '/a+b/p'
a+b=c

> echo 'a+b=c
aab
' | sed -rn '/a+b/p'
aab

# 绝对字符 \%regexp%
# 可以少打很多转义字符
sed -n '/^\/home\/alice\/documents\//p'
sed -n '\%^/home/alice/documents/%p'  # 不一定需要 %
sed -n '\;^/home/alice/documents/;p'  # 也可以替换其他字符
~~~

`[addr]`支持通过`,`标记行范围，如下例所示：
~~~
# 替换第 4 行到第 17 行的内容
sed '4,17s/hello/world/' input.txt > output.txt

# 第二项可以为正则表达式
# 匹配到则退出，因此至少包括两行
> seq 10 | sed -n '4,/[0-9]/p'
4
5  # 第一个游标是 4，因此从第 5 行开始匹配

> seq 5 | sed -n '1,/^[root]/p'
1
2
3
4
5  # 没有匹配到任何行，因此不会中途退出

# 第二项可以带 +
> seq 10 | sed -n '6, +2p'
6
7  # 加几就额外处理多少行
8  # 加几就额外处理多少行
~~~

`[addr]`支持通过`!`进行取反：
~~~
# 行中没有 apple 则替换
sed '/apple/!s/hello/world/' input.txt > output.txt

# 对 1 到 3 行以及 18 到最后的行进行替换
sed '4,17!s/hello/world/' input.txt > output.txt
~~~

`[addr]`支持`first~step`结构，每次处理第`first + (n * step)`行：
```shell
> seq 10 | sed -n 0~4p
4
8
> seq 10 | sed -n 1~3p
1
4
7
10
```