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
