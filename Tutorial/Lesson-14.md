# lesson 14: The Halide type system

## Build & Run

```bash
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ g++ lesson_14*.cpp -g -I ../include -L ../bin -lHalide -lpthread -ldl -o lesson_14 -std=c++11
dongkesi@2020:~/github/Halide/build/distrib/tutorial$ LD_LIBRARY_PATH=../bin ./lesson_14
```

## 代码分析

```c
#include "Halide.h"
#include <stdio.h>
using namespace Halide;

// This function is used to demonstrate generic code at the end of
// this lesson.
Expr average(Expr a, Expr b);

int main(int argc, char **argv) {
```
所有`Exprs`都具有标量类型，并且所有`Funcs`均对一个或多个标量类型求值。Halide中的标量类型是**各种位宽的无符号整数**，相同位宽集的有符号整数，单精度和双精度的浮点数以及不透明的句柄（等效于void *）。以下数组包含所有合法类型。
```c
    // All Exprs have a scalar type, and all Funcs evaluate to one or
    // more scalar types. The scalar types in Halide are unsigned
    // integers of various bit widths, signed integers of the same set
    // of bit widths, floating point numbers in single and double
    // precision, and opaque handles (equivalent to void *). The
    // following array contains all the legal types.

    Type valid_halide_types[] = {
        UInt(8), UInt(16), UInt(32), UInt(64),
        Int(8), Int(16), Int(32), Int(64),
        Float(32), Float(64), Handle()
    };
```
您可以以编程方式检查Halide类型的属性。当您编写具有`Expr`参数的C++函数并希望检查其类型时，这很有用：
```c
    // Constructing and inspecting types.
    {
        // You can programmatically examine the properties of a Halide
        // type. This is useful when you write a C++ function that has
        // Expr arguments and you wish to check their types:
        assert(UInt(8).bits() == 8);
        assert(Int(8).is_int());
```
您还可以通过编程方式将`Types`构造为其他`Types`的函数。
```c
        // You can also programmatically construct Types as a function of other Types.
        Type t = UInt(8);
        t = t.with_bits(t.bits() * 2);
        assert(t == UInt(16));
```
或从C++标量类型构造一个Type
```c
        // Or construct a Type from a C++ scalar type
        assert(type_of<float>() == Float(32));
```
Type结构体也可以表示**向量类型**，但这保留给Halide内部使用。您应该使用`Func::vectorize`对代码进行向量化，而不是尝试直接构造向量表达式。如果以编程方式处理降低的Halide代码，则可能会遇到向量类型，但这是高级主题（请参见`Func::add_custom_lowering_pass`）。

您可以查询任何Halide `Expr`的类型。表示`Var`的`Expr`具有`Int(32)`类型：
```c
        // The Type struct is also capable of representing vector types,
        // but this is reserved for Halide's internal use. You should
        // vectorize code by using Func::vectorize, not by attempting to
        // construct vector expressions directly. You may encounter vector
        // types if you programmatically manipulate lowered Halide code,
        // but this is an advanced topic (see Func::add_custom_lowering_pass).

        // You can query any Halide Expr for its type. An Expr
        // representing a Var has type Int(32):
        Var x;
        assert(Expr(x).type() == Int(32));
```
Halide中大多数先验函数将其输入转换为Float(32)并返回Float(32)：
```c
        // Most transcendental functions in Halide cast their inputs to a
        // Float(32) and return a Float(32):
        assert(sin(x).type() == Float(32));
```
您可以使用强制转换运算符将Expr从一种类型强制转换为另一种类型：
```c
        // You can cast an Expr from one Type to another using the cast operator:
        assert(cast(UInt(8), x).type() == UInt(8));
```
这也以采用C++类型的模板形式出现。
```c
        // This also comes in a template form that takes a C++ type.
        assert(cast<uint8_t>(x).type() == UInt(8));
```
您还可以查询任何已定义的Func产生的类型。
```c
        // You can also query any defined Func for the types it produces.
        Func f1;
        f1(x) = cast<uint8_t>(x);
        assert(f1.output_types()[0] == UInt(8));

        Func f2;
        f2(x) = {x, sin(x)};
        assert(f2.output_types()[0] == Int(32) &&
               f2.output_types()[1] == Float(32));
    }
```
当您组合不同类型的`Exprs`（例如使用'+'，'*'等）时，Halide使用类型提升规则系统。这些与C的规则不同。

