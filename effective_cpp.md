# C++ 要点记录

[toc]

## 1. 智能指针

> 智能指针是要管理不能自动释放的内存，也就是堆上的内存，对于栈上的内存不要使用智能指针！会崩溃。

1. **unique_ptr**

   - 没有copy构造函数和copy赋值函数，只能`move`

   - 可以从函数返回一个`unique_ptr`，不必使用`move`

   - 其大小`sizeof`与一个指针相同，并没有额外开销

   - 数组,原生支持数组

     `std::unique_ptr<T[]> ptr(new T[3])` using constructor: 

     ```cpp
     template<class T, class Deleter>
     class unique_ptr<T[], Deleter>;
     ```

   - deleter

     - 构造：`auto deleter = [](int *p){ delete p; }; std::unique_ptr<int, decltype(deleter)> ptr`

     - 有状态的deleter（function onject）会增加unique_ptr的大小。用函数指针作为deleter也会增加一个指针的大小

     - deleter是它类型的一部分，定义一个T的unique_ptr时需要让编译器看到T的析构函数！**Pimpl**需要把特殊函数在源文件中定义

       ```cpp
       //  static_assert(sizeof(_Tp)>0,
       // 原因：Widget的析构函数必须在cpp文件中，否则就会使用编译器生成的内联析构函数，其中会调用unique_ptr的析构，但是这个时候还在头文件里，Impl还是个incomplete type
       
       /* widget.h */
       class Widget {
         public:
         	Widget();
         	~Widget() = default; // Error! Should implement it in cpp file
       
         private:
         	struct Impl;
         	std::unique_ptr<Impl> pImpl;
       };
       
       /* widget.cpp */
       struct Widget::Impl {
         int a;
       }
       
       Widget::Widget(): pImpl(new Impl()){}
       
       /* main.cpp */
       int main() {
         Widget w;
         return 0;
       }
       
       ```

       

   - 一个简单的实现

     ```cpp
     template<typename T>
     class unique_ptr {
      private:
       T* ptr_;
       
      public:
       unique_ptr(): ptr(nullptr) {}
       unique_ptr(T* ptr): ptr_(ptr) {}
       unique_ptr(const unique_ptr&) = delete;
       unique_ptr& operatpr=(const unique_ptr&) = delete;
       T* get() { return ptr_; }
       T* operator->() {return ptr_; }
     };
     ```

     

2. **shared_ptr**

   - 可以由`unique_ptr`构造

   - 内部包含两个指针，一个是类型的指针，另外一个指向`control_block`(有引用计数，若引用计数等信息)

   - 占用空间较大，可能会造成性能开销，由于控制块占用比较大的内存

   - 何时释放所占用的内存？

     - 当引用计数为零时释放T的内存，当弱引用计数都变成0才会释放控制块

   - 使用`make_shared`和`shared_ptr<T>(new T)`构造的区别？

     - `make_shared`会将control_block和T分配在连续空间上，释放也是一起释放
     - 裸指针构造会分开分配，分开释放。这里面就有性能的衡量了！

   - deleter

     - deleter不是`shared_ptr`类型的一部分，而是作为类成员存在的，所以自定义deleter会增加shared_ptr的大小，存在于控制块中

     - 自定义deleter，必须是一个函数对象（不是unique_ptr中的类型）

       ```cpp
       /** Construtor **/
       template< class Y, class Deleter >
       shared_ptr( Y* ptr, Deleter d );
       
       std::shared_ptr<Foo> sh(new Foo, [](auto p) {
                  std::cout << "Call delete from lambda...\n";
                  delete p;
               });
       ```

       

   - 数组的支持

     `std::shared_ptr<int[]> sh(new int[10], [](auto p) { delete[] p; });`

   - 实现

     ```c++
     // TODO
     
     ```

     

3. **weak_ptr**

   - 可以看成是`shared_ptr`的一个附属，不是**standalone**的类。不能解引用，不能判断是否为nullptr，
   - 必须由`shared_ptr`构造
   - 可以提升为shared_ptr: `wptr.lock()`
   - 可以用来判断shared_ptr是否还可用，`wptr.expire()`

4. **WHY 优先使用make_shared & make_unique**

   - 异常安全。例子：`func(shared_ptr<T>(new T()), helper())`
   - 效率问题。针对`shared_ptr`
   - make_xxx 也有**限制**，比如不能直接使用initialize_list构造对象；不能自定义deleter；如果对象特别大由于controlblock不会及时释放，可能会占用过多内存

5. **enabel_shared_from_this**

   - 应用场景：需要返回智能指针保护的`this`时使用。

   - How to use？ 对象必须是一个`shared_ptr`管理的堆上对象

     ```c++
     struct Widget : public std::enable_shared_from_this<Widget>{
       std::shared_ptr get_self() {
         return std::shared_from_this();
       }
     };
     
     // main
     std::shared_ptr<Widget> sp = make_shared<Widget>();
     auto self = sp->get_self(); // use_count of sp and self is 2
     
     ```



## 2. Lambda

1. lambda只会捕获可见的**non-static**局部变量，包括`this`，函数参数，函数内的局部变量

