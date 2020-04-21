# 定制new和delete

## Item 49：new handler的行为

new 申请内存失败时会抛出 `"bad alloc"` 异常，此前会调用一个由 std::set_new_handler() 指定的错误处理函数。

“new-handler” 函数通过 std::set_new_handler() 来设置，std::set_new_handler() 定义在 `<new>` 中：

```
namespace std{
    typedef void (*new_handler)();
    new_handler set_new_handler(new_handler p) throw();
}
```

throw() 是一个异常声明，表示不抛任何异常。例如：` void func() throw(Exception1, Exception2)`  表示func 可能会抛出 Exception1，Exception2 两种异常。

set_new_handler() 的使用也很简单：

```
void outOfMem(){
    std::cout<<"Unable to alloc memory";
    std::abort();
}
int main(){
    std::set_new_handler(outOfMem);
    int *p = new int[100000000L];
}
```

当new申请不到足够的内存时，它会不断地调用 outOfMem。因此一个良好设计的系统中 outOfMem 函数应该做如下几件事情之一：

- 使更多内存可用。
- 安装一个新的 ”new-handler”。
- 卸载当前 ”new-handler”，传递 null 给 set_new_handler 即可。
- 抛出 bad_alloc（或它的子类）异常。
- 不返回，可以 abort 或者 exit 。

std::set_new_handler 设置的是全局的 bad_alloc 的错误处理函数，C++ 并未提供类型相关的 bad_alloc 异常处理机制。 但我们可以重载类的 operator new，当创建对象时暂时设置全局的错误处理函数，结束后再恢复全局的错误处理函数。

比如 Widget 类，首先需要声明自己的 set_new_handler 和 operator new：

```
class Widget{
public:
    static std::new_handler set_new_handler(std::new_handler p) throw();
    static void * operator new(std::size_t size) throw(std::bad_alloc);
private:
    static std::new_handler current;
};
 
//@ 静态成员需要定义在类的外面
std::new_handler Widget::current = 0;
std::new_handler Widget::set_new_handler(std::new_handler p) throw(){
    std::new_handler old = current;
    current = p;
    return old;
}
```

关于 abort, exit, terminate 的区别：abort 会设置程序非正常退出，exit 会设置程序正常退出，当存在未处理异常时C++会调用 terminate， 它会回调由 std::set_terminate 设置的处理函数，默认会调用 abort。

最后来实现 operator new，该函数的工作分为三个步骤：

- 调用 std::set_new_handler，把 Widget::current 设置为全局的错误处理函数。
- 调用全局的 operator new 来分配真正的内存。
- 如果分配内存失败，Widget::current 将会抛出异常。
- 不管成功与否，都卸载 Widget::current，并安装调用 Widget::operator new 之前的全局错误处理函数。

我们通过RAII类来保证原有的全局错误处理函数能够恢复，让异常继续传播。

```
class NewHandlerHolder{
public:
    explicit NewHandlerHolder(std::new_handler nh): handler(nh){}
    ~NewHandlerHolder(){ std::set_new_handler(handler); }
private:
    std::new_handler handler;
    NewHandlerHolder(const HandlerHolder&);     //@ 禁用拷贝构造函数
    const NewHandlerHolder& operator=(const NewHandlerHolder&); //@ 禁用赋值运算符
};
```

然后 Widget::operator new 的实现其实非常简单：

```
void * Widget::operator new(std::size_t size) throw(std::bad_alloc){
    NewHandlerHolder h(std::set_new_handler(current));
    return ::operator new(size);    //@ 调用全局的new，抛出异常或者成功
}   //@ 函数调用结束，原有错误处理函数恢复
```

客户使用 Widget 的方式也符合基本数据类型的惯例： 

