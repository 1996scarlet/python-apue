# Shell 的种类

**sh** -- `Bourne Shell` ：在 `v7` 版本中引入的首个标准 `Unix Shell` 程序，仅支持最基础的功能，但目前仍在 `Unix` 系统和与 `Unix` 相关的环境中使用。在 `Linux` 系统中也提供了 `sh` 用于实现对 `Unix` 程序的兼容。

**bash** -- `GNU Bourne-Again SHell` ： `GNU` 钦定的标准 `shell` ，实际上也是 大部分 `Linux` 发行版和 `Mac OS X` 上的默认 `shell` 。作为 `sh` 的超集( `superset` ), `bash` 在方便初学者使用的同时也为专业用户提供了大量的高级功能，而且有良好的兼容性。[本书](https://1996scarlet.github.io/python-apue/)中所有的练习与示例都使用 `Ubuntu 20.04 LTS` 自带的 `bash 5.3.0` 。

**csh** -- `C shell` ：语法类似 `C` 语言的 `shell` ，由 `Bill Joy` 设计，目前很少使用。

**tcsh** -- `TENEX C shell` ： `csh` 的升级版，完全兼容 `csh` ，添加了自动补全、历史记录、作业控制等功能并具有更强大的语法支持。

**zsh** -- `Z shell` ：功能更加强大的 `shell` ，也是 `Mac OS Catalina` 版本之后的默认 `shell` ，相比于 `bash` 提供了更加智能的命令补全和提示功能以及对其他功能的优化，但是由于其运行效率较差而且语法与 `bash` 有细微差别，因此目前不适合在生产环境中使用。

## 自定义解释器

在 `/etc/shells` 文件中会列出当前中系统已经安装的 `shell` 解释器：

``` bash
> cat /etc/shells
# /etc/shells: valid login shells  
/bin/sh  
/bin/bash  
/usr/bin/bash  
/bin/rbash  
/usr/bin/rbash  
/bin/dash  
/usr/bin/dash  
/bin/tcsh
/usr/bin/tcsh
```

用户登录时默认使用的 `shell` 定义在 `/etc/passwd` 文件中，配合 `USER` 变量可以查看当前用户的默认 `shell` ：

``` bash
> cat /etc/passwd | grep $USER  
remilia:x:1000:1000:Remilia Scarlet,,,:/home/remilia:/bin/bash
```

通过在文件开头指定[Shebang](https://bash.cyberciti.biz/guide/Shebang)可以选择执行 `shell` 脚本时用的解释器，在 `Python` 和 `Perl` 中也有类似的设定。

``` bash
#!/usr/bin/python
```

# Bash 编辑模式

`bash` 有 `emacs` (默认)和 `vi` 两种编辑模式, 可以通过 `bind -V` 查看当前的编辑模式, 通过 `set -o` 设置快捷键模式。

``` bash
> bind -V | grep keymap
keymap is set to 'emacs'
> set -o vi
> bind -V | grep keymap
keymap is set to `vi-insert'
```

## 光标控制快捷键

* 光标跳转到行首 `Ctrl + A` 
* 光标跳转到行尾 `Ctrl + E` 
* 光标向左移一位 `Ctrl + B` 
* 光标向右移一位 `Ctrl + F` 
* 在行首与当前光标位置之间切换 `Ctrl + XX` 

## 作业控制快捷键

* 中断前台作业(SIGINT) `Ctrl + C` 
* EOF（结束标准输入stdin）、退出前台作业、登出会话 `Ctrl + D` 
* 挂起前台作业(SIGTSTP) `Ctrl + Z` 

## 文本控制快捷键

* Backspace `Ctrl + H` 
* Delete `Ctrl + D` 
* 清屏 `Ctrl + L` 
* 删除从当前光标位置到行首的全部字符 `Ctrl + U` 
* 删除从当前光标位置到行尾的全部字符 `Ctrl + K` 

## 历史记录快捷键

* 搜索历史记录（最近最匹配原则） `Ctrl + R` 
* 由近到远获得一条历史记录 `Ctrl + P` 
* 由远到近获得一条历史记录 `Ctrl + N` 
* 由近到远获得一条历史记录的最后一个词 `Alt + .` 

``` shell
> touch /tmp/this-is-a-long-file-name.99839.op
> chmod a+x <Alt + .>
# 等价于 chmod a+x /tmp/this-is-a-long-file-name.99839.op
```

## 流控制（自信盲打模式）

* 锁定输出流(XOFF) `Ctrl + S` 
* 释放输出流(XON) `Ctrl + Q` 

    

``` shell
> <Ctrl + S> date <Ctrl + J> uname -sr <Ctrl + J> <Ctrl + Q> 
# 由于输出流会锁定，终端不会显示任何输出, 释放后会显示以下内容
> date
Sat 22 Feb 2020 01:31:04 PM CST
> uname -sr
Linux CT7GK 5.3.0-40-generic
```

# 参考内容

* UNIX环境高级编程
* [Bash快捷键大全](https://linux.cn/article-5660-1.html)
* [Linux Command Line & Bash Shell Shortcuts](https://linuxconfig.org/linux-command-line-bash-shell-shortcuts)

