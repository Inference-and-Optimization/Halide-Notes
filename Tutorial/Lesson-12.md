# lesson 12: Using the GPU

## Code

```c
#include "Halide.h"
#include <stdio.h>
using namespace Halide;

#include "halide_image_io.h"
using namespace Halide::Tools;

#include "clock.h"

Var x, y, c, i, ii, xo, yo, xi, yi;

class MyPipeline {
public:
    Func lut, padded, padded16, sharpen, curved;
    Buffer<uint8_t> input;

    MyPipeline(Buffer<uint8_t> in) : input(in) {
        lut(i) = cast<uint8_t>(clamp(pow(i / 255.0f, 1.2f) * 255.0f, 0, 255));

        padded(x, y, c) = input(clamp(x, 0, input.width()-1),
                                clamp(y, 0, input.height()-1), c);

        padded16(x, y, c) = cast<uint16_t>(padded(x, y, c));

        sharpen(x, y, c) = (padded16(x, y, c) * 2-
                            (padded16(x - 1, y, c) +
                             padded16(x, y - 1, c) +
                             padded16(x + 1, y, c) +
                             padded16(x, y + 1, c)) / 4);

        curved(x, y, c) = lut(sharpen(x, y, c));
    }

    void schedule_for_cpu() {
        lut.compute_root();

        curved.reorder(c, x, y)
              .bound(c, 0, 3)
              .unroll(c);

        Var yo, yi;
        curved.split(y, yo, yi, 16)
              .parallel(yo);

        sharpen.compute_at(curved, yi);

        sharpen.vectorize(x, 8);

        padded.store_at(curved, yo)
              .compute_at(curved, yi);

        padded.vectorize(x, 16);

        curved.compile_jit();
    }

    void schedule_for_gpu() {
        lut.compute_root();

        Var block, thread;
        lut.split(i, block, thread, 16);
        lut.gpu_blocks(block)
           .gpu_threads(thread);

        curved.reorder(c, x, y)
              .bound(c, 0, 3)
              .unroll(c);

        curved.gpu_tile(x, y, xo, yo, xi, yi, 8, 8);

        padded.compute_at(curved, xo);

        padded.gpu_threads(x, y);

        Target target = get_host_target();

        if (target.os == Target::OSX) {
            target.set_feature(Target::Metal);
        } else {
            target.set_feature(Target::OpenCL);
        }

        curved.compile_jit(target);
    }

    void test_performance() {
        Buffer<uint8_t> output(input.width(), input.height(), input.channels());

        curved.realize(output);

        double best_time = 0.0;
        for (int i = 0; i < 3; i++) {

            double t1 = current_time();

            for (int j = 0; j < 100; j++) {
                curved.realize(output);
            }

            output.copy_to_host();

            double t2 = current_time();

            double elapsed = (t2 - t1)/100;
            if (i == 0 || elapsed < best_time) {
                best_time = elapsed;
            }
        }

        printf("%1.4f milliseconds\n", best_time);
    }

    void test_correctness(Buffer<uint8_t> reference_output) {
        Buffer<uint8_t> output =
            curved.realize(input.width(), input.height(), input.channels());

        for (int c = 0; c < input.channels(); c++) {
            for (int y = 0; y < input.height(); y++) {
                for (int x = 0; x < input.width(); x++) {
                    if (output(x, y, c) != reference_output(x, y, c)) {
                        printf("Mismatch between output (%d) and "
                               "reference output (%d) at %d, %d, %d\n",
                               output(x, y, c),
                               reference_output(x, y, c),
                               x, y, c);
                        exit(-1);
                    }
                }
            }
        }

    }
};

bool have_opencl_or_metal();

int main(int argc, char **argv) {
    Buffer<uint8_t> input = load_image("images/rgb.png");

    Buffer<uint8_t> reference_output(input.width(), input.height(), input.channels());

    printf("Testing performance on CPU:\n");
    MyPipeline p1(input);
    p1.schedule_for_cpu();
    p1.test_performance();
    p1.curved.realize(reference_output);

    if (have_opencl_or_metal()) {
        printf("Testing performance on GPU:\n");
        MyPipeline p2(input);
        p2.schedule_for_gpu();
        p2.test_performance();
        p2.test_correctness(reference_output);
    } else {
        printf("Not testing performance on GPU, "
               "because I can't find the opencl library\n");
    }

    return 0;
}

#ifdef _WIN32
#include <windows.h>
#else
#include <dlfcn.h>
#endif

bool have_opencl_or_metal() {
#ifdef _WIN32
    return LoadLibrary("OpenCL.dll") != NULL;
#elif __APPLE__
    return dlopen("/System/Library/Frameworks/Metal.framework/Versions/Current/Metal", RTLD_LAZY) != NULL;
#else
    return dlopen("libOpenCL.so", RTLD_LAZY) != NULL;
#endif
}
```
## Build & Run

