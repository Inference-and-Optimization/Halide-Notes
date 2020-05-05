# lesson 10: AOT compilation 
本课分为两个文件。
- 第一个（这一个），构建一个Halide管道，并将其编译成一个**静态库和头文件**。
- 第二个（lesson_10_aot_compilation_run.cpp）使用该静态库实际运行管道。

这意味着编译此代码是一个多步骤的过程。

# part 1
## Code
```c
#include "Halide.h"
#include <stdio.h>
using namespace Halide;

int main(int argc, char **argv) {
    Func brighter;
    Var x, y;

    Param<uint8_t> offset;

    ImageParam input(type_of<uint8_t>(), 2);

    brighter(x, y) = input(x, y) + offset;

    brighter.vectorize(x, 16).parallel(y);

    brighter.compile_to_static_library("lesson_10_halide", {input, offset}, "brighter");

    printf("Halide pipeline compiled, but not yet run.\n");

    return 0;
}
```
## Build & Run
```bash
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ g++ lesson_10*generate.cpp -g -std=c++11 -I ../include -L ../bin -lHalide -lpthread -ldl -o lesson_10_generate
# 生成 
# -rwxrwxr-x 1 dongkesi dongkesi  775904 Apr  3 14:43 lesson_10_generate
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ LD_LIBRARY_PATH=../bin ./lesson_10_generate
Halide pipeline compiled, but not yet run.
# 生成 
# -rw-rw-r-- 1 dongkesi dongkesi  166094 Apr  3 14:44 lesson_10_halide.a
# -rw-rw-r-- 1 dongkesi dongkesi   91624 Apr  3 14:44 lesson_10_halide.h
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ g++ lesson_10*run.cpp lesson_10_halide.a -std=c++11 -I ../include -lpthread -ldl -o lesson_10_run
# 生成 
# -rwxrwxr-x 1 dongkesi dongkesi  100352 Apr  3 14:45 lesson_10_run
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ ./lesson_10_run
```
生成了4个文件
```bash
-rwxrwxr-x 1 dongkesi dongkesi  775904 Apr  3 14:32 lesson_10_generate
-rw-rw-r-- 1 dongkesi dongkesi  166094 Apr  3 14:32 lesson_10_halide.a
-rw-rw-r-- 1 dongkesi dongkesi   91624 Apr  3 14:32 lesson_10_halide.h
-rwxrwxr-x 1 dongkesi dongkesi  100352 Apr  3 14:33 lesson_10_run
```

## 代码分析

```c
#include "Halide.h"
#include <stdio.h>
using namespace Halide;

int main(int argc, char **argv) {
```
我们将定义一个简单的`one-stage`管道：
```c
    // We'll define a simple one-stage pipeline:
    Func brighter;
    Var x, y;
```
管道将依赖于一个标量参数。
```c
    // The pipeline will depend on one scalar parameter.
    Param<uint8_t> offset;
```
取一个灰度**8位输入缓冲区**。
- 第一个构造函数参数给出像素的类型，
- 第二个参数指定维度的数量（而不是通道的数量！）。
  - 对于灰度图像，这是2；
  - 对于彩色图像，这是3。
  - 目前，四个维度是输入和输出的最大值。
```c
    // And take one grayscale 8-bit input buffer. The first
    // constructor argument gives the type of a pixel, and the second
    // specifies the number of dimensions (not the number of
    // channels!). For a grayscale image this is two; for a color
    // image it's three. Currently, four dimensions is the maximum for
    // inputs and outputs.
    ImageParam input(type_of<uint8_t>(), 2);
```
如果我们是`jit`编译，它们只是一个`int`和一个`Buffer`，但是因为我们**只想编译一次管道并让它对参数的任何值起作用**，我们需要创建一个`Param`对象（可以像`Expr`一样使用）和一个`ImageParam`对象（可以像`Buffer`一样使用）。
```c
    // If we were jit-compiling, these would just be an int and a
    // Buffer, but because we want to compile the pipeline once and
    // have it work for any value of the parameter, we need to make a
    // Param object, which can be used like an Expr, and an ImageParam
    // object, which can be used like a Buffer.

    // Define the Func.
    brighter(x, y) = input(x, y) + offset;

    // Schedule it.
    brighter.vectorize(x, 16).parallel(y);
```
这次，我们将调用一个方法，将管道**编译为静态库和头**，而不是调用`brighter.realize(…)`，**后者将立即编译并运行管道**。

对于`AOT`编译的代码，我们**需要显式地向例程声明参数**。这个程序需要两个参数，通常是`Params`或`ImageParams`。
```c
    // This time, instead of calling brighter.realize(...), which
    // would compile and run the pipeline immediately, we'll call a
    // method that compiles the pipeline to a static library and header.
    //
    // For AOT-compiled code, we need to explicitly declare the
    // arguments to the routine. This routine takes two. Arguments are
    // usually Params or ImageParams.
    brighter.compile_to_static_library("lesson_10_halide", {input, offset}, "brighter");

    printf("Halide pipeline compiled, but not yet run.\n");

    // To continue this lesson, look in the file lesson_10_aot_compilation_run.cpp

    return 0;
}
```
- Output
> 让我们看一下`lesson_10_halide.h`中有关`brighter`的代码
```bash
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ ls -lh lesson_10_halide.*
-rw-rw-r-- 1 dongkesi dongkesi 163K Apr  3 14:44 lesson_10_halide.a
-rw-rw-r-- 1 dongkesi dongkesi  90K Apr  3 14:44 lesson_10_halide.h
```
```c
#ifdef __cplusplus
extern "C" {
#endif

HALIDE_FUNCTION_ATTRS
int brighter(struct halide_buffer_t *_p1_buffer, uint8_t _p0, struct halide_buffer_t *_f0_buffer);

HALIDE_FUNCTION_ATTRS
int brighter_argv(void **args);

HALIDE_FUNCTION_ATTRS
const struct halide_filter_metadata_t *brighter_metadata();

#ifdef __cplusplus
}  // extern "C"
#endif
```
# part 2