2. 具有`static duration storage`的变量一般定义在全局或者namspace中，这种变量不需要捕获，但是可以直接在lambda中使用

3. 避免default capture instead of specify vars explicitly

4. 如何`move` object 进行捕获，c++14才能支持`init capture`，c++11中只能使用`bind`

5. lambda与bind的区别

   - 占用空间

     - Lambda，没有捕获不占空间，有捕获就看捕获变量的大小

    - 何时计算参数

      - bind的参数是在编译期间计算出来的，而lambda内部调用的函数参数则是在调用实际发生时才计算。

        ```cpp
        // timer will be triggered at current_time + after
        
        // lambda
         // will be calculated when user call set_timer
        auto set_timer = [](int after) {
          alarm(steady_clock::now() + after);
        };
        
        // bind
        // will be calculated when compile!
        auto set_timer = bind(alarm, steady_clock::now() + after);
        ```

        

    - bind的函数有重载时，尤其是参数个数不同时。由于bind的只有一个函数名，参数作为内部实现类的成员，编译器不知道该名字具体是哪个重载函数。

      ```cpp
      // TODO:
      
      
      ```

   6. 综上所述，应该优先使用lambda。但是有时候bind有自己的优势：

   - C++11还不支持`move capture`，所以需要使用bind来实现
   - 多态函数。bind内部使用完美转发(std::forward)来传递参数，可以应对参数类型的重载。但是c++11的lambda却需要再定义时指定参数类型（C++14可以用auto）。

## 3. 类型推导

1. 模板参数推导

   我们使用下面的code来说明类型推导的规则,`ParamType`通常会带有CV和引用/指针等修饰。

   ```c++
   template<typename T>
   void f(ParamType param);
   
   // 调用
   f(expr)
   ```

   - ParamType是引用或指针

     - 忽略`expr`的引用/指针属性，然后再匹配ParamType来决定T

       ```c++
       f1(const T &param);
       f2(T &param);
       
       int x = 10;
       const int cx = x;
       const int &rx = x;
       
       
       ```

       

   - ParamType是universal reference `T&& param`

     - 如果expr是lvalue，paramtype是一个左值引用
     - 如果expr是rvalue，适用上面（paramtype是引用）的规则：忽略引用进而进行匹配

     ParamType不是指针或引用，形式是pass by value

     - 这种形式表示想要传递一份copy
     - 忽略expr的引用属性
     - 忽略cv属性

     ```c++
     
     ```

     expr是数组！

     - 如果是pass by value `f(T param)`, 推导成指针

     - 如果是pass by ref `f(T& param)`, 推导成数组！利用此规则可以写一个函数获取数组大小

       ```c++
       template<typename T, size_t N>
       constexpr size_t array_size(T (&)[N])
       {
         return N;
       }
       ```

   - expr是函数的情况与数组一致

   

2. auto 推导

   - 与template推导不同的是，如果用`{}`来初始化 `auto`声明的变量，推导成`initializer_list`，推导包括两步，一个是把{}推导成初始化列表，第二就是由于初始化列表是模板，还需要推导模板参数

   - C++14中，auto作为函数返回值或者lambda参数时，如果使用{}，会推导失败

   - auto对于不可见的proxy class不能预期工作

     `vector<bool>`

     

3. decltype

   - 总是会返回你认为的类型
   - 对于左值表达式`vector[index]`，decltype 返回T&类型

## 4. 并发编程
### 4.1 c++11提供的并发编程工具
Reference: https://baptiste-wicht.com/posts/2017/09/cpp11-concurrency-tutorial-futures.html
1. `std::lock_guard`和`std::unique_lock`的区别
   - 条件变量必须使用`std::unique_lock`
   - `std::unique_lock`拥有更过的API和构造函数，可以指定在构造时不lock，可以unlock等更丰富的功能。
3. 学会使用`recursive_mutex`, `timed_mutex`, `call_once`, `condition_variable` `atomic<T>`等组件，以及相应的API`try_lock`, `try_lock_for`等

### 4.2 Other Points
- 大多数情况下优先使用`std::async`而非`std::thread`，因为STL会帮你做一些调度优化，不会一味的创建新线程，当然你也可以使用launch policy让async总是创建新线程

- launch policy
  - std::launch::async :  立刻在新线程中执
  - std::launch::deferred：直到调用get/wait才会执行，不一定在新线程中
  - default设置是 OR，交给C++自己决定如何调度

- 何时可以使用默认的policy去执行一个task？
  - task不需要与调用`get` `wait`的线程并行
  - `wait_for`或者`wait_until`会检查defer状态
  - 一定会在`future`调用`get`或`wait`(其实就是让task执行)，或者不在乎task是否执行
  对于`wait_for`，它不会导致task被执行，所以如果不检查defer状态，可能导致bug：
  ```
  auto future = std::async(f);
  while (std::wait_for(100ms) != std::future_status::ready) // May nerver exit the loop
  {
   continue;
  }
  
  // TO FIX IT
  if (future.wait_for(0) == std::future_status::deferred)
  {
   future.wait(); // will let task go
  } else {
   while (std::wait_for(100) != std::future_status::ready) ...
  }
  ```
  
  - 调用一个joinable 线程的析构函数（内部会detach和join）将导致进程退出。你应该join 或者 detach这个线程, 如果不join直接调用结束就会崩溃
  - `std::future`工作原理：caller <---- shared states (heap) <---- callee
  - `std::future`的析构函数一般只会销毁自己的数据成员，如果它是是最后一个引用async non-deferred task 的shared states的future，它析构函数会阻塞直到线程结束

 

