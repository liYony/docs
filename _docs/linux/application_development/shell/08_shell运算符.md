# shell运算符

## 1 运算符种类

与其他编程语言相同的是，shell同样支持多种运算符：

- 算数运算符
- 关系运算符
- 布尔运算符
- 逻辑运算符
- 字符串运算符
- 文件测试运算符

shell想要使用这些运算符，需要结合其他命令和工具来使用（因为shell中不支持简单的数学运算），如使用算符运算符就需要搭配的常用的工具有两种：

- awk
- expr（使用频繁）

> [!NOTE]
> - 表达式和运算符之间必须要有空格，例如 3+2 是不对的，必须写成 3 + 2
> - 完整的表达式要被 两个" ` "包含（在 Esc 键下边那个键）

如下实例：

```bash
#!/bin/bash
 
val=`expr 3 + 2`
echo "两数之和为 : $val"
# 输出：两数之和为 ：5
```

## 2 算数运算符

shell支持的常用的如下表，举例中这里假定变量 a 为 10，变量 b 为 20。

运算符 | 说明 | 举例
------|--------|-------
\+ | 加法 | expr $a + $b 结果为 30。
\- | 减法 | expr $a - $b 结果为 -10。
\* | 乘法 | expr $a \* $b 结果为 200。
/ | 除法 | expr $b / $a 结果为 2。
% | 取余 | expr $b % $a 结果为 0。
= | 赋值 | a=$b 将把变量 b 的值赋给 a。
== | 相等。用于比较两个数字，相同则返回 true。 | [ $a == $b ] 返回 false。
!= | 不相等。用于比较两个数字，不相同则返回 true。 | [ $a != $b ] 返回 true。

> [!NOTE]
> 在windows系统中乘号(*)前边必须加反斜杠()才能实现乘法运算；

## 3 关系运算符

shell中的关系运算符和其他编程语言不同，shell中使用特殊的字符表示关系运算符，并且只支持数字，不支持字符串，除非字符串是数字，下表为常用关系运算符，同样指定a为10，b为20。

运算符 | 说明 | 举例
------|------|-------
-eq | 检测两个数是否相等，相等返回 true。 | [ $a -eq $b ] 返回 false。
-ne | 检测两个数是否不相等，不相等返回 true。 | [ $a -ne $b ] 返回 true。
-gt | 检测左边的数是否大于右边的，如果是，则返回 true。 | [ $a -gt $b ] 返回 false。
-lt | 检测左边的数是否小于右边的，如果是，则返回 true。 | [ $a -lt $b ] 返回 true。
-ge | 检测左边的数是否大于等于右边的，如果是，则返回 true。 | [ $a -ge $b ] 返回 false。
-le | 检测左边的数是否小于等于右边的，如果是，则返回 true。 | [ $a -le $b ] 返回 true。

```bash
#!/bin/bash
 
a=10
b=20
 
if [ $a -eq $b ]
then
   echo "$a -eq $b : a 等于 b"
else
   echo "$a -eq $b: a 不等于 b"
fi
if [ $a -ne $b ]
then
   echo "$a -ne $b: a 不等于 b"
else
   echo "$a -ne $b : a 等于 b"
fi 
# 输出
# 10 -eq 20: a 不等于 b
# 10 -ne 20: a 不等于 b
```

> [!NOTE]
> 运算符和数之间必须用空格隔开。

## 4 布尔运算符

shell中的布尔运算符使用如下表，同样指定a为10，b为20。
运算符 | 说明 | 举例
------|-----|------
! | 非运算，表达式为 true 则返回 false，否则返回 true。 | [ ! false ] 返回 true。
-o | 或运算，有一个表达式为 true 则返回 true。 | [ $a -lt 20 -o $b -gt 100 ] 返回 true。
-a | 与运算，两个表达式都为 true 才返回 true。 | [ $a -lt 20 -a $b -gt 100 ] 返回 false。

```bash
#!/bin/bash
 
a=10
b=20
 
if [ $a != $b ]
then
   echo "$a != $b : a 不等于 b"
else
   echo "$a == $b: a 等于 b"
fi 
# 输出：10 != 20 : a 不等于 b
```
## 5 逻辑运算符

shell中的逻辑运算符和其他编程语言有类似的地方，如下表。假定变量 a 为 10，变量 b 为 20。

运算符 | 说明 | 举例
-----|-------|----
&& | 逻辑的 AND | [[ $a -lt 100 && $b -gt 100 ]] 返回 false
\|\| | 逻辑的 OR | [[ $a -lt 100 || $b -gt 100 ]] 返回 true

```bash
#!/bin/bash
 
a=10
b=20
 
if [[ $a -lt 100 && $b -gt 100 ]]
then
   echo "返回 true"
else
   echo "返回 false"
fi 
# 输出：返回 false
```

## 6 字符串运算符

shell中常用的字符串运算符如下表。假定变量 a 为 “abc”，变量 b 为 “efg”

运算符 | 说明 | 举例
------|------|-----
= | 检测两个字符串是否相等，相等返回 true。 | [ $a = $b ] 返回 false。
!= | 检测两个字符串是否相等，不相等返回 true。 | [ $a != $b ] 返回 true。
-z | 检测字符串长度是否为0，为0返回 true。 | [ -z $a ] 返回 false。
-n | 检测字符串长度是否不为 0，不为 0 返回 true。 | [ -n “$a” ] 返回 true。
$ | 检测字符串是否为空，不为空返回 true。 | [ $a ] 返回 true。

```bash
#!/bin/bash
 
a="abc"
b="efg"
 
if [ $a != $b ]
then
   echo "$a != $b : a 等于 b"
else
   echo "$a != $b: a 不等于 b"
fi 
# 输出：abc != efg : a 不等于 b
```

## 7 文件测试运算符

shell中的文件测试运算符用于检测在类unix系统中，文件的各种属性，如下表

操作符 | 说明 | 举例
------|-----|-----
-b file | 检测文件是否是块设备文件，如果是，则返回 true。 | [ -b $file ] 返回 false。
-c file | 检测文件是否是字符设备文件，如果是，则返回 true。 | [ -c $file ] 返回 false。
-d file | 检测文件是否是目录，如果是，则返回 true。 | [ -d $file ] 返回 false。
-f file | 检测文件是否是普通文件（既不是目录，也不是设备文件），如果是，则返回 true。 | [ -f $file ] 返回 true。
-g file | 检测文件是否设置了 SGID 位，如果是，则返回 true。 | [ -g $file ] 返回 false。
-k file | 检测文件是否设置了粘着位(Sticky Bit)，如果是，则返回 true。 | [ -k $file ] 返回 false。
-p file | 检测文件是否是有名管道，如果是，则返回 true。 | [ -p $file ] 返回 false。
-u file | 检测文件是否设置了 SUID 位，如果是，则返回 true。 | [ -u $file ] 返回 false。
-r file | 检测文件是否可读，如果是，则返回 true。 | [ -r $file ] 返回 true。
-w file | 检测文件是否可写，如果是，则返回 true。 | [ -w $file ] 返回 true。
-x file | 检测文件是否可执行，如果是，则返回 true。 | [ -x $file ] 返回 true。
-s file | 检测文件是否为空（文件大小是否大于0），不为空返回 true。 | [ -s $file ] 返回 true。
-e file | 检测文件（包括目录）是否存在，如果是，则返回 true。 | [ -e $file ] 返回 true。
