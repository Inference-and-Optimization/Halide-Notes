# lesson 6: Realizing Funcs over arbitrary domains

## Code
```c
#include "Halide.h"
#include <stdio.h>

using namespace Halide;

int main(int argc, char **argv) {

    // The last lesson was quite involved, and scheduling complex
    // multi-stage pipelines is ahead of us. As an interlude, let's
    // consider something easy: evaluating funcs over rectangular
    // domains that do not start at the origin.

    // We define our familiar gradient function.
    Func gradient("gradient");
    Var x("x"), y("y");
    gradient(x, y) = x + y;

    // And turn on tracing so we can see how it is being evaluated.
    gradient.trace_stores();

    // Previously we've realized gradient like so:
    //
    // gradient.realize(8, 8);
    //
    // This does three things internally:
    // 1) Generates code than can evaluate gradient over an arbitrary
    // rectangle.
    // 2) Allocates a new 8 x 8 image.
    // 3) Runs the generated code to evaluate gradient for all x, y
    // from (0, 0) to (7, 7) and puts the result into the image.
    // 4) Returns the new image as the result of the realize call.

    // What if we're managing memory carefully and don't want Halide
    // to allocate a new image for us? We can call realize another
    // way. We can pass it an image we would like it to fill in. The
    // following evaluates our Func into an existing image:
    printf("Evaluating gradient from (0, 0) to (7, 7)\n");
    Buffer<int> result(8, 8);
    gradient.realize(result);
    // Click to show output ...

    // Let's check it did what we expect:
    for (int y = 0; y < 8; y++) {
        for (int x = 0; x < 8; x++) {
            if (result(x, y) != x + y) {
                printf("Something went wrong!\n");
                return -1;
            }
        }
    }

    // Now let's evaluate gradient over a 5 x 7 rectangle that starts
    // somewhere else -- at position (100, 50). So x and y will run
    // from (100, 50) to (104, 56) inclusive.

    // We start by creating an image that represents that rectangle:
    Buffer<int> shifted(5, 7); // In the constructor we tell it the size.
    shifted.set_min(100, 50); // Then we tell it the top-left corner.

    printf("Evaluating gradient from (100, 50) to (104, 56)\n");

    // Note that this won't need to compile any new code, because when
    // we realized it the first time, we generated code capable of
    // evaluating gradient over an arbitrary rectangle.
    gradient.realize(shifted);
    // Click to show output ...

    // From C++, we also access the image object using coordinates
    // that start at (100, 50).
    for (int y = 50; y < 57; y++) {
        for (int x = 100; x < 105; x++) {
            if (shifted(x, y) != x + y) {
                printf("Something went wrong!\n");
                return -1;
            }
        }
    }
    // The image 'shifted' stores the value of our Func over a domain
    // that starts at (100, 50), so asking for shifted(0, 0) would in
    // fact read out-of-bounds and probably crash.

    // What if we want to evaluate our Func over some region that
    // isn't rectangular? Too bad. Halide only does rectangles :)

    printf("Success!\n");
    return 0;
}
```
## Build & Run
```bash
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ g++ lesson_06*.cpp -g -I ../include -L ../bin -lHalide -lpthread -ldl -o lesson_06 -std=c++11
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ LD_LIBRARY_PATH=../bin ./lesson_06
```
## 代码分析

上一课涉及的内容很多，后面还有复杂的多级管道调度。作为一个插曲，让我们考虑一些简单的事情：在**不从原点开始的矩形域上对函数求值**。我们定义了我们熟悉的梯度函数。
```c
    Func gradient("gradient");
    Var x("x"), y("y");
    gradient(x, y) = x + y;
```
打开跟踪，这样我们就可以看到它是如何被求值的。
```c
gradient.trace_stores();
```

之前我们已经实现了这样的gradient：
```c
gradient.realize(8, 8);
```
这在内部做了三件事：
1) 生成可以计算任意矩形上的gradient。
2) 分配新的 8x8 image。
3) 运行生成的代码来计算从(0, 0)到(7, 7)的所有x，y的梯度，并将结果放入image中。
4) 作为realize调用的结果返回新image。

