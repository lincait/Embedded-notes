# C语言基础练习报错调试

## 2026.05.25

报错： `warning: format ‘%f’ expects argument of type ‘float *’, but argument 2 has type ‘double’`

原因：使用`scanf`时格式类型和传入参数的类型冲突

解决：加入取地址符号