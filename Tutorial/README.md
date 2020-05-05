# Halide

## Halide OpenGL/GLSL backend

**Halide的OpenGL后端**通过生成基于GLSL的片段着色器将图像处理操作卸载到GPU。

与CUDA和OpenCL等其他基于GPU的处理选项相比，**OpenGL具有两个主要优点**：基本上每个台式机和移动设备都可以使用OpenGL，并且通常在不同的硬件供应商中都得到很好的支持。

OpenGL作为图像处理框架的**主要缺点**是片段着色器的**计算能力受到很大限制**。通常，OpenGL提供的处理模型最适合于过滤器，其中每个输出像素都可以表示为输入像素的简单函数。这涵盖了许多有趣的操作，例如逐点过滤器和卷积。但是众所周知，在GLSL中很难表达一些常见的图像处理操作，例如直方图或递归过滤器。

### Writing OpenGL-Based Filters

要启用OpenGL的代码生成，请在传递给Halide的目标说明符中包含opengl。由于OpenGL着色器的计算能力有限，因此您还必须为无法或不应在GPU上计算的滤镜部分指定CPU目标。有效目标说明符的示例是

```
host-opengl
x86-opengl-debug
```
如第二个示例中所示，添加`debug`会添加其他日志记录输出，强烈建议在开发过程中使用。

**默认情况下，为OpenGL目标编译的过滤器完全在CPU上运行。必须通过适当的调度调用在单个Funcs上启用在GPU上执行**。

GLSL片段着色器隐式地**在两个空间维度x，y和颜色通道上进行迭代**。由于GLSL中处理颜色通道的方式，只能安排颜色索引为编译时常数的过滤器。主要结果是在调度之前必须**为输入和输出缓冲区显式指定颜色变量的范围**：
```c
ImageParam input;
Func f;
Var x, y, c;
f(x, y, c) = ...;

input.set_bounds(2, 0, 3);   // specify color range for input
f.bound(c, 0, 3);            // and output
f.glsl(x, y, c);
```

### JIT Compilation

对于JIT编译，Halide尝试为opengl加载系统库，并创建一个用于每个模块的新上下文。Windows尚不支持。在test/opengl中可以找到基于OpenGL的过滤器的JIT执行示例。

### AOT Compilation

使用AOT（提前 ahead-of-time）编译时，Halide会生成启用OpenGL的目标文件，**这些文件可以链接到主机应用程序，也可以从主机应用程序调用**。通常，这非常简单，但是必须注意一些事项。

在Linux，OS X和Android上，除非当前线程已具有活动上下文，否则Halide会创建自己的OpenGL上下文。在其他平台上，您必须将以下两个功能的实现与Halide代码链接起来：
```c
extern "C" int halide_opengl_create_context(void *) {
    return 0;  // if successful
}

extern "C" void *halide_opengl_get_proc_addr(void *, const char *name) {
    ...
}
```

Halide 会根据需要分配和删除textures。应用程序可以通过设置`halide_buffer_t::device`字段来手动管理texture。这对于重用已经存储在texture中的图像数据最有用。执行一些基本检查以确保从外部分配的texture具有正确的格式，但这通常是应用程序的责任。

可以直接将渲染渲染到当前的帧缓冲区。为此，请将输出缓冲区的`dev`字段设置为`halide_opengl_output_client_bound`返回的值。`apps/HelloAndroidGL`中的示例演示了此技术。

某些操作系统可以删除已暂停应用程序的OpenGL上下文。如果发生这种情况，在应用程序恢复后，Halide需要使用新的上下文重新初始化自身。发生这种情况后，调用`halide_opengl_context_lost`以重置Halide的OpenGL状态。

### Limitations

OpenGL后端的当前实现针对OpenGL 2.0和OpenGL ES 2.0的公共子集，该子集可在移动设备和传统计算机上广泛使用。结果，只有Halide语言的一部分可以被调度用来使用OpenGL运行。一些重要的限制是：
- Reduction无法在GLSL中实现，必须在CPU上运行。
- OpenGL ES 2.0仅支持uint8缓冲区。

  支持浮点texture，但是需要OpenGL（ES）3.0或texture_float扩展，这些扩展可能不适用于所有移动设备。
- OpenGL ES 2.0对整数算术的支持非常有限。为了获得最大的兼容性，即使使用整数texture，也应考虑使用浮点进行所有计算。
- 只能调度具有3或4个颜色通道的2D图像。具有一个或两个通道的图像需要OpenGL（ES）3.0或texture_rg扩展。
- 目前尚不支持Halide提供的所有内置函数，例如`fast_log`，`fast_exp`，`fast_pow`，`reinterpret`，位操作，`random_float`，`random_int`无法在GLSL代码中使用。