为了演示这些，我们将为每种类型制作一些`Exprs`。
```c
    // Type promotion rules.
    {
        // When you combine Exprs of different types (e.g. using '+',
        // '*', etc), Halide uses a system of type promotion
        // rules. These differ to C's rules. To demonstrate these
        // we'll make some Exprs of each type.
        Var x;
        Expr u8 = cast<uint8_t>(x);
        Expr u16 = cast<uint16_t>(x);
        Expr u32 = cast<uint32_t>(x);
        Expr u64 = cast<uint64_t>(x);
        Expr s8 = cast<int8_t>(x);
        Expr s16 = cast<int16_t>(x);
        Expr s32 = cast<int32_t>(x);
        Expr s64 = cast<int64_t>(x);
        Expr f32 = cast<float>(x);
        Expr f64 = cast<double>(x);
```
规则如下，并按照下面的顺序应用。

1. 在Handle()类型的`Exprs`上强制转换或使用算术运算符是错误的。
```c
        // The rules are as follows, and are applied in the order they are
        // written below.

        // 1) It is an error to cast or use arithmetic operators on Exprs of type Handle().
```
2. 如果类型相同，则不会发生类型转换。
```c
        // 2) If the types are the same, then no type conversions occur.
        for (Type t : valid_halide_types) {
            // Skip the handle type.
            if (t.is_handle()) continue;
            Expr e = cast(t, x);
            assert((e + e).type() == e.type());
        }
```
3. 如果一种类型是浮点型，而另一种则不是，则将非浮点型参数提升为浮点型（可能会导致大整数精度下降）。
```c
        // 3) If one type is a float but the other is not, then the
        // non-float argument is promoted to a float (possibly causing a
        // loss of precision for large integers).
        assert((u8 + f32).type() == Float(32));
        assert((f32 + s64).type() == Float(32));
        assert((u16 + f64).type() == Float(64));
        assert((f64 + s32).type() == Float(64));
```
4. 如果两种类型均为浮点型，则将较窄的参数提升为较宽的位宽。
```c
        // 4) If both types are float, then the narrower argument is
        // promoted to the wider bit-width.
        assert((f64 + f32).type() == Float(64));
```
上面的规则处理所有浮点情况。以下三个规则处理整数情况。

