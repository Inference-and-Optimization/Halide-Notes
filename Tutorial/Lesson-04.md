# lesson 4: Debugging with tracing, print, and print_when

## Code
```c
#include "Halide.h"
#include <stdio.h>
using namespace Halide;

int main(int argc, char **argv) {

    Var x("x"), y("y");

    // Printing out the value of Funcs as they are computed.
    {
        // We'll define our gradient function as before.
        Func gradient("gradient");
        gradient(x, y) = x + y;

        // And tell Halide that we'd like to be notified of all
        // evaluations.
        gradient.trace_stores();

        // Realize the function over an 8x8 region.
        printf("Evaluating gradient\n");
        Buffer<int> output = gradient.realize(8, 8);
        // Click to show output ...

        // This will print out all the times gradient(x, y) gets
        // evaluated.

        // Now that we can snoop on what Halide is doing, let's try our
        // first scheduling primitive. We'll make a new version of
        // gradient that processes each scanline in parallel.
        Func parallel_gradient("parallel_gradient");
        parallel_gradient(x, y) = x + y;

        // We'll also trace this function.
        parallel_gradient.trace_stores();

        // Things are the same so far. We've defined the algorithm, but
        // haven't said anything about how to schedule it. In general,
        // exploring different scheduling decisions doesn't change the code
        // that describes the algorithm.

        // Now we tell Halide to use a parallel for loop over the y
        // coordinate. On Linux we run this using a thread pool and a task
        // queue. On OS X we call into grand central dispatch, which does
        // the same thing for us.
        parallel_gradient.parallel(y);

        // This time the printfs should come out of order, because each
        // scanline is potentially being processed in a different
        // thread. The number of threads should adapt to your system, but
        // on linux you can control it manually using the environment
        // variable HL_NUM_THREADS.
        printf("\nEvaluating parallel_gradient\n");
        parallel_gradient.realize(8, 8);
        // Click to show output ...
    }

    // Printing individual Exprs.
    {
        // trace_stores() can only print the value of a
        // Func. Sometimes you want to inspect the value of
        // sub-expressions rather than the entire Func. The built-in
        // function 'print' can be wrapped around any Expr to print
        // the value of that Expr every time it is evaluated.

        // For example, say we have some Func that is the sum of two terms:
        Func f;
        f(x, y) = sin(x) + cos(y);

        // If we want to inspect just one of the terms, we can wrap
        // 'print' around it like so:
        Func g;
        g(x, y) = sin(x) + print(cos(y));

        printf("\nEvaluating sin(x) + cos(y), and just printing cos(y)\n");
        g.realize(4, 4);
        // Click to show output ...
    }

    // Printing additional context.
    {
        // print can take multiple arguments. It prints all of them
        // and evaluates to the first one. The arguments can be Exprs
        // or constant strings. This can be used to print additional
        // context alongside the value:
        Func f;
        f(x, y) = sin(x) + print(cos(y), "<- this is cos(", y, ") when x =", x);

        printf("\nEvaluating sin(x) + cos(y), and printing cos(y) with more context\n");
        f.realize(4, 4);
        // Click to show output ...

        // It can be useful to split expressions like the one above
        // across multiple lines to make it easier to turn on and off
        // printing certain values while debugging.
        Expr e = cos(y);
        // Uncomment the following line to print the value of cos(y)
        // e = print(e, "<- this is cos(", y, ") when x =", x);
        Func g;
        g(x, y) = sin(x) + e;
        g.realize(4, 4);
    }

    // Conditional printing
    {
        // Both print and trace_stores can produce a lot of output. If
        // you're looking for a rare event, or just want to see what
        // happens at a single pixel, this amount of output can be
        // difficult to dig through. Instead, the function print_when
        // can be used to conditionally print an Expr. The first
        // argument to print_when is a boolean Expr. If the Expr
        // evaluates to true, it returns the second argument and
        // prints all of the arguments. If the Expr evaluates to false
        // it just returns the second argument and does not print.

        Func f;
        Expr e = cos(y);
        e = print_when(x == 37 && y == 42, e, "<- this is cos(y) at x, y == (37, 42)");
        f(x, y) = sin(x) + e;
        printf("\nEvaluating sin(x) + cos(y), and printing cos(y) at a single pixel\n");
        f.realize(640, 480);
        // Click to show output ...

        // print_when can also be used to check for values you're not expecting:
        Func g;
        e = cos(y);
        e = print_when(e < 0, e, "cos(y) < 0 at y ==", y);
        g(x, y) = sin(x) + e;
        printf("\nEvaluating sin(x) + cos(y), and printing whenever cos(y) < 0\n");
        g.realize(4, 4);
        // Click to show output ...
    }

    // Printing expressions at compile-time.
    {
        // The code above builds up a Halide Expr across several lines
        // of code. If you're programmatically constructing a complex
        // expression, and you want to check the Expr you've created
        // is what you think it is, you can also print out the
        // expression itself using C++ streams:
        Var fizz("fizz"), buzz("buzz");
        Expr e = 1;
        for (int i = 2; i < 100; i++) {
            if (i % 3 == 0 && i % 5 == 0) e += fizz*buzz;
            else if (i % 3 == 0) e += fizz;
            else if (i % 5 == 0) e += buzz;
            else e += i;
        }
        std::cout << "Printing a complex Expr: " << e << "\n";
        // Click to show output ...
    }

    printf("Success!\n");
    return 0;
}
```
## Build & Run

