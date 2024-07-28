---
title: shell脚本入门
date: 2023-02-10 12:23:25
tags:
    - shell
categories:
    - linux

---



# shell基本用法

最近有写shell的需求，因此花了几个小时简单学习了下，在此做个简单的记录，学习资料为[菜鸟教程](https://www.runoob.com/linux/linux-shell.html)。个人博客：[singhe.art](singhe.art)

<!--more-->

## shell变量

```shell
for skill in Ada Coffe Action Java; do
	echo "I am good at ${skill}Script"
done
# There must be no spaces on both side of the equal sign, otherwise an error will be reported
your_name="tom"
echo $your_name
your_name="alibaba"
echo $your_name

# Read-only variable
myUrl="https://www.google.com"
readonly myUrl
# Error! variable.sh: line 13: myUrl: readonly variable
myUrl="https://www.runoob.com"

# unset url
unset_variable="https://www.runoob.com"
unset myUrl
# nothing output
echo $myUrl

```



## shell传递参数

```shell
echo "Shell 传递参数实例!";
echo "执行的文件名：$0";
echo "第一个参数为：$1";
echo "第二个参数为：$2";
echo "第三个参数为：$3";

echo "参数个数为: $#";
echo "传递的参数作为一个字符串显示：$*";

# diffrence between $* and $@
echo "-- \$* demonstration --"
for i in "$*"; do
	echo $i
done

echo "-- \$@ demonstration --"
for i in "$@"; do
	echo $i
done

```

使用：

```shell
sh ./parameter.sh 1 2 3 4 5
```



## shell数组

```shell
# Create a simple array
my_array=(A B "C" D)
# We can aslo use index to define array
array_name[0]=value0
array_name[1]=value1
array_name[2]=value2

# Read value from array
echo "The fisrt element is: ${my_array[0]}"
echo "The second element is: ${my_array[1]}"
echo "The third element is: ${my_array[2]}"
echo "The forth element is: ${my_array[3]}"

# Associative array(which is like dictionary in python)
site["google"]="www.google.com"
site["runoob"]="www.runoob.com"
site["taobao"]="www.taobao.com"
echo ${site["runoob"]}

# we can get all elements in array using @ or *
my_array[0]=A
my_array[1]=B
my_array[2]=C
my_array[3]=D
echo "All elements in array are: ${my_array[*]}"
echo "All elements in array are: ${my_array[@]}"
echo "All elements in site: ${site[*]}"
echo "All elements in site: ${site[@]}"
echo "All keys are: ${!site[*]}"

# Get The element numbers of array
echo "The number of array elements: ${#my_array[*]}"
echo "The number of array elements: ${#my_array[@]}"
```



## shell运算符

```shell
# There must be spaces on both side of '+'
# Arithmetic operator
val=`expr 2 + 2`
echo "The sum of two numers: $val"

a=10
b=20

val=`expr $a + $b`
echo "a + b: $val"

val=`expr $a - $b`
echo "a - b: $val"

val=`expr $a \* $b`
echo "a * b: $val"

val=`expr $b / $a`
echo "a / b: $val"

val=`expr $b % $a`
echo "a % b: $val"

if [ $a == $b ]
then
    echo "a is equal b"
fi

if [ $a != $b ]
then 
    echo "a is not equal b"
fi

# Relational operator
a=10
b=20
if [ $a -eq $b ]
then
    echo "$a -eq $b : a is equal b"
else
    echo "$a -eq $b : a is not equal b"
fi

if [ $a -ne $b ]
then
    echo "$a -ne $b : a is not equal b"
else
    echo "$a -ne $b : a is equal b"
fi

if [ $a -gt $b ]
then
    echo "$a -gt $b : a is greater than b"
else
    echo "$a -gt $b : a is not greater than b"
fi

if [ $a -lt $b ]
then
    echo "$a -lt $b : a is less than b"
else
    echo "$a -lt $b : a is not less than b"
fi

if [ $a -ge $b ]
then
    echo "$a -ge $b : a is greater than or equal b"
else
    echo "$a -ge $b : a is not greater than or euqal b"
fi

if [ $a -le $b ]
then
    echo "$a -le $b : a is less than or euqal b"
else
    echo "$a -le $b : a is not less than or euqal b"
fi

# Bool operator
a=10
b=20
if [ $a != $b ]
then
    echo "$a != $b : a is not equal b"
else
    echo "$a == $b : a is euqal b"
fi

# -a: and
if [ $a -lt 100 -a $b -gt 15]
then
    echo "$a is less than 100 and $b is greater than 15 is valid"
else
    echo "$a is less than 100 and $b is greater than 15 is not valid"
fi

# -o: or
if [ $a -lt 100 -o $b -gt 100 ]
then
    echo "$a is less than 100 or $b is greater than 100 is valid"
else
    echo "$a is less than 100 or $b is greater than 100 is not valid"
fi


# Logic operator
a=10
b=20

if [[ $a -lt 100 && $b -gt 100 ]]
then
    echo "return true"
else
    echo "return false"
fi

if [[ $a -lt 100 || $b -gt 100 ]]
then
    echo "return true"
else
    echo "return false"
fi

# Strign operator
a="abc"
b="efg"
if [ $a = $b ]
then
    echo "$a = $b : a is equal b"
else
    echo "$a = $b : a is not equal b"
fi

if [ $a != $b ]
then
    echo "$a != $b : a is not equal b"
else
    echo "$a != $b : a is equal b"
fi

if [ -z $a ]
then
    echo "-z $a : the length of string is 0"
else
    echo "-z $a : the length of string is not 0"
fi

if [ -n $a ]
then
    echo "-z $a : the length fo string is not 0"
else
    echo "-z $a : the length fo string is 0"
fi

if [ $a ]
then
    echo "$a : the string is not null"
else
    echo "$a : the strign is null"
fi


# file test operator
file="/Volumes/HDD/Users/singheart/Project/shell/learn/operator.sh"
if [ -r $file ]
then
    echo "file is readable"
else
    echo "file is not readable"
fi

if [ -w $file ]
then
    echo "file is writable"
else
    echo "file is not writable"
fi

if [ -x $file ]
then
    echo "file is executable"
else
    echo "file is not executable"
fi

if [ -d $file ]
then
    echo "file is directory"
else
    echo "file is not directory"
fi

if [ -s $file ]
then
    echo "file is not empty"
else
    echo "file is empty"
fi

if [ -e $file ]
then
    echo "file exists"
else
    echo "file does not exist"
fi

```



## shell echo命令

```shell
# output normal string
echo "It is a test"
# The double quotation marks can be omitted
echo It is a test

# Show escape characters
echo "\"It is a test\""

# Show varaible
echo "Please input your name:"
# read will read one line from standard input"
read name
echo "your name is $name"

# New line
echo "OK! \n"
echo "It is a test"

# Don't output new line
echo "OK! \c"
echo "It is a test"

# Redirect output to a file
echo "It is a test" > myfile

# single quotation
echo '$name\"'

# Show the result of command
echo `date`

```



## shell printf命令

```shell
printf "%-10s %-8s %-4s\n" name sex weight\(KG\)
printf "%-10s %-8s %-4.2f\n" xoxi man 66.1234
printf "%-10s %-8s %-4.2f\n" singheart man 66.1234
printf "%-10s %-8s %-4.2f\n" jiong woman 66.1234
```



## shell test命令

```shell
num1=100
num2=100
if test $[num1] -eq $[num2]
then
    echo "Two numbers are equal"
else
    echo "Two numbers are not equal"
fi

# []执行基本的算数运算
a=5
b=6
result=$[a+b]
echo "result is : $result"

# string test
num1="ru1noob"
num2="runoob"
if test $num1 = $num2
then
    echo "Two strings are equal"
else
    echo "Two strings are not equal"
fi

# file test
cd /bin
if test -e ./bash
then
    echo "file exists"
else
    echo "file does not exist"
fi

```



## shell awk命令

这里记录其最简单的用法。awk能够把输入的字符串按照空格进行分割，$0是字符串本身，$1是第一个子字符串，$2是第二个子字符串，以此类推。

下面的shell脚本能够获取某个文件的详细信息大小：

```shell
if [ ! -e $1 ]
then
    echo "$1 does not exist!"
    exit
fi

authority=$(ls -l $1 | awk '{print $1'})
file_size=$(ls -l $1 | awk '{print $5'})
echo "$1 authority is $authority"
echo "$1 size is $file_size bytes"

```



## shell tee命令

tee命令用于读取标准输入，将其内容输出到标准输出，同时保存成文件.

```shell
$ tee test_data
hello world
hello world
$ ls
test_data
$ cat test_data
hello world
```

hello world为我们在键盘上的输入，tee会将标准输入输出到标准输出，同时还会将其内容写到文件中。

## shell流程控制

```shell
a=10
b=20
# if else的[...]判断语句中大于使用-gt，小于使用-lt
# 如果使用((...))作为判断语句，大于和小于可以直接使用>和<
a=10
b=20
if [ $a == $b ]
then
    echo "a is equal b"
elif (( $a > $b ))
then
    echo "a is greater than b"
elif [ $a -lt $b ]
then
    echo "a is less than b"
else
    echo "no valid condition"
fi

# if_else is usually used with test
num1=$[2*3]
num2=$[1+5]
if test $[num1] -eq $[num2]
then
    echo "Two nums are equal"
else
    echo "Two nums are not equal"
fi

# for loop
for loop in 1 2 3 4 5
do
    echo "The value is : $loop"
done

for str in This is a string
do
    echo "value is $str"
done

# while loop
int=1
while (( $int <= 5 ))
do
    echo $int
    # use bash let command
    let "int++"
done

echo "Press <CTRL-D> exit"
echo "input your favorite site:\c"
while read FILM
do
    echo "Yes! $FILM is a greate site"
    echo "input your favorite site:\c"
done


# condition loop
a=0
until [ ! $a -lt 10 ]
do
    echo $a
    a=`expr $a + 1`
done

# case...esac
echo "input digit between 1 and 4:\c"
read aNum
case $aNum in
    1) echo "you choose 1"
    ;;
    2) echo "you choose 2"
    ;;
    3) echo "you choose 3"
    ;;
    4) echo "you choose 4"
    ;;
    *) echo "choosed number are not between 1 and 4"
    ;;
esac

site="runoob"
case "$site" in
    "runoob") echo "菜鸟教程"
    ;;
    "google") echo "google search"
    ;;
    "taoabo") echo "taobao"
    ;;
esac

# break
while :
do
    echo "input numer between 1 and 5:\c"
    read aNum
    case $aNum in
        1|2|3|4|5) echo "input number is $aNum!"
        ;;
        *) echo "input number is not between 1 and 5. Gameover!"
            break;
        ;;
    esac
done

# continue
while :
do
    echo "input number between 1 and 5:\c"
    read aNum
    case $aNum in
        1|2|3|4|5) echo "your input is $aNum!"
        ;;
        *) echo "your input is not between 1 and 5"
             continue
             echo "game over"
        ;;
    esac
done
```



## shell函数

```shell
demoFunc(){
    echo "this is a shell function"
}

demoFunc

funWithReturn(){
    echo "Please input the first number:\c"
    read aNum
    echo "Please input the second number:\c"
    read anotherNum
    echo "you input $aNum and $anotherNum"
    return $(($aNum+$anotherNum))
}
funWithReturn
# use $? to get return value from function
echo "The sum is $? !"

fun3(){
    echo "the first parameter is $1"
    echo "the second parameter is $2"
    echo "the third parameter is $3"
    echo "the tenth parameter is $10"
    echo "the tenth parameter is ${10}"
    echo "total parameter number is $#"
    echo "作为一个字符串输出所有参数 $* "
}
fun3 1 2 3 4 5 6 7 8 9 34 73

```



## shell输入输出重定向

| 符号 | 含义                                                       |
| ---- | ---------------------------------------------------------- |
| >    | 标准输出覆盖重定向：将命令的输出重定向输出到其他文件中     |
| >>   | 标准输出追加重定向：将命令的输出重定向输出到其他文件中     |
| >&   | 标识输出重定向：将一个标识的输出重定向到另一个标识的输入   |
| <    | 标准输入重定向：命令将从指定文件中读取输入而不是从键盘输入 |
| \|   | 管道符，从一个命令中读取输出并作为另一个命令的输入         |

### >的用法

```shell
$ ls
client.proto		node.proto		storage.proto
message.proto		server.proto		transaction.proto
$ ls > all_files.txt
$ ls -l all_files.txt
-rw-r--r--  1 singheart  admin  97  2 10 18:18 all_files.txt
$ cat all_files.txt
all_files.txt
client.proto
message.proto
node.proto
server.proto
storage.proto
transaction.proto
```

可以看出，语句`ls > all_files.txt` ls生成的标准输出，被重定向到了文件中，结果就是所有的输出都不见了，转而作为文件内容写入到了`all_files.txt`中.

> 值得注意的是，上面命令等价于`ls 1> all_files.txt`,也就是省略了代表标准输出的1。



### >> 用法

和`>`类似，只不过>>生成的文件内容是以追加的形式写入，而`>`是直接生成新的文件并覆盖



### >&的用法

`>&`用于将一个标识重定向到另一个标志。举个例子：

```shell
make 2>&1 | tee log.txt
```

此处的`2>&1`就是将标准错误重定向到标准输出中(也就是想把错误也记录下来，方便查日志)。`|`为管道标识符，将上一部分的标准输出作为下一个部分的标准输入。tee上面讲过，会将标准输入写入标准输出而且同时还写入文件中。

# shell实战

## new_item.sh

使用shell添加新的item，避免做一些重复的事情

```shell
if [ $# != 1 ]
then
    echo "Usage: sh new_item.sh [number]. \nfor example, sh new_item.sh 29"
    exit
else
    dir_name="item$1"
    if [ -e $dir_name ]
    then
        echo "$dir_name already exists!"
        exit
    else
        mkdir $dir_name
        touch $dir_name/CMakeLists.txt
        touch $dir_name/main.cpp
        echo "#include <iostream>\n\nint main() {\n    return 0;\n}" > $dir_name/main.cpp
        echo "add_executable(item$1_main main.cpp" > $dir_name/CMakeLists.txt
        echo "add_subdirecory(item$1)" >> CMakeLists.txt
        echo "$dir_name has successfully created!"
    fi
fi
```



## compare.sh

使用该shell来对比内联函数和普通函数的区别

```shell
# 比较普通函数和内联函数生成的可执行文件的大小
# 由此来判断内联函数的原理
inline_src_file="inline_function.cpp"
outline_src_file="outline_function.cpp"

inline_out_file="inline_out.app"
outline_out_file="outline_out.app"

# judge whether source file exists
if [ ! -e $inline_src_file ]
then
    echo "$inline_src_file does not exist!"
fi

if [ ! -e $outline_src_file ]
then
    echo "$outline_src_file does not exists!"
fi

# judge whether the binary executable file exists.
# If it exists, delete them.
if [ -e $inline_out_file ]
then
    echo "file $inline_out_file already exists, I will delete it!"
    rm -rf $inline_out_file
fi

if [ -e $outline_out_file ]
then
    echo "file $outline_out_file already exists, I will delete it!"
    rm -rf $outlie_out_file
fi

# compile with g++
g++ $inline_src_file -o $inline_out_file
g++ $outline_src_file -o $outline_out_file
if [ -e $inline_out_file -a -e $outline_out_file ]
then
    echo "$inline_out_file and $outline_out_file have generated"
else
    echo "Fail to generate $inline_out_file and $outline_out_file!"
fi

inline_app_size=$(ls -l $inline_out_file | awk '{print $5}')
outline_app_size=$(ls -l $outline_out_file | awk {'print $5}')
echo $inline_app_size
echo $outline_app_size
echo "The file size of $inline_out_file generated by $inline_src_file is $inline_app_size bytes."
echo "The file size of $outline_out_file generated by $outline_src_file is $outline_app_size bytes."
if (( $inline_app_size > $outline_app_size))
then
    echo "The executable file size generated with inline_function is bigger!"
fi
```

