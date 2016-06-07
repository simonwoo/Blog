title: awk,sed,grep,find使用
date: 2016-1-30
tags: [Linux] #文章标签，可空，多标签请用格式，注意:后面有个空格
---
Awk, sed与Grep俗称Linux下三剑客。Sed是一种非交互式且面向字符流的编辑器（a non-interactive stream editor）, awk是一门模式匹配的编程语言，它主要用于匹配文本并处理，同时它也有一些编程语言才有的语法，如函数，分支循环语句，变量等。

## awk使用
awk语法：

```sh
awk [-F ERE] [-v assignment] ... program [argument ...]
awk [-F ERE] -f progfile ... [-v assignment] ...[argument ...]
```

program一般由多个pattern和action组成，当读入的记录匹配pattern时，才会执行相应的action。awk的输入被解析成多条record，默认情况下，记录分隔符是\n, 可以通过内置变量RS更改。因此可以认为一行是一条记录。每个记录被分割成多个Field，默认情况下字段的分隔符是空白符，例如空格，制表符等等，可以通过-F ERE选项改变或者通过内置变量FS更改。在awk中可以通过$1,$2...来访问对应位置的字段，同时$0存放着整个记录。

标准awk命令参数有三个：
- -F ERE : 定义字段分隔符，该选项的值可以是扩展的正则表达式（ERE）
-  -f progfile : 指定awk脚本，可以同时指定多个脚本，他们会按照在命令行出现的顺序连接在一起。
-  -v assignment : 定义awk变量，name = value, 赋值发生在awk处理文本之前。

## sed使用

sed语法：

```sh
sed [option] commands [file-to-edit]
```

sed执行流程：

![sed](http://7xq5i5.com1.z0.glb.clouddn.com/img_sed.png)


## grep使用
grep是一种强大的文本搜索工具，它使用正则表达式搜索文本，并把匹配的行打印出来。

语法：
```sh
grep [options] pattern [files]
```

例子：

```sh
echo "this is a word\nnext line" | grep word --color=auto
#正则表达式
echo "this is a word\nnext line." | grep -E "[a-z]*\."
# 统计个数
echo -e "1,2,3,4,5\n6,7" | grep -E -o "[0-9]" | wc -l
# 在该目录下递归搜索并忽略大小写
grep "text" . -R -n -i
```

## find使用
find是一个非常有效的工具，它可以遍历当前目录甚至整个文件系统来查找某些文件或者目录。

语法：

```sh
find pathname -options [-print -exec -ok]
```

## examples

```sh
#!/bin/bash
# 文件名: word_freq.sh
# 用途: 计算单词的频率

if[ $# -ne 1 ];
then
echo "Usage: $0 filename";
exit -1
fi

filename=$1
#\b是单词分界符
grep -E -o "\b[[:alpha:]]+\b" $filename | \
awk '{ count[$0]++ }
END {printf("%-14s%s\n", "Word","Count");
for(ind in count)
{printf("%-14s%s\n",ind,count[ind]); }
}'
```
