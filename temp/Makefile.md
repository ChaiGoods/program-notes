# Makefile 编写

[TOC]

# Makefile 介绍

## 编写原则

1. 如果这个工程没有编译过，那么所有的 C 文件都需要编译并链接；
2. 如果这个工程的某几个 C 文件被修改，那么只编译被修改的 C 文件，并连接目标程序；
3. 如果这个工程的头文件被改变了，那么需要编译引用了这几个头文件的 C 文件，并连接目标程序。



## Makefile 规则

```makefile
target ...: prerequisites ...
    command
    ...
    ...
```

target 一个目标文件（Object File），或一个可执行文件，或一个标签（Label）

prerequisites 生成该 target 所依赖的文件以或/和 target

command 该 target 要执行的命令



# Makefile 示例

```makefile
edit: main.o kbd.o command.o display.o \
      insert.o search.o files.o utils.o
      cc -o edit main.o kbd.o command.o display.o \
      insert.o search.o files.o utils.o

main.o:    main.c defs.h
           cc -c main.c
kbd.o:     kdb.c defs.h command.h
           cc -c command.c
command.o: commnd.c defs.h command.h
           cc -c command.c
display.o: display.c defs.h buffer.h
           cc -c dislay.c
insert.o:  insert.c defs.h buffer.h
           cc -c insert.c
serch.o:   search.c defs.h buffer.h
           cc -c search.c
files.o:   files.c defs.h buffer.h command.h
           cc -c files.c
utils.o:   utils.c defs.h
           cc -c utils.c
           
clean:     
           rm edit main.o kbd.o command.o display.o \
           insert.o search.o files.o utils.o
```



## Make 工作方式

输入 make 命令后：

1. Make 会在当前目录下寻找名字叫做“Makefile”或“makefile”的文件；
2. 如果找到，继续找到文件中第一个target（目标文件）；
3. 如果 target 不存在，或者 target 后面依赖的“.o”文件的修改时间比 target 要新，那么，它会执行后面所定义的命令来生成 target 文件；
4. 如果 target 依赖的“.o”文件也不存在，那么 Make 会在当前文件中寻找目标为“.o”文件的依赖性，如果找到则再根据哪个规则生成“.o”文件；
5. 如果 C 文件和 H 文件存在，则 Make 会根据规则生成“.o”文件，从而生成最终的 target 文件。

Make 会层层递进的寻找依赖关系，直到编译出最终的 target，一旦依赖的文件不存在，那么 Make 就会报错。

clean 命令没有被 target 依赖，那么 Make 不会执行后面定义的命令，除非显式的执行：

```shell
make clean
```



## Makefile 中的变量

```makefile
objects = main.o kbd.o command.o display.o \
      insert.o search.o files.o utils.o

edit: $(objects)
      cc -o $(objects)

main.o:    main.c defs.h
           cc -c main.c
kbd.o:     kdb.c defs.h command.h
           cc -c command.c
command.o: commnd.c defs.h command.h
           cc -c command.c
display.o: display.c defs.h buffer.h
           cc -c dislay.c
insert.o:  insert.c defs.h buffer.h
           cc -c insert.c
serch.o:   search.c defs.h buffer.h
           cc -c search.c
files.o:   files.c defs.h buffer.h command.h
           cc -c files.c
utils.o:   utils.c defs.h
           cc -c utils.c

clean:     
           rm edit $(objects)
```



## Makefile 自动推导

Make 可以自动推导文件以及文件依赖后面的命令.

```makefile
objects = main.o kbd.o command.o display.o \
      insert.o search.o files.o utils.o

edit: $(objects)
      cc -o edit $(objects)

main.o:    defs.h
kbd.o:     defs.h command.h
command.o: defs.h command.h
display.o: defs.h buffer.h
insert.o:  defs.h buffer.h
search.o:  defs.h buffer.h
files.o:   defs.h buffer.h command.h
utils.o:   defs.h

.PHONY:    clean
clean:     
           rm edit $(objects)
```

`.PHONY` 表示 `clean` 是一个伪目标文件。


## 另类风格的 Makefile

```makefile
objects = main.o kbd.o command.o display.o \
      insert.o search.o files.o utils.o

edit: $(objects)
      cc -o edit $(objects)
      
$(objects): defs.h
            kbd.o command.o files.o: command.h
            display.o insert.o seatch.o files.o: buffer.h

.PHONY:     clean
clean:      
            rm edit $(objects) 
```



## clean 规则

```makefile
clean: 
        rm edit $(objects)
```
|
V
```makefile
.PHONY: clean
clean:  
        -rm edit $(objects)
```

“-”表示也许某些文件会出现问题，忽略继续做后面的事



## Makefile 里有什么？

