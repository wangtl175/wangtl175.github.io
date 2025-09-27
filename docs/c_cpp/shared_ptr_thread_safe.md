# shared_ptr是线程安全的吗？

`shared_ptr`**只有共享块控制块里的引用计数器是原子的**，也就是线程安全的，但是对同一个`shared_ptr`实例的并发读写与被管理对象的并发访问不是线程安全的。

## 线程安全的情况

- 多个线程各自持有不同的`shared_ptr`实例，对这些实例的拷贝、析构、赋值是线程安全的。

```cpp
std::shared_ptr<Foo> p = std::make_shared<Foo>();

std::thread t1([q = p] { q.reset(); });
std::thread t2([r = p] { auto s = r; });

```

- `weak_ptr`与`shared_ptr`之间的转换

```cpp
std::shared_ptr<Foo> p = std::make_shared<Foo>();
std::weak_ptr<Foo> q = p;

std::thread t1([r = p] { q.reset(); });
std::thread t2([s = q] { auto t = r.lock(); });

```

## 线程不安全的情况

- 多个线程同时读写同一个`shared_ptr`实例

```cpp
std::shared_ptr<Foo> p = std::make_shared<Foo>();

std::thread t1([&p] { p.reset(); });
std::thread t2([&p] { auto q = p; });
```

- `shared_ptr`没有保证被管理的对象的线程安全

```cpp
std::shared_ptr<Foo> p = std::make_shared<Foo>();

// Foo的fun实现需要保证线程安全
std::thread t1([q = p] { q->fun(); });
std::thread t2([r = p] { r->fun(); });
```