```bash
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ g++ lesson_12*.cpp -g -std=c++11 -I ../include -I ../tools -L ../bin -lHalide `libpng-config --cflags --ldflags` -ljpeg -lpthread -ldl -o lesson_12
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ LD_LIBRARY_PATH=../bin ./lesson_12
```
## 代码分析

```c
#include "Halide.h"
#include <stdio.h>
using namespace Halide;

// Include some support code for loading pngs.
#include "halide_image_io.h"
using namespace Halide::Tools;

// Include a clock to do performance testing.
#include "clock.h"

// Define some Vars to use.
Var x, y, c, i, ii, xo, yo, xi, yi;
```
我们将以多种方式调度一个管道，因此我们在一个类中定义该管道，以便可以使用不同的调度多次重新创建它。
```c
// We're going to want to schedule a pipeline in several ways, so we
// define the pipeline in a class so that we can recreate it several
// times with different schedules.
class MyPipeline {
public:
    Func lut, padded, padded16, sharpen, curved;
    Buffer<uint8_t> input;

    MyPipeline(Buffer<uint8_t> in) : input(in) {
        // For this lesson, we'll use a two-stage pipeline that sharpens
        // and then applies a look-up-table (LUT).

        // First we'll define the LUT. It will be a gamma curve.

        lut(i) = cast<uint8_t>(clamp(pow(i / 255.0f, 1.2f) * 255.0f, 0, 255));

        // Augment the input with a boundary condition.
        padded(x, y, c) = input(clamp(x, 0, input.width()-1),
                                clamp(y, 0, input.height()-1), c);

        // Cast it to 16-bit to do the math.
        padded16(x, y, c) = cast<uint16_t>(padded(x, y, c));

        // Next we sharpen it with a five-tap filter.
        sharpen(x, y, c) = (padded16(x, y, c) * 2-
                            (padded16(x - 1, y, c) +
                             padded16(x, y - 1, c) +
                             padded16(x + 1, y, c) +
                             padded16(x, y + 1, c)) / 4);

        // Then apply the LUT.
        curved(x, y, c) = lut(sharpen(x, y, c));
    }
```
#### schedule_for_cpu

```c
    // Now we define methods that give our pipeline several different
    // schedules.
    void schedule_for_cpu() {
```
提前计算LUT。
```c
        // Compute the look-up-table ahead of time.
        lut.compute_root();
```
计算最里面的颜色通道。约定它们会有三个通道，并展开它们。
```c
        // Compute color channels innermost. Promise that there will
        // be three of them and unroll across them.
        curved.reorder(c, x, y)
              .bound(c, 0, 3)
              .unroll(c);
```
查找表不能很好地向量化，所以只需**将16条扫描线的slice并行化**。
```c
        // Look-up-tables don't vectorize well, so just parallelize
        // curved in slices of 16 scanlines.
        Var yo, yi;
        curved.split(y, yo, yi, 16)
              .parallel(yo);
```
根据曲线扫描线的需要计算锐化。
```c
        // Compute sharpen as needed per scanline of curved.
        sharpen.compute_at(curved, yi);
```
将锐化向量化。它是16位的，所以我们将它向量化为8-wide。
```c
        // Vectorize the sharpen. It's 16-bit so we'll vectorize it 8-wide.
        sharpen.vectorize(x, 8);
```
根据需要计算每个曲线扫描线的`padded`输入，重用以前在16条扫描线的同一条中计算的值。
```c
        // Compute the padded input as needed per scanline of curved,
        // reusing previous values computed within the same strip of
        // 16 scanlines.
        padded.store_at(curved, yo)
              .compute_at(curved, yi);
```
同时对`padded`进行向量化。它是8位的，所以我们将16-wide向量化。
```c
        // Also vectorize the padding. It's 8-bit, so we'll vectorize
        // 16-wide.
        padded.vectorize(x, 16);
```
JIT编译CPU管道。
```c
        // JIT-compile the pipeline for the CPU.
        curved.compile_jit();
    }
```
#### schedule_for_gpu
我们决定是否对每个函数单独使用GPU。如果有一个`Func`是在CPU上计算的，而下一个`Func`是在GPU上计算的，Halide将在引擎盖下拷贝到GPU。对于这个管道，没有理由在任何阶段使用CPU。Halide将在我们第一次运行管道时将输入图像复制到GPU，并将其留在那里以便在以后的运行中重用。
和以前一样，我们将在管道开始时计算一次LUT。

