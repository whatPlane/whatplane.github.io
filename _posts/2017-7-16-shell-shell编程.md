---
layout:     post                    # 使用的布局（不需要改）
title:      shell编程              # 标题 
subtitle:     #副标题
date:       2017-7-16              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - shell
---

#### 编码报错
```
[root@instance-uukpmak2 myshell]# ./test.sh
-bash: ./test.sh: /bin/sh^M: bad interpreter: No such file or directory

```
一开始我在window上创建了一个 test.sh文件，然后上传到centos，执行是会报错。这个是编码问题。所以，最好直接在centos上创建shell脚本，然后同步到本地后修改。
#### 变量

```
#!/bin/sh
echo 'this file is create in linux'

# 变量类型有两种 字符串和数字
# 变量赋值时等于号周围不能有空格
# 使用变量一般写法是 $var ${var} 

name1=david #这种赋值不用双引号和单引号也可以
echo $name1 

#name2=david lau #这里会报错 因为中间有空格
#echo ${name2}

name3='david lau'
echo ${name3}

name4="face book"
echo ${name4}

# 单引号里面的内容全部当作字符
echo 'hi,man ${name3}'

# 双引号里面可以包括变量和转义字符 
echo "hi,man ${name3}"
echo "hi,man \"${name3}\",a nice day" #我这里用了 \" 转移字符

```
执行结果
```
david
david lau
face book
hi,man ${name3}
hi,man david lau
hi,man "david lau",a nice day

```
#### 字符串
字符串有3个基本操作
- 拼接

```
str="hi,${name3}"
echo ${str}
```


- 获得子字符串

```
echo ${name1:3:4} #字符串的下标从0开始
```

- 获得字符串长度

```
echo ${#name1} #可以认为 #name1 是在定义变量 name1 时自动定义的一个专门保存长度的变量

```

#### 字符串比较

这里要注意一下：字符串变量**是否存在**和**变量存在但是长度为0**这两个概念。我们之前已经看到了给变量赋值可以像下面这样
```
name=david
name="david"
name='david'
```
但是如果写 
```
name			#这样写会直接报错
name=			#这样写不会报错
name=""			#里面没有一个空格
name="   " 		#里面包含多个空格

function str_test(){
    #name
    #name=
    #name=""
    #name="      "

    if [ $name ] 
    then
        echo "name 存在"
    fi 
}

```
上面的结果除了第一个测试报错外，其他都表示name不存在。也就是说这几种赋值操作都相当于没做任何事情。
- [ $a = $b ]字符串如果相等 返回true
- [ $a != $b ] 字符串如果不相等 返回true
- [ -z $a ] 字符串长度为0则返回ture `在shell编程中这个跟a变量不存在仿佛是相等的？？？`
- [ -n "$a"] 字符串不为空 则返回true 注意跟上面那一行对比。




#### 整数比较
判断数字大小。举个例子判断分数大于80分。

```

if [ ${score} -gt 80 ] #注意中括号开始后又空格，结束前有空格
then
    echo "good,your score is: ${score} "
fi

```

- gt 大于
- lt 小于
- eq 等于
- ne 不等于
- ge 大于等于
- le 小于等于


比较的操作的 操作数必须都是整数。而且操作符两边要有空格。

#### 默认参数的意义
我执行 ./tmp.sh 44 88
```
#!/bin/bash
# 参数意义
param_name='$0'
echo "shell脚本文件名 ${param_name} : $0"
param_name='$1'
echo "第一个参数 ${param_name} : $1"
param_name='$2'
echo "第二个参数 ${param_name} : $2"
param_name='$*'
echo "所有参数 ${param_name} : $*"
param_name='$#'
echo "参数个数 ${param_name} : $#"

```
结果展示
```
shell脚本文件名 $0 : ./tmp.sh
第一个参数 $1 : 44
第二个参数 $2 : 88
所有参数 $* : 44 88
参数个数 $# : 2

```

#### 总结
在shell编程中，空格不能随意使用。 但是在一些运算操作的时候，空格又是必须要的。
#### 结束


