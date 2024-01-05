# shell传递参数

在执行 Shell 脚本时，向脚本传递参数，脚本内获取参数的格式为：$n。n 代表一个数字，1 为执行脚本的第一个参数，2 为执行脚本的第二个参数，以此类推，脚本编写如下,保存为test.sh。

```bash
echo "传递参数实例！";
echo "执行的文件名：$0";
echo "第一个参数为：$1";
echo "第二个参数为：$2";
echo "第三个参数为：$3";
```

在使用shell传递参数的时候，常常需要用到以下的几个字符来处理参数：

https://zhuanlan.zhihu.com/p/57784678

## 1 \$* 和 \$@

在 Bash 中没有双引号时, 它们两个被扩展后, 结果是一样的, 都是表示外部输入的参数列表。

当有双引号时, 如 “\$*”, “\$@”, 这个时候, 前者表示的是用 IFS (Internal Field Separator) 分隔符连接起来的统一字符, 后者则表示的是输入的每个参数.

```bash
#!/bin/bash

export IFS=%

cnt=1
for i in “$*”
do
    echo “Number of $cnt parameter is: $i”
    (( cnt++ ))
done

echo
echo 

cnt=1
for i in “$@”
do 
    echo “Number of $cnt parametre is: $i”
    (( cnt++ ))
done
```

执行结果：
```shell
./test_1.sh “Hello, how are you?” Second Third Fourth
# 输出：
Number of 1 parameter is: Hello, how are you?%Second%Third%Fourth

Number of 1 parameter is: Hello, how are you?
Number of 2 parameter is: Second
Number of 3 parameter is: Third
Number of 4 parameter is: Fourth
```

被双括号后, “\$*” 表示的是用内部分割符 IFS 连接起来的一个完整的统一字符串, 注意上面的打印输出只有一个参数.

而 ”\$@” 仍然表示的是各个输入的参数. 所以这也就解释了, 除非特殊情况, 为什么推荐使用 \$@ 而不是 \$* 展开参数列表了.