```bash
g++ lesson_04*.cpp -g -I ../include -L ../bin -lHalide -lpthread -ldl -o lesson_04 -std=c++11
LD_LIBRARY_PATH=../bin ./lesson_04
# 限制为一个线程执行parallel_gradient
$ HL_NUM_THREADS=1 LD_LIBRARY_PATH=../bin ./lesson_04
```
## Code 分析
### 原理

```bash
# 会打印所有求值表达式，格式是"Store <Func Name>.xx(x, y) = value"
gradient.trace_stores();
# 会打印指定的部分
print()
# 条件打印
print_when()
```

### Printing out the value of Funcs as they are computed.
#### gradient
```c
    // Printing out the value of Funcs as they are computed.
    {
        // 串行代码
        Func gradient("gradient");
        gradient(x, y) = x + y;
        gradient.trace_stores();
        printf("Evaluating gradient\n");
        Buffer<int> output = gradient.realize(8, 8);
```
- output
```bash
# 可以看到按行输出(x, y)，每一行计算完，再计算下一行
Evaluating gradient
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
#### parallel_gradient
```c
        // 多线程并行代码
        Func parallel_gradient("parallel_gradient");
        parallel_gradient(x, y) = x + y;
        parallel_gradient.trace_stores();
        parallel_gradient.parallel(y);
        printf("\nEvaluating parallel_gradient\n");
        parallel_gradient.realize(8, 8);
        // Click to show output ...
    }
