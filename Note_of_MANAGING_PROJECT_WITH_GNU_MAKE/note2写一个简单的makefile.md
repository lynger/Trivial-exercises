#规则 Rules

make 通过依赖链图来更新所要的目标。make 有许多的不同规则。

*显示规则* 前面一篇的例子就是*显示规则*，通常你的规则都是这么写的。
*模式规则* （Pattern rules）用通配符来代替显式的文件名。
*隐式规则* 就是 make 内建使用的显示规则或是后缀规则。内建规则使得写 makefile 更容易，因为一般任务下文件类型、后缀以及要更新的目标程序都是已知的。
*静态模式规则*（Pattern rules） 类似于正则表达规则（regular pattern rules），只让它们应用在特定的目标文件上。

GNU make 可以替换其它版本的 make，它包含一些兼容的特性。后缀规则是 make 的一般规则，GNU make 支持它，但是这种规则被认为将被模式规则替代，后者更清晰和一般化。

## 显式规则 Explicit Rules

大多数规则属于这种。一个规则可以有多个目标，只要依赖相同。比如：

	vpath.o variable.o: make.h config.h getopt.h gettext.h dep.h

等价于：

	vpath.o : make.h config.h getopt.h gettext.h dep.h
	variable.o: make.h config.h getopt.h gettext.h dep.h

## 通配符 Wildcards

示例

	prog: *.c
	      $(CC) -o $@ $^


## 伪目标 Phony Targets

有时候为了要执行一些命令脚本，可以做个伪目标的标签
示例：
	clean:
		rm -f *.o

make 无法区分目标和伪目标，因此上例运行 make 会得到

	$ make clean
	make: `clean‘ is up to date

为了避免这种情形，伪目标前需要被 .PHONY 依赖，表示伪目标不是一个文件，示例：

	.PHONY: clean
	clean:
		rm -f *.o

make 的输出会有些让人困惑：makefile 的书写是 top-down 的方式，但是命令执行确实 bottom-up;另外规则被执行时也没有指示。而伪目标正可以改善这些，看一个来自 bash 的 makefile 示例:

	$(Program): build_msg $(OBJECTS) $(BUILD_DEP) $(LIBDEP)
		$(RM) $@
		$(CC) $(LDFLAGS) -O $(Program) $(OBJECTS) $(LIBS)
		ls -l $(Program)
		size $(Program)

	.PHONY: build_msg
	build_msg:
		@printf "#\n# Building $(Program)\n#\n"

由于 build_msg 是伪目标，打印消息在任何依赖之前。.PHONY build_msg 目标会使 $(Program) 视为“旧的”，需要更新。所幸，在这种情况下，做中的目标文件只是需要链接一下。

make 的标准伪目标有以下几个
伪目标 | 功能
---|---
all | 运行所有任务构建应用程序
install | 安装二进制应用程序
clean | 删除生成的二进制程序
distclean | 删除所有非源代码的生成文件
TAGS | 生成一个标签表给编辑者使用
info | 从 Texinfo 源中创建 GNU info 文件
check | 运行应用程序的相关测试

## 空目标

示例

	prog: size prog.o
	      $(CC) $(LDFLAGS) -o $@ $^

	size: prog.o
	      size $^
	      touch size

上例中 size 规则使用 touch 创建一个空文件 size。当 prog.o 被更新了空文件 size 的时间戳被更新。

#变量

变量的语法为：

	$(variable-name)

##自动变量 Automatic Variables

	$@ 	     目标文件文件名
	$%	     库文件中成员的文件名
	$<	     第一个依赖的文件名
	$?	     所有新于目标的依赖的文件名，以空格隔开
	$^	     所有依赖的文件名，不含重复项，以空格隔开
	$+	     与 $^ 类似，但含重复项，常用于链接
	$*	     目标文件名，不含后缀

其它变体有 $(@D),$(<D),$(@F),$(<F)，等等，其中 D 代表路径，F 代表文件

## 用 VPATH 和 vpath 寻找文件

例如有以下的源代码目录树

	.
	├── include
	│   ├── counter.h
	│   └── lexer.h
	├── makefile
	└── src
	    ├── counter.c
	    ├── count_words.c
	    └── lexer.l

那么 VPATH 可以写为

	VPATH = src

以及头文件

	CPPFLAGS = -I include


如果用 path 则可以写为 vpath pattern directory-list，上例可改为更好的

	vpath %.c src
	vpath %.h include


##模式规则 Pattern Rules

许多程序的输入和输出文件遵循标准的约定。例如，C 编译器就假定 C 源代码是 .c 后缀的文件，目标文件是 .o 后缀（有些 Windows 平台编译器为 .obj 后缀）。于是，make 就可以通过内建规则识别文件类型来执行构建过程。makefile 就可以写的非常简洁：

	VPATH = src include
	CPPFLAGS = -I include

	count_words: counter.o lexer.o -lfl
	count_words.o: counter.h
	counter.o: counter.h lexer.h
	lexer.o: lexer.h

这省下许多构建需要执行的命令部分，只写依赖关系。上面的 makefile 可以执行，因为 make 的三个内建规则。
1.其一描述如何把 .c 文件编译为 .o 文件：

	%.o: %.c
		$(COMPILE.c) $(OUTPUT_OPTION) $<

2. 其二描述了如了从 .l 文件生成 .c 文件

	%.c: %.l
		$(RM) $@
		$(LEX.l) $< > $@

3. 其三描述了如何从 .c 文件生成无后缀的文件（通常是可执行文件）

	%: %.c
		$(LINK.c) $^ $(LOADLIBES) $(LDLIBS) -o $@

## 模式 Patterns

百分号 “%" 粗略等于 Unix shell 中的 *。

	%，v
	s%.0
	wrapper_%

以上都是合法的对%的使用

## 静态模式规则 Static Pattern Rules
例如：

	$(OBJECTS): %.o: %c
		    $(CC) -c $(CFLAGS) $< -o $@

这与一般模式规则不同在初始 $(OBJECTS): 规则。它限定了变量 $(OBJECTS) 中的文件,只匹配 %.o 文件

## 后缀规则 Suffix Rules

后缀规则属于 make 的原内定（将被废弃的）隐式规则。其他版本的 make 可能不支持 GNU make 的 模式规则，所以有时候还需要能能够认识它们。

	.c.o:
		$(COMPILE.c) $(OUTPUT_OPTION） $<

把 .c 文件编译为 .o 目标文件，等价于

      	%.o: %.c
		$(COMPILE.c) $(OUTPUT_OPTION) $<

已知后缀列表存放在 .SUFFIXES 中，默认 .SUFFIXES 的第一部分有：

	.SUFFIXES: .out .a .ln .o .c .cc .C .cpp .p .f .F .r .y .l

你也可以自己加入后缀到你的 makefile:

	.SUFFIXES: .pdf .fo .html .xml

如果想清空所有后缀

	.SUFFIXES:

## 隐式规则数据库

GNU make 3.8 有90+的各种隐式规则，或是模式规则，或是后缀规则。如果没有你需要的语言的规则，你需要自己简单添加一些规则。检测规则是否内建可以用 --print-data-base 或者 -p 选项查看。

实际 GNU make 的规则格式为：

     	 %: %.C
	 # (内建的）
		$(LINK.C) $^ $(LOADLIBES) $(LDLIBS) -o $@

makefile 中自定义的规则则为：

	 %.html %.xml
	 # 写在 `Makefile' 中的
	   	$(XMLTO) $(XMLTO_FLAGS) html-nochunks $<

