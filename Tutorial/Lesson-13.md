# lesson 13: Tuples
本课介绍如何编写**评估多个值**的函数。
## Code

```c
#include "Halide.h"
#include <stdio.h>
#include <algorithm>
using namespace Halide;

int main(int argc, char **argv) {
    Func single_valued;
    Var x, y;
    single_valued(x, y) = x + y;

    Func color_image;
    Var c;
    color_image(x, y, c) = select(c == 0, 245, // Red value
                                  c == 1, 42,  // Green value
                                  132);        // Blue value

    Func brighter;
    brighter(x, y, c) = color_image(x, y, c) + 10;

    Func func_array[3];
    func_array[0](x, y) = x + y;
    func_array[1](x, y) = sin(x);
    func_array[2](x, y) = cos(y);

    Func multi_valued;
    multi_valued(x, y) = Tuple(x + y, sin(x * y));

    {
        Realization r = multi_valued.realize(80, 60);
        assert(r.size() == 2);
        Buffer<int> im0 = r[0];
        Buffer<float> im1 = r[1];
        assert(im0(30, 40) == 30 + 40);
        assert(im1(30, 40) == sinf(30 * 40));
    }

    {
        int multi_valued_0[80*60];
        float multi_valued_1[80*60];
        for (int y = 0; y < 80; y++) {
            for (int x = 0; x < 60; x++) {
                multi_valued_0[x + 60*y] = x + y;
                multi_valued_1[x + 60*y] = sinf(x*y);
            }
        }
    }

    Func multi_valued_2;
    multi_valued_2(x, y) = {x + y, sin(x*y)};

    Expr integer_part = multi_valued_2(x, y)[0];
    Expr floating_part = multi_valued_2(x, y)[1];
    Func consumer;
    consumer(x, y) = {integer_part + 10, floating_part + 10.0f};

    {
        Func input_func;
        input_func(x) = sin(x);
        Buffer<float> input = input_func.realize(100);

        Func arg_max;

        arg_max() = {0, input(0)};

        RDom r(1, 99);
        Expr old_index = arg_max()[0];
        Expr old_max   = arg_max()[1];
        Expr new_index = select(old_max < input(r), r, old_index);
        Expr new_max   = max(input(r), old_max);
        arg_max() = {new_index, new_max};

        int arg_max_0 = 0;
        float arg_max_1 = input(0);
        for (int r = 1; r < 100; r++) {
            int old_index = arg_max_0;
            float old_max = arg_max_1;
            int new_index = old_max < input(r) ? r : old_index;
            float new_max = std::max(input(r), old_max);
            arg_max_0 = new_index;
            arg_max_1 = new_max;
        }

        {
            Realization r = arg_max.realize();
            Buffer<int> r0 = r[0];
            Buffer<float> r1 = r[1];
            assert(arg_max_0 == r0(0));
            assert(arg_max_1 == r1(0));
        }
    }

    {
        struct Complex {
            Expr real, imag;

            Complex(Tuple t) : real(t[0]), imag(t[1]) {}

            Complex(Expr r, Expr i) : real(r), imag(i) {}

            Complex(FuncRef t) : Complex(Tuple(t)) {}

            operator Tuple() const {
                return {real, imag};
            }

            Complex operator+(const Complex &other) const {
                return {real + other.real, imag + other.imag};
            }

            Complex operator*(const Complex &other) const {
                return {real * other.real - imag * other.imag,
                        real * other.imag + imag * other.real};
            }

            Expr magnitude_squared() const {
                return real * real + imag * imag;
            }

        };

        Func mandelbrot;

        Complex initial(x/15.0f - 2.5f, y/6.0f - 2.0f);

        Var t;
        mandelbrot(x, y, t) = Complex(0.0f, 0.0f);

        RDom r(1, 12);
        Complex current = mandelbrot(x, y, r-1);

        mandelbrot(x, y, r) = current*current + initial;

        Expr escape_condition = Complex(mandelbrot(x, y, r)).magnitude_squared() < 16.0f;
        Tuple first_escape = argmin(escape_condition);

        Func escape;
        escape(x, y) = first_escape[0];

        Buffer<int> result = escape.realize(61, 25);
        const char *code = " .:-~*={}&%#@";
        for (int y = 0; y < result.height(); y++) {
            for (int x = 0; x < result.width(); x++) {
                printf("%c", code[result(x, y)]);
            }
            printf("\n");
        }
    }


    printf("Success!\n");

    return 0;
}
```
## Build & Run

