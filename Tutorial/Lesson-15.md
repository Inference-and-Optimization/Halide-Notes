# lesson 15: Generators

本课演示了如何将Halide管道封装到称为生成器的**可重用**组件中。

# Part 1

## Build & Run
```bash
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ g++ lesson_15*.cpp ../tools/GenGen.cpp -g -std=c++11 -fno-rtti -I ../include -L ../bin -lHalide -lpthread -ldl -o lesson_15_generate
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ sudo apt-get install dos2unix
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ dos2unix lesson_15_generators_usage.sh
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ bash lesson_15_generators_usage.sh
```
## 代码分析

```c
#include "Halide.h"
#include <stdio.h>

using namespace Halide;
```
生成器是一种更结构化的方法，可以**提前进行Halide管道的编译**。而不是像第10节中那样使用临时的命令行界面编写`int main()`，而是定义一个继承自`Halide::Generator`的类。
```c
// Generators are a more structured way to do ahead-of-time
// compilation of Halide pipelines. Instead of writing an int main()
// with an ad-hoc command-line interface like we did in lesson 10, we
// define a class that inherits from Halide::Generator.
class MyFirstGenerator : public Halide::Generator<MyFirstGenerator> {
```
我们将Halide管道的`Input`声明为`public`成员变量。它们将**以与声明它们相同的顺序出现在生成的函数的签名**中。
```c
public:
    // We declare the Inputs to the Halide pipeline as public
    // member variables. They'll appear in the signature of our generated
    // function in the same order as we declare them.
    Input<uint8_t> offset{"offset"};
    Input<Buffer<uint8_t>> input{"input", 2};
```
我们还将`Outputs`声明为公共成员变量。
```c
    // We also declare the Outputs as public member variables.
    Output<Buffer<uint8_t>> brighter{"brighter", 2};
```
通常，您也需要在此范围内声明`Vars`，以便可以在以后添加的任何辅助方法中使用它们。
```c
    // Typically you declare your Vars at this scope as well, so that
    // they can be used in any helper methods you add later.
    Var x, y;
```
然后，我们定义一个构造函数并返回Halide管道的方法：
```c
    // We then define a method that constructs and return the Halide
    // pipeline:
    void generate() {
        // In lesson 10, here is where we called
        // Func::compile_to_file. In a Generator, we just need to
        // define the Output(s) representing the output of the pipeline.
        // 在第10课中，我们在这里调用Func::compile_to_file。在生成器中，我们只需要定义代表管道输出的Output即可。
        brighter(x, y) = input(x, y) + offset;

        // Schedule it.
        brighter.vectorize(x, 16).parallel(y);
    }
};
```
我们将这个文件与`tools/GenGen.cpp`一起编译。该文件定义了一个“int main(...)”，它提供了使用生成器类的命令行界面。 我们需要告诉有关生成器的代码。我们这样做：
```c
// We compile this file along with tools/GenGen.cpp. That file defines
// an "int main(...)" that provides the command-line interface to use
// your generator class. We need to tell that code about our
// generator. We do this like so:
HALIDE_REGISTER_GENERATOR(MyFirstGenerator, my_first_generator)
```
如果愿意，可以将多个Generator放在一个文件中。如果他们共享一些通用代码，这可能是一个好主意。让我们定义另一个更复杂的生成器：
```c
// If you like, you can put multiple Generators in the one file. This
// could be a good idea if they share some common code. Let's define
// another more complex generator:
class MySecondGenerator : public Halide::Generator<MySecondGenerator> {
public:
```
该生成器还将采用一些编译时参数。这些使您可以编译Halide管道的多个变体。我们将定义一个告诉我们是否在调度中并行化的对象：
```c
    // This generator will take some compile-time parameters
    // too. These let you compile multiple variants of a Halide
    // pipeline. We'll define one that tells us whether or not to
    // parallelize in our schedule:
    GeneratorParam<bool> parallel{"parallel", /* default value */ true};

    // ... and another representing a constant scale factor to use:
    GeneratorParam<float> scale{"scale",
            1.0f /* default value */,
            0.0f /* minimum value */,
            100.0f /* maximum value */};
```
您可以定义所有基本标量类型的`GeneratorParams`。对于数值类型，您可以选择提供最小值和最大值，就像上面对小数位数所做的那样。

