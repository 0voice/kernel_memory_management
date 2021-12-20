内存管理是主要通过运算符`new`, `new[]`, `delete`和`delete[]`来实现。按照C++标准`new/delete`和`new[]/delete[]`并不是C++中的函数，而是C++中的关键字。之所以有这样的区别，C++中在对象创建时需要自动执行构造函数，而在对象销毁时需要自动执行对象的析构函数。而`malloc()/free()`则是库函数而非运算符，不在编译器控制权限之内，无法把执行构造函数和析构函数的任务强加于`malloc()/free()`。

## 内存分配方式

在C++中，内存被分为栈、堆、全局/静态存储区、（文字）常量存储区和程序代码区5个分区。

- 栈：在执行函数时，函数内局部变量的存储单元都可以在栈上创建，函数执行结束时这些存储单元自动被释放。栈内存分配运算内置于处理器的指令集中，效率很高，但是分配的内存容量有限。
- 堆：就是那些由`new`分配的内存块，而由`delete`来释放。编译器不会参与内存块的释放，需要程序员开发的的应用程序去自行控制，一般一个`new`就要对应一个`delete`。
- 全局/静态存储区：全局变量和静态变量被分配到同一块内存中，在以前的C语言中，全局变量又分为初始化的和未初始化的，在C++里面没有这个区分了，他们共同占用同一块内存区。
- 常量存储区：这是一块比较特殊的存储区，里面存放的是常量如常量字符串，不允许修改。程序结束后由系统释放。
- 程序代码区：存放函数体的二进制代码

### 堆与栈的区别

- 管理方式
  栈由编译器自动管理，堆则是需要程序员控制，容易出现内存泄漏
- 空间大小
  栈以数据结构中的栈实现，由高地址向低地址扩展。在存储中是一块**连续的存储区域**。也就是说栈顶地址和栈的最大容量是在**程序启动时就由系统预先规定好**。所以栈的空间一般较小。当用栈过多时可导致栈溢出（无穷次（大量的）的递归调用，或者大量的内存分配）。而堆则是由低地址向高地址扩展，在内存中是**不连续的内存区域**。系统通过链表来存储空闲的内存地址。堆的大小受计算机中有效虚拟内存的限制，因此能通过堆获得较大的存储空间。
- 分配方式
  堆都是动态分配的。**栈有静态和动态两种分配方式**，其中静态分配由编译器完成比如局部变量的分配。动态分配由`alloca()`函数进行分配，但是栈的动态分配和堆是不同的，他的动态分配是由编译器进行释放，无需我们手工实现。
- 分配效率
  栈是机器系统提供的数据结构，计算机会在底层对栈提供支持：分配专门的寄存器存放栈的地址，压栈出栈都有专门的指令执行，这就决定了**栈的效率比较高**。堆则是C/C++函数库提供的，它的机制是很复杂的，例如为了分配一块内存，库函数会按照一定的算法（具体的算法可以参考数据结构/操作系统）在堆内存中搜索可用的足够大小的空间，如果没有足够大小的空间（可能是由于内存碎片太多），就有可能调用系统功能去增加程序数据段的内存空间，这样就有机会分到足够大小的内存，然后进行返回。显然，**堆的效率比栈要低得多**。
- 访问方式
  在栈上的数据可以直接访问，但是在堆上的数据只能通过指针来访问。
- 内存碎片
  对于堆来讲，频繁的new/delete势必会造成内存空间的不连续，从而造成大量的碎片，使程序效率降低。对于栈来讲，则不会存在这个问题，因为栈是先进后出的队列，他们是如此的一一对应，以至于永远都不可能有一个内存块从栈中间弹出，在他弹出之前，在他上面的后进的栈内容已经被弹出，详细的可以参考数据结构，这里我们就不再一一讨论了。

## new/delete的实现机制

在C++中`new` 与`delete`即是关键字也是一种特殊的运算符。在C++程序中的内存管理主要通过关键字`new/delete`和`new[]/delete[]`实现。

