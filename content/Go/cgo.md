---
title: Go1.6.3-cgo
date: 2017-05-16
---

[TOC]

# Go1.6.3 - cgo

## 引入C代码

若想在Go代码中使用C代码，必须在所使用的包头开头中注释处引入，紧接着下一行导入C包 `import "C"`，即可导入C中定义&实现的函数及变量。`"C"` 包并不真正存在的包，它是个伪Go包类似命名空间，能直接调用C代码导出的任何变量及函数。

### 1. Go中直接引入

```go
package main

// #include <stdio.h>
/*
int foo()
{
    printf("Hello Cgo!\n");
}
*/
import "C"

func main() {
    C.foo()
}
```

```sh
$ go run test.go
Hello Cgo!
```

### 2. 在Go中引入C文件

```c
// test.c
#include <stdio.h>
void foo()
{
    printf("Hello Cgo!\n");
}
```

```go
// test.go
package main

// #include "test.c"
import "C"

func main() {
    C.foo()
}
```

```sh
$ go run test.go
Hello Cgo!
```

### 3. 在Go中引入静态链接库

```c
// test.h
#ifndef _TEST_H_
#define _TEST_H_

#include <stdio.h>

void foo();

#endif
```

```c
// test.c
#include "test.h"

void foo()
{
    printf("Hello Cgo!\n");
}
```

```sh
# 生成静态链接库
$ gcc -c test.c && ar -rv libtest.a test.o
```

```go
package main

// #cgo LDFLAGS: -L./ -ltest
// #include "test.h"
import "C"

func main() {
    C.foo()
}
```

```sh
$ go run test.go
Hello Cgo!
```

### 4. 在Go中引入动态链接库

```sh
# 生成动态链接库
$ gcc -fPIC -shared test.c -o libtest.so
$ sudo cp libtest.so /usr/lib
```

```go
package main

// #cgo LDFLAGS: -ltest
// #include "test.h"
import "C"

func main() {
    C.foo()
}
```

```sh
$ go run test.go
Hello Cgo!
```

`// #cgo LDFLAGS: -L. -ltest` 指定当前目录去搜索动态链接库会报错误，无法运行，因此将动态链接库移至 `/usr/lib` 目录下。

## 数值类型

     C.char, C.schar (signed char), C.uchar (unsigned char), C.short, C.ushort (unsigned short), C.int, C.uint (unsigned int), C.long, C.ulong (unsigned long), C.longlong (long long), C.ulonglong (unsigned long long), C.float, C.double, C.complexfloat (complex float), and C.complexdouble (complex double). 

## 指针 && 数组

     // Go string to C string
     // The C string is allocated in the C heap using malloc.
     // It is the caller's responsibility to arrange for it to be
     // freed, such as by calling C.free (be sure to include stdlib.h
     // if C.free is needed).
     func C.CString(string) *C.char
    
     // Go []byte slice to C array
     // The C array is allocated in the C heap using malloc.
     // It is the caller's responsibility to arrange for it to be
     // freed, such as by calling C.free (be sure to include stdlib.h
     // if C.free is needed).
     func C.CBytes([]byte) unsafe.Pointer
    
     // C string to Go string
     func C.GoString(*C.char) string
    
     // C data with explicit length to Go string
     func C.GoStringN(*C.char, C.int) string
    
     // C data with explicit length to Go []byte
     func C.GoBytes(unsafe.Pointer, C.int) []byte

引自：[Go references to C](https://golang.org/cmd/cgo/#hdr-Go_references_to_C "Go references to C")

### C.CString

```go
package main

// #include <stdio.h>
// #include <stdlib.h>
/*
void print(char *buff)
{
    printf("%s\n", buff);
}
*/
import "C"

import (
    "unsafe"
)

func main() {
    var say string = "Hello Cgo!"
    cSay := C.CString(say)
    defer C.free(unsafe.Pointer(cSay))
    C.print(cSay)
}
```

```sh
$ go run test.go
Hello Cgo!
```

### C.CBytes

此函数在 Go1.7.x 被引入，对于 Go1.7.x 之前的版本，需要从 Go1.7.x 中复制一份 C.CBytes 的实现
```go
package main
// #include <stdio.h>
// #include <stdlib.h>
// #include <unistd.h>
import "C"

import (
    "unsafe"
)

// _Cfunc_CBytes copy by go 1.7.x
func _Cfunc_CBytes(b []byte) unsafe.Pointer {
    p := C.malloc(C.size_t(len(b)))
    pp := (*[1 << 30]byte)(p)
    copy(pp[:], b)
    return p
}

func main() {
    var (
        buff  []byte = []byte("Hello Cgo!")
        cBuff unsafe.Pointer
    )
    cBuff = _Cfunc_CBytes(buff)
    defer C.free(cBuff)
    C.write(C.STDOUT_FILENO, cBuff, C.size_t(len(buff)))
}
```

```sh
$ go run test.go
Hello Cgo!
```

参考：[Convert Go []byte to a C *char](http://stackoverflow.com/questions/35673161/convert-go-byte-to-a-c-char#35675259 "Convert Go []byte to a C *char")

### C.GoString

```go
package main

// #include <stdio.h>
// #include <stdlib.h>
// #include <string.h>
/*
void say(char **buff)
{
    *buff = (char *)malloc(sizeof(char) * 11);
    strncpy(*buff, "Hello Cgo!", 11);
}
*/
import "C"

import (
    "fmt"
    "unsafe"
)

func main() {
    var str *C.char
    C.say(&str)
    if str == nil {
        return
    }
    defer C.free(unsafe.Pointer(str))
    fmt.Println(C.GoString(str))
}
```

## 复杂类型（结构体）


参考资料：
https://golang.org/cmd/cgo/
http://cholerae.com/2015/05/17/%E4%BD%BF%E7%94%A8Cgo%E7%9A%84%E4%B8%80%E7%82%B9%E6%80%BB%E7%BB%93/
http://ybxu-123.blog.163.com/blog/static/594737702014818113159247/
http://blog.csdn.net/a600423444/article/details/7206015