您还可以为枚举定义`GeneratorParams`。为了完成这项工作，您必须提供从字符串到枚举值的映射。
```c
    // You can define GeneratorParams of all the basic scalar
    // types. For numeric types you can optionally provide a minimum
    // and maximum value, as we did for scale above.

    // You can also define GeneratorParams for enums. To make this
    // work you must provide a mapping from strings to your enum
    // values.
    enum class Rotation { None, Clockwise, CounterClockwise };
    GeneratorParam<Rotation> rotation{"rotation",
            /* default value */
            Rotation::None,
            /* map from names to values */
            {{ "none", Rotation::None },
             { "cw",   Rotation::Clockwise },
             { "ccw",  Rotation::CounterClockwise }}};
```
我们将使用与以前相同的输入
```c
    // We'll use the same Inputs as before:
    Input<uint8_t> offset{"offset"};
    Input<Buffer<uint8_t>> input{"input", 2};
```
和类似的输出。请注意，我们没有为`Buffer`指定类型：在编译时，必须通过“output.type” `GeneratorParam`（为该`Output`隐式定义）指定显式类型。
```c
    // And a similar Output. Note that we don't specify a type for the Buffer:
    // at compile-time, we must specify an explicit type via the "output.type"
    // GeneratorParam (which is implicitly defined for this Output).
    Output<Buffer<>> output{"output", 2};
```
我们将像以前一样在这里声明`Vars`。
```c
    // And we'll declare our Vars here as before.
    Var x, y;
```
定义`Func`。我们将使用编译时比例因子以及运行时偏移参数。
```c
    void generate() {
        // Define the Func. We'll use the compile-time scale factor as
        // well as the runtime offset param.
        Func brighter;
        brighter(x, y) = scale * (input(x, y) + offset);
```
根据枚举，我们可能会进行某种轮换。要获取GeneratorParam的值，请将其强制转换为相应的类型。这种转换在大多数时间都是隐式发生的（例如，缩放比例在上面）。
```c
        // We'll possibly do some sort of rotation, depending on the
        // enum. To get the value of a GeneratorParam, cast it to the
        // corresponding type. This cast happens implicitly most of
        // the time (e.g. with scale above).

        Func rotated;
        switch ((Rotation)rotation) {
        case Rotation::None:
            rotated(x, y) = brighter(x, y);
            break;
        case Rotation::Clockwise:
            rotated(x, y) = brighter(y, 100-x);
            break;
        case Rotation::CounterClockwise:
            rotated(x, y) = brighter(100-y, x);
            break;
        }
```
然后，我们将转换为所需的输出类型。
```c
        // We'll then cast to the desired output type.
        output(x, y) = cast(output.type(), rotated(x, y));
```
管道的结构取决于生成器参数。调度也一样。

让我们开始对输出进行向量化处理。虽然我们不知道类型，所以很难选择一个好的factor。生成器提供了一个名为“ natural_vector_size”的辅助程序，它将根据您要编译的类型和目标为您选择一个合理的factor。

```c
        // The structure of the pipeline depended on the generator
        // params. So will the schedule.

        // Let's start by vectorizing the output. We don't know the
        // type though, so it's hard to pick a good factor. Generators
        // provide a helper called "natural_vector_size" which will
        // pick a reasonable factor for you given the type and the
        // target you're compiling to.
        output.vectorize(x, natural_vector_size(output.type()));
```
现在我们可能将其并行化
```c
        // Now we'll possibly parallelize it:
        if (parallel) {
            output.parallel(y);
        }
```
如果发生旋转，我们将安排在输出的每条扫描线上进行旋转，并根据其类型进行向量化。
```c
        // If there was a rotation, we'll schedule that to occur per
        // scanline of the output and vectorize it according to its
        // type.
        if (rotation != Rotation::None) {
            rotated
                .compute_at(output, y)
                .vectorize(x, natural_vector_size(rotated.output_types()[0]));
        }
    }

};

// Register our second generator:
HALIDE_REGISTER_GENERATOR(MySecondGenerator, my_second_generator)

// After compiling this file, see how to use it in
// lesson_15_generators_build.sh
```