```c++
class CA
{
    public:
       CA():m_a(0){}
       CA(int a):m_a(a){}

       virtual void foo(){ cout<<m_a<<endl;}
       int m_a;
};

void main()
{
       CA *p1 = new CA;
       CA *p2 = new CA(10);
       CA *p3 = new CA[20];

       delete p1;
       delete p2;
       delete[] p3;
}
```

其中关键字`new/delete`和`new[]/delete[]`对内存的分配主要通过相应的运算符来实现。这些运算符的底层则也是通过调用库函数`malloc/free`来实现。

```c++
void* operator new(size_t size);
void* operator new[](size_t size);
void  operator delete(void *p);
void  operator delete[](void *p);
```

除了对内存的空间的申请，`new`还要负责调用相应的构造函数对类对象进行初始化。 所以关键字`new`在整个调用过程中完成的工作主要是1.首先通过调用`operator new`分配了指定大小的未被初始化的空间，2.此后再调用构造函数对类对象所在内存空间进行初始化，3.最后返回新分配并构造好的对象的指针。`new`本身并不直接开辟内存。而`delete`则是1.调用 pA 指向对象的析构函数，对打开的文件进行关闭，2.释放指针所指向空间的内存。这其中的内存申请和释放的底层实现还是通过`malloc`和`free`来实现的。同理`new[]`和`delete[]`会对对象数据中的每一个对象进行对象的空间的分配和销毁。

