# Note for Effective Modern C++

## 条款30：熟悉完美转发的失败情形

- 补充：
1. 关于位域
   - 位域（`Bit Field`）是 C/C++ 中的一种特性，它允许在一个结构体或联合体中定义**一组连续的位**，这些位可以用于存储多个小的整数值；
   - 位域通常用于优化内存使用，特别是在嵌入式系统或需要精确控制内存布局的场景中；
   - **位域的内存布局是编译器定义的**，不同的编译器可能会有不同的实现（如从最低有效位向最高有效位，编译器通常会提供某种机制以控制位域的内存布局）；
   - 语法：
    ```C++
    struct {
        type member_name : width;
    } struct_name;
    ```
    其中，`type` 是成员变量的数据类型，通常是整数类型（如 `int, unsigned int...`）；`member_name` 即成员变量的名称；`width` 是成员变量占用的位数；eg:
    ```C++
    struct BitField{
        unsigned int a : 1; // 1 位
        unsigned int b : 3; // 3 位
        unsigned int c : 4; // 4 位
    }
    ```
    需要注意的是，如果赋值超过位域范围，会造成未定义行为。例如赋值： `struct BitField bf; bf.a = 2`（ `a` 只有 1 位，只能存储 0 或 1）；
    （上述 `BitField` 占用 8 个位，即 1 个字节；当未显示指定成员变量占用位数时，每个成员变量将占用其数据类型所占用的位数，如 `int` 占 4 字节，32 位。）
    （编译器可能会对结构体进行对齐和填充，以优化访问速度。因此，实际的结构体大小可能与理论计算的大小不同。）
   - **在硬件层次，引用和指针是同一回事。于是，由于无法创建指涉到任意比特的指针（C++ 硬性规定，可以指涉的最小实体是单个 `char`），很自然地，也无法把引用绑定到任意比特了**。
2. 函数名、函数指针
   ```C++
    template<typename T>
    void fwd(T&& param){
    f(std::forward<T>(param));
    }
    ...
    void f(int (*pf)(int));
    ...
    int processVal(int value);
    int processVal(int value, int priority);
    ...
    f(processVal);      // 没问题
    fwd(processVal);    // 错误！哪个 processVal 重载版本？
   ```
   关于 `f(processVal);` 在书中指出：“processVal 既非函数指针，甚至连函数都是不是”：
   - 首先，`processVal` 是一个函数名（两个不同函数的名字），即函数标识符。它代表了所有可能的重载版本；
   - **函数名在某些上下文中可以被隐式转换为指向该函数的指针**（如上述代码中的 `f(processVal)`），但这种转换并不是总是发生的，特别是在模板和函数重载的上下文中；
   - 当调用 `fwd(processVal)` 时，`processVal` 是一个函数名，但它并没有被隐式转换为函数指针。相反，`processVal` 被视为一个函数标识符，它代表了所有可能的重载版本。**编译器需要在模板实例化时确定正确的重载版本**（eg，显式转换：`fwd(static_cast<int(*)(int)>(processVal));`）;
3. 仅有声明的整形 `static const` 成员变量
   ```C++
   class Widget{
    public:
        static const std::size_t MinVals = 28;  // 给出了 MinVals 的声明
        ...
   }
    ...         // 未给出 MinVals 的声明

    std::vector<int> widgetData;
    widgetData.reserve(Widget::MinVals);    // 此处用到了 MinVals
   ```
   在上述代码中，书中指出："在这里，尽管 `Widget::MinVals`（以下简称 `MinVals`） 并无定义，我们还是利用了 `MinVals` 来指定 `widgetData` 的初始容量。**编译器绕过了 `MinVals` 缺少定义的事实（编译器的行为是这样规定的），手法是把值 28 塞到所有提及 `MinVals` 之处**。未为 `MinVals` 的值保留存储这一事实并不会带来问题。如果产生了对 `MinVals` 实施取址的需求（例如，有人创建了一个指涉到 `MinVals` 的指针），`MinVals` 就得要求存储方可（因此指针才能够指涉到它），然后上面这段代码虽然仍能够通过编译，但是如果不为 `MinVals` 提供定义，它在链接期就会遭遇失败。"
   - 是声明还是定义：
     1. 补充见 `cppFundation`；
     2. 静态常量成员变量：
        ```C++
        class Widget{
        public:
            static const std::size_t MinVals = 28;  // 声明 
        }
        ```
        声明：`static const std::size_t MinVals = 28;` 告诉编译器 `MinVals` 是一个静态常量成员变量，不分配存储空间，其值为 28（这既是声明也是定义。但是，这种声明方式在**类内**初始化时，**只在编译时使用其值，而不会为该变量分配存储空间。因此，它主要被视为声明**）；
        定义：如果需要在运行时访问 `MinVals` 的地址，必须在类外提供定义：
        ```C++
        const std::size_t Widget::MinVals;  // 定义
        // 注意，定义语句没有重复指定初始化物（即 28）。如果忘记了这一点，在两处（即类内和类外）都提供了初始化物，编译器会发出控诉，从而提醒只在一处指定即可（此所谓唯一定义规则，one define rule）。
        ```
     3. 静态成员变量：
        ```C++
        class Widget{
        public:
            static int b = 10;  // 声明和定义
        }
        ```
        对于普通的静态成员变量，如上所述，情况有所不同。在类内初始化静态成员变量时，这既是声明也是定义，会分配存储空间（同其他非类内成员变量，即普通变量，见 1 所指补充内容）。**对于静态成员变量，通常建议在类外提供定义，以避免在多个翻译单元中重复定义的问题**。
        ```C++
        // .h 文件
        class Widget{
        public:
            static int b;       // 声明
        }

        // .cpp 文件
        int Widget::b = 10;     // 定义
        ```
   - 如果在链接阶段需要访问 `MinVals` 的地址（例如，创建一个指向 MinVals 的指针），那么 `MinVals` 必须有定义（即分配了存储空间）。如果没有定义，链接器会报错。