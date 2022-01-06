# N皇后的位运算解法

这个题解使用到的位运算解法比较新颖，特此记录一下。

[原题连接](https://leetcode-cn.com/problems/n-queens/)，[题解](https://leetcode-cn.com/problems/n-queens/solution/nhuang-hou-by-leetcode-solution/)

## 知识点

+ 令一个整数$x$的二进制表示中最后一个1为0，eg: $(0110)_2$ => $(0100)_2$可以这样

    ```c++
    x = x & (x - 1);
    ```

+ 令一个整数$x$的二进制表示中，除最后一个1外，其它都为0，eg: $(0110)_2$ => $(0010)_2$

    ```c++
    x = x & (-x);
    ```

    > 可以由上一个知识点推到过来，相当于 令y= x & (x - 1)，x = x & \~y。\~y相当于x取反加1，也就是-x

+ C标准库中的`__buildtin_`函数

    > GCC提供了一系列的builtin函数，可以实现一些简单快捷的功能来方便程序编写，另外，很多builtin函数可用来优化编译结果。这些函数以“\_\_builtin\_”作为函数名前缀。
    >
    > GCC provides quite a lot of builtin functions. These functions are part of standard C offered by the compiler and may come in various variants as per the gcc. These are also termed as hardware specific functions which are internally implemented in assembly or we can say machine instructions, with wide usage in low level programming and are generally target optimized.

    `int __buildtin_ctz(unsigned int x)`：计算x末尾(二进制表示)0的个数，x=0时结果未定义。

## 代码

```c++
#pragma once
#include<vector>
#include<string>
#include<set>
using namespace std;

#ifdef __GNUC__
#define clz(x) __builtin_clz(x)
#define ctz(x) __builtin_ctz(x)
#endif
#ifdef _MSC_VER
#include <intrin.h>

uint32_t __inline ctz(uint32_t value) {
	unsigned long trailing_zero = 0;

	if (_BitScanForward(&trailing_zero, value)) {
		return trailing_zero;
	} else {
		// This is undefined, I better choose 32 than 0
		return 32;
	}
}

uint32_t __inline clz(uint32_t value) {
	unsigned long leading_zero = 0;

	if (_BitScanReverse(&leading_zero, value)) {
		return 31 - leading_zero;
	} else {
		// Same remarks as above
		return 32;
	}
}
#endif

class Solution {
public:
	vector<vector<string>> solveNQueens(int n) {
		boardWidth = n;
		vector<int> queens(n, -1);
		vector<vector<string>> solution;
		solve(solution, queens, 0, 0, 0, 0);
		return solution;
	}
private:
	int boardWidth;
	// columns, diagonals1(右下方向), diagonals2(左下方向)中，为1的位表示第depth行不能放置棋子
	void solve(vector<vector<string>> &solution, vector<int> &queens, int depth, int columns, int diagonals1, int diagonals2) {
		if (depth == boardWidth) {
			solution.push_back(generateSolution(queens));
			return;
		}
		// (columns | diagonals1 | diagonals2)可以放置棋子的地方为0，取反之后，可以放置棋子的地方为1，不可以放置棋子的为0
		int available = ((1 << boardWidth) - 1) & (~(columns | diagonals1 | diagonals2));//为0的位置不能放置棋子
		while (available != 0) {
			// available取反加一，再和availabe相与，则可以获取到available最右边的那个1
			int pos = available & (-available);
			// 令available最右边的1为0
			available = available & (available - 1);
            // __builtin_ctz时gcc提供的，用mvc时，要自己实现
			int column = ctz(pos);// __builtin_ctz(x)，计算x末尾0的个数，x为0时未定义
			queens[depth] = column;
			solve(solution, queens, depth + 1, columns | pos, (diagonals1 | pos) >> 1, (diagonals2 | pos) << 1);
			queens[depth] = -1;//回溯
		}
	}
	vector<string> generateSolution(vector<int> &queens) {
		vector<string> solution(boardWidth, string(boardWidth, '.'));
		for (int i = 0; i < boardWidth; i++) {
			solution[i][queens[i]] = 'Q';
		}
		return solution;
	}
};
```