- Output
```c
$ bash lesson_15_generators_usage.sh
The halide runtime:
0000000000000000 W halide_buffer_copy
0000000000000000 W halide_buffer_copy_already_locked
0000000000000000 W halide_buffer_to_string
0000000000000000 W halide_cache_cleanup
0000000000000000 W halide_can_reuse_device_allocations
0000000000000000 W halide_can_use_target_features
0000000000000000 W halide_cond_broadcast
0000000000000000 W halide_cond_signal
0000000000000000 W halide_cond_wait
0000000000000000 W halide_copy_to_device
0000000000000000 W halide_copy_to_host
0000000000000000 W halide_current_time_ns
0000000000000000 W halide_debug_to_file
0000000000000000 W halide_default_buffer_copy
0000000000000000 W halide_default_can_use_target_features
0000000000000000 W halide_default_device_and_host_free
0000000000000000 W halide_default_device_and_host_malloc
0000000000000000 W halide_default_device_crop
0000000000000000 W halide_default_device_detach_native
0000000000000000 W halide_default_device_release_crop
0000000000000000 W halide_default_device_slice
0000000000000000 W halide_default_device_wrap_native
0000000000000000 W halide_default_do_loop_task
0000000000000000 W halide_default_do_parallel_tasks
0000000000000000 W halide_default_do_par_for
0000000000000000 W halide_default_do_task
0000000000000000 W halide_default_error
0000000000000000 W halide_default_free
0000000000000000 W halide_default_get_library_symbol
0000000000000000 W halide_default_get_symbol
0000000000000000 W halide_default_load_library
0000000000000000 W halide_default_malloc
0000000000000000 W halide_default_print
0000000000000000 W halide_default_semaphore_init
0000000000000000 W halide_default_semaphore_release
0000000000000000 W halide_default_semaphore_try_acquire
0000000000000000 W halide_default_trace
0000000000000000 W halide_device_and_host_free
0000000000000000 W halide_device_and_host_free_as_destructor
0000000000000000 W halide_device_and_host_malloc
0000000000000000 W halide_device_crop
0000000000000000 W halide_device_detach_native
0000000000000000 W halide_device_free
0000000000000000 W halide_device_free_as_destructor
0000000000000000 W halide_device_host_nop_free
0000000000000000 W halide_device_malloc
0000000000000000 W halide_device_release
0000000000000000 W halide_device_release_crop
0000000000000000 W halide_device_slice
0000000000000000 W halide_device_sync
0000000000000000 W halide_device_wrap_native
0000000000000000 W halide_do_loop_task
0000000000000000 W halide_do_parallel_tasks
0000000000000000 W halide_do_par_for
0000000000000000 W halide_do_task
0000000000000000 W halide_double_to_string
0000000000000000 W halide_error
0000000000000000 W halide_error_access_out_of_bounds
0000000000000000 W halide_error_bad_dimensions
0000000000000000 W halide_error_bad_extern_fold
0000000000000000 W halide_error_bad_fold
0000000000000000 W halide_error_bad_type
0000000000000000 W halide_error_bounds_inference_call_failed
0000000000000000 W halide_error_buffer_allocation_too_large
0000000000000000 W halide_error_buffer_argument_is_null
0000000000000000 W halide_error_buffer_extents_negative
0000000000000000 W halide_error_buffer_extents_too_large
0000000000000000 W halide_error_buffer_is_null
0000000000000000 W halide_error_constraints_make_required_region_smaller
0000000000000000 W halide_error_constraint_violated
0000000000000000 W halide_error_debug_to_file_failed
0000000000000000 W halide_error_device_dirty_with_no_device_support
0000000000000000 W halide_error_device_interface_no_device
0000000000000000 W halide_error_explicit_bounds_too_small
0000000000000000 W halide_error_extern_stage_failed
0000000000000000 W halide_error_fold_factor_too_small
0000000000000000 W halide_error_host_and_device_dirty
0000000000000000 W halide_error_host_is_null
0000000000000000 W halide_error_no_device_interface
0000000000000000 W halide_error_out_of_memory
0000000000000000 W halide_error_param_too_large_f64
0000000000000000 W halide_error_param_too_large_i64
0000000000000000 W halide_error_param_too_large_u64
0000000000000000 W halide_error_param_too_small_f64
0000000000000000 W halide_error_param_too_small_i64
0000000000000000 W halide_error_param_too_small_u64
0000000000000000 W halide_error_requirement_failed
0000000000000000 W halide_error_specialize_fail
0000000000000000 W halide_error_unaligned_host_ptr
0000000000000000 W halide_float16_bits_to_double
0000000000000000 W halide_float16_bits_to_float
0000000000000000 W halide_free
0000000000000000 W halide_get_gpu_device
0000000000000000 W halide_get_library_symbol
0000000000000000 W halide_get_symbol
0000000000000000 W halide_get_trace_file
0000000000000000 W halide_host_cpu_count
0000000000000000 W halide_int64_to_string
0000000000000000 W halide_join_thread
0000000000000000 W halide_load_library
0000000000000000 W halide_malloc
0000000000000000 W halide_memoization_cache_cleanup
0000000000000000 W halide_memoization_cache_lookup
0000000000000000 W halide_memoization_cache_release
0000000000000000 W halide_memoization_cache_set_size
0000000000000000 W halide_memoization_cache_store
0000000000000000 W halide_msan_annotate_buffer_is_initialized
0000000000000000 W halide_msan_annotate_buffer_is_initialized_as_destructor
0000000000000000 W halide_msan_annotate_memory_is_initialized
0000000000000000 W halide_msan_check_buffer_is_initialized
0000000000000000 W halide_msan_check_memory_is_initialized
0000000000000000 W halide_mutex_array_create
0000000000000000 W halide_mutex_array_destroy
0000000000000000 W halide_mutex_array_lock
0000000000000000 W halide_mutex_array_unlock
0000000000000000 W halide_mutex_lock
0000000000000000 W halide_mutex_unlock
0000000000000000 W halide_pointer_to_string
0000000000000000 W halide_print
0000000000000000 W halide_profiler_get_pipeline_state
0000000000000000 W halide_profiler_get_state
0000000000000000 W halide_profiler_memory_allocate
0000000000000000 W halide_profiler_memory_free
0000000000000000 W halide_profiler_pipeline_end
0000000000000000 W halide_profiler_pipeline_start
0000000000000000 W halide_profiler_report
0000000000000000 W halide_profiler_report_unlocked
0000000000000000 W halide_profiler_reset
0000000000000000 W halide_profiler_reset_unlocked
0000000000000000 W halide_profiler_shutdown
0000000000000000 W halide_profiler_stack_peak_update
0000000000000000 W halide_register_device_allocation_pool
0000000000000000 W halide_reuse_device_allocations
0000000000000000 W halide_semaphore_init
0000000000000000 W halide_semaphore_release
0000000000000000 W halide_semaphore_try_acquire
0000000000000000 W halide_set_custom_can_use_target_features
0000000000000000 W halide_set_custom_do_loop_task
0000000000000000 W halide_set_custom_do_par_for
0000000000000000 W halide_set_custom_do_task
0000000000000000 W halide_set_custom_free
0000000000000000 W halide_set_custom_get_library_symbol
0000000000000000 W halide_set_custom_get_symbol
0000000000000000 W halide_set_custom_load_library
0000000000000000 W halide_set_custom_malloc
0000000000000000 W halide_set_custom_parallel_runtime
0000000000000000 W halide_set_custom_print
0000000000000000 W halide_set_custom_trace
0000000000000000 W halide_set_error_handler
0000000000000000 W halide_set_gpu_device
0000000000000000 W halide_set_num_threads
0000000000000000 W halide_set_trace_file
0000000000000000 W halide_shutdown_thread_pool
0000000000000000 W halide_shutdown_trace
0000000000000000 W halide_sleep_ms
0000000000000000 W halide_spawn_thread
0000000000000000 W halide_start_clock
0000000000000000 W halide_string_to_string
0000000000000000 W halide_thread_pool_cleanup
0000000000000000 W halide_trace
0000000000000000 W halide_trace_cleanup
0000000000000000 W halide_trace_helper
0000000000000000 W halide_type_to_string
0000000000000000 W halide_uint64_to_string
ar: creating my_first_generator_multi.a
```
# Part 2

