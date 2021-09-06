# llvm-loopstat

### 功能需求
基于[Driver](https://gitee.com/s4plus/llvm-ustc-proj/blob/master/my-llvm-driver/include/Driver/driver.h#L23)，实现一个分析器，该分析器依次输出每个函数的循环信息，其中会调用对每个[Function](https://github.com/llvm/llvm-project/blob/llvmorg-11.0.0/llvm/include/llvm/IR/Function.h#L61)的分析Pass，该Pass需要做到：

- 统计给定[Function](https://github.com/llvm/llvm-project/blob/llvmorg-11.0.0/llvm/include/llvm/IR/Function.h#L61)中的循环信息
  - 并列循环的数量
  - 每个循环的深度信息

如对于给出的以下样例：

```c
int main(){
    int num = 0;
    for(int i=0;i<10;i++){
        for(int j=i;j<10;j++){
            for(int k=j;k<10;k++){
				num++;
            }
        }
        for(int j=i;j<10;j++){
            for(int k=j;k<10;k++){
				num++;
            }
        }
    }
    for(int i=0;i<10;i++){
        for(int j=i;j<10;j++){
            num++;
        }
    }
    return num;
}
```

你需要识别出main函数下有两个并列循环，依次记为`L1`、`L2`; 其中，`L1`下又有两个并列循环，依次记为`L11`，`L12`；`L11`下有一个嵌套深度为1的循环，记为`L111`;`L12`下有一个嵌套深度为1的循环，记为`L121`；`L2`下有一个嵌套深度为1的循环，记为`L21`。

因此你需要识别出如下的循环信息：

```json
{
    "L1" : {
        "depth" : 1,
        "L11" : {
            "depth" : 2,
            "L111" : {
                "depth" : 3
            }
        },
        "L12" : {
            "depth" : 2,
            "L121" : {
                "depth" : 3
            }
        }
    },
    "L2" : {
        "depth" : 1,
        "L21" : {
            "depth" : 2
        }
    }
}
```

### 算法

首先描述一下课本上的算法, 即构造回边的自然循环:

这里引用书上的说法:
> 输入: 流图 G 和 回边 $n\rightarrow d$
> 输出: 由回边 $n\rightarrow d$ 确定的自然循环中所有结点的集合 loop.
> 方法: 令 loop 初值是 {n, d}. 标记 d 为已访问, 以保证搜索不会超出 d. 从结点 n 开始, 完成对流图 G 的逆向流图的深度优先搜索, 把搜索过程中访问的所有结点都加入 loop. 该过程找出不经过 d 能到达 n 的所有结点.

我们根据这个算法进行一定的**修改**才得到获取各循环深度的算法:
1. <a id='step1'></a>对支配树进行后序遍历. 在遍历每个结点的时候, 去遍历它在流图中的前驱, 通过 `dominates` 方法来判断支配情况, 以确认是否为回边. 
2. <a id='step234'></a>找到该结点相应的所有回边后, 就开始做在逆向流图的 DFS.
3. 在该 DFS 过程中, 若有未发现过的 `BasicBlock`, 就将其标记为当前循环的 `BasicBlock`; 否则它应当已经被之前的更内层的循环标记过, 我们不必理会(不把这个内层循环内的 `BasicBlock` 加入 DFS 的栈中), 只需继续 DFS.
4. 在前面完成了对整个支配树的后序遍历后, 我们就获得了每个 `BasicBlock` 所属的最近的一层循环, 以及各个循环的父循环. 那么此时只需要遍历一遍所有的循环, 就可以建立其循环嵌套的树.
5. <a id='step5'></a>一些简单的遍历树并 print

### 代码结构


核心代码位于 [include/Analysis/LoopStatisticsPass.hpp](./include/Analysis/LoopStatisticsPass.hpp).