## 使用隐式规则
很简单，不许要指定 makefile 中目标需要的命令，make 会自动查找内建数据库中的目标适应的规则。通常情况下，这样做没什么问题。

一些特殊情况下，依靠隐式规则会产生麻烦。比如一个源代码混合了 C 和 Lisp 代码。如果 editor.l 和 editor.c 在同一个路径下，一个文件依赖另一个。make 会把 .l 后缀的 lisp 文件错认为是 flex 文件，C 源代码是 flex 的输出文件。如果 editor.o 是目标文件并且 editor.l 比 editor.c 新，那么 make 将把 .l 文件通过 flex 输出 C 文件来更新 C 文件。Oh！C文件被覆盖掉了！

你可以直接通过不写命令脚本部分直接删除内建的 flex 相关规则：

	%.o: %.l
	%.c: %.l


## 规则的构造 Rule Structure

C 文件规则示例：

	%.o: %.c
		$(COMPILE.c) $(OUTPUT_OPTION) $<

此规则包含了三个变量。我们只看到两个，但 COMPILE.c 定义还包含其它变量。

	COMPILE.c = $(CC) $(CFLAGS) $(CPPFLAGS) $(TAGET_ARCH) -c
	CC = gcc
	OUTPUT_OPTION = -o $@

C 编译器由 CC 定义，CFLAGS 定义编译选项，CPPFLAGS 定义预编译选项，TAGET_ARCH 定义平台架构设定选项。

设置这些变量需要很细致。假如在意个 makefile 中

	CPPFLAGS = -I project/include

那么当用户要在 make 时自定义为

	$ make CPPFLAGS = -DDEBUG

