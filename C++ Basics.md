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
- 
