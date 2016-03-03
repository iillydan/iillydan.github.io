Lua源码之table
=================
-----------------
Lua里面最重要的数据结构或者说唯一内置的数据结构就是table了，并且我们可以用table模拟出许多其他语言中常见的各种数据类型，甚至是用来模拟OO编程。在语言实现的层面，table同样是及其重要的数据结构，Lua解释器同样也大量使用了table。

table相关的操作函数都是以luaH开头，推测H应该是Hashtable的意思。主要功能相关的实现在ltable.c中，对应的头文件是ltable.h，table对应的结构体是定义在lobject.h中的：

    typedef struct Table{
      CommonHeader;               //对象通用头部
      lu_byte flags;              //cache tag method是否存在
      lu_byte lsizenode;          //node数组长度的log2的值
      struct Table* metatable;    //指向对应的metatable的指针
      TValue* array;              //数组部分
      Node* node;                 //table节点
      Node* lastfree;             //空闲节点
      GCObject* gclist;           //指向所在的gc队列
      int sizearray;              //数组长度
    }Table;
其中CommonHeader和gclist主要是跟GC工作相关。根据注释描述，Lua table采用的是chained scatter table和Brent's变种的混合技术。Google了一下chained scatter table，其主要特点是：

- table由一个数组构成，数组的每个元素包含key和一个指针（数组下标）；
- 当发生冲突时，元素插入到数组中尚未被占用的位置，元素中的指针来构造冲突链表；

Brent's是一种二次hash的方案，在遇到冲突时，不断加上另一个Hash函数的hash结果，直到找到空位为止。
