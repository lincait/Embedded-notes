# 2026.05.25

## 1

报错： `warning: format ‘%f’ expects argument of type ‘float *’, but argument 2 has type ‘double’`

原因：使用`scanf`时格式类型和传入参数的类型冲突

解决：加入取地址符号

## 2

报错： `undefined reference to sqrt`，引用了头文件`math.h`仍然报错

原因：C语言的编译和链接过程是分离的。`math.h`头文件仅提供了函数声明，但`sqrt`函数的实际实现位于独立的数学库文件中，在Linux/Unix系统中，这个库通常命名为`libm.so`（共享库）或`libm.a`（静态库）。但GCC编译器默认不链接数学库。

解决：需要在编译命令中显式指定对应库，即添加`-lm`选项。`-l`选项用于指定链接的库，`m`表示数学库