# 让自己习惯C++

## Item 1: 视C++为一个语言联邦

C++ 是多范式的程序设计语言。同时支持：过程式编程，面向对象编程，函数式编程，泛型编程，元编程。
### C++ 的四种次语言

- **C 语言**。C++ 是基于 C 设计的，你可以只使用 C++ 中 C 的那部分语法。此时你会发现你的程序反映的完全是C的特征：没有模板、没有异常、没有重载。
- **Object-Oriented C++**。面向对象程序设计也是 C++ 的设计初衷：构造与析构、封装与继承、多态、动态绑定的虚函数。
- **Template C++**。这是 C++ 的泛型编程部分。另外模板元编程也是一个新兴的程序设计范式，虽然有点非主流。
- **STL**。这是一个特殊的模板库，它的容器、迭代器和算法优雅地结合在一起，只是在使用时你需要遵循它的程序设计惯例。当然你也可以基于其他想法来构建模板库。

### 总结

- C++ 并非单一的一门语言，它有很多不同的规则集。
- C++ 程序设计的惯例并非一尘不变，而是取决于你使用 C++ 语言的哪一部分。



## Item 2: 尽量使用const、enum、inline等替换#define

### const 常量代替 #define

#### 普通常量

```
#define ASPECT_RATIO 1.653     		//@ 不推荐
const double AspectRatio = 1.653;	//@ 推荐		 
```

宏定义的名字不会被添加到符号表中，给调试带来很大的麻烦。而 const 常量会被编译器明确识别并确实加入符号表。

另外宏定义会造成代码中出现多份拷贝。而 const 常量就只有一份拷贝。

#### 类属常量

为了将一个常量的作用范围限制在一个类内，你必须将它作为一个类的成员，而且为了确保它最多只有一个常量拷贝，你还必须把它声明为一个静态成员。

```
class GamePlayer {
private:
  static const int NumTurns = 5;      //@ 声明一个类属常量
  int scores[NumTurns];               //@ 使用常量
  ...
};
```

上面 NumTurns 是 declaration（声明），而不是 definition（定义），可以提供一个独立的定义：

```
const int GamePlayer::NumTurns;
```

- 应该把它放在一个实现文件而非头文件中。
- 因为类属常量的初始值在声明时已经提供初值，在此无须重复给初值
- 定义时不需要 static 关键字。

#define 不考虑作用范围：一旦一个宏被定义，它将大范围影响你的代码（除非在后面某处存在 #undefine），因此无法使用 #define 来创建一个类属常量。

### the enum hack

上面的 NumTurns 编译器会为它分配内存，可以获取它的地址。

通过 enum 可以避免内存分配，也就无法获得其地址，这一点与 #define 是一致的。

```
class GamePlayer {
private:
  enum { NumTurns = 5 };                           
  int scores[NumTurns];             
  ...
};
```

### inline 函数代替 #define

```
//@ call f with the maximum of a and b
#define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))
```

编写这样的宏一定要带上括号，但是有时候即使带上括号，也无法避免错误：

```
int a = 5, b = 0;
CALL_WITH_MAX(++a, b); 		//@ a 自增了两次
CALL_WITH_MAX(++a, b+10); 	//@ a 自增了一次
```

可以通过一个内联函数的模板来获得宏的效率，以及完全可预测的行为和常规函数的类型安全：

```
template<typename T>    
inline void callWithMax(const T& a, const T& b) 
{                                       
	f(a > b ? a : b);                        
}
```

### 总结

- 对于简单常量，使用 const 代替宏定义，其优点：
  - const 常量能够出现在符号表中，方便调试。
  - 宏定义因为进行的宏替换，有时候会造成代码冗余，const 常量能够很好的避免这个问题。
  - const 常量可以作为类属成员，#define 则毫无封装性。
  - 整型族类属常量可以在类中声明时直接初始化。
- enum 也可以作为整型常量使用，并且无法取得其地址。
- 使用内联函数代替宏定义的函数将会在不损失效率的情况下降低发生错误的可能性。

## Item 3: 尽量使用 const

### const 与指针

```
char greeting[] = "Hello";
char* p = greeting;	//@ non-const data,non-const pointer
const char* p = greeting;	//@ non-const pointer,const data
char* const p = greeting;	//@ const pointer,non-const data
const char* const p = greeting; //@ const pointer,const data
```

- const 出现在 `*` 左边，则指针指向的内容是 const。
- const 出现在 `*` 右边，则指针本身是 const。
- const 出现在  `*` 两边，两者都是 const。

当指针指向的内容是常量时，将 const 放在类型前和放在类型后是没有区别的：

```
//@ 等价的形式
void f1(const Widget *pw);	
void f1(Widget const *pw);	
```

#### 变与不变的思考

当指针指向的内容是常量时，表示无法通过指针修改变量的值，但是可以通过其它方式修改指针指向变量的值：