则原来的 include 头文件就没有了，无法 make。

把变量定义得详细些就不会出现这个问题

	COMPILE.c = $(CC) $(CFLAGS) $(INCLUDES) $(CPPFLAGS) $(TAGET_ARCH) -c
	INCLUDES = -I project/include

# 自动依赖生成

由于头文件还可能依赖其它的头文件因此手工管理依赖是非常头痛的。GNU make使用这样的方式处理：生成每个源文件的依赖，放入相应 .d 文件中，然后把 .d 文件作为目标写对应的依赖规则，当源文件修改时，依赖会自动在 make 时更新。

	%.d: %.c
		$(CC) -M $(CPPFLAGS) $< > $@.$$$$;			\
		sed 's,\($*\).o[ :]*,\1.o $@ : ,g' < $@.$$$$ > $@; 	\
		rm -f $@.$$$$

这个脚本有些令人印象深刻，需要一些解释。

首先编译器 -M 选项把源代码的依赖写入目标文件 $@.$$$$，文件名为 $@ 但是又追加 $$$$ 唯一数字。在 shell 中 $$ 返回 shell 的进程号。由于进程号是唯一的，产生的文件名也是唯一的。注意，在 makefile 中使用 shell 的变量，需要以 $$ 开头，如 $$(SHELL_VAR),故此处共有4个$.

sed 把 .d 文件加为某规则的目标。sed 表达式含逗号隔开的一个搜索部分 \($*\)\.o[ :]* 一个替代部分 \1.o $@ : 。搜索表达式由 \(..\)  保留而不替换的正则表达式，目标 $* 开始，跟着后缀 \.o ，然后是0个或者多个空格或者冒号 [ :]* 。替换表达式通过引用正则表达式组并追加标签1（即 's,\($*\)）后缀为 \1.o 来恢复原目标，然后新增依赖文件目标 $@ 。

一个较完整的 makefile 的例子：

	VPATH = src include
	CPPFLAGS = -I include

	SOURCES = count_words.c	\
		  lexer.c	\
		  counter.c

	count_words: counter.o lexer.o -lfl
	count_words.o: counter.h
	counter.o: counter.h lexer.h
	lexer.o: lexer.h

	include $(subst .c, .d $(SOURCES))

	%.d: %.c
		$(CC) -M $(CPPFLAGS) $< > $@.$$$$;			\
		sed 's,\($*\).o[ :]*,\1.o $@ : ,g' < $@.$$$$ > $@; 	\
		rm -f $@.$$$$
	
include 中的 subst 把 $(SOURCES) 中的文件名字列中的 .c 替换为 .d

加 --just-print 选项运行一下 make 得到：

   	$ make --just-print
	Makefile:13: count_words.d: No such file or directory
	Makefile:13: lexer.d: No such file or directory
	Makefile:13: counter.d: No such file or directory
	gcc -M -I include src/counter.c > counter.d.$$;
	\
	sed 's,\(counter\)\.o[ :]*,\1.o counter.d : ,g' < counter.d.$$ >
	counter.d; \
	rm -f counter.d.$$
	flex -t src/lexer.l > lexer.c
	gcc -M -I include lexer.c > lexer.d.$$;
	\
	sed 's,\(lexer\)\.o[ :]*,\1.o lexer.d : ,g' < lexer.d.$$ > lexer.d;
	\
	rm -f lexer.d.$$
	gcc -M -I include src/count_words.c > count_words.d.$$;


# 管理库

创建库示例

	$ ar rv libcounter.a counter.o lexer.o
	a - counter.o
	a - lexer.o

ar 的 r 选项表示替代 (replace) .a 库中的各个 .o 项， v 表示显示详细信息 (verbose)

也可以增量式地创建库文件

	$ ar rv libcounter.a counter.o
	r - counter.o
	$ ar rv libcounter.a lexer.o

使用库文件时可以直接列出库文件

	cc count_words.o libcounter.a /lib/libfl.a -o count_words

安装到系统的标准库文件的使用，可以用编译器的 -l 选项，它会优先寻找动态库文件。

	cc count_words.o -lcounter -lfl -o count_words

上个例子实际上会出错，因为 counter 不是系统中的库文件，编译器会找不到它，我们需要用 -L 选项指定库文件的位置

	cc count_words.o -L. -lcounter -lfl -o count_words

##创建和更新库文件

简单的示例 makefile:

	libcounter.a: counter.o lexer.o
		$(AR) $(ARFLAGS) $@ $^