```bash
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ g++ lesson_13*.cpp -g -I ../include -L ../bin -lHalide -lpthread -ldl -o lesson_13 -std=c++11
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ LD_LIBRARY_PATH=../bin ./lesson_13
```

## 代码分析

```c
#include "Halide.h"
#include <stdio.h>
#include <algorithm>
using namespace Halide;

int main(int argc, char **argv) {
```
到目前为止，Funcs（如下面的函数）对于其域中的每个点都计算为单个标量值。
```c
    // So far Funcs (such as the one below) have evaluated to a single
    // scalar value for each point in their domain.
    Func single_valued;
    Var x, y;
    single_valued(x, y) = x + y;
```
编写返回集合值的`Func`的一种方法是**添加一个额外的维度来索引该集合**。这就是我们通常处理颜色的方式。例如，下面的`Func`表示由`c`索引的每个x，y坐标的三个值的集合。
```c
    // One way to write a Func that returns a collection of values is
    // to add an additional dimension that indexes that
    // collection. This is how we typically deal with color. For
    // example, the Func below represents a collection of three values
    // for every x, y coordinate indexed by c.
    Func color_image;
    Var c;
    // 通过select选择三个通道中C值，来返回一个这些值约束的一个集合
    color_image(x, y, c) = select(c == 0, 245, // Red value
                                  c == 1, 42,  // Green value
                                  132);        // Blue value
```
此方法通常很方便，因为它使对该函数的操作变得容易，从而可以平等地对待集合中的每个项：
```c
    // This method is often convenient because it makes it easy to
    // operate on this Func in a way that treats each item in the
    // collection equally:
    Func brighter;
    brighter(x, y, c) = color_image(x, y, c) + 10;
```
但是这种方法也**不方便**，原因有三。
1. Func是在一个`infinite domain`上定义的，因此该Func的用户可以访问彩色图像`(x, y, -17)`，这不是一个有意义的值，可能表示有`bug`。
2. 它需要一个`select`，如果不进行绑定和展开，则会影响性能：`brighter.bound(c, 0, 3).unroll(c);`
3. 使用此方法，集合中的所有值都**必须具有相同的类型**。虽然上述两个问题仅仅是不方便，但这一个问题是一个硬限制，使之无法用这种方式表达某些东西。

也可以将值集合表示为函数集合：
```c
    // However this method is also inconvenient for three reasons.
    //
    // 1) Funcs are defined over an infinite domain, so users of this
    // Func can for example access color_image(x, y, -17), which is
    // not a meaningful value and is probably indicative of a bug.
    //
    // 2) It requires a select, which can impact performance if not
    // bounded and unrolled:
    // brighter.bound(c, 0, 3).unroll(c);
    //
    // 3) With this method, all values in the collection must have the
    // same type. While the above two issues are merely inconvenient,
    // this one is a hard limitation that makes it impossible to
    // express certain things in this way.

    // It is also possible to represent a collection of values as a
    // collection of Funcs:
    Func func_array[3];
    func_array[0](x, y) = x + y;
    func_array[1](x, y) = sin(x);
    func_array[2](x, y) = cos(y);
```
这种方法避免了上述三个问题，但引入了一个新的烦恼。因为**这些函数是独立的**，所以很难对它们进行调度，以至于它们都在`x`，`y`上的单个循环中一起计算。

