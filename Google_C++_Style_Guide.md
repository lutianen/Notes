# Google C++ 代码规范总结

![Google C++ style guide](https://raw.githubusercontent.com/lutianen/PicBed/main/202308262111101.png)

## `.h` 和 `.cc` 文件

- 使用 `#define` 保护文件，命名规则为 `PROJECT_PATH_FILE_H_`，如 `PROJECT_PATH_FILE_H_`

## 命名规则

- 通用规则：**命名具有描述性，尽量少用缩写**，例如 `DoSpecificThing()`
- 文件命名：**下划线连接具体描述性单词，全部小写**，`my_useful_class.cc`
- 类型命名：**采用 CamelCase**，例如 `FileTransfer`
- 变量命名：

  - 普通变量：`table_name`
  - 类数据成员：`class_member_name_`，结构体成员和普通变量一致
  - 常量：`const kTableName`
  - 函数：采用 `CamelCase`，例如 `validateFuncName()`
  - 枚举命名：

    ```c++
    enum CommandType {
        kSend2One = 0,
        kSend2More,
        kShowClients,
        kSetAlias,
    };
    ```

  - 宏命名：全部大写，单词间用下划线连接，例如 `MAX_NUM`
  - 命名空间：命名独特不冲突

## 命名空间 `namespace`

- 将函数包裹在命名空间中，**不使用无意思的类和静态成员**
- 不要修改约定俗称的 namespace，如 std

## 类 `class`

- 构造函数不要使用虚函数，及时抛出异常，不要有失败初始化这种不确定操作
- 拷贝构造函数使用 `explicit` 防止隐式类型转换
- 拷贝构造、赋值构造、移动构造、移动赋值：要么成套使用，要么都不使用；尤其注意浅拷贝
- 组合优于继承，has-a 比 is-a 更更灵活；
- 使用 `override`、`final`、`delete` 明确语义
- 对于纯方法的类，建议使用 `Interface` 作为后缀
- 只在合适的时候使用**运算符重载**，不要给通俗运算符赋予让人意外的操作

## 函数

- 函数参数顺序：输入参数在前，输入能用 `const` 就用 `const`，输出参数在后
- 引用参数最好都加上 `const`；如需修改参数，使用指针更直白
- 函数如果超过 40 行，就应该考虑拆分
- 函数默认参数和重载需要清晰

## 类型转换

使用 C++ 的类型转换，不要使用 C 的类型转换

- `static_cast`：**编译时转换**，如 `int` 转 `double`
- `const_cast`：**去除 `const`**，如 `const int*` 转 `int*`
- `reinterpret_cast`：指针类型和整型或其他指针之间进行**不安全的转换**
- `dynamic_cast`：**运行时转换**，用于**多态**，如 `Base*` 转 `Derived*，不推荐使用

## 流 stream

**只在记录日志时使用流，不要在业务代码中使用流**

使用 `printf` 和 `scanf` 代替 `cout` 和 `cin`

## 整形

C++ 内建整型中，仅使用 `int`；在程序中如果需要不同大小的变量，可以使用`<cstdint>` 中长度精确的整形，如`int16_t`、`uint32_t`等，代替 `short`、`long` 等

使用断言来指出变量为非负数, 而不是使用无符号型

## 64 位下的可移植性

```cpp
// printf macros for size_t, in the style of inttypes.h
#ifdef _LP64
#define __PRIS_PREFIX "z"
#else
#define __PRIS_PREFIX
#endif

// Use these macros after a % in a printf format string
// to get correct 32/64 bit behavior, like this:
// size_t size = records.size();
// printf("%"PRIuS"\n", size);
#define PRIdS __PRIS_PREFIX "d"
#define PRIxS __PRIS_PREFIX "x"
#define PRIuS __PRIS_PREFIX "u"
#define PRIXS __PRIS_PREFIX "X"
#define PRIoS __PRIS_PREFIX "o"
```

| 类型 | 使用 | 不要使用 | 备注 |
| --- | --- | --- | --- |
| `void*` / `指针类型` | `%p` | `%lx` |
| `int64_t` | `%"PRId64"` | `%qd` / `%lld` |
| `uint64_t` | `%"PRIu64"` / `PRIx64` | |
| `size_t` | `%"PRIuS"` / `%"PRIxs"` | C99 规定 `%zu` |
| `ptrdiff_t` | `%"PRIdS"` | `%d` | C99 规定 `%zd`

注意 `PRI*` 宏会被编译器扩展为独立字符串. 因此如果使用非常量的格式化字符串, 需要将宏的值而不是宏名插入格式中.

使用 `PRI*` 宏同样可以在 % 后包含长度指示符.

`printf("x = %30"PRIuS"\n", x)` 在 32 位 Linux 上将被展开为 `printf("x = %30" "u" "\n", x)`, 编译器当成 `printf("x = %30u\n", x)` 处理

在 64 位系统中, 任何含有 `int64_t` / `uint64_t` 成员的类/结构体, 缺省都以 8 字节在结尾对齐. 如果 32 位和 64 位代码要共用持久化的结构体, 需要确保两种体系结构下的结构体对齐一致. 大多数编译器都允许调整结构体对齐. gcc 中可使用 `__attribute__((packed))`.

## 预处理宏

使用宏时要非常谨慎, 尽量以内联函数, 枚举和常量代替之.

宏意味着你和编译器看到的代码是不同的. 这可能会导致异常行为, 尤其因为宏具有全局作用域.

宏可以做一些其他技术无法实现的事情, 在一些代码库 (尤其是底层库中) 可以看到宏的某些特性 (如用 `#` 字符串化, 用 `##` 连接等等).

- 不要在 `.h` 文件中定义宏.
- 在马上要使用时才进行 `#define`, 使用后要立即 `#undef`.
- 不要只是对已经存在的宏使用`#undef`，选择一个不会冲突的名称；
- 不要试图使用展开后会导致 C++ 构造不稳定的宏, 不然也至少要附上文档说明其行为.
- 不要用 `##` 处理函数，类和变量的名字。

## 注释

**应该始终遵循自注释代码，简洁易懂，不要写无意义的注释**

- 文件开头要有版权注释（为了开源和负责）
- 类注释：让读者知道这个类大致干了啥
- TODO：
- 启用注释：// DEPRECATED

## 格式

- 一行 80 个字符，最长不超过 120 字符
- 正常缩进使用 2 个字符
- 函数返回类型和函数名在同一行，形参可以换行 w
- 函数大括号和类大括号都另起一行

## TIPS

- 静态变量和全局变量要慎用，尤其是多线程环境下：**静态变量和全局变量的初始化顺序是不确定的**
- 采用 Modern C++

  - `auto`：自动类型推导
  - `nullptr`：空指针
  - `for (const auto& item : items)`：遍历
  - `std::unique_ptr`：智能指针
  - `std::move`：移动语义

- 指针的传递代表了所有权的转移，引用的传递代表了共享所有权
- RTTI（运行时类型识别）：`dynamic_cast<**>` 能不用就不用
