# lesson 7: Multi-stage pipelines
## Code
```c
#include "Halide.h"
#include <stdio.h>

using namespace Halide;

// Support code for loading pngs.
#include "halide_image_io.h"
using namespace Halide::Tools;

int main(int argc, char **argv) {
    // First we'll declare some Vars to use below.
    Var x("x"), y("y"), c("c");

    // Now we'll express a multi-stage pipeline that blurs an image
    // first horizontally, and then vertically.
    {
        // Take a color 8-bit input
        Buffer<uint8_t> input = load_image("images/rgb.png");

        // Upgrade it to 16-bit, so we can do math without it overflowing.
        Func input_16("input_16");
        input_16(x, y, c) = cast<uint16_t>(input(x, y, c));

        // Blur it horizontally:
        Func blur_x("blur_x");
        blur_x(x, y, c) = (input_16(x-1, y, c) +
                           2 * input_16(x, y, c) +
                           input_16(x+1, y, c)) / 4;

        // Blur it vertically:
        Func blur_y("blur_y");
        blur_y(x, y, c) = (blur_x(x, y-1, c) +
                           2 * blur_x(x, y, c) +
                           blur_x(x, y+1, c)) / 4;

        // Convert back to 8-bit.
        Func output("output");
        output(x, y, c) = cast<uint8_t>(blur_y(x, y, c));

        // Each Func in this pipeline calls a previous one using
        // familiar function call syntax (we've overloaded operator()
        // on Func objects). A Func may call any other Func that has
        // been given a definition. This restriction prevents
        // pipelines with loops in them. Halide pipelines are always
        // feed-forward graphs of Funcs.

        // Now let's realize it...

        // Buffer<uint8_t> result = output.realize(input.width(), input.height(), 3);

        // Except that the line above is not going to work. Uncomment
        // it to see what happens.

        // Realizing this pipeline over the same domain as the input
        // image requires reading pixels out of bounds in the input,
        // because the blur_x stage reaches outwards horizontally, and
        // the blur_y stage reaches outwards vertically. Halide
        // detects this by injecting a piece of code at the top of the
        // pipeline that computes the region over which the input will
        // be read. When it starts to run the pipeline it first runs
        // this code, determines that the input will be read out of
        // bounds, and refuses to continue. No actual bounds checks
        // occur in the inner loop; that would be slow.
        //
        // So what do we do? There are a few options. If we realize
        // over a domain shifted inwards by one pixel, we won't be
        // asking the Halide routine to read out of bounds. We saw how
        // to do this in the previous lesson:
        Buffer<uint8_t> result(input.width()-2, input.height()-2, 3);
        result.set_min(1, 1);
        output.realize(result);

        // Save the result. It should look like a slightly blurry
        // parrot, and it should be two pixels narrower and two pixels
        // shorter than the input image.
        save_image(result, "blurry_parrot_1.png");

        // This is usually the fastest way to deal with boundaries:
        // don't write code that reads out of bounds :) The more
        // general solution is our next example.
    }

    // The same pipeline, with a boundary condition on the input.
    {
        // Take a color 8-bit input
        Buffer<uint8_t> input = load_image("images/rgb.png");

        // This time, we'll wrap the input in a Func that prevents
        // reading out of bounds:
        Func clamped("clamped");

        // Define an expression that clamps x to lie within the
        // range [0, input.width()-1].
        Expr clamped_x = clamp(x, 0, input.width()-1);
        // clamp(x, a, b) is equivalent to max(min(x, b), a).

        // Similarly clamp y.
        Expr clamped_y = clamp(y, 0, input.height()-1);
        // Load from input at the clamped coordinates. This means that
        // no matter how we evaluated the Func 'clamped', we'll never
        // read out of bounds on the input. This is a clamp-to-edge
        // style boundary condition, and is the simplest boundary
        // condition to express in Halide.
        clamped(x, y, c) = input(clamped_x, clamped_y, c);

        // Defining 'clamped' in that way can be done more concisely
        // using a helper function from the BoundaryConditions
        // namespace like so:
        //
        // clamped = BoundaryConditions::repeat_edge(input);
        //
        // These are important to use for other boundary conditions,
        // because they are expressed in the way that Halide can best
        // understand and optimize. When used correctly they are as
        // cheap as having no boundary condition at all.

        // Upgrade it to 16-bit, so we can do math without it
        // overflowing. This time we'll refer to our new Func
        // 'clamped', instead of referring to the input image
        // directly.
        Func input_16("input_16");
        input_16(x, y, c) = cast<uint16_t>(clamped(x, y, c));

        // The rest of the pipeline will be the same...

        // Blur it horizontally:
        Func blur_x("blur_x");
        blur_x(x, y, c) = (input_16(x-1, y, c) +
                           2 * input_16(x, y, c) +
                           input_16(x+1, y, c)) / 4;

        // Blur it vertically:
        Func blur_y("blur_y");
        blur_y(x, y, c) = (blur_x(x, y-1, c) +
                           2 * blur_x(x, y, c) +
                           blur_x(x, y+1, c)) / 4;

        // Convert back to 8-bit.
        Func output("output");
        output(x, y, c) = cast<uint8_t>(blur_y(x, y, c));

        // This time it's safe to evaluate the output over the some
        // domain as the input, because we have a boundary condition.
        Buffer<uint8_t> result = output.realize(input.width(), input.height(), 3);

        // Save the result. It should look like a slightly blurry
        // parrot, but this time it will be the same size as the
        // input.
        save_image(result, "blurry_parrot_2.png");
    }

    printf("Success!\n");
    return 0;
}
```
## Build & Run
```bash
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ g++ lesson_07*.cpp -g -std=c++11 -I ../include -I ../tools -L ../bin -lHalide `libpng-config --cflags --ldflags` -ljpeg -lpthread -ldl -o lesson_07
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ LD_LIBRARY_PATH=../bin ./lesson_07
```