```
void outOfMem();
Widget::set_new_handler(outOfMem);
 
Widget *p1 = new Widget;    //@ 如果失败，将会调用outOfMem
//@ 如果失败，将会调用全局的 new-handling function，当然如果没有的话就没有了
string *ps = new string;   

Widget::set_new_handler(0); //@ 把Widget的异常处理函数设为空
Widget *p2 = new Widget;    //@ 如果失败，立即抛出异常
```

仔细观察上面的代码，很容易发现自定义 ”new-handler” 的逻辑其实和 Widget 是无关的。我们可以把这些逻辑抽取出来作为一个模板基类：

```
template<typename T>
class NewHandlerSupport{
public:
    static std::new_handler set_new_handler(std::new_handler p) throw();
    static void * operator new(std::size_t size) throw(std::bad_alloc);
private:
    static std::new_handler current;
};
 
template<typename T>
std::new_handler NewHandlerSupport<T>::current = 0;
 
template<typename T>
std::new_handler NewHandlerSupport<T>::set_new_handler(std::new_handler p) throw(){
    std::new_handler old = current;
    current = p;
    return old;
}
 
template<typename T>
void * NewHandlerSupport<T>::operator new(std::size_t size) throw(std::bad_alloc){
    NewHandlerHolder h(std::set_new_handler(current));
    return ::operator new(size);
}
```

有了这个模板基类后，给 Widget 添加 ”new-handler” 支持只需要 public 继承即可：

```
class Widget: public NewHandlerSupport<Widget>{ ... };
```

其实 NewHandlerSupport 的实现和模板参数 T 完全无关，添加模板参数是因为 handler 是静态成员，这样编译器才能为每个类型生成一个 handler 实例。

1993 年之前 C++ 的 operator new 在失败时会返回 null 而不是抛出异常。如今的 C++ 仍然支持这种 nothrow 的operator new：

```
Widget *p1 = new Widget;    // @失败时抛出 bad_alloc 异常
assert(p1 != 0);            //@ 这总是成立的
Widget *p2 = new (std::nothrow) Widget;
if(p2 == 0) ...             //@ 失败时 p2 == 0
```

“nothrow new”  只适用于内存分配错误。而构造函数也可以抛出的异常，这时它也不能保证是 new 语句是 ”nothrow” 的。

### 总结

- set_new_handler 允许你指定一个当内存分配请求不能被满足时可以被调用的函数。
- nothrow new 作用有限，因为它仅适用于内存分配，随后的 constructor 调用可能依然会抛出 exceptions。



## Item 50：了解new和delete所谓合理替换时机

为什么需要自定义 operator new 或 operator delete ?

- 检测使用错误。new 得到的内存如果没有 delete 会导致内存泄露，而多次 delete 又会引发未定义行为。如果自定义 operator new 来保存动态内存的地址列表，在 delete 中判断内存是否完整，便可以识别使用错误，避免程序崩溃的同时还可以记录这些错误使用的日志。
- 提高效率。全局的 new 和 delete 被设计为通用目的（general purpose）的使用方式，通过提供自定义的new，我们可以手动维护更适合应用场景的存储策略。
- 收集使用信息。在继续自定义 new 之前，你可能需要先自定义一个 new 来收集地址分配信息，比如动态内存块大小是怎样分布的？分配和回收是先进先出 FIFO 还是后进先出 LIFO？
- 实现非常规的行为。比如考虑到安全，operator new 把新申请的内存全部初始化为0。
- 其他原因，比如抵消平台相关的字节对齐，将相关的对象放在一起等等。

自定义一个 operator new 很容易的，比如实现一个支持越界检查的 new：

```
static const int signature = 0xDEADBEEF;    //@ 边界符
typedef unsigned char Byte; 
 
void* operator new(std::size_t size) throw(std::bad_alloc) {
    //@ 多申请一些内存来存放占位符 
    size_t realSize = size + 2 * sizeof(int); 
 
    //@ 申请内存
    void *pMem = malloc(realSize);
    if (!pMem) throw bad_alloc(); 
 
    //@ 写入边界符
    *(reinterpret_cast<int*>(static_cast<Byte*>(pMem)+realSize-sizeof(int))) 
        = *(static_cast<int*>(pMem)) = signature;
 
    //@ 返回真正的内存区域
    return static_cast<Byte*>(pMem) + sizeof(int);
}
```

