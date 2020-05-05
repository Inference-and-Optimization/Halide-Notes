# lesson 1: Getting started with Funcs, Vars, and Exprs

## Code

```c
// Halide tutorial lesson 1: Getting started with Funcs, Vars, and Exprs

// This lesson demonstrates basic usage of Halide as a JIT compiler for imaging.

// On linux, you can compile and run it like so:
// g++ lesson_01*.cpp -g -I ../include -L ../bin -lHalide -lpthread -ldl -o lesson_01 -std=c++11
// LD_LIBRARY_PATH=../bin ./lesson_01

// On os x:
// g++ lesson_01*.cpp -g -I ../include -L ../bin -lHalide -o lesson_01 -std=c++11
// DYLD_LIBRARY_PATH=../bin ./lesson_01

// If you have the entire Halide source tree, you can also build it by
// running:
//    make tutorial_lesson_01_basics
// in a shell with the current directory at the top of the halide
// source tree.

// The only Halide header file you need is Halide.h. It includes all of Halide.
#include "Halide.h"

// We'll also include stdio for printf.
#include <stdio.h>

int main(int argc, char **argv) {

    // This program defines a single-stage imaging pipeline that
    // outputs a grayscale diagonal gradient.

    // A 'Func' object represents a pipeline stage. It's a pure
    // function that defines what value each pixel should have. You
    // can think of it as a computed image.
    Halide::Func gradient;

    // Var objects are names to use as variables in the definition of
    // a Func. They have no meaning by themselves.
    Halide::Var x, y;

    // We typically use Vars named 'x' and 'y' to correspond to the x
    // and y axes of an image, and we write them in that order. If
    // you're used to thinking of images as having rows and columns,
    // then x is the column index, and y is the row index.

    // Funcs are defined at any integer coordinate of its variables as
    // an Expr in terms of those variables and other functions.
    // Here, we'll define an Expr which has the value x + y. Vars have
    // appropriate operator overloading so that expressions like
    // 'x + y' become 'Expr' objects.
    Halide::Expr e = x + y;

    // Now we'll add a definition for the Func object. At pixel x, y,
    // the image will have the value of the Expr e. On the left hand
    // side we have the Func we're defining and some Vars. On the right
    // hand side we have some Expr object that uses those same Vars.
    gradient(x, y) = e;

    // This is the same as writing:
    //
    //   gradient(x, y) = x + y;
    //
    // which is the more common form, but we are showing the
    // intermediate Expr here for completeness.

    // That line of code defined the Func, but it didn't actually
    // compute the output image yet. At this stage it's just Funcs,
    // Exprs, and Vars in memory, representing the structure of our
    // imaging pipeline. We're meta-programming. This C++ program is
    // constructing a Halide program in memory. Actually computing
    // pixel data comes next.

    // Now we 'realize' the Func, which JIT compiles some code that
    // implements the pipeline we've defined, and then runs it.  We
    // also need to tell Halide the domain over which to evaluate the
    // Func, which determines the range of x and y above, and the
    // resolution of the output image. Halide.h also provides a basic
    // templatized image type we can use. We'll make an 800 x 600
    // image.
    Halide::Buffer<int32_t> output = gradient.realize(800, 600);

    // Halide does type inference for you. Var objects represent
    // 32-bit integers, so the Expr object 'x + y' also represents a
    // 32-bit integer, and so 'gradient' defines a 32-bit image, and
    // so we got a 32-bit signed integer image out when we call
    // 'realize'. Halide types and type-casting rules are equivalent
    // to C.

    // Let's check everything worked, and we got the output we were
    // expecting:
    for (int j = 0; j < output.height(); j++) {
        for (int i = 0; i < output.width(); i++) {
            // We can access a pixel of an Buffer object using similar
            // syntax to defining and using functions.
            if (output(i, j) != i + j) {
                printf("Something went wrong!\n"
                       "Pixel %d, %d was supposed to be %d, but instead it's %d\n",
                       i, j, i+j, output(i, j));
                return -1;
            }
        }
    }

    // Everything worked! We defined a Func, then called 'realize' on
    // it to generate and run machine code that produced an Buffer.
    printf("Success!\n");

    return 0;
}
```
## Build & Run

```bash
# 编译
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ g++ lesson_01*.cpp -g -I ../include -L ../bin -lHalide -lpthread -ldl -o lesson_01 -std=c++11
# 运行
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ LD_LIBRARY_PATH=../bin ./lesson_01
```

## 程序分析

该程序定义了一个输出灰度对角线梯度的单级成像管线。

Func对象：表示管道阶段。它是一个**纯函数**，用于定义每个像素应具有的值。您可以将其视为计算图像。
```c
Halide::Func gradient;
```
Var对象：是在`Func`的定义中用作**变量的名称**。他们自己没有意义。
```c
Halide::Var x, y;
```
通常，我们使用名为`x`和`y`的`Var`来对应图像的x和y轴，并按此顺序编写它们。如果您习惯于将图像视为具有`rows`和`columnes`，则x是`column`索引，而y是`row`索引。

就这些变量和其他函数而言，`Func`在其变量的任何整数坐标处定义为`Expr`。在这里，我们将定义一个`Expr`，其值为`x + y`。`Var`具有适当的运算符重载，因此`x + y`之类的表达式成为`Expr`对象。

```c
Halide::Expr e = x + y;
```
现在，我们将为Func对象添加一个定义。 在像素x，y处，图像将具有Expr e的值。左侧是我们要定义的Func和一些Vars。 在右侧，我们有一些使用相同Vars的Expr对象。
```c
gradient(x, y) = e;
```
这两种写法是一样的
```c
gradient(x, y) = x + y;
```
这是最常见的形式，但是为了完整起见，我们在此显示中间的Expr。

该行代码定义了`Func`，但实际上并没有计算输出图像。在此阶段，内存中只有`Funcs`，`Exprs`和`Vars`，代表了我们成像管道的结构。我们正在元编程。这个C++程序**正在内存中构造一个Halide程序**。实际上计算像素数据紧随其后。

现在我们“实现(realize)” `Func`，JIT编译一些代码以实现我们定义的管道，然后运行它。我们还需要告诉Halide对`Func`域求值，该域确定上述`x`和`y`的范围以及输出图像的分辨率。`Halide.h`还提供了我们可以使用的基本模板图像类型。我们将制作一个`800x600`的图像。

```c
Halide::Buffer<int32_t> output = gradient.realize(800, 600);
```
Halide为您做类型推断。`Var`对象代表`32`位整数，因此`Expr`对象'`x + y`'也代表32位整数，因此'`gradient`'定义了32位图像，因此当出现以下情况时，我们得到了32位有符号整数图像，我们称之为“realize”。Halide类型和类型转换规则等效于C。

让我们检查一切是否正常，并获得预期的输出：
```c
for (int j = 0; j < output.height(); j++) {
  for (int i = 0; i < output.width(); i++) {
    // We can access a pixel of an Buffer object using similar
    // syntax to defining and using functions.
    if (output(i, j) != i + j) {
      printf("Something went wrong!\n"
            "Pixel %d, %d was supposed to be %d, but instead it's %d\n",
            i, j, i+j, output(i, j));
      return -1;
    }
  }
}
```
一切正常！我们定义了一个Func，然后在其上调用“realize”以生成并运行产生Buffer的机器代码。