```
int a = 1;
const int *p = &a;
cout << *p << endl;	//@ 1
*p = 2;	//@ error, data is const
a = 2;
cout << *p << endl;	//@ 2
```

指针本身是常量，表示指针表示的地址是固定的，但是其指向的内容是可以改变的：

```
int a = 1, b = 2;
int* const p = &a;
cout << *p << endl;	//@ 1
p = &b;	//@ error, pointer is const
*p = b;
cout << *p << endl;	//@ 2
```

### const 与迭代器

**const iterator：**表示 iterator 本身是常量，不能将这个 iterator 指向另外一件不同的东西，但是它所指向的东西本身可以变化。

**conts_iterator：**表示  iterator 指向的内容不能发生变化，但是其本身可以变化。

```
std::vector<int> vec;

const std::vector<int>::iterator iter = vec.begin();
*iter = 10;	//@ 正确，迭代器指向的内容可以变化
iter++;		//@ 错误，迭代器本身是常量不可以变化

std::vector<int>::const_iterator cIter =  vec.begin();
*cIter = 10;	//@ 错误，迭代器指向的内容是常量不可以变化
++cIter;		//@ 正确，迭代器本身可以变化
```

### const 与函数

const 可以用在函数返回值，函数参数，对于成员函数，还可以用于整个函数。

#### 函数返回 const value

函数返回 const value 常常可以在不放弃安全和效率的前提下尽可能减少客户造成的影响：

```
class Rational{...};
const Rational operator*(const Rational& lhs,const Rational& rhs);
```

上面函数返回 const object 的原因，可以避免客户如下暴行：

```
Rational a,b,c;
...
(a * b) = c;	//@ 为两个数的乘积赋值，将返回值声明为 const 可以避免此问题
```

#### 函数参数是 const

无论何时，只要可以就应该将函数的参数声明为 const 类型，除非需要改变这个参数。

#### const 成员函数

const 成员函数的优点：

- 明确哪些方法可以供常量对象调用。
- 使用接口是否会改变对象的意图显而易见。

const 成员函数的注意事项：

- 常量对象只能调用常量方法， 非常量对象优先调用非常量方法，如不存在会调用同名常量方法。

- 常量成员函数也可以在类声明外定义，但声明和定义都需要指定 const 关键字。

- 成员方法添加常量限定符属于函数重载。

```
class TextBlock {
public:
  ...  
  //@ operator[] for const objects
  const char& operator[](std::size_t position) const  
  { return text[position]; }                          

  //@ operator[] for non-const objects
  char& operator[](std::size_t position)           
  { return text[position]; }                          

private:
   std::string text;
};

//@ 使用
TextBlock tb("Hello");
std::cout << tb[0];	//@ 调用 non-const TextBlock::operator[]
                                       
const TextBlock ctb("World");
std::cout << ctb[0];  //@ 调用 const TextBlock::operator[]
```

const objects 常常作为函数的参数传递：

```
void print(const TextBlock& ctb)      
{
  std::cout << ctb[0];   //@ 调用 const TextBlock::operator[]
  ...
}
```
对 const 和 non-const 的 TextBlocks 做不同的操作：
```
std::cout << tb[0];   
tb[0] = 'x';         //@ 因为返回类型是非常量引用，此处正确
std::cout << ctb[0]; 
ctb[0] = 'x';       //@ 错误，返回的常量对象不允许被修改
```

再请注意 non-const 版本的 operator[] 的返回类型是一个 char 的引用而不是一个 char 本身。如果 operator[] 只是返回一个简单的 char，下面的语句将无法编译：

```
tb[0] = 'x';	//@ 相当于给右值赋值
```

### 比特常量和逻辑常量

编译器强制实行 **bitwise constness** (又称 physical constness,物理上的常量性，即成员函数不更改对象的任何一个 bit 时才可以说是 const)，例如: 

```
class CTextBlock {
public:
  ...
  char& operator[](std::size_t position) const   
  { return pText[position]; }                   
                                               
private:
  char *pText;
};
```

 编译器认定它是 bitwise constness 的,但是它却允许以下代码的存在:

```
const CTextBlock cctb("Hello");  //@ 声明有一个常量对象
char *pc = &cctb[0];
*pc = 'J'; //@ cctb 当前的值是 "Jello"
```

这是由于只有 pText 是 cctb 的一部分，其指向的内存并不属于 cctb。