```
- output: **HL_NUM_THREADS=1** LD_LIBRARY_PATH=../bin ./lesson_04
  > 单线程时表现与串行的一致
```
Evaluating parallel_gradient
Begin pipeline parallel_gradient.0()
Tag parallel_gradient.0() tag = "func_type_and_dim: 1 0 32 1 2 0 8 0 8"
Store parallel_gradient.0(0, 0) = 0
Store parallel_gradient.0(1, 0) = 1
Store parallel_gradient.0(2, 0) = 2
Store parallel_gradient.0(3, 0) = 3
Store parallel_gradient.0(4, 0) = 4
Store parallel_gradient.0(5, 0) = 5
Store parallel_gradient.0(6, 0) = 6
Store parallel_gradient.0(7, 0) = 7
Store parallel_gradient.0(0, 1) = 1
Store parallel_gradient.0(1, 1) = 2
Store parallel_gradient.0(2, 1) = 3
Store parallel_gradient.0(3, 1) = 4
Store parallel_gradient.0(4, 1) = 5
Store parallel_gradient.0(5, 1) = 6
Store parallel_gradient.0(6, 1) = 7
Store parallel_gradient.0(7, 1) = 8
Store parallel_gradient.0(0, 2) = 2
Store parallel_gradient.0(1, 2) = 3
Store parallel_gradient.0(2, 2) = 4
Store parallel_gradient.0(3, 2) = 5
Store parallel_gradient.0(4, 2) = 6
Store parallel_gradient.0(5, 2) = 7
Store parallel_gradient.0(6, 2) = 8
Store parallel_gradient.0(7, 2) = 9
Store parallel_gradient.0(0, 3) = 3
Store parallel_gradient.0(1, 3) = 4
Store parallel_gradient.0(2, 3) = 5
Store parallel_gradient.0(3, 3) = 6
Store parallel_gradient.0(4, 3) = 7
Store parallel_gradient.0(5, 3) = 8
Store parallel_gradient.0(6, 3) = 9
Store parallel_gradient.0(7, 3) = 10
Store parallel_gradient.0(0, 4) = 4
Store parallel_gradient.0(1, 4) = 5
Store parallel_gradient.0(2, 4) = 6
Store parallel_gradient.0(3, 4) = 7
Store parallel_gradient.0(4, 4) = 8
Store parallel_gradient.0(5, 4) = 9
Store parallel_gradient.0(6, 4) = 10
Store parallel_gradient.0(7, 4) = 11
Store parallel_gradient.0(0, 5) = 5
Store parallel_gradient.0(1, 5) = 6
Store parallel_gradient.0(2, 5) = 7
Store parallel_gradient.0(3, 5) = 8
Store parallel_gradient.0(4, 5) = 9
Store parallel_gradient.0(5, 5) = 10
Store parallel_gradient.0(6, 5) = 11
Store parallel_gradient.0(7, 5) = 12
Store parallel_gradient.0(0, 6) = 6
Store parallel_gradient.0(1, 6) = 7
Store parallel_gradient.0(2, 6) = 8
Store parallel_gradient.0(3, 6) = 9
Store parallel_gradient.0(4, 6) = 10
Store parallel_gradient.0(5, 6) = 11
Store parallel_gradient.0(6, 6) = 12
Store parallel_gradient.0(7, 6) = 13
Store parallel_gradient.0(0, 7) = 7
Store parallel_gradient.0(1, 7) = 8
Store parallel_gradient.0(2, 7) = 9
Store parallel_gradient.0(3, 7) = 10
Store parallel_gradient.0(4, 7) = 11
Store parallel_gradient.0(5, 7) = 12
Store parallel_gradient.0(6, 7) = 13
Store parallel_gradient.0(7, 7) = 14
End pipeline parallel_gradient.0()
```
- output: **HL_NUM_THREADS=2** LD_LIBRARY_PATH=../bin ./lesson_04
> 输出会乱序
```
Evaluating parallel_gradient
Begin pipeline parallel_gradient.0()
Tag parallel_gradient.0() tag = "func_type_and_dim: 1 0 32 1 2 0 8 0 8"
Store parallel_gradient.0(0, 0) = 0
Store parallel_gradient.0(1, 0) = 1
Store parallel_gradient.0(2, 0) = 2
Store parallel_gradient.0(3, 0) = 3
Store parallel_gradient.0(4, 0) = 4
Store parallel_gradient.0(5, 0) = 5
Store parallel_gradient.0(6, 0) = 6
Store parallel_gradient.0(7, 0) = 7
Store parallel_gradient.0(0, 1) = 1
Store parallel_gradient.0(0, 2) = 2
Store parallel_gradient.0(1, 2) = 3
Store parallel_gradient.0(1, 1) = 2
Store parallel_gradient.0(2, 2) = 4
Store parallel_gradient.0(2, 1) = 3
Store parallel_gradient.0(3, 1) = 4
Store parallel_gradient.0(3, 2) = 5
Store parallel_gradient.0(4, 1) = 5
Store parallel_gradient.0(4, 2) = 6
Store parallel_gradient.0(5, 1) = 6
Store parallel_gradient.0(5, 2) = 7
Store parallel_gradient.0(6, 1) = 7
Store parallel_gradient.0(6, 2) = 8
Store parallel_gradient.0(7, 1) = 8
Store parallel_gradient.0(7, 2) = 9
Store parallel_gradient.0(0, 3) = 3
Store parallel_gradient.0(0, 4) = 4
Store parallel_gradient.0(1, 3) = 4
Store parallel_gradient.0(1, 4) = 5
Store parallel_gradient.0(2, 3) = 5
Store parallel_gradient.0(3, 3) = 6
Store parallel_gradient.0(2, 4) = 6
Store parallel_gradient.0(4, 3) = 7
Store parallel_gradient.0(3, 4) = 7
Store parallel_gradient.0(4, 4) = 8
Store parallel_gradient.0(5, 3) = 8
Store parallel_gradient.0(5, 4) = 9
Store parallel_gradient.0(6, 3) = 9
Store parallel_gradient.0(7, 3) = 10
Store parallel_gradient.0(6, 4) = 10
Store parallel_gradient.0(0, 5) = 5
Store parallel_gradient.0(7, 4) = 11
Store parallel_gradient.0(1, 5) = 6
Store parallel_gradient.0(0, 6) = 6
Store parallel_gradient.0(2, 5) = 7
Store parallel_gradient.0(1, 6) = 7
Store parallel_gradient.0(3, 5) = 8
Store parallel_gradient.0(2, 6) = 8
Store parallel_gradient.0(4, 5) = 9
Store parallel_gradient.0(3, 6) = 9
Store parallel_gradient.0(5, 5) = 10
Store parallel_gradient.0(4, 6) = 10
Store parallel_gradient.0(6, 5) = 11
Store parallel_gradient.0(5, 6) = 11
Store parallel_gradient.0(7, 5) = 12
Store parallel_gradient.0(6, 6) = 12
Store parallel_gradient.0(0, 7) = 7
Store parallel_gradient.0(7, 6) = 13
Store parallel_gradient.0(1, 7) = 8
Store parallel_gradient.0(2, 7) = 9
Store parallel_gradient.0(3, 7) = 10
Store parallel_gradient.0(4, 7) = 11
Store parallel_gradient.0(5, 7) = 12
Store parallel_gradient.0(6, 7) = 13
Store parallel_gradient.0(7, 7) = 14
End pipeline parallel_gradient.0()
```
### Printing individual Exprs.
- trace_stores()只能打印Func的值。有时您想检查子表达式的值，而不是整个Func。内置函数'print'可以包装在任何Expr周围，以在每次求值时打印该Expr的值。
```c
    // Printing individual Exprs.
    {
        Func f;
        f(x, y) = sin(x) + cos(y);

        Func g;
        g(x, y) = sin(x) + print(cos(y));

        printf("\nEvaluating sin(x) + cos(y), and just printing cos(y)\n");
        g.realize(4, 4);
    }