**第三种方法**是定义一个`Func`作为对元组的求值而不是`Expr`求值。元组是表达式的**固定大小集合**。元组中的每个`Expr`可能有不同的类型。下面的函数计算整数值`(x+y)`和浮点值`(sin(x*y))`。
```c
    // This method avoids the three problems above, but introduces a
    // new annoyance. Because these are separate Funcs, it is
    // difficult to schedule them so that they are all computed
    // together inside a single loop over x, y.

    // A third alternative is to define a Func as evaluating to a
    // Tuple instead of an Expr. A Tuple is a fixed-size collection of
    // Exprs. Each Expr in a Tuple may have a different type. The
    // following function evaluates to an integer value (x+y), and a
    // floating point value (sin(x*y)).
    Func multi_valued;
    multi_valued(x, y) = Tuple(x + y, sin(x * y));
```
实现元组值的`Func`返回缓冲区的集合。我们称其为`Realization`。它等效于Buffer对象的`std::vector`：
```c
    // Realizing a tuple-valued Func returns a collection of
    // Buffers. We call this a Realization. It's equivalent to a
    // std::vector of Buffer objects:
    {
        Realization r = multi_valued.realize(80, 60);
        assert(r.size() == 2);
        Buffer<int> im0 = r[0];
        Buffer<float> im1 = r[1];
        assert(im0(30, 40) == 30 + 40);
        assert(im1(30, 40) == sinf(30 * 40));
    }
```
所有Tuple元素**在同一循环嵌套中的同一域上一起求值，但存储在不同的内存分配中**。上面的等效C++代码是：
```c
    // All Tuple elements are evaluated together over the same domain
    // in the same loop nest, but stored in distinct allocations. The
    // equivalent C++ code to the above is:
    {
        int multi_valued_0[80*60];
        float multi_valued_1[80*60];
        for (int y = 0; y < 80; y++) {
            for (int x = 0; x < 60; x++) {
                multi_valued_0[x + 60*y] = x + y;
                multi_valued_1[x + 60*y] = sinf(x*y);
            }
        }
    }
```
提前编译时，元组值的`Func`会评估为多个不同的输出`buffer_t`结构体。它们按顺序出现在**函数签名的末尾**：
```c
    // int multi_valued(...input buffers and params...,
    //                  buffer_t *output_1, buffer_t *output_2);
```
您可以像上面一样通过将多个`Exprs`传递给Tuple构造函数来构造一个Tuple。也许更优雅些，您还可以利用C++ 11初始化程序列表，并将`Exprs`括在花括号中：
```c
    // When compiling ahead-of-time, a Tuple-valued Func evaluates
    // into multiple distinct output buffer_t structs. These appear in
    // order at the end of the function signature:
    // int multi_valued(...input buffers and params...,
    //                  buffer_t *output_1, buffer_t *output_2);

    // You can construct a Tuple by passing multiple Exprs to the
    // Tuple constructor as we did above. Perhaps more elegantly, you
    // can also take advantage of C++11 initializer lists and just
    // enclose your Exprs in braces:
    Func multi_valued_2;
    multi_valued_2(x, y) = {x + y, sin(x*y)};
```
调用多值`Func`不能视为`Exprs`。以下是语法错误：
```c
    // Func consumer;
    // consumer(x, y) = multi_valued_2(x, y) + 10;
```
 取而代之的是，您必须用方括号索引一个元组以检索各个`Exprs`：
```c
    // Calls to a multi-valued Func cannot be treated as Exprs. The
    // following is a syntax error:
    // Func consumer;
    // consumer(x, y) = multi_valued_2(x, y) + 10;

    // Instead you must index a Tuple with square brackets to retrieve
    // the individual Exprs:
    Expr integer_part = multi_valued_2(x, y)[0];
    Expr floating_part = multi_valued_2(x, y)[1];
    Func consumer;
    consumer(x, y) = {integer_part + 10, floating_part + 10.0f};
```