## 代码分析

```c
#include "Halide.h"
#include <stdio.h>

using namespace Halide;

// Support code for loading pngs.
#include "halide_image_io.h"
using namespace Halide::Tools;

int main(int argc, char **argv) {
```
首先，我们将在下面声明一些var。
```c
    Var x("x"), y("y"), c("c");
```
现在我们将表示一个多级管道，它先水平模糊图像，然后垂直模糊图像。
```c
    {
        // Take a color 8-bit input
        Buffer<uint8_t> input = load_image("images/rgb.png");

        // Upgrade it to 16-bit, so we can do math without it overflowing.
        Func input_16("input_16");
        input_16(x, y, c) = cast<uint16_t>(input(x, y, c));
```
水平模糊
```c
        // Blur it horizontally:
        Func blur_x("blur_x");
        blur_x(x, y, c) = (input_16(x-1, y, c) +
                           2 * input_16(x, y, c) +
                           input_16(x+1, y, c)) / 4;
```
垂直模糊，这里使用了函数嵌套
```c
        // Blur it vertically:
        Func blur_y("blur_y");
        blur_y(x, y, c) = (blur_x(x, y-1, c) +
                           2 * blur_x(x, y, c) +
                           blur_x(x, y+1, c)) / 4;
```
图像是8-bit, 需转换回
```c
        // Convert back to 8-bit.
        Func output("output");
        output(x, y, c) = cast<uint8_t>(blur_y(x, y, c));
```
此管道中的每个Func都使用熟悉的函数调用语法调用前一个Func（我们在Func对象上重载了operator()）。Func可以调用已给出定义的任何其他Func。此限制可防止管道中包含循环。Halide管道总是函数的前向图。

现在让我们realize。。。
```c
// Buffer<uint8_t> result = output.realize(input.width(), input.height(), 3);
```
只是上面这一行行不通。取消注释以查看发生了什么。