```
- output
```
Evaluating sin(x) + cos(y), and just printing cos(y)
1.000000
1.000000
1.000000
1.000000
0.540302
0.540302
0.540302
0.540302
-0.416147
-0.416147
-0.416147
-0.416147
-0.989992
-0.989992
-0.989992
-0.98999
```
- 加一条g.trace_stores() 作为对比
```
Evaluating sin(x) + cos(y), and just printing cos(y)
Begin pipeline f5.0()
Tag f5.0() tag = "func_type_and_dim: 1 2 32 1 2 0 4 0 4"
1.000000
Store f5.0(0, 0) = 1.000000
1.000000
Store f5.0(1, 0) = 1.841471
1.000000
Store f5.0(2, 0) = 1.909297
1.000000
Store f5.0(3, 0) = 1.141120
0.540302
Store f5.0(0, 1) = 0.540302
0.540302
Store f5.0(1, 1) = 1.381773
0.540302
Store f5.0(2, 1) = 1.449600
0.540302
Store f5.0(3, 1) = 0.681422
-0.416147
Store f5.0(0, 2) = -0.416147
-0.416147
Store f5.0(1, 2) = 0.425324
-0.416147
Store f5.0(2, 2) = 0.493151
-0.416147
Store f5.0(3, 2) = -0.275027
-0.989992
Store f5.0(0, 3) = -0.989992
-0.989992
Store f5.0(1, 3) = -0.148522
-0.989992
Store f5.0(2, 3) = -0.080695
-0.989992
Store f5.0(3, 3) = -0.848872
End pipeline f5.0()
```
### Printing additional context.
- `print`可以使用多个参数。它会打印所有内容并**为第一个参数求值**。参数可以是Exprs或常量字符串。这可以用于在值旁边打印其他上下文：
```c
    {
        Func f;
        f(x, y) = sin(x) + print(cos(y), "<- this is cos(", y, ") when x =", x);

        printf("\nEvaluating sin(x) + cos(y), and printing cos(y) with more context\n");
        f.realize(4, 4);
        Expr e = cos(y);
        // e = print(e, "<- this is cos(", y, ") when x =", x);
        Func g;
        g(x, y) = sin(x) + e;
        g.realize(4, 4);
    }