1. 显式规则，它说明了如何生成一个或者多个目标文件。这是由 Makefile 的书写者明显指出要生成的文件、文件的依赖文件和生成的命令；
2. 隐晦规则，由于 Make 有自动推导功能，所以隐晦规则支持简写 Makefile；
3. 变量定义，在 Makefile 中我们要定义一些列的变量，一般都是字符串，类似 C 语言中的宏；
4. 文件指示，包括 3 部分：在一个 Makefile 中引用另一个 Makefile，类似 C 语言中的“include”；另一个是根据某些情况指定 Makefile 的有效部分，类似 C 语言中的预编译“#if”；最后就是定义一个多行命令；
5. 注释，Makefile 中只有行注释，使用“#”字符进行注释。

Makefile 中命令必须以 `[Tab]` 开始。



## Makfefile 文件名

默认情况下，Make 会在当前目录按顺序寻找名为“GNUmakefile”、“makefile”、“Makefile”的文件。

最好使用“Makefile”，更醒目，通用性更强。

可指定 Makefile 文件名：

```shell
make -f Make.my
make -file Make.abc
```



## 引用其他的 Makfile

使用 `include` 关键值字包含别的 Makefile，类似 C 语言中的 #include，被包含的文件内容会被扩展到当前包含的位置。

```makefile
include <filename>
```

```makefile
include foo.make *.mk $(bar)
```



Make 首先寻找 include 指出其他 Makefile，并将它们的内容扩展到当前位置。

Make 首先在当前目录寻找文件，找不到时，Make 还会在如下目录寻找：

1. 如果 Make 执行时，被指定了 `-I` 或 `--include-dir` 参数，那么 Make 就会在此参数指定的目录下寻找；
2. 如果目录 `<prefix>/include`（一般为 `/usr/local/bin` 或 `/usr/include`）存在，Make 也会在里面寻找。

```makefile
-include <filename>
```

“-”表示，无论 include 过程中出现什么错误，都不要报错，继续执行。

其他版本中有类似的 `sinclude` 兼容命令。



## 环境变量 MAKEFILES

如果当前环境中定义了 `MAKFEFILES` 变量，那么 Make 会把它当作类似 `include` 的动作。

此变量中的值为其他 Makefile 的定义，使用空格隔开。



## Make 工作方式

1. 读入所有 Makefile；
2. 读入被 include 的其他 Makefile；
3. 初始化文件中的变量；
4. 推导隐晦规则，并分析所有规则；
5. 为所有的目标文件创建依赖关系链；
6. 根据依赖关系，决定那些目标要重新生成；
7. 执行生成命令。



# 书写规则

规则包含两部分，一个是依赖关系，一个是生成目标的方法。

```makefile
foo.o: foo.c defs.h
       cc -c -g foo.c
```

## 规则语法

```makefile
targets: prerequisites
  command
  ...
```

```makefile
targets: prerequisites; command
  command
  ...
```

targets 是文件名，以空格分开，可以使用通配符。

command 是命令行，如果不与前面的内容在一行，那么必须以 `[Tab]` 键开头，如果在一行，可以以分号作为分隔。

prerequisites 也就是目标所依赖的文件。如果其中的某个文件比目标文件新，那么目标文件被认为是“过时的”，需要被重新生成。

如果命令太长，可以使用 `\` 作为换行符。

一般，Make 会以 UNIX 的标准 Shell，也就是 `/bin/sh` 来执行。



## 规则中使用通配符

Make 支持 3 个通配符，`*`、`?` 和 `~`。

`~` 表示当前用户的 `$HOME` 目录。

```makefile
objects: $(wildcard *.o)
```



## 文件搜寻

`VAPTH` 可指明 Make 在当前目录找不到的情况下，去指定的目录中去寻找文件，多个路径使用 `:` 分隔。

```
VPATH = src:.../headers
```

或使用 Make 中的 `vpath` 关键字。

```
vpath <pattern> <directories>
为符合模式 <pattern> 的文件指定搜索目录 <directories>。

vapth <pattern>
清除符合模式 <pattern> 文件的搜索目录。

vpath
清除所有以被设置好了的文件搜索目录。
```

`vpath` 中的 `<pattern>` 需要包含 `%` 字符，`%` 的意思是匹配零或若干字符。

```
vpath %.h ../headers
# 表示在 ../headers 目录下搜索所有以 .h 结尾的文件
```

多行 `vpath` 关键字，Make 将会按顺序执行搜索。

```
vpath %.c foo:bar
vpath %   blish
# 表示先在 foo 目录，然后 bar 目录，最后是 blish 目录搜寻 .c 的结尾文件
```



## 伪目标

伪目标并不是一个文件，只是一个标签。

避免和文件重名，使用 `.PHONY` 标记显式指定一个“伪目标”。

当伪目标放在第一行时，可作为“默认目标”，利用它实现同时编译出多个可执行文件。

```makefile
all:    prog1 prog2 prog3
.PHONY: all