OpenGL中的最大纹理大小为`GL_MAX_TEXTURE_SIZE`，通常小于目标图像；在移动设备上，例如`GL_MAX_TEXTURE_SIZE`通常为2048。必须使用Tiling来处理较大的图像。

计划的功能：
- 支持半浮点纹理和算术
- 支持整数纹理和算术

（请注意，单独的OpenGLCompute后端支持OpenGL Compute Shaders。） 

## Halide for Hexagon HVX

Halide支持将工作卸载到Qualcomm Snapdragon 820或更高版本上的Qualcomm Hexagon DSP。Hexagon DSP提供一组64和128字节的向量指令-Hexagon向量扩展（HVX）。HVX非常适合图像处理，Halide for Hexagon HVX将从Halide中编写的程序中生成适当的HVX向量指令。

通过使用诸如`hexagon-32-qurt-hvx_64`或`hexagon-32-qurt-hvx_128`之类的目标，可以使用Halide直接编译Hexagon目标文件。

也可以使用`hexagon`调度指令将Halid用于将管道的一部分卸载到Hexagon。要启用`hexagon`调度指令，请在目标中包含`hvx_64`或`hvx_128`目标功能。当前支持的目标组合是将HVX目标功能与x86 linux主机（使用模拟器）或ARM android目标（使用Hexagon DSP硬件）一起使用。 有关在模拟器和Hexagon DSP上使用`hexagon`调度指令的示例，请参见blur示例应用程序。

要使用Hexagon目标构建并运行示例应用程序，
- 请获取并构建trunk LLVM和Clang。（早期版本的LLVM可能可以使用，但未经过积极测试，因此不建议这样做。）
- 下载并安装Hexagon SDK和8.0版Hexagon Tools。
- 构建并运行Hexagon HVX的示例

### 1. Obtain and build trunk LLVM and Clang

（前面给出的说明，只需确保签出master分支即可。）

### 2. Download and install the Hexagon SDK and version 8.0 Hexagon Tools

- https://developer.qualcomm.com/software/hexagon-dsp-sdk/tools
- 选择Hexagon Series 600软件并下载Linux的3.0版本。
- 解压安装程序
- 运行解压缩的安装程序以安装Hexagon SDK和Hexagon Tools，选择将Hexagon SDK安装到`/location/of/SDK/Hexagon_SDK/3.0`，然后将Hexagon工具安装到`/location/of/SDK/Hexagon_Tools/8.0`。
- 设置SDK安装位置的变量
  ```
  export SDK_LOC=/location/of/SDK
  ```
### 3. Build and run an example for Hexagon HVX

除了在设备上运行Hexagon代码外，Halide还支持通过Hexagon工具在模拟器上运行Hexagon代码。

要在模拟器的`Halide/apps/blur`中构建并运行模糊示例：

```
cd apps/blur
export HL_HEXAGON_SIM_REMOTE=../../src/runtime/hexagon_remote/bin/v60/hexagon_sim_remote
export HL_HEXAGON_TOOLS=$SDK_LOC/Hexagon_Tools/8.0/Tools/
LD_LIBRARY_PATH=../../src/runtime/hexagon_remote/bin/host/:$HL_HEXAGON_TOOLS/lib/iss/:. HL_TARGET=host-hvx_128 make test
```

### To build and run the blur example in Halide/apps/blur on Android:

要构建适用于Android的示例，请首先确保您具有使用`make-standalone-toolchain.sh`脚本从NDK创建的独立工具链：

```
export ANDROID_NDK_HOME=$SDK_LOC/Hexagon_SDK/3.0/tools/android-ndk-r10d/
export ANDROID_ARM64_TOOLCHAIN=<path to put new arm64 toolchain>
$ANDROID_NDK_HOME/build/tools/make-standalone-toolchain.sh --arch=arm64 --platform=android-21 --install-dir=$ANDROID_ARM64_TOOLCHAIN
```

现在，使用脚本构建并运行blur示例以在设备上运行它：

```
export HL_HEXAGON_TOOLS=$SDK_LOC/HEXAGON_Tools/8.0/Tools/
HL_TARGET=arm-64-android-hvx_128 ./adb_run_on_device.sh
```
## 在进入Tutorial之前要做以上步骤

- https://halide-lang.org/tutorials/tutorial_introduction.html
- 参考 https://github.com/halide/Halide 编译配置LLVM与Halide，我的Halide build目录在 `dongkesi@2020:~/github/Halide/build/`