---
description: 向量化执行引擎开山之作
---

# MonetDB/X100: Hyper-Pipelining Query Execution

## Part 1：现代CPU工作原理

### 原有模型的问题

#### 火山模型

* 火山模型的特点：tuple-at-a-time
* 这种执行架构（其实是大部分DBMS的执行架构）使得编译器无法进行充分优化来利用CPU的并行能力(super-scalar)
* tuple-at-a-time的模式也增加了很多interpret成本
* 由于每次只处理一行数据，难以实现基于独立数据的CPU并行

火山模型最开始提出的时候，整个执行过程的瓶颈在于磁盘，因此模型本身导致的CPU额外开销就被忽略了。但是随着磁盘速度的提升，火山模型在CPU上的浪费逐渐体现出来。

#### column-at-a-time模型

MonetDB一开始采用的是column-at-a-time的执行方式，这种模型需要在各种计算之间对全列数据做物化，虽然避免了interpret的成本，但也在执行中引入了大量的memory IO，profiling显示执行速度严重受限于memory IO bandwidth，从而影响了CPU的执行效率。

最终MonetDB选择了：tuple-at-a-time的volcano和column-at-a-time的全物化之间的折中方案，很好的提升了CPU执行效率

### 现代cpu的特点及问题

特点：流水线和超标量，并行能力大幅提升

问题：数据冒险和控制冒险的存在，影响了CPU的执行时间

Itanium2：pipeline多，但是每条pipeline的stage少

Pentium4：pipeline较少，但每条pipeline的stage很多

理想情况：CPU在任意时间的理论最大吞吐能力就是： num(pipeline) \* num(stage)，当然这只是理想情况

**loop pipelining**

大多数编程语言（包括C/C++)都不需要编程人员去指定指令之间的独立性，这需要由compiler来完成，因此compiler的优化能力对于cpu的充分利用就变得很重要，其中一个很重要的优化就是**loop pipelining（这也是vectorize所追求的)：**

```
对一个数组A中的每个元素a（元素间独立），由2个相关的计算任务F() -> G() ，且F()的执行需要2个cpu cycle
则loop pipelining可以将：
F(A[0]),G(A[0]), F(A[1]),G(A[1]),.. F(A[n]),G(A[n])
转换为：
F(A[0]),F(A[1]),F(A[2]), G(A[0]),G(A[1]),G(A[2]), F(A[3]) ...
```

这样的优化也是为何现在很多的数据库系统即使没有使用SIMD，也要先实现基于列存的向量化，就是为了利用loop pipelining的这种能力

#### predicated version

可以看到，手工重写后的版本将不带有分支的if语句，因此其执行效率更高，且和选择率无关。

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption><p>branch version -> predicated version</p></figcaption></figure>







## Part 2：介绍向量化执行引擎MonetDB/X100