其实上述代码是有一些瑕疵的：

- operator new 应当不断地调用new handler，上述代码中没有遵循这个惯例。
- 有些体系结构下，不同的类型被要求放在对应的内存位置。比如 double 的起始地址应当是 8 的整数倍，int的起始地址应当是 4 的整数倍。上述代码可能会引起运行时硬件错误。
- 起始地址对齐。C++要求动态内存的起始地址对所有类型都是字节对齐的，new 和 malloc 都遵循这一点，然而我们返回的地址偏移了一个int。 

到此为止你已经看到了，实现一个 operator new 很容易，但实现一个好的 operator new 却很难。

### 总结

- 有很多正当的编写 new 和 delete 的自定义版本的理由，包括改进性能，调试 heap（堆）用法错误，以及收集 heap（堆）用法信息。



## Item 51：编写new和delete时需固守常规

new 和delete 必须遵循的惯例：

- operator new 需要无限循环地获取资源，如果没能获取则调用 ”new handler”，不存在 ”new handler” 时应该抛出异常。 
- operator new 应该处理 size == 0 的情况。
- operator delete 应该兼容空指针。
- operator new/delete 作为成员函数应该处理 size > sizeof(Base) 的情况（因为继承的存在）。

#### 外部 operator new/delete

先看看如何实现一个外部（非成员函数）的 operator new： 

- 给出返回值很容易。当内存足够时，返回申请到的内存地址；当内存不足时，返回空或者抛出 bad_alloc 异常。
- 每次失败时调用 ”new handler”，并重复申请内存却不太容易。只有当 ”new handler” 为空时才应抛出异常。
- 申请大小为零时也应返回合法的指针。允许申请大小为零的空间确实会给编程带来方便。

考虑到上述目标，一个非成员函数的 operator new 大致实现如下：

```
void * operator new(std::size_t size) throw(std::bad_alloc){
    if(size == 0) size = 1;
    while(true){
        //@ 尝试申请
        void *p = malloc(size);
 
        //@ 申请成功
        if(p) return p;
 
        //@ 申请失败，获得new handler
        new_handler h = set_new_handler(0);
        set_new_handler(h);
 
        if(h) (*h)();
        else throw bad_alloc();
    }
}
```

- size == 0 时申请大小为 1 看起来不太合适，但它非常简单而且能正常工作。况且你不会经常申请大小为 0 的空间吧？
- 两次 set_new_handler 调用先把全局 ”new handler” 设置为空再设置回来，这是因为无法直接获取”new handler”，多线程环境下这里一定需要锁。
- while(true) 意味着这可能是一个死循环。所以Item 49提到，”new handler” 要么释放更多内存、要么安装一个新的  ”new handler”，如果你实现了一个无用的 ”new handler” 这里就是死循环了。

相比于new，实现delete的规则要简单很多。唯一需要注意的是 C++ 保证了delete 一个 NULL总是安全的，你尊重该惯例即可。

同样地，先实现一个外部（非成员）的delete：

```
void operator delete(void *rawMem) throw(){
    if(rawMem == 0) return; 
    // 释放内存
}
```

#### 成员 operator new/delete

重载 operator new 为成员函数通常是为了对某个特定的类进行动态内存管理的优化，而不是用来给它的子类用的。 因为在实现 Base::operator new() 时，是基于对象大小为 sizeof(Base) 来进行内存管理优化的。

当然，有些情况你写的 Base::operator new 是通用于整个class及其子类的，这时这一条规则不适用。 

