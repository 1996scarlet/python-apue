# 数组
## 基本用法
`awk`中的数组类似于`python`中的字典，其`index`可以为数字或字符串：
```shell
> awk 'BEGIN { a[1] = 1; a["oki"] = "poi"; a[-1] = 0} END { for (key in a) print a[key]}' /dev/null
poi
0
1
```
在上例中，我们使用`for (key in array)`遍历整个数组的`index`，其中`in`被用于判断一个`key`是否属于数组。如果`index`都是整数，也可以用传统的自增索引方式遍历数组：
```shell
{
    if ($1 > max)
        max = $1
    arr[$1] = $0
}

END {
    for (x = 1; x <= max; x++)
        print arr[x]
}

# 输入数据
5  I am the Five man
2  Who are you?  The new number two!
4  . . . And four on the floor
1  Who is number one?
3  I three you.

# 输出结果
1  Who is number one?
2  Who are you?  The new number two!
3  I three you.
4  . . . And four on the floor
5  I am the Five man
```
但要记住，数组的索引永远会被强转为字符串：
```shell
> awk 'BEGIN { a[1] = 1; a[2]=2 } END { print a[1], a["1"], a["2"]}' /dev/null 
1 1 2

# 下面这个情况是找不到的
xyz = 12.153
data[xyz] = 1  # 这里索引是 "12.153"
CONVFMT = "%2.2f"
if (xyz in data)  # 这里索引是 "12.15"
    printf "%s is in data\n", xyz
else
    printf "%s is not in data\n", xyz
```
## 未初始化的索引
未初始化的变量作为索引时会被强转为空字符串，因此需要通过`array[""]`来访问。
## 删除数组与元素
通过`delete`关键字可以删除整个数组或者其中的某个元素：
```shell
delete array
delete array[index]

a[1] = 3
delete a
a = 3  # 会报错
```
>[warning]**警告**：使用`delete`删除数组并不会改变变量的`type`，因此不能把删除后的数组名作为变量名使用。
## 多维数组
我们将输入的数组顺时针旋转 90 度：
```shell
{
     if (max_nf < NF)
          max_nf = NF
     max_nr = NR
     for (x = 1; x <= NF; x++)
          vector[x, NR] = $x
}

END {
     for (x = 1; x <= max_nf; x++) {
          for (y = max_nr; y >= 1; --y)
               printf("%s ", vector[x, y])
          printf("\n")
     }
}

# 输入
1 2 3 4 5 6
2 3 4 5 6 1
3 4 5 6 1 2
4 5 6 1 2 3

# 输出
4 3 2 1
5 4 3 2
6 5 4 3
1 6 5 4
2 1 6 5
3 2 1 6
```

# 函数
## 内置数学函数
包括`sin`, `cos`, `sqrt`, `log`, `exp`, `int`等函数，其中值得一提的有以下两个函数：
* `atan2(y, x)`：计算`y/x`的`arctangent`，例如：`pi = atan2(0, -1)`。
* `rand()`：随机产生一个在`[0,1]`之间的值，随机种子每次都相同，我们可以自己封装一个`randint`函数：
```shell
# 产生一个 [0, n) 之间的整数
function randint(n)
{
    return int(n * rand())
}

# 模拟随机掷 n 个骰子
function roll(n) { return 1 + int(rand() * n) }
function dice(n)
{
    for (i=0;i<n;i++)
        sum += roll(6)
    printf("%d points\n", sum)
}
```
* `srand([x])`：设置随机数种子，`x`为空则用当前时间，一般配合`rand`使用。
```shell
# 这样每次运行的结果就都不同
BEGIN { srand(); dice(3) }
```

## 字符串操作函数

## 输入输出函数

## 时间函数

## 位运算函数

## 用户定义的函数
自定义函数的格式如下：
```shell
function name([parameter-list])
{
     body-of-function
}
```
例如自定义的打印格式函数：
```shell
function myprint(num)
{
     printf "%6.3g\n", num
}

$3 > 0     { myprint($3) }
```
自定义的删除数组函数：
```
function delarray(a,    i)
{
    for (i in a)
        delete a[i]
}
```
递归反转字符串函数：
```
function rev(str)
{
    if (str == "")
        return ""

    return (rev(substr(str, 2)) substr(str, 1, 1))
}
```