编写程序时应该使用 **conceptual constness** (概念上的常量性或 logical constness,逻辑上的常量性）即一个const 成员函数可以处理它所修改的对象的某些 bits，但只有在客户端侦测不出的情况下才得如此。

例如对于某些特殊类，其中的某些成员的值注定是要改变的，因此可以用 mutable 关键字修饰，从而实现即使对象被设定为 const，其特定成员的值仍然可以改变的效果。此时该类符合 conceptual constness 而不符合 bitwise constness。

```
class CTextBlock {
public:
  ...
  std::size_t length() const;
private:
  char *pText;

  mutable std::size_t textLength;        
  mutable bool lengthIsValid;             
};    

std::size_t CTextBlock::length() const
{
  if (!lengthIsValid) {
    textLength = std::strlen(pText);     
    lengthIsValid = true;        
  }

  return textLength;
}
```

### 避免常量/非常量方法的重复

通常我们需要定义成对的常量和普通方法，只是返回值的修改权限不同。 当然我们不希望重新编写方法的逻辑。最先想到的方法是常量方法调用普通方法，然而这是 C++ 语法不允许的。 于是我们只能用普通方法调用常量方法，并做相应的类型转换：

```
class TextBlock {
public:
  ...
  const char& operator[](std::size_t position) const 
  {
    ...
    return text[position];
  }

  char& operator[](std::size_t position) 
  {
    return const_cast<char&>(
        static_cast<const TextBlock&>(*this)[position]);
  }
...
};
```

- `*this` 的类型是 TextBlock ，先把它强制隐式转换为 const TextBlock，这样才能调用常量方法。
- 调用 `operator[](size_t) const` ，得到的返回值类型为 const char&。
- 把返回值去掉 const 属性，得到类型为 char& 的返回值。

### 总结

- const 可被施加于任何作用域内的对象，函数参数，函数返回类型，成员函数本身。

- const 与指针：
  - const 在前表示指针指向的内容是常量。
  - 指针在前表示指针本身是常量。
- const 与迭代器：
  -  const 修饰迭代器时表示迭代器本身是常量。
  - 迭代器指向的内容是常量时应该使用 const_iterator。
- const 与函数
  - 函数返回值为 const 可以避免一些意外赋值的情况发生。
  - 尽可能的将函数参数声明为 const。
  - 常量对象只能调用常量方法， 非常量对象优先调用非常量方法，如不存在会调用同名常量方法。
  - 如果一个方法不改变对象的任何非静态变量，那么该方法是常量方法。
  - mutable 限定符对于即使是 const 的对象也可以做修改。
  - 可以利用 const 成员函数实现等价的非 const 成员函数，以避免书写更多的重复代码。

## Item 4: 确保对象在使用前被初始化
### 对象初始化方法

对于内建类型的非成员对象，初始化手动执行：

```
int x = 0; 
const char * text = "A C-style string";  
double d;
std::cin >> d; 
```

除此之外的几乎全部情况，初始化都要依靠构造函数。所以确保所有的构造函数都初始化了对象中的每一样东西。

注意区分初始化和赋值：

```
class PhoneNumber { ... };
class ABEntry {   
public:
  ABEntry(const std::string& name, const std::string& address,
          const std::list<PhoneNumber>& phones);
private:
  std::string theName;
  std::string theAddress;
  std::list<PhoneNumber> thePhones;
  int num TimesConsulted;
};

//@ 构造函数
ABEntry::ABEntry(const std::string& name, const std::string& address,
                 const std::list<PhoneNumber>& phones)

{
  //@ 下面执行的是赋值不是初始化
  theName = name;                       
  theAddress = address;                
  thePhones = phones;
  numTimesConsulted = 0;
}
```

C++ 的规则规定一个对象的数据成员在进入构造函数的函数体之前被初始化：

- 在 ABEntry 的构造函数内，theName，theAddress 和 thePhones 不是被初始化，而是被赋值。
- 在进入 ABEntry 的构造函数的函数体之前，theName，theAddress 和 thePhones 的缺省的构造函数已经被自动调用。然而很快又在缺省构造的值之上赋予新值，那些缺省构造函数所做的工作被浪费了。
- numTimesConsulted 是一个内建类型，不能保证它在被赋值之前被初始化。

一个更好的写 ABEntry 构造函数的方法是用成员初始化列表来代替赋值：

```
ABEntry::ABEntry(const std::string& name, const std::string& address,
                 const std::list<PhoneNumber>& phones)
: theName(name),
  theAddress(address),                  //@ 执行初始化
  thePhones(phones),
  numTimesConsulted(0)
{}   //@ 构造函数体为空
```

初始化列表中的参数就可以作为各种数据成员的构造函数所使用的参数。在这种情况下，theName 从 name 中 copy-constructed（拷贝构造），theAddress 从 address 中 copy-constructed（拷贝构造），thePhones 从 phones 中 copy-constructed（拷贝构造）。对于大多数类型来说，只调用一次拷贝构造函数的效率比先调用一次缺省构造函数再调用一次拷贝赋值运算符的效率要高（有时会高很多）。

对于 numTimesConsulted 这样的内建类型的对象，初始化和赋值没有什么不同，但为了统一性，最好是经由成员初始化来初始化每一件东西。

类似地，‘当你只想缺省构造一个数据成员时也可以使用成员初始化列表，只是不必指定初始化参数而已。例如，如果 ABEntry 有一个不取得参数的构造函数，它可以像这样实现：

```
ABEntry::ABEntry()
:theName(),                         //@ 调用 theName 的默认构造函数
 theAddress(),                      //@ 调用 theAddress 的默认构造函数
 thePhones(),                       //@ 调用 thePhones 的默认构造函数
 numTimesConsulted(0)               //@ 显式初始化
{}      
```

在初始化列表中总是列出每一个数据成员，这就可以避免一旦发生疏漏就必须回忆起可能是哪一个数据成员没有被初始化。因为 numTimesConsulted 是一个内建类型，如果将它从成员初始化列表中删除，就为未定义行为打开了方便之门。 

有时候即使是内建类型，初始化列表也必须使用：比如，const 或 references data members 是必须被初始化的，它们不能被赋值。

C++ 对象的数据被初始化的顺序总是相同的：

- 基类在派生类之前被初始化。
- 在一个类内部，数据成员按照它们被声明的顺序被初始化。

例如，在 ABEntry 中，theName 总是首先被初始化，theAddress 是第二个，thePhones 第三，numTimesConsulted 最后。即使它们在成员初始化列表中以一种不同的顺序排列，这依然是成立的。为了避免读者混淆，以及一些模糊不清的行为引起错误的可能性，初始化列表中的成员的排列顺序应该总是与它们在类中被声明的顺序保持一致。

### 跨转换单元的静态对象初始化问题

#### 静态对象和转换单元

一个静态对象的生存期是从它创建开始直到程序结束。程序结束时静态对象会自动销毁，也就是当 main 停止执行时会自动调用它们的析构函数。

静态对象按照定义的位置可以分为：

- 在函数内部的静态对象称为 **局部静态对象**。
- 全局对象、定义在命名空间范围内的对象、在类内部声明为静态的对象、在文件范围内被声明为静态的对象称为 **非局部静态对象**。

一个**转换单元**是可以形成一个单独的目标文件的源代码。基本上是一个单独的源文件，再加上它全部的 #include 文件。

#### 跨转换单元的初始化问题

```
class FileSystem { 
public:
  ...
  std::size_t numDisks() const;  
  ...
};
extern FileSystem tfs;  
```

现在假设一些客户为一个文件系统中的目录创建了一个类，他们的类使用了对象：

```
class Directory { 
public:
   Directory(params);
  ...
};

Directory::Directory(params)
{
  ...
  std::size_t disks = tfs.numDisks();   //@ 使用 tfs 对象
  ...
}
```

更进一步，假设这个客户决定为临时文件创建一个单独的对象：

```
Directory tempDir(params); 
```

现在初始化顺序的重要性变得明显了：除非 tfs 在 tempDir 之前初始化，否则，tempDir 的构造函数就会在 tfs 被初始化之前试图使用它。但是，tfs 和 tempDir 是被不同的人于不同的时间在不同的源文件中创建的——它们是定义在不同转换单元中的非局部静态对象。因此，无法确定它们的初始化顺序。

正确的做法是将每一个非局部静态对象移到它自己的函数中，在那里它被声明为静态。这些函数返回它所包含的 对象的引用。换一种说法，就是用局部静态对象取代非局部静态对象。

```
class FileSystem { ... };          

FileSystem& tfs()                   
{                                  
  static FileSystem fs;          
  return fs;                      
}

class Directory { ... };           

Directory::Directory( params )     
{                                
  ...
  std::size_t disks = tfs().numDisks();
  ...
}

Directory& tempDir()             
{                                  
  static Directory td;              
  return td;                      
}
```

任何种类的非常量静态对象——局部的或非局部的，在多线程存在的场合都会发生麻烦。解决这个麻烦的方法之一是在程序的单线程的启动部分手动调用所有的返回引用的函数。以此来避免与初始化相关的混乱环境。

### 总结

- 对象初始化
  - 手动初始化内建类型的对象，因为 C++ 只在某些时候才会自己初始化它们。
  - C++ 的规则规定一个对象的数据成员在进入构造函数的函数体之前被初始化。
  - 列表初始化通常比在构造函数中赋值效率更高。
  - 在构造函数中，用成员初始化列表代替函数体中的赋值初始化列表中数据成员的排列顺序要与它们在类中被声明的顺序相同。
  - 列表初始化时要初始化每一个成员，防止遗漏。
  - 类中的 const 成员和引用成员必须使用初始化列表初始化。
- 静态对象
  - 定义在不同转换单元内的非局部静态对象的初始化的相对顺序是未定义的。
  - 通过用局部静态对象代替非局部静态对象来避免跨转换单元的初始化顺序问题。

