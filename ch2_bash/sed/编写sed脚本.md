# 基本语法
`sed`表达式的基本语法如下：
```shell
[addr]X[options]
```
其中`X`表示单字母操作模式，`[addr]`代表可选的行，其值可以为**空**、**单个行号**、**正则表达式**、**行号范围**，如果`[addr]`非空，则当前操作只在指定行生效。某些特殊命令还需要额外指定`[options]`参数。
~~~
# 删除文件第 30 行到第 35 行
# ’30,35‘是’[addr]‘表示范围，‘d'是’X‘表示删除操作模式
sed '30,35d' input.txt > output.txt
~~~

下面这个命令的功能是打印每一行直到找到以`foo`开头的行，如果找到了就以状态码`42`退出，否则退出状态码为`0`：

~~~
# /^foo/ 是 [addr] 正则表达式
# q 是 X 用于设置退出状态
# 42 是 [options]
sed '/^foo/q42' input.txt > output.txt
~~~

`sed`支持使用`;`进行命令组合，也支持`-e`和`-f`混用，也就是可以输入多个模式，以下命令是等价的，都用于删除所有以`foo`开头的行并将文件中所有的`hello`替换为`world`：

~~~
sed '/^foo/d ; s/hello/world/' input.txt > output.txt

sed -e '/^foo/d' -e 's/hello/world/' input.txt > output.txt

echo '/^foo/d' > script.sed
echo 's/hello/world/' >> script.sed
sed -f script.sed input.txt > output.txt

echo 's/hello/world/' > script2.sed
sed -e '/^foo/d' -f script2.sed input.txt > output.txt
~~~

>[warning]**警告**：由于命令`a`，`c`，`i`的语法限制，不能使用分号作为命令分隔符，因此应以换行符终止或将其放在脚本文件的末尾。 命令之前也可以带有可选的非有效空白字符。

# 支持的命令
## s 命令
用于实现字符串替代功能的`s`命令是`sed`中最重要的命令，其语法如下：
```shell
s/regexp/replacemen/flags
```
它的基本概念很简单：`s`命令尝试将模式空间（当前缓存的行）与提供的正则表达式`regexp`相匹配； 如果匹配成功，则将匹配的模式空间部分替换为`replacemen`。

`s`命令同样支持回溯，也就是在匹配项中使用`\(...\)`捕获到的局部内容可以在替换项中用`\ + 数字`的方式访问，并且`GNU`对该功能做了拓展，使其支持以下`\ + 字符`功能：
`\L`：将之后的匹配项转换为小写字母直到遇到`\U`或`\E`。
`\l`：将下一个字符转换为小写。
`\U`：将之后的匹配项转换为大写直到遇到`\L`或`\E`。
`\u`：将下一个字符转换为大写。
`\E`：停止由`\L`或`\U`发起的大小写转换。

当使用g标志时，大小写转换不会从一次正则表达式传播到另一次。 例如，在模式空间中使用`a-b-`执行以下命令时：
```shell
# 只对第一个匹配项进行替换
# 由于第一个匹配项的第一个捕获区域为空
# 导致 \u 无法起作用
> echo "a-b-" | sed -E 's/(b?)-/x\u\1/'
axb-

# 加了 g 标志，对所有匹配项替换
# 第二个匹配项的第一个捕获区域为 b
# \u 让它的下一个字符大写
> echo "a-b-" | sed -E 's/(b?)-/x\u\1/g'
axxB

# 第一个匹配项的第一个捕获区域为空
# 导致紧挨着 \u 的 x 大写
# 第二个匹配项的第一个捕获区域为 b
# \u 让它的下一个字符大写
> echo "a-b-" | sed -E 's/(b?)-/\u\1x/g'
aXBx
```

>[info]**提示**：在替换结构中需要对如`\`或`&`等的特殊字符添加转义`\\`、`\&`。

每个`s`命令可以携带任意个`flags`，它们的功能描述如下：
* `g`：对模式空间中所有的匹配项都进行替换，如果不带这个标记则只会替换第一个匹配项。
* `number`：是个数字，表示只替换第`number`个匹配项，例如：`s/pa/pb/2`表示只替换第`2`个匹配项。

>[warning]**警告**：`GNU`规定，当`g`与`number`一起使用时将替换从第`number`个之后的所有匹配项。

* `p`：打印发生替换的行。
* `w filename`：将输出结果保存（打印）到文件。
```shell
# 以下两条命令等价
sed 's/hello/world/w output.txt' input.txt
sed 's/hello/world/' input.txt > output.txt
```

* `e`：替换之后直接执行命令。
```shell
> echo 'acho cc' | sed 's/a/e/eg'
cc  # 等价于执行 echo cc
```

* `I或i`：大小写不敏感匹配模式。
* `M或m`：多行匹配模式。

## 其他常用命令
* `#`：`sed`脚本中可以添加注释，不支持`[addr]`。

>[warning]**警告**：如果`sed`脚本的前两个字符为`#n`则会强制开启静默模式。

* `=`：用于显示行号。

* `q [exit-code]`：退出，只能接收一个`[addr]`，例如打印到第二行就退出:

~~~
> seq 3 | sed 2q9
1
2  # 会打印退出的那一行
> echo $?
9  # 自定义返回值

> seq 3 | sed 2Q
1  # 不会打印退出的那一行
~~~

* `d`：删除指定行（模式空间）:。

~~~
$ seq 3 | sed 2d
1
3
~~~

* `p`：打印指定行，通常配合`-n`选项使用。
~~~
$ seq 3 | sed -n 2p
2
~~~

