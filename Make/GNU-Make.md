**注：本文参考了《【正点原子】I.MX6U嵌入式Linux驱动开发指南V1.6.pdf》一书中关于Makefile的大部分内容。**



# GNU GCC简介

关于GCC的使用，可以参考我的另一篇博客：[GNU GCC学习 - zhengcixi - 博客园 (cnblogs.com)](https://www.cnblogs.com/mrlayfolk/p/17121077.html)

GCC简介：GCC（GNU Compiler Collection，GNU编译器集合），用于编译源程序为可执行文件、BIN文件或KO等。

## GCC编译选项

GCC命令格式如下：

```shell
gcc [选项] [文件名字]
```

常用的一些编译选项如下：

**-c**：只编译不链接为可执行文件，编译器将输入的.c 文件编译为.o 的目标文件。

**-o**：<输出文件名>用来指定编译结束以后的输出文件名，如果不使用这个选项的话 GCC 默认编译出来的可执行文件名字为 a.out。 

**-g**：添加调试信息，如果要使用调试工具(如 GDB)的话就必须加入此选项，此选项指示编译的时候生成调试所需的符号信息。

**-O**：对程序进行优化编译，如果使用此选项的话整个源代码在编译、链接的的时候都会进行优化，这样产生的可执行文件执行效率就高。

**-M**：自动生成依赖（c文件所需要的头文件），-M会包含系统头文件，-MM不包含系统头文件：

```shell
$ gcc -MM main.c
main.o: main.c add.h sub.h multi.h

$ gcc -M main.c
main.o: main.c /usr/include/stdc-predef.h /usr/include/stdio.h \      
 /usr/include/features.h /usr/include/sys/cdefs.h \
 /usr/include/bits/wordsize.h /usr/include/gnu/stubs.h \
 /usr/include/gnu/stubs-64.h \
 /usr/lib/gcc/x86_64-redhat-linux/4.8.5/include/stddef.h \
 /usr/include/bits/types.h /usr/include/bits/typesizes.h \
 /usr/include/libio.h /usr/include/_G_config.h /usr/include/wchar.h \ 
 /usr/lib/gcc/x86_64-redhat-linux/4.8.5/include/stdarg.h \
 /usr/include/bits/stdio_lim.h /usr/include/bits/sys_errlist.h add.h \
 sub.h multi.h
```

**-Werror**：警告当做错误处理，并且不会编译通过，比如未使用变量。

## GCC编译流程

GCC编译器的编译流程是：**预处理、编译、汇编和链接**。

**预处理：**就是展开所有的头文件、替换程序中的宏、解析条件编译并添加到文件中。

**编译：**是将经过预编译处理的代码编译成汇编代码，也就是我们常说的程序编译。

**汇编：**就是将汇编语言文件编译成二进制目标文件。

**链接：**就是将汇编出来的多个二进制目标文件链接在一起，形成最终的可执行文件，链接的时候还会涉及到静态库和动态库等问题。

```shell
$ gcc -E main.c > main.ii
$ gcc -S main.ii
$ gcc -c main.s
$ gcc main.o
main.o: In function `main':
main.c:(.text+0xf): undefined reference to `printf(char const*, ...)'
collect2: error: ld returned 1 exit status
```

## 编译动态库

动态链接库（.so库文件）：不会把代码编译到二进制文件中，而是在运行时去加载，只需要维护一个地址。

- -fPIC：产生位置无关的代码
- -shared：共享

- -l(小L)：指定动态库

- -I(大i)：指定头文件目录，默认当前目录

- -L：手动指定库文件搜索目录，默认只链接共享目录

动态：运行时才去加载，动态加载

链接：指库文件和二进制程序分离，用某种特殊手段维护两者之间的关系

库：库文件，.SO

- 编译动态库：

```shell
$ gcc -shared -fPIC add.c -o libadd.so
$ gcc -lcalc -L./ main.c 
$ ./a.out 
./a.out: error while loading shared libraries: libcalc.so: cannot open shared object file: No such file or directory
```

- 链接动态库

解决方式2种：

1）直接拷贝libcalc.so到标准库；

2）使用export LD_LIBRARY_PATH="./"

```shell
$ export LD_LIBRARY_PATH="./"
$ ./a.out 
10 + 5 = 115
10 - 5 = 5
10 * 5 = 50
```

使用Makefile：

```makefile
calc:libcalc.so
	$(CC) -lcalc -L./ main.c -o calc

libcalc.so:
	$(CC) -shared -fPIC add.c sub.c multi.c -o libcalc.so

clean:
	$(RM) calc
```

```shell
$ make
cc -shared -fPIC add.c sub.c multi.c -o libcalc.so
cc -lcalc -L./ main.c -o calc
```

# GNU Make说明

通常，在一个大型工程（有很多源文件）中，链接的速度快于编译的速度，重新编译仅更新过的文件可以节省大量的成本，只编译项目中修改过的文件可以通过`GNU Make`自动完成，GNU Make通过识别Makefile文件来对工程进行编译。

## make选项

Makefile常用选项（可通过命令make Makefile -h查看）

```shell
make [-f file] [options] [target]
```

Make默认在当前文件中寻找GNU makefile，makefile，Makefile的文件作为make的配置文件。

- -f：可以指定除上述文件名之外的文件作为输入文件
- -v：显示版本号
- -n：只输入命令，但并不执行，一般用来调试
- -s：只执行命令，但不显示具体命令，可在命令中用@符号抑制命令输出
- -w：显示执行前执行后的路径
- -C dir：执行makefile所在的目录

```shell
$ mv Makefile mk
$ make -f mk 
hello b
hello world

$ make -n
echo "hello b"
echo "hello world"

$ make -s
hello b
hello world

$ make -w
make: Entering directory `/home/xxx/Makefile_Study/code'
echo "hello b"
hello b
echo "hello world"
hello world
make: Leaving directory `/home/xxx/Makefile_Study/code'

$ cd ..
$ make -C code/
make: Entering directory `/home/xxx/Makefile_Study/code'
echo "hello b"
hello b
echo "hello world"
hello world
make: Leaving directory `/home/xxx/Makefile_Study/code'
```

## Makefile格式

Makefile由一系列规则组成，规则的格式如下：

```makefile
TARGET ... : PREREQUISITES... 
	COMMAND 
	... 
	...
```

TARGET（目标）：一般指编译的目标，也可以是一个目标，可以有多个目标。

PREREQUISITES（依赖）：指执行当前目标所要依赖的先项，包括其它目标，某个具体文件或库等，一个目标可以有多个依赖，也可以没有依赖。

COMMAND（命令）：该目标下要执行的具体命令，可以没有，也可以有多条，多条时，每条命令一行。

如果没有指定目标，默认使用第一个目标，如果执行，则执行对应的命令。

比如，规则的目标是main，生成main依赖于main.o input.o calcu.o。：

```makefile
main : main.o input.o calcu.o
	gcc -o main main.o input.o calcu.o
```

比如，目标a的依赖是b，目标b没有依赖：

```makefile
a:b
	@echo "hello world"

b:
	@echo "hello b"
```

```shell
$ make
hello b
hello world
```

比如：目标文件可以有多个：

```makefile
A B:
	@echo "hello"
```

```shell
$ make A
hello
$ make B
hello
```

## Makefile变量

### 系统变量

后面会详细的介绍。

- $*：不包括扩展名的目标文件名称
- $+：所有的依赖文件，以空格分隔
- $<：表示规则中的第一个依赖
- $?：所有时间戳比目标文件晚的依赖文件，以空格分隔
- $@：目标文件的完整名称
- $^：所有不重复的依赖文件，以空格分隔
- $%：如果目标是归档成员，则该变量表示目标的归档成员名称

### 系统常量

系统常量可用`make -p`命令查看

- AS：汇编程序的名称，默认为as

- CC：C编译器名称，默认为cc

- CPP：C预编译器名称，默认为$(CC) -E

- CXX：C++编译器名称，默认为g++

- RM：文件删除程序别名，默认为rm -f


### 自定义变量

Makefile中的变量都是字符串。

定义变量方式：

```
变量名=变量值
```

使用变量方式：

```
$(变量名)
${变量名}
```

比如：

```makefile
objects = main.o input.o calcu.o  # 定义一个objects变量
main : main.o input.o calcu.o
	gcc -o main $(objects)
```

比如：最基本的Makefile：

```makefile
calc:add.o sub.o multi.o main.o
	gcc add.o sub.o multi.o main.o -o calc

main.o:main.c
	gcc -c main.c -o main.o

add.o:add.c
	gcc -c add.c -o add.o

sub.o:sub.c
	gcc -c sub.c -o sub.o

multi.o:multi.c
	gcc -c multi.c -o multi.o
```

```shell
# 编译的顺序是写的顺序
$ make
gcc -c add.c -o add.o
gcc -c sub.c -o sub.o
gcc -c multi.c -o multi.o
gcc -c main.c -o main.o
gcc add.o sub.o multi.o main.o -o calc

# 修改了其中的一个文件，只会重新编译更新的文件
$ make
gcc -c add.c -o add.o
gcc add.o sub.o multi.o main.o -o calc
```

使用系统常量以及自定义变量进行替换：

```makefile
OBJ=add.o sub.o multi.o main.o
TARGET=calc

$(TARGET):$(OBJ)
	$(CC) $(OBJ) -o $(TARGET)

main.o:main.c
	$(CC) -c $^ -o $@

add.o:add.c
	$(CC) -c $^ -o $@

sub.o:sub.c
	$(CC) -c $^ -o $@

multi.o:multi.c
	$(CC) -c $^ -o $@

clean:
	$(RM) $(OBJ) $(TARGET)

show:
	@echo $(AS)
	@echo $(CC)
	@echo $(RM)
```

## Makefile赋值符

Makefile变量的赋值符如下。

### 赋值符“=”

```makefile
objects = main.o input.o calcu.o  # 定义一个objects变量
print:  
	@echo $(objects)  # “@”不输出命令执行过程
```

```shell
$ make print
main.o input.o calcu.o
```

### 赋值符“:=”

":="赋值符使用的是变量前一次的定义。

```makefile
name = linux
curname := $(name)
name = Linux4_10

print:
    @echo $(curname)  # 输出为linux
```

### 赋值符“?=”

```makefile
name ?= linux
name ?= Linux4_10

print:
    @echo $(name)  # 输出为linux
```

如果变量name前面没有被赋值，那么此变量就是“Linux4_10”，如果前面已经赋过值了，那么就使用前面赋的值。

### 变量追加“+=”

给已经定义好的变量添加一些字符串进去，就要使用到符号“+=”。

```makefile
name = linux
name += 4_10

print:
    @echo $(name)
```

```shell
$ make print   
linux 4_10
```

## Makefile模式规则

在编译.c源文件为.o源文件时，如果每一个C文件都写一个对应的规则，则文件增多的时候维护就很复杂，这时候就可以使用模式规则。

模式规则中，在规则的目标定义中要包含”%“，目标中的”%“表示对文件名的匹配，”%“表示长度任意的非空字符串，比如"%.c"就是所有的以.c结尾的字符串。当”%“出现在目标中的时候，目标中的”%“代表的值决定了依赖中的”%“值，用法如下：

```makefile
%.o:%.c
	命令
```

比如：

```makefile
objects = main.o input.o calcu.o
main: $(objects)
	gcc -o main $(objects)
	
%.o:%.c
	命令 # 命令的实现需要借助自动化变量，见下一章节

clean:
	rm *.o
	rm main
```



## Makefile自动化变量

**自动化变量：**把模式中所定义的一系列的文件自动的挨个取出，直至所有的符合模式的文件都取完，自动化变量只应该出现在规则的命令中。常见的自动化变量如下：

| **变量** | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| $@       | 规则中的目标集合，在模式规则中，如果有多个目标的话，“$@”表示匹配模式中定义的目标集合。 |
| $%       | 当目标是函数库的时候表示规则中的目标成员名，如果目标不是函数库文件，那么其值为空。 |
| $<       | 依赖文件集合中的第一个文件，如果依赖文件是以模式（即“%”）定义的，那么“$<”就是符合模式的一系列的文件集合。 |
| $?       | 所有比目标新的依赖目标集合，以空格分开。                     |
| $^       | 所有依赖文件的集合，使用空格分开，如果在依赖文件中有多个重复的文件，“$^”会去除重复的依赖文件，值保留一份。 |
| $+       | 和“$^”类似，但是当依赖文件存在重复的话不会去除重复的依赖文件。 |
| $*       | 这个变量表示目标模式中"%"及其之前的部分，如果目标是 test/a.test.c，目标模式为 a.%.c，那么“$*”就是 test/a.test。 |

7个自动化变量中，常用的三种：$@、$<和$^。

增加自动变量的Makefile如下：

```makefile
objects = main.o input.o calcu.o
main: $(objects)
	gcc -o main $(objects)
	
%.o:%.c
	gcc -c $<

clean:
	rm *.o
	rm main
```

```shell
$ make
gcc -c main.c
gcc -c input.c
gcc -c calcu.c
gcc -o main main.o input.o calcu.o
```

模式匹配：

```makefile
OBJ=add.o sub.o multi.o main.o
TARGET=calc

.PHONY:show

$(TARGET):$(OBJ)
	$(CC) $(OBJ) -o $(TARGET)

%.o:%.c
	$(CC) -c $^ -o $@

clean:
	$(RM) $(OBJ) $(TARGET)

show:
	@echo $(patsubst %.c, %.o, $(wildcard ./*.c))
```

```shell
$ make show
./multi.o ./main.o ./add.o ./sub.o
```

```Makefile
OBJ=$(patsubst %.c, %.o, $(wildcard ./*.c))
TARGET=calc

.PHONY:show

$(TARGET):$(OBJ)
	$(CC) $(OBJ) -o $(TARGET)

clean:
	$(RM) $(OBJ) $(TARGET)

show:
	@echo $(patsubst %.c, %.o, $(wildcard ./*.c))
```

wildcard：$(wildcard ./\*.c)获取当前目录下所有的c文件

patsubst：$(patsubst %.c,%.o,./*.c)将对应的c文件名替换成.o文件名

## Makefile伪目标

伪目标不代表真正的目标名，在执行make命令的时候通过指定这个伪目标来执行其所在规则的定义的命令。

使用伪目标主要是为了避免 Makefile 中定义的执行命令的目标和工作目录下的实际文件出现名字冲突，有时候我们需要编写一个规则用来执行一些命令，但是这个规则不是用来创建文件。

比如：清理工程的代码

```makefile
clean:
	rm *.o
	rm main
```

当我们输入“make clean”以后，后面的“rm *.o”和“rm main”总是会执行。如果当前目录下有一个名为“clean”的文件，当执行“make clean”的时候，规则因为没有依赖文件，所以目标被认为是最新的，因此后面的 rm 命令也就不会执行，我们预先设想的清理工程的功能也就无法完成。

```makefile
$ touch clean
$ make clean
make: `clean' is up to date.
```

伪目标可以解决上述的问题，伪目标声明方式如下：

```makefile
.PHONY:clean

clean:
    rm *.o
    rm main
```



## Makefile条件判断

Makefile支持条件判断，语法有两种，如下：

```makefile
<条件关键字>
	<条件为真时执行的语句>
```

```makefile
<条件关键字>
	<条件为真时执行的语句>
else
	<条件为假时执行的语句>
endif
```

其中条件关键字有4个：ifeq、ifneq、ifdef 和 ifndef，这四个关键字其实分为两对：ifeq与ifneq、ifdef 与 ifndef。

ifeq和ifneq，ifeq用来判断是否相等，ifneq就是判断是否不相等，ifeq 用法如下：

```makefile
ifeq (<参数 1>, <参数 2>)
ifeq '<参数 1 >','<参数 2>'
ifeq "<参数 1>", "<参数 2>"
ifeq "<参数 1>", '<参数 2>'
ifeq '<参数 1>', "<参数 2>"
```

```makefile
strip = need
foo = need

ifeq ($(strip), $(foo))
    ret = yes
endif

print:
    @echo $(ret)
```

```shell
$ make print   
yes
```

比较“参数 1”和“参数 2”是否相同，如果相同则为真，“参数 1”和“参数 2”可以为函数返回值。ifneq 的用法类似，只不过 ifneq 是用来了比较“参数 1”和“参数 2”是否不相等。

ifdef 和 ifndef 的用法如下：

```makefile
ifdef <变量名>
```

如果“变量名”的值非空，那么表示表达式为真，否则表达式为假。“变量名”同样可以是一个函数的返回值。

```makefile
bar =
foo = $(bar)
ifdef foo 
    frobozz = yes
else
    frobozz = no
endif

print:
    @echo $(frobozz)
```

```shell
$ make print
yes
```



## Makefile函数使用

Makefile中的函数是已经定义好的，我们直接使用，不支持我们自定义函数。

函数的用法如下：

```makefile
$(函数名 参数集合)
```

或者：

```makefile
${函数名 参数集合}
```

调用函数和调用普通变量一样，使用符号“$”来标识。参数集合是函数的多个参数，参数之间以逗号“,”隔开，函数名和参数之间以“空格”分隔开，函数的调用以“$”开头。

### 函数subst

函数subst用来完成字符串替换，调用形式如下：

```makefile
$(subst <from>,<to>,<text>)
```

此函数的功能是将字符串\<text>中的\<from>内容替换为\<to>，函数返回被替换以后的字符串。

```makefile
print:
    @echo $(subst zzq,ZZQ,my name is zzq)
```

```shell
$ make print   
my name is ZZQ
```

### 函数patsubst

函数 patsubst 用来完成模式字符串替换，使用方法如下：

```makefile
$(patsubst <pattern>,<replacement>,<text>)
```

此函数查找字符串\<text>中的单词是否符合模式\<pattern>，如果匹配就用\<replacement>来替换掉，\<pattern>可以使用通配符“%”，表示任意长度的字符串，函数返回值就是替换后的字符串。如果\<replacement>中也包涵“%”，那么\<replacement>中的“%”将是\<pattern>中的那个“%”所代表的字符串。

```makefile
print:
    @echo $(patsubst %.c, %.o, a.c b.c c.c)
```

```shell
$ make print   
a.o b.o c.o
```

### 函数dir

函数 dir 用来获取目录，使用方法如下：

```makefile
$(dir <names...>)
```

此函数用来从文件名序列\<names>中提取出目录部分，返回值是文件名序列\<names>的目录部分。

```makefile
print:
    @echo $(dir src/a.c)
```

```shell
$ make print   
src/
```

### 函数notdir

去除文件中的目录部分，也就是提取文件名，用法如下：

```makefile
$(notdir <names...>)
```

名称：取文件函数
功能：从文件名序列\<names>中取出非目录部分。非目录部分是指最后一个反斜杠（“/”）之后的部分。 
返回：返回文件名序列\<names>的非目录部分。 
示例：$(notdir src/foo.c hacks)返回值是“foo.c hacks”。

```makefile
print:
    @echo $(notdir src/a.c)
```

```shell
$ make print   
a.c
```

### 函数foreach

foreach 函数用来完成循环，用法如下：

```makefile
$(foreach <var>,<list>,<text>)
```

把参数\<list>中的单词逐一取出来放到参数\<var>中，然后再执行\<text>所包含的表达式。每次\<text>都会返回一个字符串，循环的过程中，\<text>中所包含的每个字符串会以空格隔开，最后当整个循环结束时，\<text>所返回的每个字符串所组成的整个字符串将会是函数 foreach 函数的返回值。

```makefile
names := a b c d
files := $(foreach n,$(names),$(n).o)
print:
    @echo $(files)
```

```shell
$ make print   
a.o b.o c.o d.o
```

### 函数wildcard

通配符“%”只能用在规则中，只有在规则中它才会展开，如果在变量定义和函数使用时，通配符不会自动展开，这个时候就要用到函数wildcard，使用方法如下：

```makefile
$(wildcard PATTERN...)
```

比如：

```makefile
print:
    @echo $(wildcard *.c)  # 获取当前目录下的所有C文件
```

```shell
$ make print   
calcu.c main.c input.c
```

### 函数addprefix

```makefile
$(addprefix <suffix>,<names...>)
```

名称：加前缀函数。 
功能：把前缀\<suffix>加到\<names>中的每个单词后面。 
返回：返回加过前缀的文件名序列。

```makefile
name=$(addprefix cc., a b c)
print:
	@echo $(name)
```

```shell
$ make
cc.a cc.b cc.c
```

### 函数addsuffix 

```makefile
$(addsuffix <suffix>,<names...>)
```

名称：加后缀函数。 
功能：把后缀\<suffix>加到\<names>中的每个单词后面。 
返回：返回加过后缀的文件名序列。

```makefile
name=$(addsuffix .txt, a b c)
print:
        @echo $(name)
```

```shell
$ make
a.txt b.txt c.txt
```

### 函数filter

```makefile
$(filter <pattern...>,<text>) 
```

名称：过滤函数。 
功能：以\<pattern>模式过滤\<text>字符串中的单词，保留符合模式\<pattern>的单词，可以有多个模式。 
返回：返回符合模式\<pattern>的字串。

```makefile
sources := foo.c bar.c baz.s ugh.h
name := $(filter %.c %.s, $(sources))
print:
	@echo $(name)
```

```shell
$ make
foo.c bar.c baz.s
```

### 函数filter-out

```makefile
$(filter-out <pattern...>,<text>) 
```

名称：反过滤函数。 
功能：以\<pattern>模式过滤\<text>字符串中的单词，去除符合模式\<pattern>的单词。可以有多个模式。 
返回：返回不符合模式\<pattern>的字串。

```makefile
sources := foo.c bar.c baz.s ugh.h
name := $(filter-out %.c %.s, $(sources))
print:
	@echo $(name)
```

```shell
$ make
ugh.h
```

### 函数export 

```makefile
export <variable ...>
```

功能：传递变量到下级Makefile中。



## 静态模式

静态模式可以更加容易地定义多目标的规则，可以让定义的规则变得更加的有弹性和灵活，语法如下：

```makefile
<targets ...> : <target-pattern> : <prereq-patterns ...>
	<commands>
	... ...
```

如果\<target-parrtern>定义成“%.o”，意思是我们的\<target>集合中都是以“.o”结尾的，而如果\<prereq-parrterns>定义成“%.c”，意思是对\<target-parrtern>所形成的目标集进行二次定义，其计算方法是，取\<target-parrtern>模式中的“%”（也就是去掉了[.o]这个结尾），并为其加上[.c]这个结尾，形成的新集合。所以“目标模式”或是“依赖模式”中都应该有“%”这个字符。

```makefile
OBJ=$(patsubst %.c, %.o, $(wildcard ./*.c))

all : $(OBJ)

$(OBJ) : %.o : %.c
	$(CC) -c $(CFLAGS) $< -o $@
```

上面的例子中，指明了我们的目标从$OBJ中获取，“%.o”表明要所有以“.o”结尾的目标，也就是变量$OBJ集合的模式，而依赖模式“%.c”则取模式“%.o”的“%”，并为其加下“.c”的后缀。而命令中的“$<”和“$@”则是自动化变量，“$<”表示所有的依赖目标集（也就是“\*.c”），“$@”表示目标集（也就是“\*.o”）。

## 通用Makefile

下面给出一个比较通用的Makefile，编译工程的目录结构如下：

```
├── header
│   └── xxx.h
├── include
│   ├── xxx.h
├── main.c
├── Makefile
└── src
    ├── xxx.c
    ├── drv
    │   └── xxx.c
    └── test
        └── xxx.c
```

Makefile文件：

```Makefile
# 编译工具链
# CROSS_COMPILE ?= arm-linux-gnueabihf-
CROSS_COMPILE ?=

# 生成的目标
TARGET ?= print

CC := $(CROSS_COMPILE)gcc
EXECUTABLE:= $(TARGET)

RM-F := rm -f

# 依赖的库文件目录
LIBDIR :=

# 依赖的库
LIBS := pthread

# 头文件目录
INCLUDES := header \
	include

# 源文件目录
SRCDIR := src \
	src/drv \
	src/test

# 编译选项
CFLAGS := -g -Wall -O0
CFLAGS += $(addprefix -I,$(INCLUDES))
CFLAGS += -I.

SRCS:= $(wildcard *.c) $(wildcard $(addsuffix /*.c, $(SRCDIR)))
OBJS:= $(patsubst %.c, %.o, $(SRCS))

.PHONY : all objs clean

all: $(EXECUTABLE)

objs: $(OBJS)

clean:
	@$(RM-F) $(OBJS)
	@$(RM-F) $(TARGET)

$(EXECUTABLE) : $(OBJS)
	$(CC) -o $(EXECUTABLE) $(OBJS) $(addprefix -L,$(LIBDIR)) $(addprefix -l,$(LIBS))
```



# Shell命令

在Linux下，Makefile中也可以直接执行shell命令，这里列出一些常用Shell命令：

- 找当前路径下的一级目录：

```shell
find . -maxdepth 1 -type d
```

- 查找当前路径下的所有目录包括子目录：

```shell
find -type d
```

# 常见问题

1、Makefile中字符串操作不需要加上双引号，如果加上了可能会有问题。

