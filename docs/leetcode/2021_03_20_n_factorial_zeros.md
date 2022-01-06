# n的阶乘结果后面有多少个0

[原题](https://leetcode-cn.com/problems/factorial-trailing-zeroes/)

## 分析

首先，一个5最多可以产生1个0，例如：5\*2=10，15\*4=60。但是，注意到25\*4=100，75\*4=300，这种会产生2两个0。经过观察可以得出这样的结论：一个数字的质因分解中有多少个5，就会产生多少个0。

注意，我们不单独考虑10的倍数，因为，例如：10=2\*5，100=2\*2\*5\*5，都符合上面的结论。

## 代码实现

```c++
class Solution {
public:
    int trailingZeroes(int n) {
        int factor=5;
        int num=0;
        while(factor<=n){
            int t=n/factor;
            num+=t;
            factor*=5;
        }
        return num;
    }
};
```

