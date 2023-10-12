# Golang Notes

## 数据类型

### 布尔型

- `bool`

### 整型

- `int` / `int8` / `int16` / `int32` / `int64`
- `uint` / `uint8` / `uint16` / `uint32` / `uint64`
- `uintptr`

### 浮点型

- `float32` / `float64`

### 复数

- `complex64`
- `complex128`

### 字符

- `byte`: `int8` 的别名
- `rune`: `int32` 的别名

> 声明格式：`type new_type old_type`
> 等价于 C++ 中 `using new_type = old_type;`

### Unamed Type

- `array` 数组:
- `slice` 切片:
- `map` 字典:
- `pointer` 指针:
- `channel` 通道:
- `struct` 结构:

    为命名类型定义方法的语法格式：

    ```go
    // 类型方法接收者是值类型
    func (t TypeName) MethodName (ParamList) (ReturnList) {
        // method body
    }

    // 类型方法接收者是指针
    func (t *TypeName) MethodName (ParamList) (ReturnList) {
        // method body
    }
    ```

    - `t` 是接收者，可以自由指定名称
    - `TypeName` 为命令类型的类型名
    - `MethodName` 为方法名，是一个自定义标识符
    - `ParamList` 是形参列表
    - `ReturnList` 是返回值列表

    > **Go 语言的类型方法本质上是一个函数，没有使用隐式的指针: Go 的方法底层是基于函数实现的，只是语法格式不同，本质是一样的**

    将类型的方法改写为常规函数：

    ```go
    // 类型方法接收者是值类型
    func MethodName (t TypeName, ParamList) (ReturnList) {
        // method body
    }

    // 类型方法接收者是指针
    func MethodName (t *TypeName, ParamList) (ReturnList) {
        // method body
    }
    ```

    - **可以为命令类型增加方法（除了接口），非命名类型不能自定义方法**: 比如不能为 `[]int` 类型增加方法；命名接口类型本身就是一个方法的签名集合，故不能为其增加具体的实现方法
    - **为类型增加方法有一个限制，就是方法的定义必须和类型的定义在同一个包中**：
    - **方法的命令空间的可见性和变量一样，大写开头的方法可以在包外被访问，否则只能在包内可见**
    - **使用 `type` 定义的自定义类型是一个新类型，新类型不能调用原有类型的方法，但是支持底层类型支持的运算可以被新类型继承**

- `interface` 接口:
- `function` 函数:

## 循环语句

与其它主流编程语言不同的的是，Go 语言中的循环语句只支持 `for` 关键字，而不支持 `while` 和 `do-while` 结构

`for` 循环三要素：

```go
for init; conditoin; post {
    // ...
}
```

- `init`: init语句，一般为赋值表达式，给控制变量赋值
- `conditoin`: 条件判断，一般是关系表达式或逻辑表达式，循环控制条件
- `post`: 步进语句，一般为赋值表达式，控制变量增量或减量

退出循环:

- `break`: 跳出整个循环语句, 跳出所有循环次数
- `continue`: 跳出当次循环，即跳过当前的一次循环

## struct 结构体

在Go语言中，结构体承担着面向对象语言中类的作用

Go语言中，结构体本身仅用来定义属性

还可以通过接收器函数来定义方法，使用内嵌结构体来定义继承

### Declare

Go语言通过type和struct关键字声明结构体

```go
type StructName struct {
    val_1 val_1_type
    val_2 val_2_type
    val_3 val_3_type
    // ...
}
```

Go语言结构体（Struct）从本质上讲是一种自定义的数据类型，只不过这种数据类型比较复杂，是由 int、char、float 等基本类型组成的

### Instantiation

结构体的定义只是一种内存布局的描述，只有当结构体实例化时，才会真正地分配内存，因此必须在定义结构体并实例化后才能使用结构体的字段

实例化就是根据结构体定义的格式创建一份与格式一致的内存区域，结构体实例与实例间的内存是完全独立的

值类型是变量对应的地址直接存储值，而引用类型是变量对应地址存储的是地址

结构体因为是值类型，所以结构体实例 `p` 的地址与存储的第一个值的地址是相同的，而后面每一个成员变量的地址是连续的

- 方式一：先声明结构体，再赋值
- 方式二：`val := StructName{val_1: 12, val_2: "Lute", val_3: []string{"Chinese", "Math", "English"}}`
- 方式三：``