prog1: prog1.o utils.o
       cc -o prog1 prog1.o utils.o

prog2: prog2.o
       cc -o prog2 prog2.o

prog3: prog3.o sort.o utils.o
       cc -o prog3 prog3.o sort.o utils.o
```

伪目标也可以成为依赖：

```makefile
.PHONY: cleanall cleanobj cleandiff

cleanall:  cleanobj cleandiff
           rm program

cleanobj:
           rm *.o

cleandiff:
           rm *.diff
```



## 多目标

Makefile 支持多个目标，有时多个目标会依赖于同一个文件，并且生成命令类似，可以将其合并起来。

自动化变量 `$@` 表示目前规则中所有目标的集合。

```makefile
bigoutput littleoutput: text.g
    generate text.g -$(subst output,,$@) > $@
```

等价于

```makefile
bigoutout: text.g
    gererate text.g -big > bigoutput
littleoutput: text.g
    generate text.g -little > littleoutput
```



## 静态模式

静态模式可以更容易的定义多目标的规则，使得规则更加灵活。

```makefile
<target ...>: <target-pattern>: <prereq-patterns ...>
    <commands>
    ...
```

targets 定义了一系列的目标文件，可以有通配符，是目标的集合。

target-pattern 指明了 targets 的模式，也就是目标集模式。

prereq-patterns 是目标的依赖模式，对 target-pattern 再进行一次依赖目标的定义。

```makefile
objects = foo.o bar.o

all: $(objects)

$(objects): %.o: %.c
        $(CC) -c $(CFLAGS) $< -o $@
```

上面指明了目标从 `$object` 中获取，`%.o` 表明所有以 `.o` 结尾的目标，后面的 `%.c` 取模式 `%.o` 的 `%` 部分，也就是 `foo bar`，并添加上 `.c` 的后缀，表示依赖的目标是 `foo.c bar.c`。

展开上述规则：

```makefile
foo.o: foo.c
       $(CC) -c $(CFLAGS) foo.c -o foo.o
bar.o: bar.c
       $(CC) -c $(CFLAGS) bar.c -o bar.o
```



## 自动生成依赖性

使用编译器命令自动生成依赖关系：

```shell
cc -M main.c
```

会生成：

```
main.o: main.c defs.h
```

GNU 建议把每一个源文件的依赖关系保存到一个对应的 `.d` 文件中。

可以让 Make 自动生成 `.d` 文件，并包含在 Makefile 中。

```makefile
%.d: %.c
    @set -e; rm -f $@; \
    $(CC) -M $(CPPFLAGS) $< > $@.$$$$; \
    sed 's,\($*\)\.o[ :]*,\1.o $@ : ,g' < $@.$$$$ > $@; \
    rm -f $@.$$$$
```

意思是，所有的 `.d` 文件依赖于 `.c` 文件， `rm -f $@` 意思是删除所有的目标，也就是 `.d` 文件，第二行意思是，为每个依赖文件 `$<` ，也就是 `.c` 文件生成依赖文件， `$@` 表示模式 `%.d` 文件，如果有一个 C 文件是 `a.c`，那么 `%` 就是 `a` ， `$$$$` 表示一个随机编号，第二行生成的文件可能是“name.d.12345”，第三行使用 `sed` 命令做了替换。

从而在编译器生成的依赖关系中加入了 `.d` 文件依赖。即：

```makefile
main.o: main.c defs.h
```

变成了

```makefile
main.o main.d: main.c defs.h
```



# 书写命令

## 显示命令

通常 Make 会把执行的命令在命令执行前打印出来，使用 `@` 字符可以不让 Make 显示命令。

```
@echo 正在编译 xxx 模块……
```

Make 执行时，只会显示出 `正在编译 xxx 模块……`，如果没有 `@`，那么显示：

```
@echo 正在编译 xxx 模块……
正在编译 xxx 模块……
```

使用 Make 带上 `-n` 或 `--just-print` 参数，那么只显示命令，不执行命令。

`-s` 和 `--silent` 或 `--quiet` 是全面禁止命令的显式。



## 命令执行

如果需要在前一个命令执行的基础上执行下一个命令，需要放在一行，以 `;` 隔开：

不能分两行执行。

```
exec:
        cd /home/l0neman/; pwd
```



## 嵌套执行 Make

大工程中，可能会把不同模块或者不同功能的源文件放在不同目录中，每一个目录中可以写一个 Makefile，有利于让 Makefile 变得更简洁，而不是全部都写在一个 Makefile 中。

```makefile
subsystem:
    cd subdir && $(MAKE)
```

等价于：

```makefile
subsystem:
    $(MAKE) -C subdir