如果我们正在小心地管理内存，不想让Halide为我们分配一个新的image呢？我们可以用另一种方式来实现。我们可以给它一个我们想要它来填充的image。下面将在现有的image内求值Func：
```c
printf("Evaluating gradient from (0, 0) to (7, 7)\n");
Buffer<int> result(8, 8);
gradient.realize(result);
```
- Output
```c
Evaluating gradient from (0, 0) to (7, 7)
Begin pipeline gradient.0()
Tag gradient.0() tag = "func_type_and_dim: 1 0 32 1 2 0 8 0 8"
Store gradient.0(0, 0) = 0
Store gradient.0(1, 0) = 1
Store gradient.0(2, 0) = 2
Store gradient.0(3, 0) = 3
Store gradient.0(4, 0) = 4
Store gradient.0(5, 0) = 5
Store gradient.0(6, 0) = 6
Store gradient.0(7, 0) = 7
Store gradient.0(0, 1) = 1
Store gradient.0(1, 1) = 2
Store gradient.0(2, 1) = 3
Store gradient.0(3, 1) = 4
Store gradient.0(4, 1) = 5
Store gradient.0(5, 1) = 6
Store gradient.0(6, 1) = 7
Store gradient.0(7, 1) = 8
Store gradient.0(0, 2) = 2
Store gradient.0(1, 2) = 3
Store gradient.0(2, 2) = 4
Store gradient.0(3, 2) = 5
Store gradient.0(4, 2) = 6
Store gradient.0(5, 2) = 7
Store gradient.0(6, 2) = 8
Store gradient.0(7, 2) = 9
Store gradient.0(0, 3) = 3
Store gradient.0(1, 3) = 4
Store gradient.0(2, 3) = 5
Store gradient.0(3, 3) = 6
Store gradient.0(4, 3) = 7
Store gradient.0(5, 3) = 8
Store gradient.0(6, 3) = 9
Store gradient.0(7, 3) = 10
Store gradient.0(0, 4) = 4
Store gradient.0(1, 4) = 5
Store gradient.0(2, 4) = 6
Store gradient.0(3, 4) = 7
Store gradient.0(4, 4) = 8
Store gradient.0(5, 4) = 9
Store gradient.0(6, 4) = 10
Store gradient.0(7, 4) = 11
Store gradient.0(0, 5) = 5
Store gradient.0(1, 5) = 6
Store gradient.0(2, 5) = 7
Store gradient.0(3, 5) = 8
Store gradient.0(4, 5) = 9
Store gradient.0(5, 5) = 10
Store gradient.0(6, 5) = 11
Store gradient.0(7, 5) = 12
Store gradient.0(0, 6) = 6
Store gradient.0(1, 6) = 7
Store gradient.0(2, 6) = 8
Store gradient.0(3, 6) = 9
Store gradient.0(4, 6) = 10
Store gradient.0(5, 6) = 11
Store gradient.0(6, 6) = 12
Store gradient.0(7, 6) = 13
Store gradient.0(0, 7) = 7
Store gradient.0(1, 7) = 8
Store gradient.0(2, 7) = 9
Store gradient.0(3, 7) = 10
Store gradient.0(4, 7) = 11
Store gradient.0(5, 7) = 12
Store gradient.0(6, 7) = 13
Store gradient.0(7, 7) = 14
End pipeline gradient.0()
```

让我们检查一下它是否达到了我们的预期:

```c
for (int y = 0; y < 8; y++) {
    for (int x = 0; x < 8; x++) {
        if (result(x, y) != x + y) {
            printf("Something went wrong!\n");
            return -1;
        }
    }
}
```

现在让我们计算一个从其他地方开始的`5x7`矩形上的梯度——位置`(100, 50)`。所以`x`和`y`从`(100, 50)`到`(104, 56)`。

我们首先创建一个表示该矩形的image：

```c
Buffer<int> shifted(5, 7); // In the constructor we tell it the size.
shifted.set_min(100, 50); // Then we tell it the top-left corner.

printf("Evaluating gradient from (100, 50) to (104, 56)\n");
```
注意，这不需要编译任何新代码，因为当我们第一次实现它时，我们生成了能够计算任意矩形上的梯度的代码。

```c
gradient.realize(shifted);
```
- Output
```c
Evaluating gradient from (100, 50) to (104, 56)
Begin pipeline gradient.0()
Tag gradient.0() tag = "func_type_and_dim: 1 0 32 1 2 100 5 50 7"
Store gradient.0(100, 50) = 150
Store gradient.0(101, 50) = 151
Store gradient.0(102, 50) = 152
Store gradient.0(103, 50) = 153
Store gradient.0(104, 50) = 154
Store gradient.0(100, 51) = 151
Store gradient.0(101, 51) = 152
Store gradient.0(102, 51) = 153
Store gradient.0(103, 51) = 154
Store gradient.0(104, 51) = 155
Store gradient.0(100, 52) = 152
Store gradient.0(101, 52) = 153
Store gradient.0(102, 52) = 154
Store gradient.0(103, 52) = 155
Store gradient.0(104, 52) = 156
Store gradient.0(100, 53) = 153
Store gradient.0(101, 53) = 154
Store gradient.0(102, 53) = 155
Store gradient.0(103, 53) = 156
Store gradient.0(104, 53) = 157
Store gradient.0(100, 54) = 154
Store gradient.0(101, 54) = 155
Store gradient.0(102, 54) = 156
Store gradient.0(103, 54) = 157
Store gradient.0(104, 54) = 158
Store gradient.0(100, 55) = 155
Store gradient.0(101, 55) = 156
Store gradient.0(102, 55) = 157
Store gradient.0(103, 55) = 158
Store gradient.0(104, 55) = 159
Store gradient.0(100, 56) = 156
Store gradient.0(101, 56) = 157
Store gradient.0(102, 56) = 158
Store gradient.0(103, 56) = 159
Store gradient.0(104, 56) = 160
End pipeline gradient.0()
```
从C++，我们也使用从`(100, 50)`开始的坐标访问image对象。
```c
for (int y = 50; y < 57; y++) {
    for (int x = 100; x < 105; x++) {
        if (shifted(x, y) != x + y) {
            printf("Something went wrong!\n");
            return -1;
        }
    }
}
```
image “shifted”将Func的值存储在一个从`(100, 50)`开始的域上，因此请求`shifted(0, 0)`实际上会读取越界，并可能崩溃。

如果我们想计算某个非矩形区域上的Func怎么办？太糟糕了。Halide只做矩形。