如果这样构建的 libcounter.a 是 count_words 的一个依赖那么在链接出可执行文件前 make 总是会更新库文件。这里需要小小的做个更改：

	libcounter.a: counter.o lexer.o
		$(AR) $(ARFLAGS） $@ $?

这样只有 .o 目标文件新于目标，make才把它更新到函数库。

还能更好些吗？make 设计初衷之一是提高构建的效率，只更新旧的文件。不幸的是，这种方式调用 ar 每次尝试更新一个目标文件会浪费很多 fork/exec 调用的开销，另外许多系统上的 ar 在用 r 参数时效率不高。在目标文件非常多的时候，还不如用 $? 一次把所有目标文件一起从头构建函数库文件省事些。当然对于小的函数库和处理器很快的情况就不许要考虑那么多了。

	libgraphics.a(bitblt.o): bitblt.o
		$(AR) $(ARFLAGS) $@ $<

这里函数库名为 libgraphics.a 成员名为 bitblt.o。语法 libgraphics.a(bitblt.o) 表示库中 的模块。$^ 表示第一个依赖。注意，此处 $@ 会解释为函数库名 libgraphics.a。

有的老版 ar 没有直接管理函数库索引添加模块的功能，需要使用 ranlib。

	libcounter.a: libcounter.a(lexer.o) libcounter.a(counter.o)
	 	$(RANLIB) $@

或者对于大的函数库，使用

	libcounter.a: counter.o lexer.o
		$(RM) $@
		$(AR) $(ARFLAGS) $@ $^
		$(RANLIB) $@


## 使用函数库作为依赖

直接使用文件名作为依赖：

	xpong: $(OBJECTS) /lib/X11/libX11.a /lib/X11/libXaw.a
	       $(LINK) $^ -o $@

或者使用 -l 选项使用函数库

	xpong: $(OBJECTS) -lX11 -lXaw
	       $(LINK) $^ -o $@

使用 -l make 将搜寻系统中的函数库（动态库优先），替代到 $^ 和 $? 中。这样做的好处在于即使链接器不能去搜寻动态库，make 也可以去找，另外，make 既可以指定找自己应用程序的函数库，也可以找系统的库。注意，向下面这样直接用 -l 选项使用函数库中目标文件的名字而不是函数库名的话会出错，make 是无法把其扩展成函数库名：
     	
	count_words: count_words.o -lcounter -lfl
		     $(CC) $^ -o $@

对于链接复杂的程序而想不出错会有些像是魔法艺术。链接器以命令行中的顺序去搜索函数库。那么如果函数库 A 中有一个未定义的符号但定义在函数库 B 中，链接就命令必须把 A 列在 B 前边（即，A 依赖 B），否则在链接器找不到 A 库中的未定义符号时，它是无法再倒回去找函数库 B 了。假设现在函数库 B 做了修改，它也引用了函数库 A 中的符号，那了两个函数库 A 和 B 互相依赖，麻烦来了。这时候需要写成这样: -lA -lB -lA。在大型复杂程序中函数库常常会这样重复，有时候不止两次。

											xpong: xpong.o libui.a libdynamics.a libui.a -lX11
		$(CC) $^ -o $@

但是这回出现问题，$^ 会把依赖中去重，命令会处理成这样：

	gcc xpong.o libui.a libdynamics.a /usr/lib/X11R6/libX11.a -o xpong

Oh，链接器会在 libX11.a 引用 libui.a 时 找不到它。这时需要用 $+ 不是 $^

	xpong: xpong.o libui.a libdynamics.a libui.a -lX11
		$(CC) $+ -o $@

这样依赖将会处理为：
	gcc xpong.o libui.a libdynamics.a libui.a /usr/lib/X11R6/libX11.a -o xpong

## 双冒号规则
双冒号规则有些隐晦，它允许有多个不同的执行命令分支共用一个目标，而运行哪个分支取决于哪个分支的依赖比目标新。对于给定目标，要么都是双冒号规则，要么就是单冒号规则。

实际上，这个特性的比较有用的例子会非常难得（所以隐晦），不过这有个编的例子

	file-list:: generate-list-script
		    chmod +x $<
		    generate-list-script $(files) > file-list

	file-list:: $(files)
		    generate-list-script $(files) > file-list

当 generate-list-script 脚本被修改了，增加执行权限然后运行它生成 file-list；如果$(files) 更改了，运行 generate-list-script 生成 file-list。


#后记
makefile 的相关内容了解这么多，对于小项目手写 makefile 来管理应该足够了。如果需要了解更多，可以参考 [GNU make](http://www.gnu.org/software/make/manual/make.html) 文档和 [免费 make 书](http://oreilly.com/catalog/make3/book/index.csp)，或者考虑其它工具如 auto-tools、cmake、SCons等等。