```

`$(MAKE)` 是默认变量表示 Make 本身，使我们可以使用 Make 携带参数，上面两个 Makefile 都表示进入 subdir，然后执行 make 命令。

此 Makefile 被称为总控 Makefile，它可以将一些参数带入下一级 Makefile。

如果要传递变量到下一级，使用 `export`，如果不想，那么使用 `unexport`。

```makefile
export <varible ...>;
```

```makefile
unexport <varible ...>;
```

如：

```makefile
export variable = value
# 或
variable = value
export variable
# 或
export variable := value
# 或
export variable += value
```

当单独使用 `export` 占一行时，那么表示传递所有变量。

两个特殊变量一定会传递，`SHELL` 和 `MAKEFLAGS`。

`-w` 或是 `--print-directory` 会在 `Make` 的过程中输出目前的工作目录信息：

```
make: Entering directory `/home/l0neman/hello'.
make: Leaving directory `/home/l0neman/hello'
```



## 定义命令包

使用 `define` 开始，`endef` 结束，可以将一组命令定义为一个变量，成为命令包。

```makefile
define run-yacc
yacc $(firstword $^)
mv y.tab.c $@
endef
```

```makefile
foo.c: foo.y
       $(run-yacc)

```



# 使用变量

## 变量基础

使用变量，需要在变量名前加上 `$` 符号，最好使用 `()` 和 `{}` 被变量括起来，使用真实 `$` 变量，需要用 `$$` 表示。

```makefile
objects = program.o foo.o utils.o
program : $(objects)
    cc -o program $(objects)

$(objects) : defs.h
```

变量和 C 语言的宏一样，会在使用它的位置展开。

```makefile
foo = c
prog.o: prog.$(foo)
      $(foo)$(foo) -$(foo) prog.$(foo)
```

得到：

```makefile
prog.o: prog.c
        cc -c prog.c
```



## 变量的变量

使用 `=` 来将变量值赋值给另一个变量

```makefile
foo = $(bar)
bar = $(ugh)
ugh = Huh?

all:
    echo $(foo)
```

好处是可以把变量移动到后面定义：

```makefile
CFLAGS = $(include_dirs) -O
include_dirs = -Ifoo -Ibar
```

但可能会出现递归定义，Make 会检测到这种定义，从而报错。

```makefile
CFLAGS = $(CFLAGS) -o

A = $(B)
B = $(A)
```

可以使用 `:=` 来避免这种情况，`:=` 不允许使用后面定义的变量。

```makefile
x := foo
y := $(x) bar
x := later
```

一个复杂的变量例子：

```makefile
ifeq (0,${MAKELEVEL})
cur-dir   := $(shell pwd)
whoami    := $(shell whoami)
host-type := $(shell arch)
MAKE := ${MAKE} host-type=${host-type} whoami=${whoami}
endif
```

定义一个空格变量：

```makefile
nullstring :=
space := $(nullstring) # end of the line
```

`nullstring` 为一个 Empty 变量，表示什么都没有，那么 `space` 表示一个空格（在 `#` 号前面）。

注意，`#` 号符号前面的空格将会包含在变量中：

```makefile
dir := /foo/bar    # directory to put the frobs in
# 上面路径后面还包含了 4 个空格。
```

使用 `?=`，表示如果变量没有被定义过，那么变量的值就是右边的，否则什么也不做。

```makefile
FOO ?= bar
```

等价于：

```makefile
ifeq ($(origin FOO), undefined)
    FOO = bar
endif
```



## 变量高级用法

可以替换变量中共有的部分，格式是 `$(var:a=b` 或 `$(var:a=b)`，表示把变量 `foo` 中所有以 `a` 字符串结尾的 `a` 部分替换成 `b`。

```makefile
foo := a.o b.o c.o
bar := $(foo:.o=.c)
```

或“静态模式”

```makefile
foo := a.o b.o c.o
bar := $(foo:%.o=%.c)
```

- 变量再次当作变量

```makefile
x = y
y = z
a := $($(x))
# 那么 a = z
```

```makefile
x = $(y)
y = z
z = Hello
a := $($(x))
# 那么 a = Hello
```

```makefile
first_second = Hello
a = first
b = second
all = $($a_$b)
# 那么 all = $(first_second) = Hello
```

```makefile
a_objects := a.o b.o c.o
1_objects := 1.o 2.o 3.o

sources := $($(a1)_objects:.o=.c)
```

```makefile
ifdef do_sort
    func := sort
else
    func := strip
endif

bar := a d b g q c

foo := $($(func) $(bar))
```

```makefile
dir = foo
$(dir)_sources := $(wildcard $(dir)/*.c)
define $(dir)_print
lpr $($(dir)_sources)
endef
```



## 追加变量值

使用 `+=` 给变量追加值。

```makefile
objects = main.o foo.o bar.o utils.o
objects += another.o
```

