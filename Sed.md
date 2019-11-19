# Sed

* sed: streaming editor. (流式编译器)

"sed"命令是一个可以将文件作为流进行编译的编译器。

		* 搜索

* 替换
* 删除
* 添加
* 改变

**sed [options] cmd file**

I.E.

~~~shell
sed -n 's/pattern/pattern/p' file
~~~

-n: --quiet, --silent

p:      Print the current pattern space

&: 使用相同字符串替换

sed -n 's/pattern/&/p' file

sed 's/-/*/2' file : 2 表示要替换每行上的第二个-（如果存在).

sed 's/-/*/2g' file : 替换从第二次开始出现的破折号。

sed '/--/ a "double dash before this line"' //匹配行后面加入

sed '/--/ i "double dash after this line"' //匹配行前加入

sed '/PATTERN/ c "this line is Top secret"' file  //替换

sed 'y/a/A' file //将所有a替换为A

sed -i 's/PATTERN/pattern/' file //就地更改

sed '/PATTERN/d' file