元组在归约中特别有用，因为它们允许归约沿着其域行走时保持复杂状态。最简单的示例是`argmax`。

首先，我们创建一个缓冲区以接收`argmax`。

```c
    // Tuple reductions.
    {
        // Tuples are particularly useful in reductions, as they allow
        // the reduction to maintain complex state as it walks along
        // its domain. The simplest example is an argmax.

        // First we create a Buffer to take the argmax over.
        Func input_func;
        input_func(x) = sin(x);
        Buffer<float> input = input_func.realize(100);
```
然后，我们定义一个2值元组，该元组跟踪最大值的索引和值本身。
```c
        // Then we define a 2-valued Tuple which tracks the index of
        // the maximum value and the value itself.
        Func arg_max;

        // Pure definition.
        arg_max() = {0, input(0)};

        // Update definition.
        RDom r(1, 99);
        Expr old_index = arg_max()[0];
        Expr old_max   = arg_max()[1];
        Expr new_index = select(old_max < input(r), r, old_index);
        Expr new_max   = max(input(r), old_max);
        arg_max() = {new_index, new_max};
```
等价C++
```c
        // The equivalent C++ is:
        int arg_max_0 = 0;
        float arg_max_1 = input(0);
        for (int r = 1; r < 100; r++) {
            int old_index = arg_max_0;
            float old_max = arg_max_1;
            int new_index = old_max < input(r) ? r : old_index;
            float new_max = std::max(input(r), old_max);
            // In a tuple update definition, all loads and computation
            // are done before any stores, so that all Tuple elements
            // are updated atomically with respect to recursive calls
            // to the same Func.
            arg_max_0 = new_index;
            arg_max_1 = new_max;
        }
```
让我们验证Halide和C++是否找到相同的最大值和索引。
```c
        // Let's verify that the Halide and C++ found the same maximum
        // value and index.
        {
            Realization r = arg_max.realize();
            Buffer<int> r0 = r[0];
            Buffer<float> r1 = r[1];
            assert(arg_max_0 == r0(0));
            assert(arg_max_1 == r1(0));
        }
```
Halide提供`argmax`和`argmin`作为内置的归约，类似于求和，乘积，最大值和最小值。它们返回一个元组，该元组由归约域中与该值对应的点以及值本身组成。如果是ties，则它们返回找到的第一个值。 在下一节中，我们将使用其中之一。
```c
        // Halide provides argmax and argmin as built-in reductions
        // similar to sum, product, maximum, and minimum. They return
        // a Tuple consisting of the point in the reduction domain
        // corresponding to that value, and the value itself. In the
        // case of ties they return the first value found. We'll use
        // one of these in the following section.
    }
```
元组也可以是表示复合对象（如复数）的便捷方法。定义可以在元组之间进行转换的对象是用用户定义的类型扩展Halide的类型系统的一种方法。
```c
    // Tuples for user-defined types.
    {
        // Tuples can also be a convenient way to represent compound
        // objects such as complex numbers. Defining an object that
        // can be converted to and from a Tuple is one way to extend
        // Halide's type system with user-defined types.
        struct Complex {
            Expr real, imag;

            // Construct from a Tuple
            Complex(Tuple t) : real(t[0]), imag(t[1]) {}

            // Construct from a pair of Exprs
            Complex(Expr r, Expr i) : real(r), imag(i) {}

            // Construct from a call to a Func by treating it as a Tuple
            Complex(FuncRef t) : Complex(Tuple(t)) {}

            // Convert to a Tuple
            operator Tuple() const {
                return {real, imag};
            }

            // Complex addition
            Complex operator+(const Complex &other) const {
                return {real + other.real, imag + other.imag};
            }

            // Complex multiplication
            Complex operator*(const Complex &other) const {
                return {real * other.real - imag * other.imag,
                        real * other.imag + imag * other.real};
            }

            // Complex magnitude, squared for efficiency
            Expr magnitude_squared() const {
                return real * real + imag * imag;
            }

            // Other complex operators would go here. The above are
            // sufficient for this example.
        };
```
TODO: 这例子还没看懂