可以解释为：

```makefile
objects = main.o foo.o bar.o utils.o
objects := $(objects) another.o
```

如果变量之前没有定义过，那么 `+=` 会自动变成 `=`。`+=` 会继承上次操作的赋值符。如果前一次的是 := ，那么 += 会以 `:=` 作为其赋值符，那么 `+=` 会以 `:=` 作为其赋值符。



## override 指示符

如果有变量是通过 Make 的命令行设置的，那么 Makefile 对这个变量的赋值将会被忽略。

如果想要设置这类参数，可使用 `override` 指示符。

```makefile
override <variable>; = <value>;

override <variable>; := <value>;
```

在 `define` 前使用 `override` 进行多行形式的变量定义：

```makefile
override define foo
bar
endef
```



## 多行变量

使用 `define` 关键字可以设置带有换行的变量值。

`define` 指示符后面跟的变量的名字，而重起一行定义变量的值，定义是以 `endef` 关键字结束。

```
define two-lines
echo foo
ech $(bar)
endef
```



## 环境变量

Make 运行时的系统环境变量在 Make 开始执行时被载入到 Makefile 文件中，如果 Makefile 中已经定义了这个变量，或者变量由 make 命令带入，那么系统的环境变量的值将被覆盖。
 
如果系统中定义了 `CFLAGS` 环境变量，那么就可以在所有的 `Makefile` 中使用这个变量了。



## 目标变量

可以为某个目标设置局部变量，被称为 `Target-specific Variable`，它可以和全局变量同名，由于它的作用域值在这条规则以及连带规则中，所以其值只在作用范围内有效，不会影响规则链以外的全局变量的值。

```makefile
<target ...>: <variable-assignment>

<target ...>: overide <variable-asignment>
```

<variable-assignment> 指的是各类赋值表达式，如 `=`、`:=`、`+=` 或 `?=`。

第二行针对 make 命令带入的变量，或是系统环境变量。

```makefile
prog : CFLAGS = -g
prog : prog.o foo.o bar.o
    $(CC) $(CFLAGS) prog.o foo.o bar.o

prog.o : prog.c
    $(CC) $(CFLAGS) prog.c

foo.o : foo.c
    $(CC) $(CFLAGS) foo.c

bar.o : bar.c
    $(CC) $(CFLAGS) bar.c
```

上面的示例，不管全局的 `$(CFLAGS)` 的值是什么，在目标 `prog` 中，以及其所引发的所有规则中（prog.o foo.o bar.o 的规则）， `$(CFLAGS)` 的值都是 `-g`。



## 模式变量

GNU 的 Make 还支持模式变量（Pattern-specific Variable），可以给定一种模式，把变量定义在符合这种模式的所有目标上。

模式（pattern）至少含有一个 `%`，定义如下，给所有以 `.o` 结尾的目标定义目标变量。

```makefile
%.o: FLAGS = -o
```

模式变量的语法和目标变量一样：

```makefile
<pattern ...>; : <variable-assignment>

<pattern ...>; : override <variable-assignment>
```



# 使用条件判断

## 示例

条件判断，可以让 Make 根据运行时的不同情况选择不同分支。

```makefile
libs_for_gcc = -lgnu
normal_libs =

foo: $(objects)
ifeq ($(CC),gcc)
    $(CC) -o foo $(objects) $(libs_for_gcc)
else
    $(CC) -o foo $(objects) $(normal_libs)
endif
```

上面的示例，目标 `foo` 根据变量 `$(CC)` 的值来选取不同的函数库编译程序。

上面也可写成如下：

```makefile
libs_for_gcc = -lgnu
normal_libs =

ifeq ($(CC),gcc)
    libs=$(libs_for_gcc)
else
    libs=$(normal_libs)
endif

foo: $(objects)
    $(CC) -o foo $(objects) $(libs)
```



## 语法

```makefile
<conditional-directive>
<text-if-true>
endif
```

```
<conditional-directive>
<text-if-true>
else
<text-if-false>
endif
```

<conditional-directive> 表示条件关键字。

- ifeq

比较两个参数的值是否相同，参数中还可以使用 Make 支持的函数。

```makefile
ifeq (<arg1>, <arg2>)
ifeq '<arg1>' '<arg2>'
ifeq "<arg1>" "<arg2>"
ifeq "<arg1>" '<arg2>'
ifeq '<arg1>' "<arg2>"
```

包含函数：

```makefile
ifeq ($(strip $(foo)),)
<text-if-empty>
endif
```

上面表示如果函数的返回值是空（Empty），那么 <text-if-empty> 生效。



- ifneq

比较两个参数是否不相同。

```makefile
ifneq (<arg1>, <arg2>)
ifneq '<arg1>' '<arg2>'
ifneq "<arg1>" "<arg2>"
ifneq "<arg1>" '<arg2>'
ifneq '<arg1>' "<arg2>"
```



