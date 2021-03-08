# C++ 要点记录

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

     // TODO

     

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

   - 应用场景：需要返回智能指针管理的`this`时使用。注意只有堆上（使用shared_ptr管理）的对象才能使用`shared_from_this`。
   - How to use ?
     ```cpp
     class Base {
      public:
       shared_ptr<Base> get_self() {
         return shared_from_this();
       }
     };
     
     int main() {
       shared_ptr<Base> ptr(nwe Base()); // ptr use count is 1
       auto ptr2 = ptr->get_slef(); // ptr and ptr2 use count is both 2
       return 0;
     }
     ```


## 2. Lambda

1. lambda只会捕获可见的**non-static**局部变量，包括`this`，函数参数，函数内的局部变量

2. 具有`static duration storage`的变量一般定义在全局或者namspace中，这种变量不需要捕获，但是可以直接在lambda中使用

3. 避免default capture instead of specify vars explicitly

4. 如何`move` object 进行捕获，c++14支持

5. lambda与bind的区别

    

   - 占用空间

   - 何时计算参数

   - bind的函数有重载时

     ```cpp
     // TODO:
     
     ```

     

## 3. 类型推导



## 4. 并发编程