我们将使用另一个元组归约来计算迭代次数，其中值首先逃逸半径为4的圆。这可以表示为布尔值的argmin-我们希望给定布尔表达式第一次为false时的索引（我们认为false小于true）。argmax将在表达式首次为真时返回索引。

```c
        // Let's use the Complex struct to compute a Mandelbrot set.
        Func mandelbrot;

        // The initial complex value corresponding to an x, y coordinate
        // in our Func.
        Complex initial(x/15.0f - 2.5f, y/6.0f - 2.0f);

        // Pure definition.
        Var t;
        mandelbrot(x, y, t) = Complex(0.0f, 0.0f);

        // We'll use an update definition to take 12 steps.
        RDom r(1, 12);
        Complex current = mandelbrot(x, y, r-1);

        // The following line uses the complex multiplication and
        // addition we defined above.
        mandelbrot(x, y, r) = current*current + initial;

        // We'll use another tuple reduction to compute the iteration
        // number where the value first escapes a circle of radius 4.
        // This can be expressed as an argmin of a boolean - we want
        // the index of the first time the given boolean expression is
        // false (we consider false to be less than true).  The argmax
        // would return the index of the first time the expression is
        // true.

        Expr escape_condition = Complex(mandelbrot(x, y, r)).magnitude_squared() < 16.0f;
        Tuple first_escape = argmin(escape_condition);

        // We only want the index, not the value, but argmin returns
        // both, so we'll index the argmin Tuple expression using
        // square brackets to get the Expr representing the index.
        Func escape;
        escape(x, y) = first_escape[0];

        // Realize the pipeline and print the result as ascii art.
        Buffer<int> result = escape.realize(61, 25);
        const char *code = " .:-~*={}&%#@";
        for (int y = 0; y < result.height(); y++) {
            for (int x = 0; x < result.width(); x++) {
                printf("%c", code[result(x, y)]);
            }
            printf("\n");
        }
    }


    printf("Success!\n");

    return 0;
}
```
- Output: row 25; column 61

```
:::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::
:::::::::::::::::::::-------------------:::::::::::::::::::::
:::::::::::::::-------------------------------:::::::::::::::
:::::::::::---------------------------------------:::::::::::
::::::::----------------------~~~~~~~~~--------------::::::::
::::::------------------~~~~~~~~~*****~~~~~~-----------::::::
::::----------------~~~~~~~~~****={#%@=**~~~~~-----------::::
:::-------------~~~~~~~~~****==={%    %{=***~~~~----------:::
:------------~~~~~~~******==& @%       @#}} =*~~~~----------:
:--------~~~~~~~***======{{&               #{=*~~~~---------:
-------~~~~~~***==} @% %%&%                  =**~~~----------
------~~~***==={}&#                         }=**~~~~---------
------~*                                   &{=**~~~~---------
------~~~***==={}&#                         }=**~~~~---------
-------~~~~~~***==} @% %%&%                  =**~~~----------
:--------~~~~~~~***======{{&               #{=*~~~~---------:
:------------~~~~~~~******==& @%       @#}} =*~~~~----------:
:::-------------~~~~~~~~~****==={%    %{=***~~~~----------:::
::::----------------~~~~~~~~~****={#%@=**~~~~~-----------::::
::::::------------------~~~~~~~~~*****~~~~~~-----------::::::
::::::::----------------------~~~~~~~~~--------------::::::::
:::::::::::---------------------------------------:::::::::::
:::::::::::::::-------------------------------:::::::::::::::
:::::::::::::::::::::-------------------:::::::::::::::::::::
:::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::
```