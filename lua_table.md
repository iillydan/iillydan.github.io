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
      Node* lastfree;             //用于指示空闲节点的后界
      GCObject* gclist;           //指向所在的gc队列
      int sizearray;              //数组长度
    } Table;

    typedef struct Node {
      TValue i_value;
      TKey i_key;
    } Node;

其中CommonHeader是对象通用头部；gclist主要是跟GC工作相关。根据注释描述，Lua table采用的是chained scatter table和Brent's变种的混合技术。Google了一下chained scatter table，其主要特点是：

- table由一个数组构成，数组的每个元素包含key和一个指针（数组下标）；
- 当发生冲突时，元素插入到数组中尚未被占用的位置，元素中的指针来构造冲突链表；

Brent's是一种二次hash的方案，在遇到冲突时，不断加上另一个Hash函数的hash结果，直到找到空位为止。

Lua table的做法大同小异：当插入一个key到hashtable的时候，先会根据对象的类型选择相应的Hash函数计算hash值，并通过求模（为了减少潜在的冲突，对于数字和字符串类型采用了不同的模数）得到该key在Node* node数组中应该的插入位置即mainposition。得到了mainposition后，先判断mainposition是否已经被占用，若是该位置没有被占用则直接使用该位置；若该位置被占用但当前该位置上的key不是处于自己的mainposition，即只是当成冲突拉链上的节点使用，则将其移动到另外的空闲节点给新插入的；如果该位置上被占用同时，该位置是占用节点的mainposition，则新插入的元素自己找个空位置，并建立从mainposition到自己的冲突拉链。寻找空闲节点时根据lastfree指示的位置从后往前找，并更新lastfree的位置，若是所有节点都已经被占用，则重新分配空间进行rehash。

rehash部分比较复杂。Lua的Table由Hashtable部分和数组部分组成，从而同时支持能够这两种数据结构。这也是Lua的table最有意思的地方：当往table中插入一个整数（大于0同时小于等于数组最大长度）为key的对象时，如果这个key的值比当前的数组部分的长度即sizearray要小，则直接以key为index插入到数组中，如若不是，则插入到Hashtable中。不仅如此，每次进行rehash的时候，都会重新统计array和hashtable中满足数组要求的整数为key的元素个数，这个过程中会决定哪些key会放入到数组，哪些会被存放到hashtable。具体的做法是：将数组索引划分成0~MAXBITS个区间，统计当前array部分加上hash部分中的整数key的落在每个区间2^k~2^(k+1)中的数目。然后据此来确定数组部分的长度即也是key落到array part的区间，其要求是落在数组内的元素要大于数组长度的一半。

比如：

    t = {}
    t[1] = 1
    print(#t) ==> 1   
    t = {}
    t[2] = 1
    print(#t) ==> 0
    --再比如：
    i = 9
    t = {}
    while i <= 16 do
      t[i] = 1
      i = i + 1
    end
    print(#t) ==> 0  --没有满一半,所有数据都在hashtable中
    t[5] = 1
    print(#t) ==> 16  --满了一半，所有数据都迁移到了array中
    t[5] = nil
    t["a"] = 1       --触发rehash
    print(#t) ==> 0  --又不满一半,所有数据又都在hashtable中

-------------------------------------------------------------
###### 2016/03/04
