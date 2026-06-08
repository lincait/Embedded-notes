# 2026.05.25

## 1

报错： `warning: format ‘%f’ expects argument of type ‘float *’, but argument 2 has type ‘double’`

原因：使用`scanf`时格式类型和传入参数的类型冲突

解决：加入取地址符号

## 2

报错： `undefined reference to sqrt`，引用了头文件`math.h`仍然报错

原因：C语言的编译和链接过程是分离的。`math.h`头文件仅提供了函数声明，但`sqrt`函数的实际实现位于独立的数学库文件中，在Linux/Unix系统中，这个库通常命名为`libm.so`（共享库）或`libm.a`（静态库）。但GCC编译器默认不链接数学库。

解决：需要在编译命令中显式指定对应库，即添加`-lm`选项。`-l`选项用于指定链接的库，`m`表示数学库



# 2026.05.26

报错：`Segmentation fault (core dumped)`，段错误

原因：内存访问越界。由于使用错误的下标，导致数组访问越界

解决：在循环中由于`i`和`j`混用漏改了，检查并改回去

# 2026.05.29

警告：`assignment discards ‘const’ qualifier from pointer target type`

原因：把一个指向只读数据的指针`const char *`，赋给一个可以修改数据的指针`char *`，这可能导致后续通过这个可修改的指针误修改只读数据。

解决：把可以修改数据的指针也用`const`修饰。

# 2026.06.02

报错：stack smashing detected: terminated  Aborted (core dumped)

原因：栈溢出触发栈保护，程序发生了**栈缓冲区溢出**，导致栈上的关键数据被非法改写。

解决：定义数组时，没限定长度却赋值了，`char ch_num2[] = {0};`，导致数组默认只有一位，后续操作直接溢出。

# 2026.06.08

报错：xxxxxx’declared inside parameter list will not be visible of this definition or declare

原因：某个函数声明时，需要的参数类型没有被外部可见、没有定义或没有声明

解决：在定义结构体时，先写结构体的描述，再声明函数，注意先后顺序。

