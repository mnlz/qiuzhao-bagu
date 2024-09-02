# Java基础相关

## 1.集合相关

HashMap和ArrayList相关

### HashMap的基础参数：

桶树化的阈值：64

链表树化的阈值：8

树链表化的阈值：6（红黑树的长度低于6）

装载因子：0.75

初始化容量：16*0.75 = 12

扩容机制：两倍扩容

### HashMap的底层数据结构：

JDK1.7：数组+链表 采用对链表的头插法解决hash冲突

- 采用头插法解决冲突

JDK1.8：数组+链表+红黑树

- 在链表的尾部插入解决冲突
- 树化条件
  - 链表长度大于8
  - hash桶的长度大于64

### HashMap的put流程

首次插入会进行初始化一个：HashMap的长度为16*0.75 = 12

先根据key计算出hash值，进行hash扰动得到新的hash值，新hash &（length-1） 计算出hash桶的下标

桶是否为空

- 为空：直接插入
- 不为空：
  - 是红黑树节点，添加或者更新，判断是否满足扩容阈值，扩容
  - 不是红黑树节点：从头开始遍历链表节点，更新或者添加，插入链表的尾部，判断是否需要扩容

<img src="https://mmbiz.qpic.cn/mmbiz_png/K1iaod6Eca0RaN7sWmfSXcFRkkqvjtVt79KnBN0AGFLjwFyZx9pR0v3Rajh9ufDDASPwQqImPT7XtYhuveYdnicA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1" alt="图片" style="zoom: 50%;" />



### HashMap的扩容机制

明确：首次添加元素和非首次添加元素

- 首次添加元素：
  - 懒加载机制：16*0.75进行扩容
- 非首次添加元素：
  - 红黑树节点插入之前，会判断是否需要扩容
  - 节点插入之后，也会判断是否需要扩容

扩容的流程：

计算下标：hash&oldcap 相当于高的一位参与计算，如果是0，还是在原来的位置，如果是1，相当于原来的位置+容量

## 2.String相关

 ==和equals
