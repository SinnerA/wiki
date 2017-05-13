---
title: PHP内核及扩展入门
date: 2017-05-12
---

[TOC]

#### CGI

```
CGI处理步骤： 
1. 通过Internet把用户请求送到服务器
2. 服务器接收用户请求并交给CGI程序处理
3. CGI程序把处理结果传送给服务器
4. 服务器把结果送回到用户
```

​	这里的CGI 是狭义上的 CGI，即不包含 FastCGI。
​	对一个 CGI 程序，做的工作其实只有：从**环境变量(environment variables)**和**标准输入(standard input)**中读取数据、处理数据、向**标准输出(standard output)**输出数据。
​	环境变量中存储的叫 **Request Meta-Variables**，也就是诸如 **QUERY_STRING**、**PATH_INFO** 之类的东西，这些是由 Web Server 通过环境变量传递给 CGI 程序的，CGI 程序也是从环境变量中读取的。
​	标准输入中存放的往往是用户通过 PUTS 或者 POST 提交的数据，这些数据也是由 Web Server 传过来的。

​	现在用 CGI 的已经很少了，因为每个 CGI 进程只处理一个请求，换句话说，每个请求都需要创建一个 CGI 进程处理，CGI 程序处理完毕后就退出了。
​	FastCGI 正是对 CGI 的改进，而且改进了不是一点点。从总体上看，一个 FastCGI 进程可以处理若干请求（一般 FastCGI 进程是驻留着的，但不排除 IIS 之类的 Web Server 限制其空闲时间，在一段时间内没有请求就自动退出的可能），Web Server 或者 fpm 会控制 FastCGI 进程的数量。
​	细节方面，FastCGI 是一套协议，不再是通过简单的环境变量、标准输入和标准输出来接收和传递数据了。一般来说，FastCGI 用 TCP 或者**命名管道(Named Pipe)**传输数据。
​	现在绝大多数 PHP 网站都是在用 FastCGI 的。

#### SAPI(Server abstraction API)

​	简单来说, SAPI就是PHP和外部环境的代理器。它把外部环境抽象后, 为内部的PHP提供一套固定的, 统一的接口, 使得PHP自身实现能够不受错综复杂的外部环境影响，保持一定的独立性

#### 变量类型

​	PHP在内核中是通过zval这个结构体来存储变量的，它的定义在Zend/zend.h文件里，简短精炼，只有四个成员组成：

```
struct _zval_struct {
    zvalue_value value; /* 变量的值 */
    zend_uint refcount__gc;
    zend_uchar type;    /* 变量当前的数据类型 */
    zend_uchar is_ref__gc;
};
typedef struct _zval_struct zval;
 
//在Zend/zend_types.h里定义的：
typedef unsigned int zend_uint;
typedef unsigned char zend_uchar;
```

​	PHP语言得以实现了8种数据类型：