```c
    // Now a schedule that uses CUDA or OpenCL.
    void schedule_for_gpu() {
        // We make the decision about whether to use the GPU for each
        // Func independently. If you have one Func computed on the
        // CPU, and the next computed on the GPU, Halide will do the
        // copy-to-gpu under the hood. For this pipeline, there's no
        // reason to use the CPU for any of the stages. Halide will
        // copy the input image to the GPU the first time we run the
        // pipeline, and leave it there to reuse on subsequent runs.

        // As before, we'll compute the LUT once at the start of the
        // pipeline.
        lut.compute_root();
```
让我们使用16宽一维线程块中的GPU计算查找表。首先，我们将索引分成大小为16的块：
```c
        // Let's compute the look-up-table using the GPU in 16-wide
        // one-dimensional thread blocks. First we split the index
        // into blocks of size 16:
        Var block, thread;
        lut.split(i, block, thread, 16);
```
然后我们告诉cuda，我们的变量`block`和`thread`对应于`CUDA`的块和线程的概念，或者`OpenCL`的线程组和线程的概念。
```c
        // Then we tell cuda that our Vars 'block' and 'thread'
        // correspond to CUDA's notions of blocks and threads, or
        // OpenCL's notions of thread groups and threads.
        lut.gpu_blocks(block)
           .gpu_threads(thread);
```
这是GPU上一种非常常见的调度模式，因此有一个简写：`lut.gpu_tile(i, block, thread, 16)`；`Func::gpu_tile`的行为与`Func::tile`相同，只是它还指定`tile`坐标对应于GPU块，并且每个`tile`中的坐标对应于GPU线程。

计算最里面的颜色通道。约定它们会有三个通道，并展开。
```c
        // This is a very common scheduling pattern on the GPU, so
        // there's a shorthand for it:

        // lut.gpu_tile(i, block, thread, 16);

        // Func::gpu_tile behaves the same as Func::tile, except that
        // it also specifies that the tile coordinates correspond to
        // GPU blocks, and the coordinates within each tile correspond
        // to GPU threads.

        // Compute color channels innermost. Promise that there will
        // be three of them and unroll across them.
        curved.reorder(c, x, y)
              .bound(c, 0, 3)
              .unroll(c);
```
使用GPU计算二维8x8 tiles中的曲线。
```c
        // Compute curved in 2D 8x8 tiles using the GPU.
        curved.gpu_tile(x, y, xo, yo, xi, yi, 8, 8);
```
等价表示
```c
        // This is equivalent to:
        // curved.tile(x, y, xo, yo, xi, yi, 8, 8)
        //       .gpu_blocks(xo, yo)
        //       .gpu_threads(xi, yi);
```
我们将使锐化成为内联的曲线。

根据需要计算每个GPU块的`padded`输入，将中间结果存储在共享内存中。在上面的调度中，`xo`对应于GPU blocks。

```c
        // We'll leave sharpen as inlined into curved.

        // Compute the padded input as needed per GPU block, storing
        // the intermediate result in shared memory. In the schedule
        // above xo corresponds to GPU blocks.
        padded.compute_at(curved, xo);
```
将GPU线程用于`padded`输入的x和y坐标。
```c
        // Use the GPU threads for the x and y coordinates of the
        // padded input.
        padded.gpu_threads(x, y);
```
JIT编译GPU管道。**默认情况下不启用CUDA、OpenCL或Metal**。我们**必须构造一个目标对象，启用其中一个**，然后将该目标对象传递给`compile_jit`。否则你的CPU会很慢地模拟它是一个GPU，并且每输出像素使用一个线程。

