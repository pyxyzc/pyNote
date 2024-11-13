# 语法



# 内存管理

## 内存分区

从低地址到高地址：

| .text(Code Segment) | Global/Static Storage | Stack -> | ... | <- Heap |

##  指针和引用

老生长谈，

| 指针                                          | 引用                                 |
| --------------------------------------------- | ------------------------------------ |
| 一个变量，保存了另一个变量的内存地址，8个字节 | 别名，与原变量共享内存地址           |
| 可以被重新赋值，指向不同的变量                | 初始化后不能更改，始终指向同一个变量 |
| 可以为 nullptr，表示不指向任何变量            | 必须绑定到一个变量，不能为 nullptr   |

实质上，在 C++ 编译器层面，引用会被当做 const 指针来进行操作。



## this









## RAII

资源获取即初始化，离开作用域即自动销毁 / 使用局部对象来管理资源。解决了什么问题？首先考虑资源的管理难处，就像随手用东西不「归位」一样，资源的「销毁」是最容易忘记的，碰巧也是损伤最大的；解决方法：利用起来栈上局部对象的自动销毁。

> 其实比起忘记更大的困难是：保证资源在分配和释放的过程中不被打断，也就是「异常安全性」。
>
> ```text
> P* a = new P;  //(1)
> dosomething(); //(2)
> delete a; // (3)
> ```
>
> 如果 2 抛出异常，则 3 一定不会被执行到，则会发生内存泄漏。不引入 raii 之前，解决方法是

一个简单的例子：封装 pthread_mutex_t 到 Mutex 对象里，Mutex 的构造 / 析构承载其 init / destroy，则 mu_ 的生命周期实质上由执行流是否离开利用域决定，很巴适。

```cpp
class Mutex {
public:
    Mutex();
    ~Mutex();

    void Lock();
    void Unlock();
private:
    pthread_mutex_t mu_;

    Mutex(const Mutex&) = delete;
    void operator=(const Mutex&) = delete;
};
```

```cpp
static void PthreadCall(const char* label, int result) {
    if (result != 0) { fprintf(stderr, "pthread %s: %s\n", label, strerror(result)); }
}
Mutex::Mutex() {
    PthreadCall("init mutex", pthread_mutex_init(&mu_, NULL));
}
Mutex::~Mutex() {
    PthreadCall("destroy mutex", pthread_mutex_destroy(&mu_));
}
void Mutex::Lock() {
    PthreadCall("lock", pthread_mutex_lock(&mu_));
}
void Mutex::Unlock() {
    PthreadCall("unlock", pthread_mutex_unlock(&mu_));
}
```

值得注意的是，Lock / Unlock 也可以封装在更上一层的抽象中，使得开锁解锁可以自动进行 -> 😈于是我们实现了 lock_guard。🤔所以，凡是「对立」性质的正反操作都可以放在更上一层的抽象中，很有 C++ 特色的代码风格了。

```cpp
class MutexLock {
public:
    explicit MutexLock(Mutex* mu) : mu_(mu) {
        this->mu_->Lock();
    }
    ~MutexLock() {
        this->mu_->Unlock();
    }

private:
    Mutex* const mu_;
    MutexLock(const MutexLock&)      = delete;
    void operator=(const MutexLock&) = delete;
};
```

## Smart ptr

指针带来的便利性和编码上的苦难（memory leak 和内存非法访问）的 tradeoff。Smart ptr 用于减少手动管理指针（准确来说是指针指向资源的）生命周期的心智负担。

1. std::unique_ptr：独占所有权。
2. std::shared_ptr：共享所有权，允许多个 shared_ptr 指向同一个对象，当最后一个shared_ptr超出作用域时，所指向的内存才会被自动释放。

## shared_ptr

首先来看 shared_ptr 的常用 API：

```cpp
// ctor
std::shared_ptr<int> ptr;
std::shared_ptr<int> ptr = std::make_shared<int>(42);
// release
ptr.reset();
ptr.reset(new int(42)); // 释放当前 shared_ptr 的所有权，并使其指向新的对象
// get
int* raw_ptr = ptr.get();

```















## weak_ptr

主要与 shared_ptr 配合使用，解决循环引用问题，观察 shared_ptr 对象而不不影响引用计数，以及在需要时提供对底层资源的访问。

实质上，weak_ptr 真正代表的概念是「弱引用」，即并不「拥有」对象本身。从一个点可以看出，即在另一个 shared_ptr 







## malloc / free & new / delete

### 区别