| **常量名称：**   |                                          |
| ----------- | ---------------------------------------- |
| IS_NULL     | 第一次使用的变量如果没有初始化过，则会自动的被赋予这个常量，当然我们也可以在PHP语言中通过null这个常量来给予变量null类型的值。 这个类型的值只有一个 ，就是NULL，它和0与false是不同的。 |
| IS_BOOL     | 布尔类型的变量有两个值，true或者false。在PHP语言中，while、if等语句会自动的把表达式的值转成这个类型的。 |
| IS_LONG     | PHP语言中的整型，在内核中是通过所在操作系统的signed long数据类型来表示的。 在最常见的32位操作系统中，它可以存储从-2147483648 到 +2147483647范围内的任一整数。 有一点需要注意的是，如果PHP语言中的整型变量超出最大值或者最小值，它并不会直接溢出， 而是会被内核转换成IS_DOUBLE类型的值然后再参与计算。 再者，因为使用了signed long来作为载体，所以这也就解释了为什么PHP语言中的整型数据都是带符号的了。[?](http://www.cunmou.com/phpbook/2.1.md#)`$a=2147483647;``$a++;``echo $a;``//会正确的输出    2147483648；` |
| IS_DOUBLE   | PHP中的浮点数据是通过C语言中的signed double型变量来存储的， 这最终取决与所在操作系统的浮点型实现。 我们做为程序猿，应该知道计算机是无法精准的表示浮点数的， 而是采用了科学计数法来保存某个精度的浮点数。 用科学计数法，计算机只用8位便可以保存2.225x10^(-308)~~1.798x10^308之间的浮点数。 用计算机来处理浮点数简直就是一场噩梦，十进制的0.5专成二进制是0.1， 0.8转换后是0.1100110011....。 但是当我们从二进制转换回来的时候，往往会发现并不能得到0.8。 我们用1除以3这个例子来解释这个现象：1/3=0.3333333333.....，它是一个无限循环小数， 但是计算机可能只能精确存储到0.333333，当我们再乘以三时， 其实计算机计算的数是0.333333*3=0.999999，而不是我们平时数学中所期盼的1.0. |
| IS_STRING   | PHP中最常用的数据类型——字符串，在内存中的存储和C差不多， 就是一块能够放下这个变量所有字符的内存，并且在这个变量的zval实现里会保存着指向这块内存的指针。 与C不同的是，PHP内核还同时在zval结构里保存着这个字符串的实际长度， 这个设计使PHP可以在字符串中嵌入‘\0’字符，也使PHP的字符串是二进制安全的， 可以安全的存储二进制数据！本着艰苦朴素的作风，内核只会为字符串申请它长度+1的内存， 最后一个字节存储的是‘\0’字符，所以在不需要二进制安全操作的时候， 我们可以像通常C语言的方式那样来使用它。 |
| IS_ARRAY    | 数组是一个非常特殊的数据类型，它唯一的功能就是聚集别的变量。 在C语言中，一个数组只能承载一种类型的数据，而PHP语言中的数组则灵活的多， 它可以承载任意类型的数据，这一切都是HashTable的功劳， 每个HashTable中的元素都有两部分组成：索引与值， 每个元素的值都是一个独立的zval（确切的说应该是指向某个zval的指针）。 |
| IS_OBJECT   | 和数组一样，对象也是用来存储复合数据的，但是与数组不同的是， 对象还需要保存以下信息：方法、访问权限、类常量以及其它的处理逻辑。 相对与zend engine V1，V2中的对象实现已经被彻底修改， 所以我们PHP扩展开发者如果需要自己的扩展支持面向对象的工作方式， 则应该对PHP5和PHP4分别对待！ |
| IS_RESOURCE | 有一些数据的内容可能无法直接呈现给PHP用户的， 比如与某台mysql服务器的链接，或者直接呈现出来也没有什么意义。 但用户还需要这类数据，因此PHP中提供了一种名为Resource(资源)的数据类型。 有关这个数据类型的事宜将在第九章中介绍，现在我们只要知道有这么一种数据类型就行了。 |

#### 内存管理

- PHP变量的名称和值在内核中是保存在两个不同的地方的，值是通过一个与名字毫无关系的zval结构来保存，而这个变量的名字a则保存在符号表里，两者之间通过指针联系着
- 引用计数：赋值的时候，使用引用计数机制，如果删除某个变量，它删除符号表里的信息，并将计数减一，计数为0才释放内存
- 写时复制：修改某个变量时，计数 > 2，则开辟新内存复制一份，并修改返回
- 写时更改：一个变量a引用另一个变量b时，b改变，a也相应改变

#### PHP编译

PHP编译前的configure有两个特殊的选项，打开它们对我们开发PHP扩展或者进行PHP嵌入式开发时非常有帮助。但是当我们正常使用PHP的时候，则不应该开启这两个选项。

#### Zend API

- ##### 变量

  ```c
  获取类型：
  #define Z_TYPE(zval)        (zval).type
  #define Z_TYPE_P(zval_p)    Z_TYPE(*zval_p)
  #define Z_TYPE_PP(zval_pp)  Z_TYPE(**zval_pp)

  打印：
  php_printf()  内核对printf()函数的一层封装，我们可以像使用printf()函数那样使用它
  PHPWRITE(Z_STRVAL_P, Z_STRLEN_P)  打印string

  有关值操作的宏都定义在./Zend/zend_operators.h文件里：
  //操作整数的
  #define Z_LVAL(zval)            (zval).value.lval
  #define Z_LVAL_P(zval_p)        Z_LVAL(*zval_p)
  #define Z_LVAL_PP(zval_pp)      Z_LVAL(**zval_pp)
   
  //操作IS_BOOL布尔型的
  #define Z_BVAL(zval)            ((zend_bool)(zval).value.lval)
  #define Z_BVAL_P(zval_p)        Z_BVAL(*zval_p)
  #define Z_BVAL_PP(zval_pp)      Z_BVAL(**zval_pp)
   
  //操作浮点数的
  #define Z_DVAL(zval)            (zval).value.dval
  #define Z_DVAL_P(zval_p)        Z_DVAL(*zval_p)
  #define Z_DVAL_PP(zval_pp)      Z_DVAL(**zval_pp)
   
  //操作字符串的值和长度的
  #define Z_STRVAL(zval)          (zval).value.str.val
  #define Z_STRVAL_P(zval_p)      Z_STRVAL(*zval_p)
  #define Z_STRVAL_PP(zval_pp)    Z_STRVAL(**zval_pp)
   
  #define Z_STRLEN(zval)          (zval).value.str.len
  #define Z_STRLEN_P(zval_p)      Z_STRLEN(*zval_p)
  #define Z_STRLEN_PP(zval_pp)    Z_STRLEN(**zval_pp)
   
  #define Z_ARRVAL(zval)          (zval).value.ht
  #define Z_ARRVAL_P(zval_p)      Z_ARRVAL(*zval_p)
  #define Z_ARRVAL_PP(zval_pp)    Z_ARRVAL(**zval_pp)
   
  //操作对象的
  #define Z_OBJVAL(zval)          (zval).value.obj
  #define Z_OBJVAL_P(zval_p)      Z_OBJVAL(*zval_p)
  #define Z_OBJVAL_PP(zval_pp)    Z_OBJVAL(**zval_pp)
   
  #define Z_OBJ_HANDLE(zval)      Z_OBJVAL(zval).handle
  #define Z_OBJ_HANDLE_P(zval_p)  Z_OBJ_HANDLE(*zval_p)
  #define Z_OBJ_HANDLE_PP(zval_p) Z_OBJ_HANDLE(**zval_p)
   
  #define Z_OBJ_HT(zval)          Z_OBJVAL(zval).handlers
  #define Z_OBJ_HT_P(zval_p)      Z_OBJ_HT(*zval_p)
  #define Z_OBJ_HT_PP(zval_p)     Z_OBJ_HT(**zval_p)
   
  #define Z_OBJCE(zval)           zend_get_class_entry(&(zval) TSRMLS_CC)
  #define Z_OBJCE_P(zval_p)       Z_OBJCE(*zval_p)
  #define Z_OBJCE_PP(zval_pp)     Z_OBJCE(**zval_pp)
   
  #define Z_OBJPROP(zval)         Z_OBJ_HT((zval))->get_properties(&(zval) TSRMLS_CC)
  #define Z_OBJPROP_P(zval_p)     Z_OBJPROP(*zval_p)
  #define Z_OBJPROP_PP(zval_pp)   Z_OBJPROP(**zval_pp)
   
  #define Z_OBJ_HANDLER(zval, hf)     Z_OBJ_HT((zval))->hf
  #define Z_OBJ_HANDLER_P(zval_p, h)  Z_OBJ_HANDLER(*zval_p, h)
  #define Z_OBJ_HANDLER_PP(zval_p, h) Z_OBJ_HANDLER(**zval_p, h)
   
  #define Z_OBJDEBUG(zval,is_tmp)       (Z_OBJ_HANDLER((zval),get_debug_info)?  \
                          Z_OBJ_HANDLER((zval),get_debug_info)(&(zval),&is_tmp TSRMLS_CC): \
                          (is_tmp=0,Z_OBJ_HANDLER((zval),get_properties)?Z_OBJPROP(zval):NULL)) 
  #define Z_OBJDEBUG_P(zval_p,is_tmp)   Z_OBJDEBUG(*zval_p,is_tmp) 
  #define Z_OBJDEBUG_PP(zval_pp,is_tmp) Z_OBJDEBUG(**zval_pp,is_tmp)
   
  //操作资源的
  #define Z_RESVAL(zval)          (zval).value.lval
  #define Z_RESVAL_P(zval_p)      Z_RESVAL(*zval_p)
  #define Z_RESVAL_PP(zval_pp)    Z_RESVAL(**zval_pp)
  ```

- ##### 创建变量

  ```c
  ZVAL_NULL(pvz)                 注意这个Z和VAL之间没有下划线
  ZVAL_BOOL(pzv, b)              将pzv所指的zval设置为IS_BOOL类型，值是b
  ZVAL_TRUE(pzv)                 将pzv所指的zval设置 IS_BOOL类型，值是true
  ZVAL_FALSE(pzv)                将pzv所指的zval设置为IS_BOOL类型，值是false
  ZVAL_LONG(pzv, l)              将pzv所指的zval设置为IS_LONG类型，值是l	
  ZVAL_DOUBLE(pzv, d)            将pzv所指的zval设置为IS_DOUBLE类型，值是d
  ZVAL_STRINGL(pzv,str,len,dup)  dup指明了str是否需要被复制，1表示申请新内存，0表示直接返回str地址
  ZVAL_STRING(pzv, str, dup)	   想在某一位置截取str或已经知道str的长度，使用该函数
  ZVAL_RESOURCE(pzv, res)        资源类型的值其实就是一个整数，所以ZVAL_RESOURCE和ZVAL_LONG差不多
  ```

- ##### 类型转换

  ```c
  void convert_to_long(zval *op)
  void convert_to_double(zval *op)
  void convert_to_null(zval *op)
  void convert_to_boolean(zval *op)
  void convert_to_array(zval *op)
  void convert_to_object(zval *op)
  ```

- ##### 内存管理

  ```c
  //有些时候，某次申请的内存需要在一个请求结束后仍然存活一段时间，也就是持续性存在于各个请求之间。这种类型的分配被称为"永久性分配"，因此多了persistent参数
  void *emalloc(size_t count);                        void *malloc(size_t count);	
  void *pemalloc(size_t count, char persistent);

  void *ecalloc(size_t count);                        void *calloc(size_t count);	
  void *pecalloc(size_t count, char persistent);

  void *erealloc(void *ptr, size_t count);            void *realloc(void *ptr, size_t count);	
  void *perealloc(void *ptr, size_t count, char persistent);

  void *estrdup(void *ptr);                           void *strdup(void *ptr);	
  void *pestrdup(void *ptr, char persistent);

  void efree(void *ptr);                              void free(void *ptr);
  void pefree(void *ptr, char persistent);
  ```

- ##### 定义函数

  ```c
  ZEND_FUNCTION                       函数声明，与PHP_FUNCTION同义(前者写法更标准)
  #define ZEND_FUNCTION(name)         ZEND_NAMED_FUNCTION(ZEND_FN(name))
  #define ZEND_NAMED_FUNCTION(name)   void name(INTERNAL_FUNCTION_PARAMETERS)
  #define ZEND_FN(name)               zif_##name
  ```

- ##### 返回值

  ```c
  //这些宏都定义在Zend/zend_API.h文件里
  #define RETVAL_RESOURCE(l)              ZVAL_RESOURCE(return_value, l)
  #define RETVAL_BOOL(b)                  ZVAL_BOOL(return_value, b)
  #define RETVAL_NULL()                   ZVAL_NULL(return_value)
  #define RETVAL_LONG(l)                  ZVAL_LONG(return_value, l)
  #define RETVAL_DOUBLE(d)                ZVAL_DOUBLE(return_value, d)
  #define RETVAL_STRING(s, duplicate)     ZVAL_STRING(return_value, s, duplicate)
  #define RETVAL_STRINGL(s, l, duplicate) ZVAL_STRINGL(return_value, s, l, duplicate)
  #define RETVAL_EMPTY_STRING()           ZVAL_EMPTY_STRING(return_value)
  #define RETVAL_ZVAL(zv, copy, dtor)     ZVAL_ZVAL(return_value, zv, copy, dtor)
  #define RETVAL_FALSE                    ZVAL_BOOL(return_value, 0)
  #define RETVAL_TRUE                     ZVAL_BOOL(return_value, 1)
   
  #define RETURN_RESOURCE(l)              { RETVAL_RESOURCE(l); return; }
  #define RETURN_BOOL(b)                  { RETVAL_BOOL(b); return; }
  #define RETURN_NULL()                   { RETVAL_NULL(); return;}
  #define RETURN_LONG(l)                  { RETVAL_LONG(l); return; }
  #define RETURN_DOUBLE(d)                { RETVAL_DOUBLE(d); return; }
  #define RETURN_STRING(s, duplicate)     { RETVAL_STRING(s, duplicate); return; }
  #define RETURN_STRINGL(s, l, duplicate) { RETVAL_STRINGL(s, l, duplicate); return; }
  #define RETURN_EMPTY_STRING()           { RETVAL_EMPTY_STRING(); return; }
  #define RETURN_ZVAL(zv, copy, dtor)     { RETVAL_ZVAL(zv, copy, dtor); return; }
  #define RETURN_FALSE                    { RETVAL_FALSE; return; }
  #define RETURN_TRUE                     { RETVAL_TRUE; return; }
  ```

- ##### 传递引用

  ```c
  /*首先先用一个宏函数来生成头部，然后用第二个宏生成具体的数据，最后用一个宏生成尾部代码*/

  /*生成头部*/
  ZEND_BEGIN_ARG_INFO(name, pass_rest_by_reference)
  ZEND_BEGIN_ARG_INFO_EX(name, pass_rest_by_reference, return_reference,required_num_args)

  /*生产具体数据*/
  ZEND_ARG_PASS_INFO(by_ref)
  //强制所有参数使用引用的方式传递
   
  ZEND_ARG_INFO(by_ref, name)
  //如果by_ref为1，则名称为name的参数必须以引用的方式传递，
   
  ZEND_ARG_ARRAY_INFO(by_ref, name, allow_null)
  ZEND_ARG_OBJ_INFO(by_ref, name, classname, allow_null)
  //这两个宏实现了类型绑定，也就是说我们在传递某个参数时，必须是数组类型或者某个类的实例。如果最后的参数为真，则除了绑定的数据类型，还可以传递一个NULL数据。
   
  /*生产尾部*/
   #define ZEND_END_ARG_INFO()
   
  /*我们组合起来使用*/
  ZEND_BEGIN_ARG_INFO(byref_compiletime_arginfo, 0)
  ZEND_ARG_PASS_INFO(1)
  ZEND_END_ARG_INFO()
  ```

- ##### 传递参数

  ```c
  /*
  前几个参数我们直接用内核里宏来生成，形式为：ZEND_NUM_ARGS() TSRMLS_CC，注意没有逗号
  (ZEND_NUM_ARGS()提供参数个数，TSRMLS_CC用于保证线程安全以及返回值未SUCCESS或者FAILURE)
  紧接着需要传递给zend_parse_parameters()函数的参数是一个用于格式化的字符串，类似printf的第一个参数
  */
  zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, type_spec, ...)

  type_spec是格式化字符串，其常见的含义如下：
  参数   代表着的类型
  b   Boolean
  l   Integer 整型
  d   Floating point 浮点型
  s   String 字符串
  r   Resource 资源
  a   Array 数组
  o   Object instance 对象
  O   Object instance of a specified type 特定类型的对象
  z   Non-specific zval 任意类型～
  Z   zval**类型
  f   表示函数、方法名称，PHP5.1里貌似木有... ...
  ```