- ifdef

判断变量的值不为空，则通过。

```makefile
ifdef <vaiable-name>
```

它不会扩展变量值到当前位置，只会判断是否有值。

- ifndef

与 ifdef 相反。

```makefile
ifndef <variable-name>
```

在 <conditional-directive> 这行，允许后面存在多余的空格，但是不能以 `[Tab]` 键作为开始。那么注释符 `#` 是安全的。

Make 在读取 Makefile 时就计算条件表达式的值，所以避免将自动化变量如 `$@` 放入条件表达式，因为它们是运行时才有的。

Make 不允许把整个条件语句分成两部分放在不同的文件中。



# 使用函数

Makefile 中可以使用函数来处理变量。

## 函数的调用语法

```makefile
$(<function> <arguments>)
```

或：

```makefile
${<function> <arguments>}
```

<function> 是函数名，<arguments> 为函数参数，多个参数使用 `,` 分隔。

```makefile
comma:= ,
empty:=
space:= $(empty) $(empty)
foo:= a b c
bar:= $(subst $(space),$(comma),$(foo))
```

上面使用了字符串替换函数 `subst`，将 `$(foo)` 中的空格替换成了 `,` 结果是 `$(bar)=a,b,c`


- 字符串处理函数

```makefile
# subst
$(subst, <from>,<to>,<text>)

# 名称：字符串替换函数
# 功能：把字符串 <text> 中的 <from> 字符串替换成 <to>
# 结果：返回被替换过后的字符串
# 示例：fEEt on the strEEt = $(subst ee,EE,feet on the street)
```

```makefile
# patsubst
$(patsubst <pattern>,<replacement>,<text>)

# 名称：模式字符串替换函数
# 功能：查找 <text> 中的单词（单词以“空格”、“Tab”或“回车”、“换行”分隔）是否符合模式 <pattern> ，如果匹配的话，则以 <replacement> 替换。这里， <pattern> 可以包括通配符 % ，表示任意长度的字串。如果 <replacement> 中也包含 % ，那么， <replacement> 中的这个 % 将是 <pattern> 中的那个 % 所代表的字串。（可以用 \ 来转义，以 \% 来表示真实含义的 % 字符）
# 结果：返回被替换过后的字符串
# 示例：x.c.o bar.o = $(patsubst %.c,%.o,x.c.c bar.c)
# 备注：$(var:<pattern>=<replacement>;) 相当于 $(patsubst <pattern>,<replacement>,$(var))，而 $(var: <suffix>=<replacement>) 则相当于 $(patsubst %<suffix>,%<replacement>,$(var)) 
```

```makefile
# strip
$(strip <string>)

# 名称：去除空格函数
# 功能：去除 <string> 字符串中开头和结尾的空字符
# 结果：返回被去掉空格的字符串
# 示例：a b c = $(strip a b c )
```

```makefile
# findstring
$(findstring <find>,<in>)

# 名称：查找字符串函数
# 功能：在字串 <in> 中查找 <find> 字串
# 结果：如果找到，那么返回 <find> ，否则返回空字符串
# 示例：
        a = $(findstring a,a b c)
        = $(findstring a,b c)
```

```makefile
# filter
$(filter <pattern...>,<text>)

# 名称：过滤函数
# 功能：以 <pattern> 模式过滤 <text> 字符串中的单词，保留符合模式 <pattern> 的单词，可以有多个模式
# 结果：返回符合模式 <pattern> 的字串
# 示例：
        sources := foo.c bar.c baz.s ugh.h
        foo: $(sources)
            cc $(filter %.c %.s,$(sources)) -o foo
        # 返回值：foo.c bar.c baz.s
```

```makefile
# filter-out
$(filter-out <pattern...>,<text>)

# 名称：反过滤函数
# 功能：以 <pattern> 模式过滤 <text> 字符串中的单词，去除符合模式 <pattern> 的单词，可以有多个模式
# 结果：返回不符合模式 <pattern> 的字串
# 示例： 
        objects = main1.o foo.o main2.o bar.o
        mains = main1.o main2.o
        # 返回值：foo.o bar.o
```

```makefile
# sort
$(sort <list>)

# 名称：排序函数
# 功能：给字符串 <list> 中的单词排序（升序）
# 结果：返回排序后的字符串
# 示例：bar foo lose = $(sort foo bar lose)
# 备注：sort 函数会去掉 <list> 中相同的单词
```

```makefile
# word
$(word <n>,<text>)

# 名称：取单词函数
# 功能：取字符串 <text> 中第 <n> 个单词。（从一开始）
# 结果：返回字符串 <text> 中第 <n> 个单词。如果 <n> 比 <text> 中的单词数要大，那么返回空字符串
# 示例：bar = $(word 2, foo bar baz)
```

