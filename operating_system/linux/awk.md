> https://www.cnblogs.com/quincyhu/p/5884390.html

## 语法格式

```bash
awk [options] 'script' var=value file(s) 
awk [options] -f scriptfile var=value file(s)
```

## 常用选项

- -F fs fs指定输入分隔符，可以是字符串或者正则表达式
- -v var=value 赋值一个用户定义变量 将外部变量传递给awk
- -f scriptfile 从脚本文件读取awk命令

## awk脚本

由模式和操作组成

```shell
awk 'BEGIN{ commands } pattern{ commands } END{ commands }' file 
```

**END 和 BEGIN 必须写 而且要大写**

## 模式

- 正则表达式
- 关系表达式
- 模式匹配表达式 ~匹配 !~不匹配

## awk执行过程分析

- 执行 BEGIN{commands} 语句
- 从文件中读取一行，然后执行 pattern{ commands }，逐行扫描文件，挨个执行 pattern{ commands }，知道文件被读取完
- 读取完文件之后执行

## awk内置变量

- **$n** : 当前行的第n个字段(以空格为分隔符)
- **$0** : 当前行
- **FILENAME** : 当前输入文件的名。
- **NR** : 表示记录数，在执行过程中对应于当前的行号
- **FNR** : 同NR :，但相对于当前文件。

- **FS** : 字段分隔符（默认是任何空格）
- **NF** : 表示字段数，在执行过程中对应于当前的字段数。 `print $NF`答应一行中最后一个字段

- **OFS** : 输出字段分隔符（默认值是一个空格）。
- **ORS** : 输出记录分隔符（默认值是一个换行符）。
- **RS** : 记录分隔符（默认是一个换行符）。



- **ARGC** : 命令行参数的数目。
- **ARGIND** : 命令行中当前文件的位置（从0开始算）。
- **ARGV** : 包含命令行参数的数组。
- **CONVFMT** : 数字转换格式（默认值为%.6g）。
- **ENVIRON** : 环境变量关联数组。
- **ERRNO** : 最后一个系统错误的描述。
- **FIELDWIDTHS** : 字段宽度列表（用空格键分隔）
- **IGNORECASE** : 如果为真，则进行忽略大小写的匹配。

- **OFMT** : 数字的输出格式（默认值是%.6g）。
- **RSTART** : 由match函数所匹配的字符串的第一个位置。
- **RLENGTH** : 由match函数所匹配的字符串的长度。
- **SUBSEP** : 数组下标分隔符（默认值是34）。

## 将外部变量值传递给awk

- awk -v varchar=$xx '{xxx}'
- awk '{}' v1=$v1 v2=​\$v2

## 循环

### for

```shell
for(xx in xxx) {
	语句
}
for(i=0; i<n; i++) {}
```

### while

同java 除了变量声明

### do…while

同java 除了变量声明

- break、continue 同
- next 读取下一输入行
- exit 退出主输入循环 进入end 

