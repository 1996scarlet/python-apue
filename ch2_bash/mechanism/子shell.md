 `bash` 中的子 `shell` 可以分为以下两类：

* `subshell` ：通过进程替换、 `(LIST)` 、 `|` 或 `&` 创建的子 `shell` 。因为这种子 `shell` 在创建时只使用了 `fork` 函数，导致其可以继承父 `shell` 的全部变量（局部变量也会被继承，甚至包括` `bash` 中的子 `shell` 可以分为以下两类：和 `$PID` ）、别名和函数。
* `child shell` ：我们调用 `bash` 命令生成交互式 `shell` 或执行脚本都会得到这种子 `shell` 。这种子 `shell` 在创建时会通过 `fork` 和 `exec` 函数将其代码、数据、堆栈替换为全新的 `bash` 进程，因此只会继承父 `shell` 传递来的全局变量。

shell启动的shell子进程称为子shell。直接以文件名运行可执行文件时，bash并不知道它调用的一个可执行是二进制文件还是脚本，只是在exec过程中交给系统内核处理。对于shell脚本，通常以“#![shell可执行文件名]”开头，“#!”是一种magic number。当内核通过magic number断定执行的是脚本时，就会调用一个新的指定的shell的实例来解释执行脚本，这个实例就是子shell。父子shell是两个进程，所以各自的变量是独立的。除非父shell将自己的变量导出到环境中，否则子shell无法获得父shell中定义的变量。

bash通过变量SHLVL记录自己是进程调用栈中哪一层的shell，即bash被嵌套的深度。bash启动时，调用variables.c中的initialize_shell_level()设置SHLVL。系统login之后启动的bash的SHLVL为1，每层shell启动的子shell的SHLVL在其环境中读到的SHLVL基础上加1。

使用source命令（“.”命令）执行脚本时，不开启子shell。bash内部的实现是将脚本文件内容读入一个缓冲区，然后执行语法分析，因此效果与直接从键盘输入脚本内容相同。

# 创建 subshell

``` bash
# 创建 subshell 的方法
> echo `echo $BASH_SUBSHELL` 
1  # 命令替换会创建 subshell
> echo $BASH_SUBSHELL &
1  # 后台执行会创建 subshell
> (echo $BASH_SUBSHELL)
1  # 一对小括号会创建 subshell

> function f1()
> {
> echo $BASH_SUBSHELL >&2
> }
> f1
0
> f1 | f1 | f1
1  # 三个独立的 subshell
1
1

# 管道陷阱
> a=10
> a=5 | echo $a
10  # 左右两侧都在各自创建的 subshell 中执行 互不影响
> echo $a
10  # 在 subshell 中修改变量不会影响父 shell

# 证明确实创建了子 shell
> pstree -p $; echo $ $SHLVL $BASH_SUBSHELL;
bash(2695)───pstree(2702)  
2695 1 0
> (pstree -p $; echo $ $SHLVL $BASH_SUBSHELL;)
bash(2695)───bash(2714)───pstree(2715)
2695 1 1  # $也被继承了，但注意其值是 invorking shell 的 PID

# LIST 必须为多条命令才会创建子 shell
> pidof bash; (pidof bash;)
2695
2695
> pidof bash; (echo>/dev/null;pidof bash;)
2695  
3163 2695

# 只有 subshell 可以继承局部变量
> unset a; a=1
> (echo "a is $a in the subshell")
a is 1 in the subshell  # 仅 fork 完全拷贝
> bash -c 'echo "a is $a in the child shell"'
a is  in the child shell  # fork 并 exec 导致信息被覆盖

# 查看嵌套层级
> (echo $SHLVL $BASH_SUBSHELL;(echo $SHLVL $BASH_SUBSHELL;(echo $SHLVL $BASH_SUBSHELL)))
1 1
1 2
1 3
> bash
> (echo $SHLVL $BASH_SUBSHELL;(echo $SHLVL $BASH_SUBSHELL;(echo $SHLVL $BASH_SUBSHELL)))
2 1
2 2
2 3
```

`SHLVL` 变量用于记录 `shell` 的嵌套层级， `BASH_SUBSHELL` 用于记录 `subshell` 的嵌套层级。

# 执行脚本

使用 `bash` 命令（相当于在当前 `shell` 中执行了外部命令）执行 `shell` 脚本时会创建 `child shell` ；使用 `source` 、 `.` 或 `exec` 则会直接在当前 `shell` 中执行脚本，不会额外创建 `child shell` （不包括脚本中的命令可能会自行创建的子 `shell` ）。使用 `bash` 命令得到的 `child shell` 每次从脚本读取一行，每行命令都像直接来自键盘一样被读取、解释和执行，父 `shell` 在此过程中将通过 `wait` 系统调用等待其子进程完成。

``` bash
bash my.sh  # 会创建子 shell
./my.sh  # 会创建子 shell
. my.sh  # 不会创建子 shell，因为是内置命令
source my.sh  # 不会创建子 shell，因为 `source` 是 `.` 的别称
```

简单总结一下子进程和子 `shell` ：

* 执行内置命令和 `shell` 脚本中的函数时，连子进程都不会创建，自然也不会创建子 `shell` 。

``` bash
> mkfifo a
> fc1(){ a<a; }  # 定义一个 shell 脚本函数
> echo a<a &
> pstree -pa $
bash,6080
  ├─bash,11092  # 这个 subshell 是 & 创建的
  └─pstree,11095 -pa 6080
# 内置命令 echo 直接在 11092 中执行，甚至都没有建立子进程
# 可以另开一个终端然后 pstree -pa <上面那个bash的pid> 进行验证
> fc1 &
> pstree -pa 6080  # 在另一个终端中输入
bash,6080  # 只显示这个 没有创建任何子进程
```

* 执行外部命令时会使用 `fork-exec` 模式创建子进程，如果外部命令是 `shell` 类型则会创建 `child shell` 。
* 使用 `source` 、 `.` 、 `exec` 执行脚本以及用 `{ LIST }` （注意是大括号）包裹命令列表相当于在当前 `shell` 中逐行输入命令，要注意使用 `exec` 执行完脚本后会退出 `shell` 。
* 使用 `bash xxx.sh` 或 `./xxx.sh` 运行脚本会创建 `child shell` 。

``` bash
> cat test.sh
#!/bin/bash
sleep 30;

> bash test.sh  # 回车之后 ctrl+z
> pstree -pa $
bash,6080
  ├─bash,12602 test.sh  # 自动创建的 child shell
  │   └─sleep,12603 30  # 逐行解释执行的命令
  └─pstree,12606 -pa 6080
```

* 使用 `( LIST )` 执行命令列表会自动创建 `subshell` 。

``` shell
> (echo;sleep 30;)  # 回车之后按下 ctrl + z
                
> pstree -pa $
bash,6080  # 敲命令的 invoking shell
  ├─bash,9780  # 自动创建的 subshell，可与父 shell 共享局部变量
  │   └─sleep,9781 30  # 执行外部命令创建的 child process
  └─pstree,9782 -pa 6080
> fg 1  # 可以等待完成 也可以直接 ctrl+c
```

