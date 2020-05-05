# lesson 11: Cross-compilation

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
    std::vector<Argument> args(2);
    args[0] = input;
    args[1] = offset;

    brighter(x, y) = input(x, y) + offset;

    brighter.vectorize(x, 16).parallel(y);

    brighter.compile_to_file("lesson_11_host", args, "brighter");

    Target target;
    target.os = Target::Android; // The operating system
    target.arch = Target::ARM;   // The CPU architecture
    target.bits = 32;            // The bit-width of the architecture
    std::vector<Target::Feature> arm_features; // A list of features to set
    target.set_features(arm_features);
    brighter.compile_to_file("lesson_11_arm_32_android", args, "brighter", target);

    target.os = Target::Windows;
    target.arch = Target::X86;
    target.bits = 64;
    std::vector<Target::Feature> x86_features;
    x86_features.push_back(Target::AVX);
    x86_features.push_back(Target::SSE41);
    target.set_features(x86_features);
    brighter.compile_to_file("lesson_11_x86_64_windows", args, "brighter", target);

    target.os = Target::IOS;
    target.arch = Target::ARM;
    target.bits = 32;
    std::vector<Target::Feature> armv7s_features;
    armv7s_features.push_back(Target::ARMv7s);
    target.set_features(armv7s_features);
    brighter.compile_to_file("lesson_11_arm_32_ios", args, "brighter", target);

    uint8_t arm_32_android_magic[] = {0x7f, 'E', 'L', 'F', // ELF format
                                      1,       // 32-bit
                                      1,       // 2's complement little-endian
                                      1};      // Current version of elf

    FILE *f = fopen("lesson_11_arm_32_android.o", "rb");
    uint8_t header[32];
    if (!f || fread(header, 32, 1, f) != 1) {
        printf("Object file not generated\n");
        return -1;
    }
    fclose(f);

    if (memcmp(header, arm_32_android_magic, sizeof(arm_32_android_magic))) {
        printf("Unexpected header bytes in 32-bit arm object file.\n");
        return -1;
    }

    uint8_t win_64_magic[] = {0x64, 0x86};

    f = fopen("lesson_11_x86_64_windows.obj", "rb");
    if (!f || fread(header, 32, 1, f) != 1) {
        printf("Object file not generated\n");
        return -1;
    }
    fclose(f);

    if (memcmp(header, win_64_magic, sizeof(win_64_magic))) {
        printf("Unexpected header bytes in 64-bit windows object file.\n");
        return -1;
    }

    uint32_t arm_32_ios_magic[] = {0xfeedface, // Mach-o magic bytes
                                   12,  // CPU type is ARM
                                   11,  // CPU subtype is ARMv7s
                                   1};  // It's a relocatable object file.
    f = fopen("lesson_11_arm_32_ios.o", "rb");
    if (!f || fread(header, 32, 1, f) != 1) {
        printf("Object file not generated\n");
        return -1;
    }
    fclose(f);

    if (memcmp(header, arm_32_ios_magic, sizeof(arm_32_ios_magic))) {
        printf("Unexpected header bytes in 32-bit arm ios object file.\n");
        return -1;
    }

    printf("Success!\n");
    return 0;
}
```
## Build & Run
```bash
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ g++ lesson_11*.cpp -g -std=c++11 -I ../include -L ../bin -lHalide -lpthread -ldl -o lesson_11
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ LD_LIBRARY_PATH=../bin ./lesson_11
```
生成了以下文件
```bash
  92 -rw-rw-r-- 1 dongkesi dongkesi   91632 Apr  3 15:21 lesson_11_arm_32_ios.h
 120 -rw-rw-r-- 1 dongkesi dongkesi  118824 Apr  3 15:21 lesson_11_arm_32_ios.o
  92 -rw-rw-r-- 1 dongkesi dongkesi   91651 Apr  3 15:21 lesson_11_x86_64_windows.h
 108 -rw-rw-r-- 1 dongkesi dongkesi  109802 Apr  3 15:21 lesson_11_x86_64_windows.obj
  92 -rw-rw-r-- 1 dongkesi dongkesi   91640 Apr  3 15:21 lesson_11_arm_32_android.h
 156 -rw-rw-r-- 1 dongkesi dongkesi  157084 Apr  3 15:21 lesson_11_arm_32_android.o
  92 -rw-rw-r-- 1 dongkesi dongkesi   91620 Apr  3 15:21 lesson_11_host.h
 152 -rw-rw-r-- 1 dongkesi dongkesi  154848 Apr  3 15:21 lesson_11_host.o
 788 -rwxrwxr-x 1 dongkesi dongkesi  804904 Apr  3 15:21 lesson_11
