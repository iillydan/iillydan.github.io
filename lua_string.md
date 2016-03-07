Lua源码之字符串
=============================
-----------------------------
为了提高效率，Lua针对长短字符串采用不同的处理方式，用LUAI_MAXSHORTLEN来划分长短字符串，小于等于LUAI_MAXSHORTLEN的被当成短字符串处理，5.2版本中默认LUAI_MAXSHORTLEN设置为40。Lua中的字符串由TString头部加上实际的字符串内容构成，并最终以'\0'结尾。TString结构如下：

    typedef union TString {
        L_Umaxalign dummy;   /* ensures maximum alignment for strings */
        struct {
          CommonHeader;      //对象通用头部
          lu_byte extra;     //保留关键字（短字符串）的ID或者是长字符串是否有hash值的标志位
          unsigned int hash; //hash值
          size_t len;        /* number of characters in string */
        } tsv;
    } TString;

TString实际是个Union，dummy字段除了确保TString结构体的对其长度外没有其他用处。extra对于长短字符串的含义不同，对于短字符串如果命中Lua的保留关键字，则其值等于该关键字在关键字列表中的索引偏移，对于长字符串用于表示该字符串是否已经计算过hash值。hash和len分别是字符串的hash值及其长度。

Lua中使用一个全局的stringtable来管理短字符串，对于短字符串全局只有一个实例。例如每次创建一个短字符串的时候，首先去stringtable里面寻找是否存在该短字符串，如果有直接返回该实例，若没有则创建后更新stringtable。这也说明Lua的字符串是不可修改的，只能新建。stringtable并没有直接使用lua的table，而是直接采用外部拉链的方式，直接利用CommonHeader中的next指针来建立冲突链表。所以短字符串比较时只需要判断指针是否相等即可。

另外，Lua的Userdata的结构也类似，也是一个头部后面紧跟一段内存，其头部结构也类似：

      typedef union Udata {
        L_Umaxalign dummy;  /* ensures maximum alignment for `local' udata */
        struct {
          CommonHeader;
          struct Table *metatable;
          struct Table *env;
          size_t len;  /* number of bytes */
          } uv;
        } Udata;
