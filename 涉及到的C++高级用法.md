# 涉及到的C++高级用法

# 1.函数绑定对象，绑定器

![image-20241211154020235](C:\Users\18324\AppData\Roaming\Typora\typora-user-images\image-20241211154020235.png)

# 2.多态，使用基类指针

rpc框架不单单给某个服务使用，而是给所有服务使用

![image-20241211164733650](C:\Users\18324\AppData\Roaming\Typora\typora-user-images\image-20241211164733650.png)

# 3.日志

异步日志队列使用模板

```C++
private:
    std::queue<T> m_queue; // 用于存储数据的队列
    std::mutex m_mutex; // 互斥锁，保护队列
    std::condition_variable m_condvariable; // 条件变量，用于线程同步
```

可以存储任何类型数据

线程同步和互斥

线程安全 lock_guard

`Logger` 类采用了单例模式，确保系统中只有一个日志实例，同时通过 `std::thread` 实现了后台写日志的异步操作。

定义两个级别的日志信息，用宏，然后是可变参数





# 4.单例模式

框架，日志

单例模式用于确保整个程序中只有一个 `Logger` 实例。日志系统通常是全局性的，多个模块需要共享同一个日志对象。

单例模式保证了 `Logger` 对象的全局唯一性，避免资源竞争和重复实例化。

static Logger logger;

 表示 `logger` 是一个静态局部变量，它只会被初始化一次，并在程序生命周期内一直存在。

# 5.lambda表达式

在线程注册函数：多线程或者需要临时定义的逻辑时。