```
class Base{
public:
    static void* operator new(std::size_t size) throw(std::bad_alloc);
};
class Derived: public Base{...};
 
Derived *p = new Derived;       // 调用了 Base::operator new ！
```

子类继承 Base::operator new() 之后，因为当前对象不再是假设的大小，该方法不再适合管理当前对象的内存了。 可以在 Base::operator new 中判断参数 size，当大小不为 sizeof(Base) 时调用全局的new：

```
void *Base::operator new(std::size_t size) throw(std::bad_alloc){
    if(size != sizeof(Base)) return ::operator new(size);
    ...
}
```

上面的代码没有检查 size == 0！这是 C++ 神奇的地方，大小为 0 的独立对象会被插入一个char。 所以sizeof(Base) 永远不会是0，所以 size == 0 的情况交给 ::operator new(size) 去处理了。

这里提一下 operator new[]，它和 operator new 具有同样的参数和返回值， 要注意的是你不要假设其中有几个对象，以及每个对象的大小是多少，所以不要操作这些还不存在的对象。因为：

- 你不知道对象大小是什么。上面也提到了当继承发生时 size 不一定等于sizeof(Base)。
- size 实参的值可能大于这些对象的大小之和。因为 Item 16 中提到，数组的大小可能也需要存储。

成员函数的 delete 也很简单，但要注意如果你的 new 转发了其他 size 的申请，那么 delete 也应该转发其他 size 的申请。

```
class Base{
public:
    static void * operator new(std::size_t size) throw(std::bad_alloc);
    static void operator delete(void *rawMem, std::size_t size) throw();
};
void Base::operator delete(void *rawMem, std::size_t size) throw(){
    if(rawMem == 0) return;     // 检查空指针
    if(size != sizeof(Base)){
        ::operator delete(rawMem);
    }
    // 释放内存
}
```

注意上面的检查的是 rawMem 为空，size 是不会为空的。

其实 size 实参的值是通过调用者的类型来推导的（如果没有虚析构函数的话）：

```
Base *p = new Derived;  // 假设Base::~Base不是虚函数
delete p;  // 传入`delete(void *rawMem, std::size_t size)`的`size == sizeof(Base)`。
```

如果 Base::~Base() 声明为 virtual，则上述 size 就是正确的 sizeof(Derived)。 这也是为什么Item 7 指出析构函数一定要声明 virtual。

### 总结

- operator new 应该包含一个设法分配内存的无限循环，如果它不能满足一个内存请求，应该调用 new-handler，还应该处理零字节请求。class-specific（类专用）版本应该处理对比预期更大的区块的请求。
- operator delete 如果收到一个空指针应该什么都不做。class-specific（类专用）版本应该处理比预期更大的区块。



## Item 52：写了placement new就要写placement delete

“placement new”  通常是专指指定了位置的 new(std::size_t size, void *mem)，用于 vector 申请 capacity 剩余的可用内存。 但广义的 ”placement new” 指的是拥有额外参数的 operator new。

new 和 delete 是要成对的，因为当构造函数抛出异常时用户无法得到对象指针，因而 delete 的责任在于 C++ 运行时。 运行时需要找到匹配的 delete 并进行调用。因此当我们编写了 ”placement new” 时，也应当编写对应的 ”placement delete”， 否则会引起内存泄露。在编写自定义 new 和 delete 时，还要避免不小心隐藏它们的正常版本。

当构造函数抛出异常时，C++ 会调用与 new 同样签名的 delete 来撤销 new。但如果我们没有声明对应的delete：

```
class Widget{
public:
    static void* operator new(std::size_t size, std::ostream& log) throw(std::bad_alloc);
    Widget(){ throw 1; }
};
 
Widget *p = new(std::cerr) Widget;
```

构造函数抛出了异常，C++ 运行时尝试调用 delete(void *mem, std::ostream& log)， 但Widget没有提供这样的delete，于是 C++ 不会调用任何 delete，这将导致内存泄露。 所以在 Widge t中需要声明同样签名的 delete：