* `n`：打印当前模式空间并立即读取下一行到模式空间，也就是跳过一行并打印，看下面这个例子可以更直观的理解`n`命令：
~~~
# 每三行处理一次
> seq 6 | sed 'n;n;s/./x/'
1  # 被第一个 n 打印 然后读取下一行
2  # 被第二个 n 打印 然后读取下一行
x  # 被 s 命令处理 然后重复匹配模式
4
5
x

# GNU 拓展的 [addr] 语法可以达到同样的效果
> seq 6 | sed '0~3s/./x/'
~~~

* `N`：在当前模式空间中添加换行符并读取下一行到当前模式空间，如果没有下一行就立刻退出。注意与`n`的区别，`N`会将读入的下一行和当前行一起保存在模式空间中，而`n`会直接将当前模式空间打印然后再读取下一行。
```shell
> seq 7 | sed 'N;s/./X/'
X  
2  
X  
4  
X  
6  
7  # 不够读入下一行，因此不处理
```

* `P`：该命令配合`N`使用可以输出第一个换行符之前的数据，如果不和`N`一起用则与`p`命令相同。
```shell
> seq 7 | sed -n 'N;N;P;l'
1  # P 的输出
1\n2\n3$  # 拼接 3 行的真实输出
4
4\n5\n6$
```


* `{commands}`：我们可以用一对`{}`完成命令扩展与嵌套，如下例：

~~~
# 处理并打印第二行
> seq 3 | sed -n '2{s/2/X/ ; p}'
X

# 等价于
> seq 3 | sed -n '2s/2/X/ ; 2p'


# 命令嵌套
> cat elec.txt
This is my computer
  its price is 9000
This is my keyboard
  its price is 1500
This is my mouse
  its price is 750
This is my phone
  its price is 3000

# 删除第一行及其之后的两行中包含 This 的行
> sed '1,+2{/This/d}' elec.txt 
  its price is 9000
  its price is 1500
This is my mouse
  its price is 750
This is my phone
  its price is 3000

# 从第二行到最后一行 如果包含 This
# 则取出最后一个单词并大写首字母拼接到下一行
> sed -r '2,${/This/{N;s/.*\W(\w+)\n *its/\u\1`s/}}' elec.txt
This is my computer  
 its price is 9000  
Keyboard`s price is 1500
Mouse`s price is 750
Phone`s price is 3000
~~~

## 生僻命令

* `y/source-chars/dest-chars/`：集合映射替换，`source-chars`和`dest-charslists`在转义之后必须包括相同数量的字符，如下例：
~~~
$ echo hello world | sed 'y/abcdefghij/0123456789/'
74llo worl3
~~~
* `a text`：行后插入。
~~~
> seq 3 | sed '2a hello'
1
2
hello
3

> seq 3 | sed -e '2a\' -e hello
1
2
hello
3
~~~
* `i text`：行前插入。
~~~
> seq 3 | sed '2i hello'
1
hello
2
3
~~~
* `c text`：多行替换。
~~~
> seq 10 | sed '2,9c hello'
1
hello
10
~~~

# 多命令语法
在`sed`脚本中我们一般每一行实现一个命令，但是在命令行中，为了方便测试，我们可以使用`; -e`实现多命令。

```
> seq 6 | sed '1d
3d
5d'  # 换行实现多命令

2
4
6

> seq 6 | sed -e 1d -e 3d -e 5d
2  # -e 实现多命令
4
6

> seq 6 | sed '1d;3d;5d'
2  # 分号实现多命令
4
6
```


## 语法限制

以下命令由于语法限制，必须使用换行和`-e`才能实现多命令。

* `a`,`c`,`i`：
~~~
# 分号不能作为命令分隔
> seq 2 | sed '1aHello ; 2d'
1
Hello ; 2d
2

# 使用 -e 可以
> seq 2 | sed -e 1aHello -e 2d
1
Hello

# 使用换行可以
> seq 2 | sed '1aHello
2d'
1
Hello

# POSIX 经典用法
$ seq 2 | sed '1a\
Hello
2d'
1
Hello
~~~

* `#`注释符号：
~~~
$ seq 3 | sed '# this is a comment ; 2d'
1
2
3

$ seq 3 | sed '# this is a comment
2d'
1
3
~~~

* `r`,`R`,`w`,`W`这些命令用于读写文件，因此不能使用`;`作为命令分隔：
~~~
> seq 2 | sed '1w hello.txt ; 2d'
1
2

> cat 'hello.txt ; 2d'
1

# sed 会自动忽视错误命令
# 如找不到文件 hello.txt ; N
> echo x | sed '1rhello.txt ; N'
x  # 什么也不做
~~~

* `e`用于执行命令，逗号、空白字符和分号都会被送到`shell`中去执行：
~~~
$ echo a | sed '1e touch foo#bar'
a

$ ls -1
foo#bar

$ echo a | sed '1e touch foo ; s/a/b/'
sh: 1: s/a/b/: not found
a
~~~

* `s///[w]`带有`w`（写入文件）标志的`s`命令：
~~~
$ echo a | sed 's/a/b/w1.txt#foo'
b

$ ls -1
1.txt#foo
~~~


# 附录
[更多生僻的sed命令](https://www.gnu.org/software/sed/manual/html_node/Other-Commands.html#Other-Commands)
[更深入的sed命令](https://www.gnu.org/software/sed/manual/html_node/Programming-Commands.html#Programming-Commands)
[GNU扩展的sed命令](https://www.gnu.org/software/sed/manual/html_node/Extended-Commands.html#Extended-Commands)