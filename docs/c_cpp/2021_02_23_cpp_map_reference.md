# C++ Map使用汇总


总结Map的常用方法；以及如何在对Map进行遍历的同时，进行删除操作。

[文档](http://www.cplusplus.com/reference/map/map/)

## Map特性

### Associative

Elements in associative containers are referenced by their *key* and not by their absolute position in the container.

### Ordered

The elements in the container follow **a strict order** at all times. All inserted elements are given a position in this order.

### Map

Each element associates a *key* to a *mapped value*: Keys are meant to identify the elements whose main content is the *mapped value*.

### Unique keys

No two elements in the container can have equivalent *keys*.

### Allocator-aware

The container uses an allocator object to dynamically handle its storage needs.

## 常用方法（C++11）

### 声明和初始化

```c++
#include<iostream>
#include<map>
#include<string>

using namespace std;

int main() {
    // 使用{}初始化时从c++11开始的
	map<string, int> words = { {"Jim",2012} ,{"Tom",2023} };
}
```

### 更新

#### 插入

##### 使用\[\]插入

```c++
map<string, int> words;
words["apple"] = 2;
```

如果key已经存在，则会作赋值修改，没有则插入。使用\[\]访问一个不存在的key时，值会初始化位0或'\0'或""(感觉不安全)。

##### 使用insert插入

c++11中有4种方法，新增了[interlizer_list](http://www.cplusplus.com/reference/initializer_list/initializer_list/)这种方法。[具体文档](http://www.cplusplus.com/reference/map/map/insert/)

```c++
#include <iostream>
#include <map>

int main ()
{
  std::map<char,int> mymap;

  // first insert function version (single parameter):
  mymap.insert ( std::pair<char,int>('a',100) );
  mymap.insert ( std::pair<char,int>('z',200) );
  // 使用返回值判断插入是否成功
  std::pair<std::map<char,int>::iterator,bool> ret;
  ret = mymap.insert ( std::pair<char,int>('z',500) );
  if (ret.second==false) {  // 当成功插入时，为true
    std::cout << "element 'z' already existed";
    std::cout << " with a value of " << ret.first->second << '\n';
  }

  // second insert function version (with hint position):
  std::map<char,int>::iterator it = mymap.begin();
  mymap.insert (it, std::pair<char,int>('b',300));  // max efficiency inserting
  mymap.insert (it, std::pair<char,int>('c',400));  // no max efficiency inserting
  // 注意, it只是一个hint，并不强制把新元素插入该位置(Map是严格有序的)

  // third insert function version (range insertion):
  std::map<char,int> anothermap;
  anothermap.insert(mymap.begin(),mymap.find('c'));
  
  // c++11开始支持用列表插入
  anothermap.insert({ { 'd', 100 }, {'e', 200} });
  
  // showing contents:
  std::cout << "mymap contains:\n";
  for (it=mymap.begin(); it!=mymap.end(); ++it)
    std::cout << it->first << " => " << it->second << '\n';

  std::cout << "anothermap contains:\n";
  for (it=anothermap.begin(); it!=anothermap.end(); ++it)
    std::cout << it->first << " => " << it->second << '\n';

  return 0;
}
```

#### 删除元素

##### erase

```c++
iterator  erase (const_iterator position);

size_type erase (const key_type& k);  //删除key为k的元素，返回删除了多少个元素

iterator  erase (const_iterator first, const_iterator last);  // 删除多个元素
```

第1和第3个erase返回删除元素的指向下一个的iterator

> The other versions return an iterator to the element that follows the last element removed (or [map::end](http://www.cplusplus.com/map::end), if the last element was removed).

#### clear

```c++
void clear() noexcept;
```

删除所有元素

#### 交换swap

```c++
void swap (map& x);
```

和另一个相同类型的map交换元素。

```c++
std::map<char,int> foo,bar;
foo.swap(bar);
```

#### emplace和emplace\_hint

类似于插入，为c++11新增

```c++
template <class... Args>
pair<iterator,bool> emplace (Args&&... args);

template <class... Args>
iterator emplace_hint (const_iterator position, Args&&... args);
```

示例

```c++
// map::emplace
#include <iostream>
#include <map>

int main ()
{
  std::map<char,int> mymap;

  mymap.emplace('x',100);
  mymap.emplace('y',200);
  mymap.emplace('z',100);

  std::cout << "mymap contains:";
  for (auto& x: mymap)
    std::cout << " [" << x.first << ':' << x.second << ']';
  std::cout << '\n';

  return 0;
}
```



### 取值

#### 操作符\[\]

访问到不存在的key时，会引发插入操作。如果没有赋值给新的元素，则使用默认值

> If *k* does not match the key of any element in the container, the function inserts a new element with that key and returns a reference to its mapped value. Notice that this always increases the [container size](http://www.cplusplus.com/map::size) by one, even if no mapped value is assigned to the element (the element is constructed using its default constructor).

#### at()

c++11新增，或检测查询的key是否存在。如果不存在，则触发[out_of_range](http://www.cplusplus.com/out_of_range)异常。

### 容量查询

```c++
// 查询map是否为空
bool empty();

// 查询map中键值对的数量
size_t size();

// 查询map所能包含的最大键值对数量，和系统和应用库有关。
// 此外，这并不意味着用户一定可以存这么多，很可能还没达到就已经开辟内存失败了
size_t max_size();
```

### 迭代器

#### begin和end，以及cbegin和cend

其中cbegin和cend是c++11新增的

```c++
/*begin & end*/
iterator begin() noexcept;
const_iterator begin() const noexcept;

iterator end() noexcept;
const_iterator end() const noexcept;

/*cbegin & cend*/
const_iterator cbegin() const noexcept;
const_iterator cend() const noexcept;
```

> A `const_iterator` is an iterator that points to const content. This iterator can be increased and decreased (unless it is itself also const), just like the `iterator` returned by [map::begin](http://www.cplusplus.com/map::begin), but it cannot be used to modify the contents it points to, even if the [map](http://www.cplusplus.com/map) object is not itself const.

#### rbegin和rend，以及crbegin和crend

其中crbegin和crend是c++11新增的

```c++
/*rbegin & rend*/
// 返回一个指向最后一个元素的reverse_iterator
reverse_iterator rbegin() noexcept;
const_reverse_iterator rbegin() const noexcept;

// 返回指向第一个元素之前的reverse_iterator
reverse_iterator rend() noexcept;
const_reverse_iterator rend() const noexcept;

/*crbegin & crend*/
const_reverse_iterator crbegin() const noexcept;
const_reverse_iterator crend() const noexcept;
```

示例

```c++
// map::rbegin/rend
#include <iostream>
#include <map>

int main ()
{
  std::map<char,int> mymap;

  mymap['x'] = 100;
  mymap['y'] = 200;
  mymap['z'] = 300;

  // show content:
  std::map<char,int>::reverse_iterator rit;
  for (rit=mymap.rbegin(); rit!=mymap.rend(); ++rit)
    std::cout << rit->first << " => " << rit->second << '\n';

  return 0;
}
```

输出

```
z => 300
y => 200
x => 100
```

### 其它操作

```c++
// 查找key为k的元素，如果不存在返回end
iterator find (const key_type& k);
const_iterator find (const key_type& k) const;


// 查询key为k的元素的个数，对于map来说，如果存在，则一定为1
size_type count (const key_type& k) const;


// 比较
key_compare key_comp() const; // 返回一个用于比较key的比较器
value_compare value_comp() const; // 返回一个用于比较value的比较器
//示例
map<char,int> mymap;
map<char,int>::key_compare mycomp = mymap.key_comp();
mymap['a']=100;
mymap['b']=200;
mycomp('a', 'b');  // a排在b前面，因此返回结果为true


//上下界
iterator lower_bound (const key_type& k); // 指向大于等于k的第一个元素
const_iterator lower_bound (const key_type& k) const;

iterator upper_bound (const key_type& k);  // 指向小于等于k的第一个元素
const_iterator upper_bound (const key_type& k) const;

pair<const_iterator,const_iterator> equal_range (const key_type& k) const; // 等于key的元素范围，对于map来说，最多只有1个
pair<iterator,iterator>             equal_range (const key_type& k);
```

### Map遍历删除

对于c++里面的容器, 我们可以使用iterator进行方便的遍历. 但是当我们通过iterator对vector/map等进行删除时, 我们就要小心了, 因为操作往往会导致iterator失效, 之后的行为都变得不可预知.

```c++
int main(int argc, char* argv[])
{
    map<string, string> mapData;
    
    mapData["a"] = "aaa"; 
    mapData["b"] = "bbb"; 
    mapData["c"] = "ccc"; 
    for (map<string, string>::iterator iter=mapData.begin(); iiter!=mapData.end();)
    {
        if (i->first == "b")
        {
            mapData.erase(iter++);
        }
        else
        {
            iter++;
        }
    }
    return 0;
}
```

分析mapData.erase(i++)语句，map中在删除iter的时候，先将iter做缓存，然后执行iter++使之指向下一个结点，再进入erase函数体中执行删除操作，删除时使用的iter就是缓存下来的iter(也就是当前iter(做了加操作之后的iter)所指向结点的上一个结点)。

可以看出（mapData.erase(iter++) ）和（mapData.erase(iter); iter++; ）这个执行序列是不相同的。前者在erase执行前进行了加操作，在it被删除(失效)前进行了加操作，是安全的；后者是在erase执行后才进行加操作，而此时iter已经被删除(当前的迭代器已经失效了)，对一个已经失效的迭代器进行加操作，行为是不可预期的，这种写法势必会导致 map操作的失败并引起进程的异常。

而对于vector，可以这样

```c++
iter=v.erase(iter);
```

