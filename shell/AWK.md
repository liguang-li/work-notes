# AWK

awk不仅仅是Linux系统中的一个命令，而且是一种编程语言;可以用来处理数据和生成报告。

sed处理stream editor文本流。

## awk环境简介

~~~
$ ll `which awk`
lrwxrwxrwx. 1 root root 4 Oct 28 14:19 /usr/bin/awk -> gawk*

$ awk --version
GNU Awk 4.0.2
~~~

## awk格式

**awk [ options ] 'pattern { action }' file**

| name   | meaning            |
| ------ | ------------------ |
| record | 记录，行           |
| field  | 域，区域，字段，行 |

1. NF (number of field)表示一行中的区域数量，$NF取最后一个区域
2. $符号表示取某个列，$1,$2,$NF
3. NR (number of record)行号，awk对每一行的记录号都有一个内置变量NR来保存，每处理完一条记录NR的值就会自动+1.
4. FS (-F) field separator列分隔符，以什么把行分割成多列。

~~~shell
awk -F "#" '{ print $NF }' awk.txt
awk -F '[#$]' '{ print $NF }' awk.txt
awk -F "#" 'NR==1 { print $1 }' awk.txt
awk -F "#" 'NR==1' awk.txt #default print $0
awk -F "#" 'NF==1 {print $NF}NR==3 {print $NF}' awk.txt
~~~

$0: 表示整行

**FNR**: 对多文件记录不递增，每个文件都从１开始

~~~shell
#awk '{print NR}' awk.txt awk_space.txt
1
2
3
4
5
6
#awk '{print FNR}' awk.txt awk_space.txt
1
2
3
1
2
3
~~~

## AWK支持的正则表达式

| 元符号 | 功能                       | 实例        | 解释                             |
| ------ | -------------------------- | ----------- | -------------------------------- |
| ^      | 字符串开头                 | ^creditease | 匹配所有以creditease开头的字符串 |
| $      | 字符串结尾                 | creditease$ | 匹配所有以creditease结尾的字符串 |
| .      | 匹配任意单个字符           |             |                                  |
| *      | 重复０或多次前一个字符     |             |                                  |
| +      | 重复１次或多次前个字符     |             |                                  |
| ?      | 匹配０和１次前字符         |             |                                  |
| []     | 匹配指定字符组内的任一字符 |             |                                  |
| [^]    | 匹配不在指定字符组内的字符 |             |                                  |
| 0      | 子表达式组合               |             |                                  |
| \|     | 或                         | /(cool)\|B/ | 匹配cool或Ｂ                     |

| ~    | 用于对记录或区域的表达式进行匹配 |
| ---- | -------------------------------- |
| !~   | 用于表达与~相反的意思            |

**显示包含321的行**

~~~shell
$ awk '/321/{print $0}' awk.txt
CBA#DEF#GHI#321
~~~

**以#为分隔符，显示第一列以B或C开头的行**

~~~shell
awk -F "#" '$1~/^B|^C/{print $0}' awk.txt  #~表示对记录或区域的表达式进行匹配
awk -F "#" '$1~/^[BC]/{print $0}' awk.txt
awk -F "#" '$1~/^(B|C)/{print $0}' awk.txt
~~~

**比较表达式**

awk是一种编程语言，能够进行更为复杂的判断，当条件为真时，awk就执行相关的action,主要是针对某一区域做出相关的判断。

| 运算符 | 含义 | 实例 |
| ------ | ---- | ---- |
| <      |      |      |
| <=     |      |      |
| ==     |      |      |
| !=     |      |      |
| >=     |      |      |
| >      |      |      |

## Awk模块，变量与执行

~~~shell
awk 'BEGIN{coms} /pattern/{coms} END{coms}'
~~~

* BEGIN模块

  BEGIN模块在awk读取文件之前就执行，BEGIN模式常常被用来修改内置变量ORS,RS,FS,OFS, etc.

* AWK内置变量

  | 变量名      | 属性                                         |
  | ----------- | -------------------------------------------- |
  | $0          | 当前记录，整行                               |
  | $1,$2,...$a | 当前记录的第n个区域，区域间由FS分割          |
  | FS          | 分隔符                                       |
  | NF          | 当前记录的区域个数，就是有多少列             |
  | NR          | 已经读出的记录数，行号                       |
  | RS          | 输入的记录分隔符默认为空格. Record separator |
  | OFS         | 输出区域分隔符，output record separator      |
  | FNR         | 当前文件的读入记录号，每个文件重新计算       |
  | FILENAME    | 当前正在处理的文件的文件名                   |

* END模块

  END在awk读取完所有的文件再执行。

## awk数组

**arrayname[string]=value**

people[police]=110

~~~shell
awk 'BEGIN{word[0]="credit";word[1]="easy";for(i in word) print word[i]}'
~~~

* 数组分类

  * 索引数组：以数字为下标
  * 关联数组：以字符串为下标

* awk关联数组

  ~~~shell
  [root@creditease awk]# cat url.txt
  http://www.baidu.com 3
  http://mp4.video.com
  http://www.qq.com
  http://www.qq.com
  http://www.baidu.com
  http://www.baidu.com
  [root@creditease awk]# awk -F "[/]+" '{h[$2]++}END{for(i in h) print i,h[i]}' url.txt
  www.baidu.com 3
  mp4.video.com 1
  www.qq.com 2
  ~~~

## awk语法

* sub gsub

  * sub(r,s,目标)	:替换行内匹配的第一次内容

  * gsub(r,s,目标)  :匹配行内所有内容

~~~shell
awk '{sub(/A/,"a");print $0}' sub.txt
awk '{gsub(/A/,"a");print $0}' sub.txt
~~~

* if/else

~~~shell
awk '{if($0~/AA/){print $0" YES"}else{print $0" No YES"}' ifelse.txt
~~~

* next:跳过后面的所有代码

~~~c
awk '$0~/AA/{print $0" YES";next}{print $0" NO, YES"}' ifelse.txt
~~~