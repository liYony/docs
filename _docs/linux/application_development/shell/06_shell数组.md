# shell数组

在bash下，仅仅支持一维数组，并且没有限定数组的大小，不支持多维数组。类似于 C 语言，数组元素的下标由 0 开始编号（上述字符串也是这样）。获取数组中的元素要利用下标，下标可以是整数或算术表达式，其值应大于或等于 0。

## 1 定义数组

在 Shell 中，用括号()来定义表示数组，数组中元素用"空格"符号分割开。定义数组的一般形式为（三种定义形式均可）：

```bash
# 一般定义
array_name=(value1 value2 value3 value4)
 
# 多级定义
array_test=(
value1 
value2 
value3 
value4
)

# 
array_text[0]=value0
array_text[1]=value1
array_text[3]=value3
... 
...
```

## 2 数组操作

1. 读取数组：和读取变量名相同，使用$符号，需要加上下标名

```bash
valuen=${array_name[n]}
echo ${array_name[@]} # 读取所有
```

2. 获取数组长度：获取数组长度的方法与获取字符串长度的方法相同，如所示：

```bash
# 取得数组元素的个数
length=${#array_name[@]}	# 从头到尾取
# 或者
length=${#array_name[*]}	# 取所有
# 取得数组单个元素的长度
lengthn=${#array_name[n]}	# 取特定
```