此shell脚本演示了如何从命令行使用包含`Generators`的二进制文件。通常，您会从您选择的构建系统中调用这些二进制文件，而不是像我们在此处那样手动运行它们。

该脚本假定您位于`tutorials`目录中，并且生成器已针对当前系统进行编译，并且称为“lesson_15_generate”。

要运行此脚本：bash lesson_15_generators_usage.sh首先，我们定义一个帮助程序功能，该功能检查文件是否存在

```bash
# This shell script demonstrates how to use a binary containing
# Generators from the command line. Normally you'd call these binaries
# from your build system of choice rather than running them manually
# like we do here.

# This script assumes that you're in the tutorials directory, and the
# generator has been compiled for the current system and is called
# "lesson_15_generate".

# To run this script:
# bash lesson_15_generators_usage.sh

# First we define a helper function that checks that a file exists
check_file_exists()
{
    FILE=$1
    if [ ! -f $FILE ]; then
        echo $FILE not found
        exit -1
    fi
}

# And another helper function to check if a symbol exists in an object file
check_symbol()
{
    FILE=$1
    SYM=$2
    if !(nm $FILE | grep $SYM > /dev/null); then
        echo "$SYM not found in $FILE"
    exit -1
    fi
}

# Bail out on error
#set -e

# Set up LD_LIBRARY_PATH so that we can find libHalide.so
export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:../bin
export DYLD_LIBRARY_PATH=${DYLD_LIBRARY_PATH}:../bin

#########################
# Basic generator usage #
#########################

# First let's compile the first generator for the host system:
./lesson_15_generate -g my_first_generator -o . target=host

# That should create a pair of files in the current directory:
# "my_first_generator.a", and "my_first_generator.h", which define a
# function "my_first_generator" representing the compiled pipeline.

check_file_exists my_first_generator.a
check_file_exists my_first_generator.h
check_symbol my_first_generator.a my_first_generator

#####################
# Cross-compilation #
#####################

# We can also use a generator to compile object files for some other
# target. Let's cross-compile a windows 32-bit object file and header
# for the first generator:

./lesson_15_generate \
    -g my_first_generator \
    -f my_first_generator_win32 \
    -o . \
    target=x86-32-windows

# This generates a file called "my_first_generator_win32.lib" in the
# current directory, along with a matching header. The function
# defined is called "my_first_generator_win32".

check_file_exists my_first_generator_win32.lib
check_file_exists my_first_generator_win32.h

################################
# Generating pipeline variants #
################################

# The full set of command-line arguments to the generator binary are:

# -g generator_name : Selects which generator to run. If you only have
# one generator in your binary you can omit this.

# -o directory : Specifies which directory to create the outputs
# in. Usually a build directory.

# -f name : Specifies the name of the generated function. If you omit
# this, it defaults to the generator name.

# -n file_base_name : Specifies the basename of the generated file(s). If
# you omit this, it defaults to the name of the generated function.

# -e static_library,o,h,assembly,bitcode,stmt,html: A list of
# comma-separated values specifying outputs to create. The default is
# "static_library,h". "assembly" generates assembly equivalent to the
# generated object file. "bitcode" generates llvm bitcode for the pipeline.
# "stmt" generates human-readable pseudocode for the pipeline (similar to
# setting HL_DEBUG_CODEGEN). "html" generates an html version of the
# pseudocode, which can be much nicer to read than the raw .stmt file.

# -r file_base_name : Specifies that the generator should create a
# standalone file for just the runtime. For use when generating multiple
# pipelines from a single generator, to be linked together in one
# executable. See example below.

# -x .old=new,.old2=.new2,... : A comma-separated list of file extension
# pairs to substitute during file naming.

# target=... : The target to compile for.

# my_generator_param=value : The value of your generator params.

# Let's now generate some human-readable pseudocode for the first
# generator:

./lesson_15_generate -g my_first_generator -e stmt -o . target=host

check_file_exists my_first_generator.stmt

# The second generator has generator params, which can be specified on
# the command-line after the target. Let's compile a few different variants:
./lesson_15_generate -g my_second_generator -f my_second_generator_1 -o . \
target=host parallel=false scale=3.0 rotation=ccw output.type=uint16

./lesson_15_generate -g my_second_generator -f my_second_generator_2 -o . \
target=host scale=9.0 rotation=ccw output.type=float32

./lesson_15_generate -g my_second_generator -f my_second_generator_3 -o . \
target=host parallel=false output.type=float64

check_file_exists my_second_generator_1.a
check_file_exists my_second_generator_1.h
check_symbol      my_second_generator_1.a my_second_generator_1
check_file_exists my_second_generator_2.a
check_file_exists my_second_generator_2.h
check_symbol      my_second_generator_2.a my_second_generator_2
check_file_exists my_second_generator_3.a
check_file_exists my_second_generator_3.h
check_symbol      my_second_generator_3.a my_second_generator_3

# Use of these generated object files and headers is exactly the same
# as in lesson 10.

######################
# The Halide runtime #
######################

# Each generated Halide object file contains a simple runtime that
# defines things like how to run a parallel for loop, how to launch a
# cuda program, etc. You can see this runtime in the generated object
# files.

echo "The halide runtime:"
nm my_second_generator_1.a | grep "[SWT] _\?halide_"

# Let's define some functions to check that the runtime exists in a file.
check_runtime()
{
    if !(nm $1 | grep "[TSW] _\?halide_" > /dev/null); then
        echo "Halide runtime not found in $1"
    exit -1
    fi
}

check_no_runtime()
{
    if nm $1 | grep "[TSW] _\?halide_" > /dev/null; then
        echo "Halide runtime found in $1"
    exit -1
    fi
}

# Declarations and documentation for these runtime functions are in
# HalideRuntime.h

# If you're compiling and linking multiple Halide pipelines, then the
# multiple copies of the runtime should combine into a single copy
# (via weak linkage). If you're compiling and linking for multiple
# different targets (e.g. avx and non-avx), then the runtimes might be
# different, and you can't control which copy of the runtime the
# linker selects.

# You can control this behavior explicitly by compiling your pipelines
# with the no_runtime target flag. Let's generate and link several
# different versions of the first pipeline for different x86 variants:

# (Note that we'll ask the generators to just give us object files ("-e o"), 
# instead of static libraries, so that we can easily link them all into a 
# single static library.)

./lesson_15_generate \
    -g my_first_generator \
    -f my_first_generator_basic \
    -e o,h \
    -o . \
    target=host-x86-64-no_runtime

./lesson_15_generate \
    -g my_first_generator \
    -f my_first_generator_sse41 \
    -e o,h \
    -o . \
    target=host-x86-64-sse41-no_runtime

./lesson_15_generate \
    -g my_first_generator \
    -f my_first_generator_avx \
    -e o,h \
    -o . \
    target=host-x86-64-avx-no_runtime

# These files don't contain the runtime
check_no_runtime my_first_generator_basic.o
check_symbol     my_first_generator_basic.o my_first_generator_basic
check_no_runtime my_first_generator_sse41.o
check_symbol     my_first_generator_sse41.o my_first_generator_sse41
check_no_runtime my_first_generator_avx.o
check_symbol     my_first_generator_avx.o my_first_generator_avx

# We can then use the generator to emit just the runtime:
./lesson_15_generate \
    -r halide_runtime_x86 \
    -e o,h \
    -o . \
    target=host-x86-64
check_runtime halide_runtime_x86.o

# Linking the standalone runtime with the three generated object files     
# gives us three versions of the pipeline for varying levels of x86,      
# combined with a single runtime that will work on nearly all x86     
# processors.
ar q my_first_generator_multi.a \
    my_first_generator_basic.o \
    my_first_generator_sse41.o \
    my_first_generator_avx.o \
    halide_runtime_x86.o

check_runtime my_first_generator_multi.a
check_symbol  my_first_generator_multi.a my_first_generator_basic
check_symbol  my_first_generator_multi.a my_first_generator_sse41
check_symbol  my_first_generator_multi.a my_first_generator_avx

echo "Success!"
```