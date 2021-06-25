## 类型推导
- `auto` 不能用作函数形参； 可以推到数组类型
- 尾返回类型
  ```
  // C++ 11
  template<typename T, typename U>
  auto add(T x, U y) -> decltype(x+y) {
    return x+y;
  }
  
  // C++ 14
  auto add(T x, U y) { return x + y;}
  ```
 ## 初始化列表
 - 如果使用copy 构造，不能用 `{}`

## 模板
- 外部模板
  C++98 要在每个编译单元中对模板进行实例化，现在可以用 `external` 来修饰模板，只需要实例化一次即可？
- 模板别名 `using` 而不用 `typename`
- 默认模板参数 `template<typename T = int> print(T a) {}`
- 变长模板参数
  - 可以用sizeof获取参数个数
  - 解包：递归模板函数，必须定义一个终止递归的函数
    ```
    template<typename T>
    void print(T value) { cout << value << endl; }
    
    template<typename T, typename... Args>
    void print(T value, Args... args) {
      cout << value << endl;
      print(args...);
    }
    ````
- 解包：初始化列表展开 C++14
  ```
  template<typename T, typename... Args> auto print(T value, Args... args) {
    std::cout << value << std::endl;
    return std::initializer_list<T>{([&] {
        std::cout << args << std::endl;
    }(), value)...};
   }
   ```

## 面向对象特性
- 委托构造
- 继承构造，可以使用`using`来继承Base的构造函数
- override:`void func() override`
- final: `void func() final; &&. class A final : public Base{}`
- delete / default: `Contructot() = delete/default`

## 强类型枚举
`enum class E : int {}`

## lambda
- 表达式捕获C++14: 允许捕获的变量用任意表达式进行初始化
- C++14允许lambda的形参用`auto`修饰

## std::function, std::bind, std::placeholders

## std::forward 与万能引用

## STL
- std::array<int, 4>
- std::forward_list:  单向列表 没有size()
- unordered_xxx
- std::tuple: `make_tuple`, `std::get<>`, `std::tie`拆包. C++14中还可以使用`get`来根据类型获取元组中的对象 `std::get<int>(t)`
  ```
  std:::tie(a, b, c) = get_tuple();
  auto t = make_tuple(1,2,3);
  std::get<0>(t) == 1;
  std::get<1>(t) == 2;
  std::get<2>(t) == 3;
  ```
- 元组的合并与遍历
  - `new = std::tuple_cat(t1, t2)`
  - 遍历，需要知道元组长度 `std::tuple_size<T>::value`

- 智能指针
- regx
- 并发编程
 
### 原始字符串字面量
`string s = R"hell\\you";`
- 可以自定义字面量，重载"", `std::string operator"" _w(const char *w, size_t len) { return string(w) + ",";}`