在与输入图像相同的域上实现此管道需要**读取超出输入边界的像素**，因为`blur_x`mstage水平向外延伸，`blur_y` stage垂直向外延伸。Halide通过在管道顶部**注入一段代码来检测这一点**，该代码计算将在其上读取输入的区域。当它开始运行管道时，它首先运行此代码，确定将读取超出界限的输入，并拒绝继续。在内部循环中没有实际的边界检查；这会很慢。

那我们该怎么办？有几个选择。如果我们意识到在一个域上向内移动了一个像素，我们就不会要求Halide程序读取越界。我们在上一课中看到了如何做到这一点：

```c

        Buffer<uint8_t> result(input.width()-2, input.height()-2, 3);
        result.set_min(1, 1);
        output.realize(result);
```

保存结果。它应该看起来像一只略带模糊的鹦鹉，并且应该比输入图像窄两个像素，短两个像素。

```c
        save_image(result, "blurry_parrot_1.png");
```
- Output
```bash
# 第一种方案少了两个像素
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ file blurry_parrot_1.png 
blurry_parrot_1.png: PNG image data, 766 x 1278, 8-bit/color RGB, non-interlaced
```
这通常是处理边界的最快方法：**不要编写读取越界的代码** :) 下一个示例是更一般的解决方案。
```c
    // The same pipeline, with a boundary condition on the input.
    {
        // Take a color 8-bit input
        Buffer<uint8_t> input = load_image("images/rgb.png");
```
这次，我们将在Func中包装输入，以防止reading越界：
```c
        Func clamped("clamped");
```
定义一个表达式，将`x`钳制在`range[0，input.width()-1]`范围内。
```c        
        Expr clamped_x = clamp(x, 0, input.width()-1);
        // clamp(x, a, b) is equivalent to max(min(x, b), a).

        // Similarly clamp y.
        Expr clamped_y = clamp(y, 0, input.height()-1);
        // Load from input at the clamped coordinates. This means that
```
不管我们如何计算Func `claped`，我们**永远不会读取输入的越界值**。这是一个钳制到边(clamp-to-edge)的边界条件，是用Halide表示的最简单的边界条件。
```c
        clamped(x, y, c) = input(clamped_x, clamped_y, c);
```
使用`BoundaryConditions`名称空间中的helper函数可以更简洁地定义`clamped`，例如
```c
// clamped = BoundaryConditions::repeat_edge(input);
```
这些对于其他边界条件很重要，因为它们是以Halide最能理解和优化的方式表达的。如果使用正确，它们的代价就和没有边界条件一样低。

将它提升到16位，这样我们就可以在不溢出的情况下进行计算。这次我们将引用我们的新函数`clamped`，而不是直接引用输入图像。

```c

        Func input_16("input_16");
        input_16(x, y, c) = cast<uint16_t>(clamped(x, y, c));
```
接下来的流程一样
```c
        // Blur it horizontally:
        Func blur_x("blur_x");
        blur_x(x, y, c) = (input_16(x-1, y, c) +
                           2 * input_16(x, y, c) +
                           input_16(x+1, y, c)) / 4;

        // Blur it vertically:
        Func blur_y("blur_y");
        blur_y(x, y, c) = (blur_x(x, y-1, c) +
                           2 * blur_x(x, y, c) +
                           blur_x(x, y+1, c)) / 4;

        // Convert back to 8-bit.
        Func output("output");
        output(x, y, c) = cast<uint8_t>(blur_y(x, y, c));

        // This time it's safe to evaluate the output over the some
        // domain as the input, because we have a boundary condition.
        Buffer<uint8_t> result = output.realize(input.width(), input.height(), 3);

        // Save the result. It should look like a slightly blurry
        // parrot, but this time it will be the same size as the
        // input.
        save_image(result, "blurry_parrot_2.png");
    }

    printf("Success!\n");
    return 0;
}
```
- Output
```bash
# 第二种方案正常
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ file blurry_parrot_2.png 
blurry_parrot_2.png: PNG image data, 768 x 1280, 8-bit/color RGB, non-interlaced
```