```

```bash
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ file lesson_11*
lesson_11_arm_32_android.o:      ELF 32-bit LSB relocatable, ARM, EABI5 version 1 (SYSV), not stripped
lesson_11_arm_32_ios.o:          Mach-O arm subarchitecture=11 object
lesson_11_host.o:                ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
lesson_11_x86_64_windows.obj:    data
```
## 代码分析

```c
#include "Halide.h"
#include <stdio.h>
using namespace Halide;

int main(int argc, char **argv) {
```
我们将定义在第10课中使用的简单的单阶段管道。
```c
    // We'll define the simple one-stage pipeline that we used in lesson 10.
    Func brighter;
    Var x, y;
```
声明参数
```c
    // Declare the arguments.
    Param<uint8_t> offset;
    ImageParam input(type_of<uint8_t>(), 2);
    std::vector<Argument> args(2);
    args[0] = input;
    args[1] = offset;
```
定义Func，并调度
```c
    // Define the Func.
    brighter(x, y) = input(x, y) + offset;

    // Schedule it.
    brighter.vectorize(x, 16).parallel(y);
```
下面这一行是我们在第十课中所做的。它编译一个适合于运行此程序的系统的对象文件。例如，如果使用`sse4.1`在x86 CPU上的64位linux上编译并运行此文件，则生成的代码将**适用于使用sse4.1的x86上的64位linux**。
```c
    // The following line is what we did in lesson 10. It compiles an
    // object file suitable for the system that you're running this
    // program on.  For example, if you compile and run this file on
    // 64-bit linux on an x86 cpu with sse4.1, then the generated code
    // will be suitable for 64-bit linux on x86 with sse4.1.
    brighter.compile_to_file("lesson_11_host", args, "brighter");
```
我们还可以编译适用于其他CPU和操作系统的对象文件。您可以使用第三个可选参数来编译指定要编译的目标`compile_to_file`。

让我们用这个来编译这个代码的32位arm android版本：
```c
    // We can also compile object files suitable for other cpus and
    // operating systems. You do this with an optional third argument
    // to compile_to_file which specifies the target to compile for.

    // Let's use this to compile a 32-bit arm android version of this code:
    Target target;
    target.os = Target::Android; // The operating system
    target.arch = Target::ARM;   // The CPU architecture
    target.bits = 32;            // The bit-width of the architecture
    std::vector<Target::Feature> arm_features; // A list of features to set
    target.set_features(arm_features);
    // We then pass the target as the last argument to compile_to_file.
    brighter.compile_to_file("lesson_11_arm_32_android", args, "brighter", target);
```
现在是64位x86的Windows对象文件，带有AVX和SSE 4.1：
```c
    // And now a Windows object file for 64-bit x86 with AVX and SSE 4.1:
    target.os = Target::Windows;
    target.arch = Target::X86;
    target.bits = 64;
    std::vector<Target::Feature> x86_features;
    x86_features.push_back(Target::AVX);
    x86_features.push_back(Target::SSE41);
    target.set_features(x86_features);
    brighter.compile_to_file("lesson_11_x86_64_windows", args, "brighter", target);
```
最后是一个苹果32位ARM处理器A6的iOS mach-o对象文件。它用在iPhone 5上——A6。使用了一个稍微修改过的ARM架构，叫做`ARMv7s`，我们使用`TargetFeatures`字段来指定它。在`llvm`中，对苹果64位ARM处理器的支持是非常新的，而且还有些不稳定。
```
    // And finally an iOS mach-o object file for one of Apple's 32-bit
    // ARM processors - the A6. It's used in the iPhone 5. The A6 uses
    // a slightly modified ARM architecture called ARMv7s. We specify
    // this using the target features field.  Support for Apple's
    // 64-bit ARM processors is very new in llvm, and still somewhat
    // flaky.
    target.os = Target::IOS;
    target.arch = Target::ARM;
    target.bits = 32;
    std::vector<Target::Feature> armv7s_features;
    armv7s_features.push_back(Target::ARMv7s);
    target.set_features(armv7s_features);
    brighter.compile_to_file("lesson_11_arm_32_ios", args, "brighter", target);
```
现在让我们通过检查文件的**前几个字节**来检查这些文件是否是它们声称的文件。
```c
    // Now let's check these files are what they claim, by examining
    // their first few bytes.
```
32个arm android对象文件以magic字节开头
```c
    // 32-arm android object files start with the magic bytes:
    uint8_t arm_32_android_magic[] = {0x7f, 'E', 'L', 'F', // ELF format
                                      1,       // 32-bit
                                      1,       // 2's complement little-endian
                                      1};      // Current version of elf

    FILE *f = fopen("lesson_11_arm_32_android.o", "rb");
    uint8_t header[32];
    if (!f || fread(header, 32, 1, f) != 1) {
        printf("Object file not generated\n");
        return -1;
    }
    fclose(f);

    if (memcmp(header, arm_32_android_magic, sizeof(arm_32_android_magic))) {
        printf("Unexpected header bytes in 32-bit arm object file.\n");
        return -1;
    }
```
64位windows对象文件以16位值`0x8664`开始
```c
    // 64-bit windows object files start with the magic 16-bit value 0x8664
    // (presumably referring to x86-64)
    uint8_t win_64_magic[] = {0x64, 0x86};

    f = fopen("lesson_11_x86_64_windows.obj", "rb");
    if (!f || fread(header, 32, 1, f) != 1) {
        printf("Object file not generated\n");
        return -1;
    }
    fclose(f);

    if (memcmp(header, win_64_magic, sizeof(win_64_magic))) {
        printf("Unexpected header bytes in 64-bit windows object file.\n");
        return -1;
    }
```
32位arm iOS mach-o文件以以下Magic字节开头
```c
    // 32-bit arm iOS mach-o files start with the following magic bytes:
    uint32_t arm_32_ios_magic[] = {0xfeedface, // Mach-o magic bytes
                                   12,  // CPU type is ARM
                                   11,  // CPU subtype is ARMv7s
                                   1};  // It's a relocatable object file.
    f = fopen("lesson_11_arm_32_ios.o", "rb");
    if (!f || fread(header, 32, 1, f) != 1) {
        printf("Object file not generated\n");
        return -1;
    }
    fclose(f);

    if (memcmp(header, arm_32_ios_magic, sizeof(arm_32_ios_magic))) {
        printf("Unexpected header bytes in 32-bit arm ios object file.\n");
        return -1;
    }
```
看起来我们生成的目标文件对这些目标来说是合理的。在本教程中，我们将认为这是一个成功。对于一个真正的应用程序，您**需要找出如何将Halide集成到交叉编译工具链中**。在`apps`文件夹下的Halide存储库中有几个小例子。请参见此处的`HelloAndroid`和`helloiOS`: https://github.com/halide/Halide/tree/master/apps/
```c
    // It looks like the object files we produced are plausible for
    // those targets. We'll count that as a success for the purposes
    // of this tutorial. For a real application you'd then need to
    // figure out how to integrate Halide into your cross-compilation
    // toolchain. There are several small examples of this in the
    // Halide repository under the apps folder. See HelloAndroid and
    // HelloiOS here:
    // https://github.com/halide/Halide/tree/master/apps/
    printf("Success!\n");
    return 0;
}
```
