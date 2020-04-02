本文的操作环境为使用`UEFI`采用`GPT`分区表的硬盘并且已经包含了`FAT32`格式的`EFI`系统分区(`EFI FAT32 ESP`).
# 文件的元信息
在`UEFI`启动模式下, 外部存储介质中`Linux`类型的分区一般由以下几个部分组成:
* 引导区(`boot sector`): 一个分区中只有一个启动区块, 
* 超级区块(`super block`): 
* 索引节点(`index nodes`):
* 数据区块(`data blocks`)

文件的元信息被保存在`inode`中, 


我们可以通过`stat`, `fstat`, `lstat`等系统调用以`stat`结构体的形式获取指定文件的元信息, 
# 目录文件
# 文件的时间戳
用`stat`命令可以显示文件, 文件夹和文件系统的详细状态信息, 其中包括访问时间, 修改时间, 更改时间. 对于刚创建的文件这些时间是相等的, 这些信息都被保存在文件的`inode`中:
```bash
> seq 5 > peko
> stat peko
  File: peko
  Size: 10              Blocks: 8          IO Block: 4096   regular file
Device: fd00h/64768d    Inode: 3489414     Links: 1
Access: (0664/-rw-rw-r--)  Uid: ( 1000/ remilia)   Gid: ( 1000/ remilia)
Access: 2020-03-27 23:06:41.101590385 +0800  # 访问时间
Modify: 2020-03-27 23:06:41.101590385 +0800  # 修改时间
Change: 2020-03-27 23:06:41.101590385 +0800  # 状态时间
 Birth: -  # Ubuntu 中尚不支持该时间戳
```
**访问时间**(`last access timestamp, atime`): 对应`stat`结构体中的`stat.st_atime`, 用于记录对文件最后一次进行读操作的时间, 包括但不限于使用`cat`, `less`, `touch -a`等命令对文件内容进行读取或直接在程序中使用`execve`, `mknod`, `utime`, `read`(大于零字节)等系统调用对该文件进行操作. 
```bash
> touch peko
> stat peko > s1
> cat peko
> stat peko > s2
> diff s1 s2  # 只有访问时间改变了
5c5
< Access: 2020-03-28 12:15:15.473599342 +0800
---
> Access: 2020-03-28 12:15:24.758021562 +0800
```
但要注意, 有些发行版为了提升磁盘性能, 默认会在部分文件系统中禁用访问时间记录功能或延迟更新访问时间, 如: 使用`relatime`模式挂在文件系统则只有在`atime`早于`ctime`或`mtime`时才会更新, 而`strictatime`模式则会在每次访问文件时都更新`atime`. 此外, 使用`O_NOATIME`标记打开文件也不会更新该文件的访问时间. 
```bash
> cat /proc/mounts | grep /dev/sda2  # 查看挂在模式
/dev/sda2 /boot/efi vfat rw,relatime,fmask=0077,dmask=0077,codepage=437,iocharset=iso8859-1,shortname=mixed,errors=remount-ro 0 0
# 当前为 relatime 模式
> cat peko > s3  # 在 relatime 模式下不会更新 atime
> diff s2 s3  # 无差别
```
**创建时间**(`birth timestamp, btime`): 由于`stat`结构体中不存在该字段, 在文件创建时会被自动记录到`statx.stx_btime`中, 之后不会被更改. 但要注意, 大部分`Linux`发行版都不支持该时间戳.
**修改时间**(`last modification timestamp, mtime`): 对应`stat`结构体中的`stat.st_mtime`字段, 用于记录上一次文件数据块内容变动的时间, 当使用`truncate`, `mknod`, `write`(大于零字节)等系统调用对文件进行操作时, 例如: 修改文件内容, 会更新文件的`mtime`. 特别的, 在目录中添加删除文件也会更新该目录的`mtime`, 但要注意对文件`inode`内容的修改不会触发`mtime`记录器.
```bash
> seq 5 > peko
> stat peko | sed -n '/^Access.*[0-9]$/ {N;N;p}'
Access: 2020-03-28 16:03:27.603808847 +0800
Modify: 2020-03-28 16:03:27.603808847 +0800
Change: 2020-03-28 16:03:27.603808847 +0800
> echo "nginx" >> peko
> stat peko | sed -n '/^Access.*[0-9]$/ {N;N;p}'
Access: 2020-03-28 16:03:27.603808847 +0800
Modify: 2020-03-28 16:03:58.960563102 +0800
Change: 2020-03-28 16:03:58.960563102 +0800  # ctime 也被更新了
```
**更改时间**(`last status change timestamp, ctime`): 对应`stat`结构体的`stat.st_ctime`字段, 用于记录上一次修改文件`inode`数据或数据块的时间, 例如: 修改文件内容, 修改文件权限, 更新链接数, 修改文件所属的组或用户. 值得注意的是, 重命名文件时会先删除原有文件名硬链接, 然后再建立新的硬链接, 这个过程会修改`inode`中的链接数, 因此`ctime`也会被更新. 大部分情况下, 更新`mtime`时会同时更新`ctime`, 更新`ctime`却不一定会更新`mtime`.
```bash
> chmod o+w peko
> stat peko | sed -n '/^Access.*[0-9]$/ {N;N;p}'
Access: 2020-03-28 16:03:27.603808847 +0800
Modify: 2020-03-28 16:03:58.960563102 +0800
Change: 2020-03-28 16:04:29.461303202 +0800
```
# find 命令
## 对比 -exec 与 xargs