从一个适合你运行的机器的目标开始。
```c
        // JIT-compile the pipeline for the GPU. CUDA, OpenCL, or
        // Metal are not enabled by default. We have to construct a
        // Target object, enable one of them, and then pass that
        // target object to compile_jit. Otherwise your CPU will very
        // slowly pretend it's a GPU, and use one thread per output
        // pixel.

        // Start with a target suitable for the machine you're running
        // this on.
        Target target = get_host_target();
```
然后启用`OpenCL`或`Metal`，这取决于我们在哪个平台上。OSX不更新`OpenCL`驱动程序，所以它们很容易损坏。`CUDA`在配备NVidia GPU的机器上也是一个不错的选择。
```c
        // Then enable OpenCL or Metal, depending on which platform
        // we're on. OS X doesn't update its OpenCL drivers, so they
        // tend to be broken. CUDA would also be a fine choice on
        // machines with NVidia GPUs.
        if (target.os == Target::OSX) {
            target.set_feature(Target::Metal);
        } else {
            target.set_feature(Target::OpenCL);
        }
```
如果您想看到管道完成的所有OpenCL、Metal或CUDA API调用，还可以启用Debug标志。这有助于找出哪些阶段是缓慢的，或者什么时候CPU->GPU拷贝发生。不过，这会影响性能。
```c
        // Uncomment the next line and comment out the lines above to
        // try CUDA instead.
        // target.set_feature(Target::CUDA);

        // If you want to see all of the OpenCL, Metal, or CUDA API
        // calls done by the pipeline, you can also enable the Debug
        // flag. This is helpful for figuring out which stages are
        // slow, or when CPU -> GPU copies happen. It hurts
        // performance though, so we'll leave it commented out.
        // target.set_feature(Target::Debug);

        curved.compile_jit(target);
    }
```
#### test_performance
```c
    void test_performance() {
        // Test the performance of the scheduled MyPipeline.

        Buffer<uint8_t> output(input.width(), input.height(), input.channels());

        // Run the filter once to initialize any GPU runtime state.
        curved.realize(output);

        // Now take the best of 3 runs for timing.
        double best_time = 0.0;
        for (int i = 0; i < 3; i++) {

            double t1 = current_time();

            // Run the filter 100 times.
            for (int j = 0; j < 100; j++) {
                curved.realize(output);
            }

            // Force any GPU code to finish by copying the buffer back to the CPU.
            output.copy_to_host();

            double t2 = current_time();

            double elapsed = (t2 - t1)/100;
            if (i == 0 || elapsed < best_time) {
                best_time = elapsed;
            }
        }

        printf("%1.4f milliseconds\n", best_time);
    }
```
#### test_correctness
```c
    void test_correctness(Buffer<uint8_t> reference_output) {
        Buffer<uint8_t> output =
            curved.realize(input.width(), input.height(), input.channels());

        // Check against the reference output.
        for (int c = 0; c < input.channels(); c++) {
            for (int y = 0; y < input.height(); y++) {
                for (int x = 0; x < input.width(); x++) {
                    if (output(x, y, c) != reference_output(x, y, c)) {
                        printf("Mismatch between output (%d) and "
                               "reference output (%d) at %d, %d, %d\n",
                               output(x, y, c),
                               reference_output(x, y, c),
                               x, y, c);
                        exit(-1);
                    }
                }
            }
        }

    }
};
```
#### main
```c
bool have_opencl_or_metal();

int main(int argc, char **argv) {
    // Load an input image.
    Buffer<uint8_t> input = load_image("images/rgb.png");

    // Allocated an image that will store the correct output
    Buffer<uint8_t> reference_output(input.width(), input.height(), input.channels());

    printf("Testing performance on CPU:\n");
    MyPipeline p1(input);
    p1.schedule_for_cpu();
    p1.test_performance();
    p1.curved.realize(reference_output);

    if (have_opencl_or_metal()) {
        printf("Testing performance on GPU:\n");
        MyPipeline p2(input);
        p2.schedule_for_gpu();
        p2.test_performance();
        p2.test_correctness(reference_output);
    } else {
        printf("Not testing performance on GPU, "
               "because I can't find the opencl library\n");
    }

    return 0;
}

// A helper function to check if OpenCL seems to exist on this machine.

#ifdef _WIN32
#include <windows.h>
#else
#include <dlfcn.h>
#endif

bool have_opencl_or_metal() {
#ifdef _WIN32
    return LoadLibrary("OpenCL.dll") != NULL;
#elif __APPLE__
    return dlopen("/System/Library/Frameworks/Metal.framework/Versions/Current/Metal", RTLD_LAZY) != NULL;
#else
    return dlopen("libOpenCL.so", RTLD_LAZY) != NULL;
#endif
}
```
- Output
```c
// 这个输出使用的新代码，跟以上的略不匹配
Running pipeline on CPU:
Running pipeline on GPU:
Target: x86-64-linux-avx-avx2-f16c-fma-opencl-sse41
Testing GPU correctness:
Testing performance on CPU:
0.4891 milliseconds
Testing performance on GPU:
0.3559 milliseconds
```