# 作用及说明

编写`Makefile`就是编写一系列规则，用于告诉`make`如何执行这些规则，最终生成期望的文件。在编译大型程序时，全量编译非常耗时，在只改动几个文件时就需要用`Makefile`实现增量编译

`Makefile`的优先级比`makefile`低。

# 规则

## 基本规则

```makefile
目标文件: 依赖文件1 依赖文件2
	命令（必须tab缩进）
```

make 在执行这条规则前，会比较目标文件与所有依赖文件的最后修改时间。如果任何一个依赖文件比目标文件新，make就会执行下面的命令。

make默认执行第一条规则，因此把最终要的可执行文件放第一个。遇到没有的文件会自己去执行后面的规则。

## 模式规则

`%` 最常用于定义**模式规则**，一种可以批量生成同类文件的通用规则。例如：

```makefile
%.o :%.c
	命令
```

使用`%.o :%.c`后，第一条规则发现找不到所需的.o文件时，会找到这条语句并自动生成规则`xy.o: xy.c`，前面后名字相同。

# 变量

## 自动变量

`$^`：代表当前规则所有依赖文件的列表，并自动移除重复项。

`$@`：代表当前规则的目标文件。

`$<`：在编译单个源文件时，它准确指向了唯一的`.c`文件（依赖）。

## 预定义变量

定义用`变量名 = 值`，通常变量名全大写。引用变量用`$(变量名)`，使用时相当于直接替换对应的值。

```makefile
TARGET = world			#目标
OBJS = main.o hello.o	#对象
CC = gcc				#编译器
CFLAGS = -c -Wall -g	#编译器选项
LDLIBS = -lm			#链接库
```

# 编译C程序

## 需求

假设有一个项目，包含`hello.c`、`hello.h`和`main.c`。且`main.c`引用了头文件`hello.h`。逻辑如下：

先用`hello.c`编译出`hello.o`，用`main.c`和`hello.h`编译出`main.o`，然后再链接成`world.out`。

## 编写Makefile

```makefile
# 生成可执行文件:
world: hello.o main.o
	gcc hello.o main.o -o world

# 编译 hello.c:
hello.o: hello.c
	gcc hello.c -c -Wall -g -o hello.o

# 编译 main.c:
main.o: main.c hello.h
	gcc main.c -c -Wall -g -o main.o

#删除所有.o和可执行文件
clean:
	rm -f *.o world
```

使用变量、通配符后：

```makefile
TARGET = world
OBJS = main.o hello.o	
CC = gcc				
CFLAGS = -c -Wall -g	

$(TARGET): $(OBJS)
	$(CC) $^ -o $@

%.o: %.c
	$(CC) $^ $(CFLAGS) -o $@

clean:
	rm -f *.o $(TARGET)
```

但这种写法在.h文件被改时由于没有被加入到依赖，所以make不会发现。
