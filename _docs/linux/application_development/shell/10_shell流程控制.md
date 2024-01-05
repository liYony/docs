# shell流程控制

## 1 if else条件

shell中的if else条件具有一定的模版。其调用格式为：`if - then - fi`：

```bash
# 写成多行
if condition
then
    command1 
    command2
    ...
    commandN 
fi
 
# 写成单行
if condition;then command1;command2;fi

# 存在不满足情况
if condition
then
    command1 
    command2
    ...
    commandN
else
    command
fi

# 多层嵌套的情况
if condition1
then
    command1
elif condition2 
then 
    command2
else
    commandN
fi
```

```bash
num1=$[6]
num2=$[8]
if test $[num1] -eq $[num2]
then
    echo '两个数字相等!'
else
    echo '两个数字不相等!'
fi
# 输出： 两个数字不相等
```
## 2 case条件

shell中case语句为多功能选择语句，与其他语言相通的是，可以用case语句匹配一个值与一个模式，如果匹配成功，执行相匹配的命令。case语句调用格式如下：

```bash
case 值 in
模式1)
    command1
    command2
    ...
    commandN
    ;;
模式2）
    command1
    command2
    ...
    commandN
    ;;
esac
```

```bash
echo '输入 1 到 4 之间的数字:'
echo '你输入的数字为:'
read num
case $num in
    1)  echo '你选择了 1'
    ;;
    2)  echo '你选择了 2'
    ;;
    3)  echo '你选择了 3'
    ;;
    4)  echo '你选择了 4'
    ;;
    *)  echo '你没有输入 1 到 4 之间的数字'
    ;;
esac
```

case中想要跳出循环有两个命令：break和continu。

1. break：允许跳出所有循环（中止执行后面所有的循环）

```bash
#!/bin/bash
while :
do
    echo -n "输入 1 到 5 之间的数字:"
    read num
    case $num in
        1|2|3|4|5) echo "你输入的数字为 $num!"
        ;;
        *) echo "你输入的数字不是 1 到 5 之间的! 游戏结束"
            break
        ;;
    esac
done
# 输出：
# 输入 1 到 5 之间的数字:3
# 你输入的数字为 3!
# 输入 1 到 5 之间的数字:7
# 你输入的数字不是 1 到 5 之间的! 游戏结束
```

2. contimue：shell中的continue命令与break命令类似，只有一点差别，它不会跳出所有循环，仅仅跳出当前循环。这一点和其他类型的语言相同。

```bash
#!/bin/bash
while :
do
    echo -n "输入 1 到 5 之间的数字: "
    read num
    case $num in
        1|2|3|4|5) echo "你输入的数字为 $num!"
        ;;
        *) echo "你输入的数字不是 1 到 5 之间的!"
            continue
            echo "游戏结束"
        ;;
    esac
done
```

## 3 for循环

shell中的for循环调用格式和python中的for循环有点类似，也是有模板的：

```bash
for var in item1 item2 ... itemN
do
    command1
    command2
    ...
    commandN
done
 
# 写成一行同样使用分号将语句分开 
```

> 需要注意的是：
> - in列表中可以包含替换、字符串和文件名等
> - in列表是可选的，如果默认不适用，将会循环使用命令行中的位置参数

## 4 while循环

shell中的while循环用于不断执行一系列命令，也用于从输入文件中读取数据，调用格式如下

```bash
while condition
do
    command
done
```

## 5 until循环

until 循环执行一系列命令直至条件为 true 时停止。until 循环与 while 循环在处理方式上刚好相反。一般 while 循环优于 until 循环，但在某些时候—也只是极少数情况下，until 循环更加有用。until循环调用格式：

```bash
until condition
do
    command
done
```

```bash
#!/bin/bash
 
a=0
until [ ! $a -lt 5 ]
do
   echo $a
   a=`expr $a + 1`
done
# 输出：
# 0
# 1
# 2
# 3
# 4
# 5
```