## 5. C++11特性

#### `{}`与`()`初始化

1. `{}`

   - 防止类型narrowing。`int a{1.0}; // WRONG`

   - 可以就地初始化非静态成员，注意不能使用`=`

   - 防止`most vexing parse`，也就是把`Widget w()` 解析成函数声明

   - 构造函数解析：

     - 如果构造 函数有使用`initialize_list`做参数的重载，当使用`{}`时，会优先使用该构造函数，除非实在是类型不匹配！

     - vector有多个构造函数，如何选择使用 `()`或`{}` ?

       ```c++
       vector<int> v1(10); // a vector with 10 elements, v1.size() == 10
       vector<int> v2{10}; // a vector with 1 elements, v1.size() == 1
       vector<int> v3(10, 20); // a vector with 10 elements with same value 20, v1.size() == 10
       vector<int> v4{10, 20}; // a vector with 2 elements, v1.size() == 2
       ```

2. Edge Case

   ```c++
   class Widget {
    public:
     Widget(){}
     Widget(std::initialize_list<int> l) {}
   };
   
   int main() {
     Widget w{}; // Will call default constructor
     Widget w({}); // will call initialize_list constructor
     Widget w{{}}; // will call initialize_list constructor
   }
   ```

   

#### nullptr

使用`0`或者`NULL`会在重载解析是遇到问题：永远不会使用void*的重载。

```c++
func(int);
func(void* );
```

#### `using` and `typedef`

- `typedef` 不能支持模板化，`using`可以
- 在模板type traits中通常要使用`typename T::type`，使用`using`则很直接

#### scopped enum

Both scoped and unstopped enum can specify underlying type

#### 一些新加的关键字

##### 1. deleted / default

关注`deleted` 与 `private`的区别：private函数需要实现！

所有的函数都可以`deleted`，即使不是成员函数

#####2. override

#####3. final

方便编译器优化

#####4. 成员函数后面的`&`, `&&`

##### 5. constexpr

- 是否在编译期间估值视情况而定，实在不行就当成普通函数
- C++11中，`constexpr`成员函数默认是`const`的，不能修改`this`
- C++11中，如果return `void`，也不可以是constexpr，因为void不是`litreal type`

##### 6. 编译器生成的函数

part-1：

- C++98 copy constructor 与copy assignment operator相互独立，定义其中一个不会影响编译器生成另外一个

- move construtor与moveassignment operator会相互影响

- 如果你声明了任何一个copy操作，编译器都不会生成move操作

- 如果你声明 move操作，编译器也不会生成copy操作

- Rule of Three

  part-2

- 编译器在三种情况下会生成move 操作

  - 没有自定义copy
  - 没有自定义move
  - 没有自定义析构函数

- 总结一下 C++11中的规则

  • Default constructor: Same rules as C++98. Generated only if the class contains
  no user-declared constructors.
  • Destructor: Essentially same rules as C++98; sole difference is that destructors
  are noexcept by default (see Item 14). As in C++98, virtual only if a base class
  destructor is virtual.
  • Copy constructor: Same runtime behavior as C++98: memberwise copy con‐
  struction of non-static data members. Generated only if the class lacks a userdeclared copy constructor. Deleted if the class declares a move operation.
  Generation of this function in a class with a user-declared copy assignment oper‐
  ator or destructor is deprecated.
  • Copy assignment operator: Same runtime behavior as C++98: memberwise
  copy assignment of non-static data members. Generated only if the class lacks a
  user-declared copy assignment operator. Deleted if the class declares a move
  operation. Generation of this function in a class with a user-declared copy con‐
  structor or destructor is deprecated

  • Move constructor and move assignment operator: Each performs memberwise
  moving of non-static data members. Generated only if the class contains no userdeclared copy operations, move operations, or destructor.

  • Member function templates never suppress generation of special member
  functions.

# 一些技巧

1. 如果无法避免copy，而且又可以move，在传递参数时可以传value。

   有一些函数你不确定传的是lvalue还是rvalue，但是又想前者copy后者move，这时候就可以`pass by value`，否则就要写两个重载方法

   ```c++
   class Widget {
    public:
     void AddName(std::string name) {
       name_ = std::move(name);
     }
   };
   
   Widget w;
   w.AddName("Jim"); // rvalue
   
   std::string name{"Green"};
   w.AddName(name); // lvalue
   ```

   **Consider pass by value only for copyable parameters.**

   **Consider pass by value only for parameters that are always copied.**

   **Pass by value only for parameters that are cheap to move**

2. 尽可能使用`emplace`而非`insert` ，就地构造