| malloc / free                                                | new / delete                                                 |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 库函数（需要 include 头文件才能用）；更多的是和 operator new/operator delete 类似 | 关键字                                                       |
| malloc 申请内存需要显示填入大小：`int* ptr = (int) malloc(4);` | `int* ptr = new int;`                                        |
| 返回类型：malloc 返回的是 void* 需要进行强转                 | new 返回的是对应类型的指针，无需强转                         |
| 内存分配失败，malloc 返回 NULL                               | new 不会返回，只会抛出异常，如果不捕获异常，会导致程序崩溃   |
| 内存扩容机制，malloc 分配的内存空间如果不足了，可以用 realloc 来扩容，扩容过程是 realloc 先检查当前内存空间后面是否还有足够的连续空间，如果有就后面继续申请，并返回原来的指针，如果后面没有足够的，就会另外在别的地方申请一片内存空间，并把原来的内容拷贝到新的内存中，返回新的地址指针 | new 是动态分配内存，不存在扩容操作，如果需要更大的空间，就必须显式地分配一块新的内存，然后手动将旧内存的数据复制到新内存中，最后释放旧内存 |

new 实现类似扩容的操作：

```cpp
int main() {
    // 初始分配 5 个整数的空间
    int* arr = new int[5];
    for (int i = 0; i < 5; ++i) {
        arr[i] = i;
    }

    // 需要更大的空间，因此手动扩容到 10 个整数
    int  newSize = 10;
    int* newArr  = new int[newSize];

    // 将旧数组内容复制到新数组
    std::memcpy(newArr, arr, 5 * sizeof(int));

    // 释放旧数组
    delete[] arr;
    arr = newArr;

    // 输出新数组内容
    for (int i = 0; i < newSize; ++i) {
        std::cout << arr[i] << " ";
    }
    std::cout << std::endl;

    // 释放新数组
    delete[] arr;
    return 0;
}
```

> new 更偏向 OOP，即偏向于管理对象，而非内存分配，初始化对象调用构造函数，而释放内存时会调用析构函数。



### 实现机制

#### new

![img](./imgs/v2-3354a0b30af6d1d9cfbfae46a2ccb138_1440w.jpg)



#### delete

![img](./imgs/v2-a66e18d6aefc95dba444018c35dac214_1440w.jpg)

### new[] / delete[]

分配数组和释放数组，最好成对出现

#### 基本数据类型

delete / delete[] 都是正确的，无 ctor，最后 ptr 指向的内存块会被释放掉，不会有问题。

```cpp
int *ptr = new int[10];
delete ptr; // delete[] ptr;
```

#### 自定义

必须成对出现

```cpp
A* a = new A[3];
delete[] a;
```

Why 必须成对出现？首先看 new[] 的过程：

![img](./imgs/v2-4be3e2fe73c436729792cfb17b6e85ba_1440w.jpg)

于是 delete[] 就很清晰了，一定要取出这个数组长度做操作：

![img](./imgs/v2-81ff34511de59c58b38dab436fc6cac2_1440w.jpg)



> 同理，why  new 一个对象用 delete[] 释放会有问题？
>
> delete[] 过程：placement delete[] 会取当前指针指向地址之前的 4 个字节空间中的内容来作为数组长度依次调用析构，调用 operator delete[] 释放指针指向（地址 - 4 字节）为首地址的内存空间，那么指针（地址 - 4 字节）的内存空间中存的数据是未知的，可能会造成严重的问题。

# 并发

为何需要多进程（或者多线程），为何需要并发？同时间，多条执行流。

多进程

进程？running program

```cpp
void print_exit() {
    printf("the exit pid: %d\n", getpid());
}

int main() {
    pid_t pid;
    atexit(print_exit);  //注册该进程退出时的回调函数
    pid = fork();
    if (pid < 0) {
        printf("error in fork!");
    } else if (pid == 0) {
        printf("i am the child process, my process id is %d\n", getpid());
    } else {
        printf("i am the parent process, my process id is %d\n", getpid());
        wait(NULL);
    }
}
```

```
i am the parent process, my process id is 230217
i am the child process, my process id is 230218
the exit pid: 230218
the exit pid: 230217
```

子进程是 fork 出来的额外的执行流，和父进程有什么关系呢？子进程会复制父进程的 task_struct 结构，并为子进程的堆栈分配物理页。理论上来说，子进程应该完整地复制父进程的堆，栈以及数据空间，但是 2 者共享正文段。

> 写时复制：由于一般 fork 后面都接着 exec，所以，现在的 fork 都在用写时复制的技术，顾名思意，就是，数据段 / 堆 / 栈，一开始并不复制，由父 / 子进程共享，并将这些内存设置为只读。直到父 / 子进程一方尝试写这些区域，则内核才为需要修改的那片内存拷贝副本。这样做可以提高 fork 的效率。

多线程

running program 的可分配单元。单个 running program 可以并发执行多个「任务」。

> Why？极少的系统开销，如拷贝等。

```
int pthread_create(pthread_t *restrict tidp,
                   const pthread_attr_t *restrict attr,
                   void *(*start_rtn)(void), 
                   void *restrict arg);
```

1. 第一个参数为指向线程标识符的指针
2. 第二个参数用来设置线程属性
3. 第三个参数是线程运行函数的起始地址
4. 最后一个参数是运行函数的参数

```

```