```makefile
# wordlist
$(wordlist <ss>,<e>,<text>)

# 名称：取单词串函数
# 功能：从字符串 <text> 中取从 <ss> 开始到 <e> 的单词串。 <ss> 和 <e> 是一个数字
# 结果：返回字符串 <text> 中从 <ss> 到 <e> 的单词字串。如果 <ss> 比 <text> 中的单词数要大，那么返回空字符串。如果 <e> 大于 <text> 的单词数，那么返回从 <ss> 开始，到 <text> 结束的单词串
# 示例：bar baz = $(wordlist 2, 3, foo bar baz)
```

```makefile
# words
$(words <text>)

# 名称：单词个数统计函数
# 功能：统计 <text> 中字符串中的单词个数
# 结果：返回 <text> 中的单词数
# 示例：3 = $(words, foo bar baz)
# 备注：如果要取 <text> 中最后的一个单词，可以这样：$(word $(words <text>),<text>)
```

```makefile
# firstword
$(firstword <text>)

# 名称：首单词函数
# 功能：取字符串 <text> 中的第一个单词
# 结果：返回字符串 <text> 的第一个单词
# 示例：foo = $(firstword foo bar)
# 备注：可以用 word 函数来实现：$(word 1,<text>)
```

字符串函数实例：

利用搜索路径 `VPATH` 来指定编译器对头文件的搜索路径参数 CFLAGS

```makefile
override CFLAGS += $(patsubst %,-I%,$(subst :, ,$(VPATH)))
# $(VPATH) 值是 src:../headers ，那么 $(patsubst %,-I%,$(subst :, ,$(VPATH))) 将返回 -Isrc -I../headers 
```



- 文件名操作函数

每个函数的参数字符串都会被当做一个或是一系列的文件名来对待

```makefile
# dir
$(dir <names...>)

# 名称：取目录函数
# 功能：文件名序列 <names> 中取出目录部分。目录部分是指最后一个反斜杠（ / ）之前的部分。如果没有反斜杠，那么返回 ./
# 结果：返回文件名序列 <names> 的目录部分
# 示例：src/ ./ = $(dir src/foo.c hacks)
```

```makefile
# notdir
$(notdir <names...>)

# 名称：取文件函数
# 功能：从文件名序列 <names> 中取出非目录部分。非目录部分是指最後一个反斜杠（ / ）之后的部分
# 结果：返回文件名序列 <names> 的非目录部分
# 示例：foo.c hacks = $(notdir src/foo.c hacks)
```

```makefile
# suffix
$(suffix <names...>)

# 名称：取后缀函数
# 功能：从文件名序列 <names> 中取出各个文件名的后缀
# 结果：返回文件名序列 <names> 的后缀序列，如果文件没有后缀，则返回空字串
# 示例：.c .c = $(suffix src/foo.c src-1.0/bar.c hacks)
```

```makefile
# basename
$(basename <names...>)

# 名称：取前缀函数
# 功能：从文件名序列 <names> 中取出各个文件名的前缀部分
# 结果：返回文件名序列 <names> 的前缀序列，如果文件没有前缀，则返回空字串
# 示例：src/foo src-1.0/bar hacks = $(basename src/foo.c src-1.0/bar.c hacks)
```

```makefile
# addsuffix
$(addsuffix <suffix>,<names...>)

# 名称：加后缀函数
# 功能：把后缀 <suffix> 加到 <names> 中的每个单词后面
# 结果：返回加过后缀的文件名序列
# 示例：foo.c bar.c = $(addsuffix .c,foo bar)
```

```makefile
# addprefix
$(addprefix <prefix>,<names...>)

# 名称：加前缀函数
# 功能：把前缀 <prefix> 加到 <names> 中的每个单词后面
# 结果：返回加过前缀的文件名序列
# 示例：src/foo src/bar = $(addprefix src/,foo bar)
```

```makefile
# join
$(join <list1>,<list2>)

# 名称：连接函数
# 功能：把 <list2> 中的单词对应地加到 <list1> 的单词后面。如果 <list1> 的单词个数要比 <list2> 的多，那么， <list1> 中的多出来的单词将保持原样。如果 <list2> 的单词个数要比 <list1> 多，那么， <list2> 多出来的单词将被复制到 <list1> 中
# 结果：返回连接过后的字符串
# 示例：aaa111 bbb222 333 = $(join aaa bbb , 111 222 333)
```

- foreach 函数

```makefile
$(foreach <var>,<list>,<text>)
```

，把参数 `<list>` 中的单词逐一取出放到参数 `<var>` 所指定的变量中，然后再执行 `<text>` 所包含的表达式。每一次 `<text>` 会返回一个字符串，循环过程中， `<text>` 的所返回的每个字符串会以空格分隔，最后当整个循环结束时， `<text>` 所返回的每个字符串所组成的整个字符串（以空格分隔）将会是 `foreach` 函数的返回值