```
static void operator delete(void *mem, std::ostream& log);
```

但客户还可能直接调用 delete p，这时 C++ 运行时不会把它解释为 ”placement delete”，这样的调用会使得Widget 抛出异常。 所以在 Widget 中不仅要声明 ”placement delete”，还要声明一个正常的 delete。

```
class Widget{
public:
    static void* operator new(std::size_t size, std::ostream& log) throw(std::bad_alloc);
    static void operator delete(void *mem, std::ostream& log);
    static void operator delete(void *mem) throw();
    Widget(){ throw 1; }
};
```

这样，无论是构造函数抛出异常，还是用户直接调用 delete p，内存都能正确地回收了。

类中的名称会隐藏外部的名称，子类的名称会隐藏父类的名称。 所以当你声明一个 ”placement new” 时：

```
class Base{
public:
    static void* operator new(std::size_t size, std::ostream& log) throw(std::bad_alloc);
};
Base *p = new Base;     //@ 错误
Base *p = new (std::cerr) Base;     //@ 正确
```

普通的 new 将会抛出异常，因为 ”placement new” 隐藏了外部的 ”normal new”。同样地，当你继承时：

```
class Derived: public Base{
public:
    static void* operator new(std::size_t size) throw(std::bad_alloc);
};
Derived *p = new (std::clog) Derived;       //@ 错误
Derived *p = new Derived;       //@ 正确
```

这是因为子类中的 ”normal new” 隐藏了父类中的 ”placement new”，虽然它们的函数签名不同。 但 Item 33 中提到，按照 C++ 的名称隐藏规则会隐藏所有同名（name）的东西，和签名无关。

为了避免全局的 ”new” 被隐藏，先来了解一下 C++ 提供的三种全局 ”new”：

```
void* operator new(std::size_t) throw(std::bad_alloc);     
void* operator new(std::size_t, void*) throw();             
void* operator new(std::size_t, const std::nothrow_t&) throw();     
```

为了避免隐藏这些全局 ”new”，你在创建自定义的 ”new” 时，也分别声明这些签名的 ”new” 并调用全局的版本。 为了方便，我们可以为这些全局版本的调用声明一个父类 StandardNewDeleteForms：

```
class StandardNewDeleteForms {
public:
  //@ normal new/delete
  static void* operator new(std::size_t size) throw(std::bad_alloc) { return ::operator new(size); }
  static void operator delete(void *pMemory) throw() { ::operator delete(pMemory); }
 
  //@ placement new/delete
  static void* operator new(std::size_t size, void *ptr) throw() { return ::operator new(size, ptr); }
  static void operator delete(void *pMemory, void *ptr) throw() { return ::operator delete(pMemory, ptr); }
 
  //@ nothrow new/delete
  static void* operator new(std::size_t size, const std::nothrow_t& nt) throw() { return ::operator new(size, nt); }
  static void operator delete(void *pMemory, const std::nothrow_t&) throw() { ::operator delete(pMemory); }
};
```

然后在用户类型 Widget 中 using StandardNewDeleteForms::new/delete 即可使得这些函数都可见：

```
class Widget: public StandardNewDeleteForms {           // inherit std forms
public:
   using StandardNewDeleteForms::operator new;         
   using StandardNewDeleteForms::operator delete;     
 	
   //@ 自定义 placement new
   static void* operator new(std::size_t size, std::ostream& log) throw(std::bad_alloc);    //@ 对应的 placement delete
   static void operator delete(void *pMemory, std::ostream& logStream) throw();           
};
```

### 总结

- 在编写一个 operator new 的 placement 版本时，确保同时编写 operator delete 的相应的 placement 版本。否则，你的程序可能会发生微妙的，断续的 memory leaks（内存泄漏）。
- 当你声明 new 和 delete 的 placement 版本时，确保不会无意中覆盖这些函数的常规版本。