```
- output
```
Evaluating sin(x) + cos(y), and printing cos(y) with more context
1.000000 <- this is cos( 0 ) when x = 0
1.000000 <- this is cos( 0 ) when x = 1
1.000000 <- this is cos( 0 ) when x = 2
1.000000 <- this is cos( 0 ) when x = 3
0.540302 <- this is cos( 1 ) when x = 0
0.540302 <- this is cos( 1 ) when x = 1
0.540302 <- this is cos( 1 ) when x = 2
0.540302 <- this is cos( 1 ) when x = 3
-0.416147 <- this is cos( 2 ) when x = 0
-0.416147 <- this is cos( 2 ) when x = 1
-0.416147 <- this is cos( 2 ) when x = 2
-0.416147 <- this is cos( 2 ) when x = 3
-0.989992 <- this is cos( 3 ) when x = 0
-0.989992 <- this is cos( 3 ) when x = 1
-0.989992 <- this is cos( 3 ) when x = 2
-0.989992 <- this is cos( 3 ) when x = 3
```
### Conditional printing
- `print`和`trace_stores`均可产生大量输出。如果您正在寻找一个罕见的事件，或者只是想看看单个像素发生了什么，那么很难挖掘出如此大量的输出。而是可以使用函数`print_when`有条件地打印Expr。`print_when`的第一个参数是`布尔Expr`。
  - 如果Expr的计算结果为true，则返回第二个参数并打印所有参数。
  - 如果Expr的计算结果为false，则仅返回第二个参数，并且不打印。
- 条件打印eg1
```c
    // Conditional printing
    {
        Func f;
        Expr e = cos(y);
        e = print_when(x == 37 && y == 42, e, "<- this is cos(y) at x, y == (37, 42)");
        f(x, y) = sin(x) + e;
        printf("\nEvaluating sin(x) + cos(y), and printing cos(y) at a single pixel\n");
        f.realize(640, 480);
```
- output
```
Evaluating sin(x) + cos(y), and printing cos(y) at a single pixel
-0.399985 <- this is cos(y) at x, y == (37, 42)
```
- 条件打印eg2
```
        Func g;
        e = cos(y);
        e = print_when(e < 0, e, "cos(y) < 0 at y ==", y);
        g(x, y) = sin(x) + e;
        printf("\nEvaluating sin(x) + cos(y), and printing whenever cos(y) < 0\n");
        g.realize(4, 4);
    }
```
- output
```
Evaluating sin(x) + cos(y), and printing whenever cos(y) < 0
-0.416147 cos(y) < 0 at y == 2
-0.416147 cos(y) < 0 at y == 2
-0.416147 cos(y) < 0 at y == 2
-0.416147 cos(y) < 0 at y == 2
-0.989992 cos(y) < 0 at y == 3
-0.989992 cos(y) < 0 at y == 3
-0.989992 cos(y) < 0 at y == 3
-0.989992 cos(y) < 0 at y == 3
```
### Printing expressions at compile-time.
- 上面的代码跨几行代码构建了Halide Expr。如果您正在以编程方式构造一个复杂的表达式，并且想要检查所创建的Expr是否符合您的想法，则还可以使用C++流打印出表达式本身：
```c
    // Printing expressions at compile-time.
    {
        Var fizz("fizz"), buzz("buzz");
        Expr e = 1;
        for (int i = 2; i < 100; i++) {
            if (i % 3 == 0 && i % 5 == 0) e += fizz*buzz;
            else if (i % 3 == 0) e += fizz;
            else if (i % 5 == 0) e += buzz;
            else e += i;
        }
        std::cout << "Printing a complex Expr: " << e << "\n";
    }
```
- output

```
Printing a complex Expr: ((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((((1 + 2) + fizz) + 4) + buzz) + fizz) + 7) + 8) + fizz) + buzz) + 11) + fizz) + 13) + 14) + (fizz*buzz)) + 16) + 17) + fizz) + 19) + buzz) + fizz) + 22) + 23) + fizz) + buzz) + 26) + fizz) + 28) + 29) + (fizz*buzz)) + 31) + 32) + fizz) + 34) + buzz) + fizz) + 37) + 38) + fizz) + buzz) + 41) + fizz) + 43) + 44) + (fizz*buzz)) + 46) + 47) + fizz) + 49) + buzz) + fizz) + 52) + 53) + fizz) + buzz) + 56) + fizz) + 58) + 59) + (fizz*buzz)) + 61) + 62) + fizz) + 64) + buzz) + fizz) + 67) + 68) + fizz) + buzz) + 71) + fizz) + 73) + 74) + (fizz*buzz)) + 76) + 77) + fizz) + 79) + buzz) + fizz) + 82) + 83) + fizz) + buzz) + 86) + fizz) + 88) + 89) + (fizz*buzz)) + 91) + 92) + fizz) + 94) + buzz) + fizz) + 97) + 98) + fizz)
```