![img](https://jacktang816.github.io/img/cpp/newDelete/newDeleteCall.png)

- 由于内部数据类型的“对象”没有构造与析构的过程，对其使用`malloc/free`或者`new/delete`是等价的。
- 对于`new[]`分配的对象数组，最后返回的指针与其内部通过`operator new[]`中返回的指针相差4个字节，多出的四字节空间用于存储对象数组的对象个数。因此在通过`delete[]`释放内存时，不用指定对象个数，只需要将`new[]`返回的指针前移四个字节就可以了。
- `free`或者`delete`只会释放指针指向空间的内存，但不会对指针本身做任何处理，所以为了防止野指针，**在释放了内存后要及时将指针置为NULL**。

```c++
CA *p = new CA[10];
delete p;
```

以上代码中`new[]`首先调用函数`operator new[]`分配了足够的内存大小（需要多出四个字节存放数组的大小），然后在刚分配的内存上调用构造函数初始化对象，最后返回对象数组的指针（不是分配内存空间的首地址，因为首地址存放的是数组的大小，返回地址即内存首地址+4的地址）。`delete`只完成了首先调用一次p指向的对象的析构函数，然后调用operator delete(p)释放内存。因此由于只对第一个元素调用了析构函数而另外9一个对象并没有析构，这有可能会**造成内存泄漏**。此外因为分配内存的首地址是p所指地址-4位置，所以直接释放p指向的空间会引起段错误。应该传入的参数为p-4。

## new/delete 重载

因为`new/delete`不仅是关键字也是运算符，故C++允许对其进行重载。系统默认的new关键字会分配堆内存和进行构造函数调用对内存进行初始化，而在实际应用中我们可能不希望频繁进行堆内存的申请和释放，因为频繁使用`new/delete`会造成内存碎片、内存不足等问题。所以可以通过实现内存池的方法，一次开辟一个较大的内存空间。每当需要创建对象时候，不用再去开辟内存只需要调用构造函数。

```c++
class String{
public:
  String(const char* str = ""){
  cout << "Create" << endl;
    if(NULL == str){
      data = new char[1];
      data[0] = '\0';
    }
    else{
      data = new char[strlen(str) + 1];
      strcpy(data, str);
    }
  }
  ~String(){
  cout << "Free" << endl;
    delete []data;
    data = NULL;
  }
private:
  char* data = NULL;
};
//重载方式1
void* operator new(size_t sz){
  cout << "in operator new" << endl;
  void* o = malloc(sz);
  return o;
}
void operator delete(void *o){
  cout << "in operator delete" << endl;
  free(o);
}
//重载方式2
void* operator new[](size_t sz){
  cout << "in operator new[]" << endl;
  void* o = malloc(sz);
  return o;
}
void operator delete[](void *o){
  cout << "in operator delete[]" << endl;
  free(o);
}

//重载方式3
//第一个参数size_t即使不适用，也必须有                          
void* operator new(size_t sz, String* s, int pos){
  return s + pos;
}
int main(){
  String *s = new String("abc");
  delete s;

  String *sr = new String[3];
  delete []sr;

  //开辟内存池，但是还没有调用过池里对象的构造方法                  
  String *ar = (String*)operator new(sizeof(String) * 2);
  //调用池里第一个对象的构造方法，不再开辟空间
  new(ar, 0)String("first0");
  //调用池里第二个对象的构造方法 ，不再开辟空间 
  new(ar, 1)String("first1");
  //调用池里第一个对象的析构方法，注意不会释放到内存
  (&ar[0])->~String();
  //调用池里第二个对象的析构方法，注意不会释放到内存
  (&ar[1])->~String();
  //下面语句执行前，内存池里的对象可以反复利用 
  operator delete(ar);
}
```

###  new/delete运算符重载的一些规则

- new和delete运算符重载必须成对出现
- new运算符的第一个参数必须是size_t类型，delete运算符的第一个参数则必须是要销毁释放的内存对象。
- 系统默认实现了`new/delete`、`new[]/delete[]`、`placement new/delete` 6个运算符函数。
- 当delete运算符的参数大于等于2时，就需要自己负责析构函数的盗用，并且以运算符函数的形式来调用delete运算符。

~~~c++
class A {
  public；
    A(){}
    void * operator new(size_t size);
    void * operator new[](size_t size);
    void * operator new(size_t size, void *p);
    void * operator new(size_t size, int a, int b);
    
    void operator delete(void *p);
    void operator delete[](void *p);
    void operator delete(void *p, void *p1);
    void operator delete(void *p, int a, int b);
};

class B {
  public:
    B(){}
}

//全局运算符函数，请谨慎重写覆盖全局运算符函数。
void * operator new(size_t size);
void * operator new[](size_t size);
void * operator new(size_t size, void *p) noexcept;
void * operator new(size_t size, int a, int b);

void operator delete(void *p);
void operator delete[](void *p);
void operator delete(void *p, void *p1);
void operator delete(void *p, int a, int b);

int main(){
  char buf[100];

  A *a1 = new CA();   //调用void * A::operator new(size_t size)
  
  A *a2 = new CA[10];  //调用void * A::operator new[](size_t size)
  
  A *a3 = new(buf)CA();  //调用void * A::operator new(size_t size, void *p)
  
  A *a4 = new(10, 20)CA();  //调用void* A::operator new(size_t size, int a, int b)
  
  
  delete a1;  //调用void A::operator delete(void *p)
  
  delete[] a2;  //调用void A::operator delete[](void *p)
  
  //a3用的是placement new的方式分配，因此需要自己调用对象的析构函数。
  a3->~CA();
  A::operator delete(a3, buf);  //调用void CA::operator delete(void *p, void *p1)，记得要带上类命名空间。

  //a4的运算符参数大于等于2个所以需要自己调用对象的析构函数。
  a4->~CA();
  A::operator delete(a4, 10, 20); //调用void CA::operator delete(void *p, int a, int b)

  //B类没有重载运算符，因此使用的是全局重载的运算符。
    
  B *b1 = new B();  //调用void * operator new(size_t size)

  B *b2 = new B[10]; //调用void * operator new[](size_t size)
  
  //这里你可以看到同一块内存可以用来构建CA类的对象也可以用来构建CB类的对象
  B *b3 = new(buf)B();  //调用void * operator new(size_t size, void *p)
  
  B *b4 = new(10, 20)CB(); //调用void* operator new(size_t size, int a, int b)
  

  delete b1;  //调用void operator delete(void *p)

  
  delete[] b2;   //调用void operator delete[](void *p)
  
  
  //b3用的是placement new的方式分配，因此需要自己调用对象的析构函数。
  b3->~B();
  ::operator delete(b3, buf);  //调用void operator delete(void *p, void *p1)
  
  //b4的运算符参数大于等于2个所以需要自己调用对象的析构函数。
  b4->~B();
  ::operator delete(b4, 10, 20);  //调用void operator delete(void *p, int a, int b)
 
 return 0;
    
}

## placement new 方法
`placement new`允许我们将object创建与 已经申请好的内存中。C++中`new/delete`除了默认实现外，还实现了对其的重载

```c++
void* operator new(size_t  size, void *p)
{
   return p;
}

void* operator new[](size_t size, void *p)
{
   return p;
}

void operator delete(void *p1, void *p2)
{
   // do nothing..
}

void operator delete[](void *p1, void *p2)
{
   // do nothing..
}
~~~

而这四个重载的运算符则是C++中的`placement new`和`placement delete`。由于`placement new`是在已经申请好的内存中创建对象，故重载的`operator new`并未真正申请内存`operator delete`也就不用释放内存了。因此在使用`placement new`和`placement delete`时要成对使用。

```c++
char *buf = new char[sizeof(A)*3]; //申请3个A大小的空间
A *p = new(buf) A(); //通过placement new 在申请好的buf的内存上面赋值
```

编译器遇到上面的代码会翻译成

```c++
A * p;
try {
    void* mem = operator new(sizeof(A), buf); //借用内存
    p = static_cast<A*>(mem);//安全转换
    p->A::A();//构造函数
}
catch (std::bad_alloc){
    //若失败 不执行构造函数
}
```

## 对象的自动删除

通过之前的分析我们知道，`new`关键字创建对象并非一步完成，而是通过先分配未初始化内存和调用构造函数初始化两步实现的。那么在这个过程中如果是第一步出错，那么内存分配失败不会调用构造函数，这是没有问题的。但是如果第一步已经完成在堆中已经成功分配了内存之后，在第二步调用构造函数时异常导致创建对象失败，那么就应该将第一步中申请的内存释放。C++中规定，如果一个对象无法完全构造，那么这个对象就是一个无效对象，也不会调用析构函数。因此为了保证对象的完整性，当通过new分配的堆内存对象在构造函数执行过程中出现异常时，就会停止构造函数的执行并且自动调用对应的`delete`运算符来对已经分配好的对内存执行销毁处理，即对象的自动删除技术。

虽然全局delete运算符所支持的对象删除技术能够解决对象本身的内存泄漏问题，但是却不能解决对象构造函数内部数据成员的内存泄漏。

```c++
class A {
  public:
    A(){
      m_a = new int(0);
      throw 1;
    }

    ~A(){
      delete m_a;
      m_a = nullptr;
    }

  private:
    int *m_a;
};

int main(){
  try {
    A *p = new A();
    delete p; //这一句不会执行，m_a的内存不会被释放
  } catch(int) {
    cout << "Oops!" << endl;
  }
}
```

上面代码中类A在构造函数中抛出异常，虽然系统会对p所指对象执行自动删除来销毁已经申请的内存，但是对于A的数据成员m_a，因为没有调用析构函数所以导致了其内存的泄露。那么如何来解决这问题呢？

数据成员的内存泄露主要是因为全局`operator delete`只对对象本身进行了内存释放，所以当自动删除的时候并不会考虑对象的成员。因此可以**通过C++中的关于`operator new/operator delete`运算符发重载技术来重载`operator delete`解决这一问题**。

```c++
class A {
  public:
    A(){
      m_a = new int(0);
      throw 1;
    }

    ~A(){
      delete m_a;
      m_a = nullptr;
    }

    void *operator new(size_t size){
      return malloc(size);
    }

    void operator delete(void *p){
      A *pb = (A*)p;
      if (pb->m_a != nullptr)
        delete pb->m_a;

      free(p);
    }

  private:
    int *m_a;
};
```
