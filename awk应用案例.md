1. 查询占用百分比排前 3 位的文件系统：
```
> df -h | grep -v snap | sort -rnk 5 | head -3 | awk '$0!~/snap/ { printf "Partition %-10s: %s full.\n",$6,$5}'
Partition /         : 48% full.
Partition /boot/efi : 34% full.
Partition /dev/shm  : 1% full.
```
2. 查询所有 `\dev` 目录下的文件系统的占用率：
```
> df -h | awk '$1~/dev/ { print $1,"\t",$5 }'
udev     0%
/dev/mapper/vgkubuntu-root       48%
/dev/loop1       100%
/dev/loop0       100%
/dev/loop2       100%
/dev/loop3       100%
/dev/sda2        34%
```
3. 查询所有`a`或`x`开头的`.conf`文件：
```
> locate *.conf | awk -v FS="/" '$NF ~ /^[ax][[:alpha:]]*.conf/{ i=index($0, $NF); print $NF," at ", substr($0, 0, i-1) }'
xorg.conf  at  /usr/share/doc/xserver-xorg-video-intel/
auto.conf  at  /usr/src/linux-headers-5.3.0-29-generic/include/config/
```
4. 根据输入的纯文本生成`html`页面：
```
> cat make-html-from-text.awk
BEGIN { print "<html>\n<head><title>Awk-generated HTML</title></head>\n<body bgcolor=\"#ffffff\">\n<pre>" }
{ print $0 }
END { print "</pre>\n</body>\n</html>" }
> awk -f make-html-from-text.awk inventory-shipped > file.html
> google-chrome-stable file.html
```

5. 根据输入的以`TAB`作为字段分隔的纯文本转换为`XML`：
```
{
    template = "<row>\n<entry>" $1 "</entry>\n"
    for (i=2;i<=NF;i++)
        template = template "<entry>\n" $i "\n</entry>\n"
        
    print template "</row>"
}
```

6. 合并文件为一行
```
> seq 5 | awk -v ORS="" '{ print } END { print "\n"}'
12345
```
7. 范围模式.
```
# awk '/P1/, /P2/'
> seq 5 | awk '/2/, /4/'
2
3
4
```
8. 统计`/etc/passwd`中各种`shell`的数量
```
> awk -F: '{ s[$NF]++ } END { for (i in s) { print i,s[i]}}' /etc/passwd
/bin/sync 1
/bin/bash 2
/sbin/nologin 1
/bin/false 5
/usr/sbin/nologin 40
```
9. 统计端口的连接状态并排序
```
> netstat -ant | awk 'NR>2 { s[$NF]++ } END { for (i in s) { print i,s[i] }}' | sort -k2 -n
TIME_WAIT 2
SYN_SENT 4
ESTABLISHED 7
LISTEN 14
```
10. 查找用户名为四个字符的用户:
```
> awk -F: '$1 ~ /^.{4}$/ { counter++; print $1 } END { print "total:",counter }' /etc/passwd
# 或 awk -F: 'length($1)==4 { counter++; print $1 } END { print "total:",counter }' /etc/passwd
root
sync
mail
news
uucp
list
_apt
sddm
epmd
sshd
total: 10
```
11. 清除本机ARP缓存.
```
> arp -n | awk 'NR>1 { print $1 }' | xargs arp -d
```
12. 判断系统是`redhat`还是`debian`分支:
```

```

13. 分析系统资源瓶颈.
```
# cpu利用率与负载: top vmstat sar
# 磁盘和Inode的使用率与I/O负载: df iostat iotop sar dstat
# 内存利用率: free vmstat
# TCP连接状态: netstat ss
# CPU与内存占用率最高的10个进程: top ps
# 查看网络流量: ifconfig iftop iptraf

cpu_stat () {
    
}
```
