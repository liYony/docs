# shell变量

## 1 命名变量

shell编程中，定义变量是直接定义的，没有明确的数据类型，shel允许用户建立变量存储数据，但是将认为赋给变量的值都解释为一串字符，如下:

```bash
cout=1			    # 定义变量		
name="ohuohuo"	    # 定义变量
echo $cout		    # 取变量值
echo $name	        # 取变量值
```

shell中，英文符号"$"用于取变量值。

> [!NOTE]shell编程的变量名的命名和其他语言一样，需要遵循一定的规则，规则如下：
> - 命名只能使用英文字母，数字和下划线，首个字符不能以数字开头
> - 中间不能有空格，可以使用下划线（_）
> - 不能使用标点符号
> - 不能使用bash里的关键字（可用help命令查看保留关键字）

如果在变量中使用系统命令，需要加上"`"符号（ESC键下方），如下所示（两者功能相同）：

```bash
DATE1=`date`	
DATE2=$(date)
```

## 2 使用变量

使用变量的时，用英文符号"$"取变量值，对于较长的变量名，建议加上{ }花括号，帮助解释器识别变量的边界，如下：

```bash
name="test_name"
echo "My name is ${name}and you"
```

加上方括号时即所有便后面的语句不留空格，shell也会自动识别边界，默认添加一个空格。
此外，已经定义过的变量，可以二次定义并重新被赋值覆盖上一次的变量值，这点如同其他语言。

## 3 变量类型

shell编程中也同样存在变量类型，在运行shell时会同时存在三种变量

- 局部变量：在脚本或命令中定义，仅在当前shell实例中有效，其他shell启动的程序不能访问局部变量。
- 环境变量：所有的程序，包括shell启动的程序，都能访问环境变量，必要的时候shell脚本也可以定义环境变量。
- shell变量：由shell程序设置的特殊变量。shell变量中有一部分是环境变量，有一部分是局部变量，不同类型的变量保证了shell的正常运行。

## 4 变量操作

shell中的变量，默认为可读可写类型，如果想要其只可读，如同url一样，需要将其声明为只读类型变量（如同const），使用readonly命令，如下脚本：

```bash
#!/bin/bash
Url="http://www.baidu.com"
readonly Url
Url="http://www.csnd.net"
```

这样的话，这句就会报错，提示/bin/sh: NAME: This variable is read only.此变量为只读变量。

如果想要删除变量，使用unset命令解除命令赋值，但是unset不能删除只读变量，如下所示：

```bash
#!/bin/sh
name="ohuohuo"
Url="http://www.baidu.com"
readonly Url	# 设置可读变量
unset name		# 可以被删除
unset Url		# 不可被删除
echo $name		# 不被打印出
echo $Url		# 打印出
```
