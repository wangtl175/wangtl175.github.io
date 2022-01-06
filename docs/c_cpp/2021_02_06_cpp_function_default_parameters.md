# C++的缺省函数到底是在声明中确定还是在定义中确定？

首先，编译器是禁止在同时在声明和定义中确定缺省参数，只能在其中一个地方确定缺省参数。注意，在函数定义中确定缺省参数时，定义要在调用之前，即

```c++
#include<iostream>
using namespace std;
int sum(int x,int y);  // 函数声明
int main(){
    cout<<sum()<<endl; // error: too few arguments to function 'int sum(int, int)'
}
int sum(int x=1,int y=2){  // 函数定义
    return x+y;
}
```

要改成

```c++
#include<iostream>
using namespace std;
int sum(int x,int y);  // 函数声明
int sum(int x=1,int y=2){  // 函数定义
    return x+y;
}
int main(){
    cout<<sum()<<endl; // error: too few arguments to function 'int sum(int, int)'
}
```

那么，C++的缺省函数到底是在声明中确定还是在定义中确定？

声明是用户可以看到的部分，而定义通常是用户不可见的，所以如果在定义中确定缺省值的话，用户看不到，因此，通常在声明中确定缺省值。