5. 如果参数之一是C++ `int`，另一个参数是`Halide::Expr`，则将`int`强制转换为表达式的类型。
```c
        // The rules above handle all the floating-point cases. The
        // following three rules handle the integer cases.

        // 5) If one of the arguments is an C++ int, and the other is
        // a Halide::Expr, then the int is coerced to the type of the
        // expression.
        assert((u32 + 3).type() == UInt(32));
        assert((3 + s16).type() == Int(16));
```
如果此规则将导致整数溢出，则Halide将触发错误，例如，取消注释以下行将导致该程序终止并出现错误。
```c
  // Expr bad = u8 + 257;  
```
6. 如果两种类型都是无符号整数，或者两种类型都是带符号整数，则将窄型参数提升为宽型。
```c
        // If this rule would cause the integer to overflow, then Halide
        // will trigger an error, e.g. uncommenting the following line
        // will cause this program to terminate with an error.
        // Expr bad = u8 + 257;

        // 6) If both types are unsigned integers, or both types are
        // signed integers, then the narrower argument is promoted to
        // wider type.
        assert((u32 + u8).type() == UInt(32));
        assert((s16 + s64).type() == Int(64));
```
7. 如果一种类型是带符号的，而另一种类型是无符号的，则两个参数都将被提升为两个宽度较大的带符号的整数。
```c
        // 7) If one type is signed and the other is unsigned, both
        // arguments are promoted to a signed integer with the greater of
        // the two bit widths.
        assert((u8 + s32).type() == Int(32));
        assert((u32 + s8).type() == Int(32));
```
请注意，在位宽度相同的情况下，这可能会静默地溢出无符号类型。
```c
        // Note that this may silently overflow the unsigned type in the
        // case where the bit widths are the same.
        assert((u32 + s32).type() == Int(32));
```
通过这种方式将无符号`Expr`转换为较宽的带符号类型时，首先将其**扩展为较宽的无符号类型**（零扩展），然后将其重新解释为有符号整数。即，将`UInt(8)`值`255`强制转换为`Int(32)`会产生`255`，而不是`-1`。
```c
        // When an unsigned Expr is converted to a wider signed type in
        // this way, it is first widened to a wider unsigned type
        // (zero-extended), and then reinterpreted as a signed
        // integer. I.e. casting the UInt(8) value 255 to an Int(32)
        // produces 255, not -1.
        int32_t result32 = evaluate<int>(cast<int32_t>(cast<uint8_t>(255)));
        assert(result32 == 255);
```
当使用强制转换运算符将有符号类型显式转换为较宽的无符号类型时（类型提升规则将永远不会自动执行此操作），首先将其转换为较宽的有符号类型（对符号进行扩展），然后将其重新解释为无符号整数 。即，将`Int(8)`值`-1`转换为`UInt(16)`会产生`65535`，而不是`255`。
```c
        // When a signed type is explicitly converted to a wider unsigned
        // type with the cast operator (the type promotion rules will
        // never do this automatically), it is first converted to the
        // wider signed type (sign-extended), and then reinterpreted as
        // an unsigned integer. I.e. casting the Int(8) value -1 to a
        // UInt(16) produces 65535, not 255.
        uint16_t result16 = evaluate<uint16_t>(cast<uint16_t>(cast<int8_t>(-1)));
        assert(result16 == 65535);
    }
```
`Handle`用于表示不透明的指针。将`type_of`应用于任何指针类型将返回`Handle()`
```
    // The type Handle().
    {
        // Handle is used to represent opaque pointers. Applying
        // type_of to any pointer type will return Handle()
        assert(type_of<void *>() == Handle());
        assert(type_of<const char * const **>() == Handle());
```
不管编译目标如何，`Handle`始终以64位存储。
```c
        // Handles are always stored as 64-bit, regardless of the compilation
        // target.
        assert(Handle().bits() == 64);

        // The main use of an Expr of type Handle is to pass
        // it through Halide to other external code.
    }
```
Halide中`Type`的主要显式用途是编写由`Type`参数化的Halide代码。在C++中，您可以使用模板执行此操作。在Halide中不需要-您可以在C++运行时动态检查和修改类型。下面定义的函数对任何相等数值类型的两个表达式求平均值。
```c
    // Generic code.
    {
        // The main explicit use of Type in Halide is to write Halide
        // code parameterized by a Type. In C++ you'd do this with
        // templates. In Halide there's no need - you can inspect and
        // modify the types dynamically at C++ runtime instead. The
        // function defined below averages two expressions of any
        // equal numeric type.
        Var x;
        assert(average(cast<float>(x), 3.0f).type() == Float(32));
        assert(average(x, 3).type() == Int(32));
        assert(average(cast<uint8_t>(x), cast<uint8_t>(3)).type() == UInt(8));
    }

    printf("Success!\n");

    return 0;
}
```

```c
Expr average(Expr a, Expr b) {
    // Types must match.
    assert(a.type() == b.type());

    // For floating point types:
    if (a.type().is_float()) {
        // The '2' will be promoted to the floating point type due to
        // rule 3 above.
        return (a + b)/2;
    }

    // For integer types, we must compute the intermediate value in a
    // wider type to avoid overflow.
    Type narrow = a.type();
    Type wider = narrow.with_bits(narrow.bits() * 2);
    a = cast(wider, a);
    b = cast(wider, b);
    return cast(narrow, (a + b)/2);
}
```