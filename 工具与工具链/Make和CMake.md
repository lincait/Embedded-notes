# 作用及说明

`Make`和`CMake`都用于管理项目构建，可以配合使用。**CMake负责生成构建规则，Make负责执行规则，GCC负责实际编译和链接。**

| 工具 | 读取的文件 | 作用 |
| --- | --- | --- |
| `make` | `Makefile` | 根据依赖关系执行命令，完成增量构建 |
| `cmake` | `CMakeLists.txt` | 根据项目描述生成Makefile等构建文件 |
| `gcc / g++` | C / C++源文件等 | 编译源文件、链接目标文件和库 |

使用CMake时，通常只维护`CMakeLists.txt`，不用手写或修改它生成的Makefile。CMake也能生成Ninja等其他工具使用的构建文件，因此**使用CMake不一定使用Make**。

# Make基本使用

在Makefile所在目录执行：

```bash
make                 # 构建默认目标，通常是Makefile中的第一个普通目标
make world           # 构建指定目标world
make -j4             # 最多同时执行4个构建任务
make clean           # 执行Makefile中定义的clean规则
make -f demo.mk      # 指定读取的Makefile
```

`clean`不是make内置的清理命令，需要自己编写规则，并用`.PHONY: clean`声明伪目标。

# CMake基本使用

## 环境

Ubuntu安装所需工具：`sudo apt install build-essential cmake`。用`cmake --version`查看版本。

## 编写CMakeLists.txt

沿用`main.c`调用`hello.c`中函数的例子，将文件按下表放置；两个源文件都通过`#include "hello.h"`引用头文件。

| 相对于项目根目录的路径 | 内容 |
| --- | --- |
| `src/main.c` | main函数，调用hello函数 |
| `src/hello.c` | hello函数的实现 |
| `include/hello.h` | hello函数的声明 |
| `CMakeLists.txt` | 项目的构建配置 |

根目录下的`CMakeLists.txt`：

```cmake
cmake_minimum_required(VERSION 3.16)      # 要求CMake版本不低于3.16
project(HelloProject LANGUAGES C)        # 项目名，启用C语言

add_executable(world src/main.c src/hello.c)  # 定义可执行目标world
target_include_directories(world PRIVATE include)  # 头文件搜索目录
target_compile_features(world PRIVATE c_std_11)    # 至少支持C11
target_compile_options(world PRIVATE -Wall -Wextra) # GCC/Clang警告选项
```

**目标名`world`不必与项目名`HelloProject`相同。** 目标可以是可执行文件，也可以是库；后面的`target_...`命令为已定义的目标设置属性。

`include`在这里相对于当前`CMakeLists.txt`所在目录，相当于告诉编译器去哪里找头文件。头文件不必写入`add_executable`，CMake会安排构建工具跟踪编译器发现的头文件依赖。

## 配置、编译和运行

在项目根目录执行：

```bash
cmake -S . -B build -G "Unix Makefiles"  # 配置项目并生成Makefile
cmake --build build -j4                 # 调用make完成编译、链接
./build/world                          # 运行程序
```

- `-S .`：源码目录为当前目录，即根`CMakeLists.txt`所在位置。
- `-B build`：构建目录为`build`，不存在时会自动创建；中间文件、缓存和本例的可执行文件放在这里。
- `-G "Unix Makefiles"`：指定生成Makefile；省略时使用环境的默认生成器。

`cmake --build build`会调用对应的底层构建工具，不需要自己判断应该执行make还是ninja。上面明确选择Make后，也可以用`make -C build -j4`完成构建。

**修改已有`.c`或`.h`后，再执行构建命令即可增量编译。** 新增源文件时，要将它加入`add_executable`，再构建；CMake通常会自动重新配置。

## 常用构建命令

以下命令都在项目根目录执行：

```bash
cmake --build build --target world     # 只构建world及其依赖
cmake --build build --target clean     # 清理构建产物，保留构建配置
cmake --build build --verbose          # 查看实际执行的编译、链接命令
```

需要完全重新配置时，删除本例的`build`目录，再执行配置和构建命令即可。**不要手动修改build里的Makefile，重新生成时会被覆盖。**

## Debug与Release

对于上面的Makefile生成器，在配置时用`-D变量名=值`设置选项：

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build
```

`Debug`用于调试，通常带调试信息；`Release`用于发布，通常开启优化。两种配置可以分别使用`build-debug`和`build-release`目录，避免混用构建结果。

配置选项会保存在构建目录的`CMakeCache.txt`中，后续普通构建不用重复传入。

# 常用写法

## 变量与源文件

`set(变量名 值...)`定义变量，`${变量名}`引用变量。前面的`add_executable`也可以改写为：

```cmake
set(SRCS src/main.c src/hello.c)
add_executable(world ${SRCS})
```

源文件少时直接列出即可；新增`.c`文件时同步更新列表。CMake的命令不需要像Makefile的命令行一样使用Tab缩进。

## 生成和链接库

将`hello.c`先编译成静态库，再让`world`链接它。将根`CMakeLists.txt`整体改为：

```cmake
cmake_minimum_required(VERSION 3.16)
project(HelloProject LANGUAGES C)

add_library(hello STATIC src/hello.c)
target_include_directories(hello PUBLIC include)

add_executable(world src/main.c)
target_link_libraries(world PRIVATE hello)
target_compile_features(hello PRIVATE c_std_11)
target_compile_features(world PRIVATE c_std_11)
```

- `add_library(hello STATIC ...)`：在Linux下生成`libhello.a`；改成`SHARED`则生成动态库`libhello.so`。
- `target_link_libraries(world PRIVATE hello)`：让world链接hello，CMake会安排先构建库，不需要手写库文件路径或`-lhello`。
- `PRIVATE`：设置只供当前目标使用。
- `PUBLIC`：设置既供当前目标使用，也传给链接它的其他目标。因此world链接hello后，也能使用其`include`头文件目录。

这里`src/hello.c`只放在库的源文件列表中，world的源文件列表只需要`src/main.c`。构建和运行命令与前面相同。

## 链接线程库

Linux程序使用pthread时，在定义world后添加：

```cmake
set(THREADS_PREFER_PTHREAD_FLAG TRUE)            # 优先使用-pthread选项
find_package(Threads REQUIRED)                  # 查找线程支持
target_link_libraries(world PRIVATE Threads::Threads)
```

CMake会提供当前平台需要的线程编译、链接设置，比直接写死`-lpthread`更方便。

## 用于C++项目

将`project`改成`project(HelloProject LANGUAGES CXX)`，源文件列表换成实际的`.cpp`文件，标准设置改为`target_compile_features(world PRIVATE cxx_std_17)`即可要求至少支持C++17。若项目同时包含C和C++，使用`LANGUAGES C CXX`。