## Code
```c
#include "lesson_10_halide.h"

#include "HalideBuffer.h"

#include <stdio.h>

int main(int argc, char **argv) {
    Halide::Runtime::Buffer<uint8_t> input(640, 480), output(640, 480);

    int offset = 5;
    int error = brighter(input, offset, output);

    if (error) {
        printf("Halide returned an error: %d\n", error);
        return -1;
    }

    for (int y = 0; y < 480; y++) {
        for (int x = 0; x < 640; x++) {
            uint8_t input_val = input(x, y);
            uint8_t output_val = output(x, y);
            uint8_t correct_val = input_val + offset;
            if (output_val != correct_val) {
                printf("output(%d, %d) was %d instead of %d\n",
                       x, y, output_val, correct_val);
                return -1;
            }
        }
    }

    printf("Success!\n");
    return 0;
}
```
## Build & Run

在Part1已经编译了

## 代码分析
这是实际使用我们编译的Halide管道的代码。它不依赖于Halide，所以我们不包括Halide。
相反，它取决于运行`lesson_10_generate`生成的头文件：
```c
// This is the code that actually uses the Halide pipeline we've
// compiled. It does not depend on libHalide, so we won't be including
// Halide.h.
//
// Instead, it depends on the header file that lesson_10_generate
// produced when we ran it:
#include "lesson_10_halide.h"
```
我们希望继续使用`Halide::Buffer`和`AOT`编译的代码，所以我们显式地包含它。这是一个只有header类，不需要`libHalide`。
```c
// We want to continue to use our Halide::Buffer with AOT-compiled
// code, so we explicitly include it. It's a header-only class, and
// doesn't require libHalide.
#include "HalideBuffer.h"
```
查看上面的头文件（在运行`lesson_10_generate`之前它不会存在）。底部是我们生成的函数的签名：
```c
int brighter(halide_buffer_t *_input_buffer, uint8_t _offset, halide_buffer_t *_brighter_buffer);
```
- `ImageParam`输入已经变成了指向`halide_buffer_t`结构体的指针。这是Halide用来表示数据数组的结构体。除非你是从纯C代码调用Halide管道，否则你不想直接使用它。
- `Halide::Runtime::Buffer`是一个简单的`halide_buffer_t`包装器，它将隐式转换为`halide_buffer_t *`。我们将在这些插槽中传递`Halide::Runtime::Buffer`对象。

我们在JIT代码中使用的`Halide::Buffer`类实际上只是一个指向更简单的`Halide::Runtime::Buffer`类的共享指针。它们共享相同的API。
 
最后，`brighter`的返回值是一个错误代码。成功是零。
```c
#include <stdio.h>

int main(int argc, char **argv) {
    // Have a look in the header file above (it won't exist until you've run
    // lesson_10_generate). At the bottom is the signature of the function we generated:

    // int brighter(halide_buffer_t *_input_buffer, uint8_t _offset, halide_buffer_t *_brighter_buffer);

    // The ImageParam inputs have become pointers to "halide_buffer_t"
    // structs. This is struct that Halide uses to represent arrays of
    // data.  Unless you're calling the Halide pipeline from pure C
    // code, you don't want to use it
    // directly. Halide::Runtime::Buffer is a simple wrapper around
    // halide_buffer_t that will implicitly convert to a
    // halide_buffer_t *. We will pass Halide::Runtime::Buffer objects
    // in those slots.

    // The Halide::Buffer class we have been using in JIT code is in
    // fact just a shared pointer to the simpler
    // Halide::Runtime::Buffer class. They share the same API.

    // Finally, the return value of "brighter" is an error code. It's
    // zero on success.
```
让我们为输入和输出创建一个缓冲区。
```c
    // Let's make a buffer for our input and output.
    Halide::Runtime::Buffer<uint8_t> input(640, 480), output(640, 480);
```
`Halide::Runtime::Buffer`也有**包装现有数据而不是分配新内存的构造函数**。如果要使用自己的图像类型，请使用这些。
```c
    // Halide::Runtime::Buffer also has constructors that wrap
    // existing data instead of allocating new memory. Use these if
    // you have your own Image type that you want to use.

    int offset = 5;
    int error = brighter(input, offset, output);

    if (error) {
        printf("Halide returned an error: %d\n", error);
        return -1;
    }
```
检测是否匹配
```c
    // Now let's check the filter performed as advertised. It was
    // supposed to add the offset to every input pixel.
    for (int y = 0; y < 480; y++) {
        for (int x = 0; x < 640; x++) {
            uint8_t input_val = input(x, y);
            uint8_t output_val = output(x, y);
            uint8_t correct_val = input_val + offset;
            if (output_val != correct_val) {
                printf("output(%d, %d) was %d instead of %d\n",
                       x, y, output_val, correct_val);
                return -1;
            }
        }
    }

    // Everything worked!
    printf("Success!\n");
    return 0;
}
```