实例：

```makefile
names := a b c d
files := $(foreach n,$(names),$(n).o)
```

`$(name)` 中的单词会被挨个取出，并存到变量 `n` 中，`$(n).o` 每次根据 `$(n)` 计算出一个值，这些值以空格分隔，最后作为foreach函数的返回，那么，`$(files)` 的值是 `a.o b.o c.o d.o`



- if 函数

```makefile
$(if <condition>,<then-part>)
# 或
$(if <condition>,<then-part>,<else-part>)
```

`if` 函数可以包含“else”部分，或是不含。即 `if` 函数的参数可以是两个，也可以是三个。

`<condition>` 参数是 `if` 的表达式，如果其返回的为非空字符串，那么这个表达式就相当于返回真，于是，`<then-part>` 会被计算，否则 `<else-part>` 会被计算。

`if` 函数的返回值：如果 `<condition>` 为真（非空字符串），那个 `<then-part>` 会是整个函数的返回值，如果 `<condition>` 为假（空字符串），那么 `<else-part>` 会是整个函数的返回值，此时如果 `<else-part>` 没有被定义，那么，整个函数返回空字串。

所以，`<then-part>` 和 `<else-part>` 只会有一个被计算



- call 函数

唯一一个可以用来创建新的参数化的函数，可以写一个非常复杂的表达式，这个表达式中，可以定义许多参数，然后可以用 `call` 函数来向这个表达式传递参数。

```makefile
$(call <expression>,<parm1>,<parm2>,...,<parmn>)
```

当 Make 执行这个函数时，`<expression>` 参数中的变量，如 `$(1)` 、`$(2)` 等，会被参数 `<parm1>` 、`<parm2>` 、`<parm3>` 依次取代。而 `<expression>` 的返回值就是 `call` 函数的返回值。

示例：

```makefile
reverse =  $(1) $(2)
foo = $(call reverse,a,b)
```

`foo` 的值就是 `a b` ，参数的次序可以是自定义的，不一定是顺序的。

```makefile
reverse =  $(2) $(1)
foo = $(call reverse,a,b)
```

此时的 `foo` 的值就是 `b a`

备注：在向 `call` 函数传递参数时要尤其注意空格的使用。`call` 函数在处理参数时，第 `2` 个及其之后的参数中的空格会被保留，因而可能造成一些奇怪的效果。因而在向 `call` 函数提供参数时，最安全的做法是去除所有多余的空格。



- origin 函数

它并不操作变量的值，它告诉你这个变量的来源。

```makefile
$(origin <variable>)
```

<variable> 是变量的名字，不应该是引用（使用 `$` 符号）

返回值：

```
undefined    -> 如果 <variable> 从来没有定义过
default      -> 如果 <variable> 是一个默认的定义，比如“CC”这个变量
environment  -> 如果 <variable> 是一个环境变量，并且当 Makefile 被执行时，-e 参数没有被打开
file         -> 如果 <variable> 这个变量被定义在 Makefile 中
command line -> 如果 <variable> 这个变量是被命令行定义的
override     -> 如果 <variable> 是被 override 指示符重新定义的
automatic    -> 如果 <variable> 是一个命令运行中的自动化变量
```

用法实例：

有一个 Makefile 包含了一个定义文件 Make.def，在 Make.def 中定义了一个变量“bletch”，而此时环境中也有一个环境变量“bletch”，此时，判断如果变量来源于环境，那么就把之重定义，如果来源于 Make.def 或是命令行等非环境的，那么就不重新定义它。

```makefile
ifdef bletch
    ifeq "$(origin bletch)" "environment"
        bletch = barf, gag, etc.
    endif
endif
```



- shell 函数

它的参数就是操作系统 Shell 的命令，shell 函数把执行操作系统命令后的输出作为函数返回。

示例：

```makefile
contents := $(shell cat foo)
files := $(shell echo *.c)
```

备注：这个函数会新生成一个 Shell 程序来执行命令，所以你要注意其运行性能。



- 控制 Make 的函数

```makefile
$(error <text ...>)
# 产生一个致命的错误，<text ...> 是错误信息

$(warning <text ...>)
# 输出一段警告信息
```

可以将函数提前保存到变量，在合适的时候使用：

```makefile
ifdef ERROR_001
    $(error error is $(ERROR_001))
endif
```



# Make 的运行

## Make 的退出码

make 命令执行后有三个退出码：

0 表示成功执行

1 Make 运行时出现任何错误

2 如果使用了 Make 的 `-q` 选项，并且 `Make` 使得一些目标不需要更新

## 指定 Makefile
