# shell command

## tr

* translate or delete characters

**tr [OPTION]... SET1 [SET2]**

* -d, --delete :  delete characters in SET1, do not translate
* -t, --truncate-set1: first truncate SET1 to length of SET2

I.E. 

~~~shell
$ echo "hello |word!" | tr "|" "\n"
hello 
word
~~~

## cut

* cut - remove sections from each line of files

**cut OPTION... [FILE]...**

* -c, --characters=LIST :  select only these characters
* -d, --delimiter=DELIM : use DELIM instead of TAB for field delimiter
* -f, --fields=LIST : select only these fields;  also print any line that contains no delimiter character, unless the -s option is specified

I.E.

~~~shell
$ echo "1:hello" | cut -c 2
:
$ echo "1:hello" | cut -d ":" -f1
1
~~~

