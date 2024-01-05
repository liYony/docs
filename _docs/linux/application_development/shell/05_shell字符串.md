# shell字符串

## 1 字符串类型

在shell中字符串是shell编程中最常用最有用的数据类型，字符串可以用单引号，也可以用双引号，也可以不用引号。

1. 使用单引号

```bash
str='this is a string'
```

使用单引号的不足：
- 单引号里的任何字符都会原样输出，单引号字符串中的变量是无效的。
- 单引号字串中不能出现单独一个的单引号（对单引号使用转义符后也不行），但可成对出现，作为字符串拼接使用。

2. 使用双引号

```bash
name="ohouhuoo"
str="please input your \"$name"\"
echo -e $str
# 输出：please input your ohouhuoo
```

使用双引号的优势：
- 可以在双引号中使用变量。
- 可以在双引号中使用转移字符。

## 2 字符串操作

1. 获取字符串长度：在对变量进行取值时，使用`#`符号对字符串进行取值。

```bash
string="abcd"
echo ${#string}
# 输出：4
```

2. 提取子字符串：使用字符串的截取命令，用于提取部分字符串。

```bash
string="this is a test"
echo ${string:2:6} # 表示从第3个字符开始截取
# 输出：is is
```

3. 查找字符串：用于查找字符的位置，输出结果为字符在字符串中所占的数据位置，如果查找多个字符，那哪个字母先出现就计算哪个，如下查找it中i和t两个字符，t先出现，输出为1

```bash
string="this is a test"
echo `expr index "$string" it`
# 输出：1
```
