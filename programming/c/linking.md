# Linking

## Basic
通常情况下每个 *.c* 文件是一个独立的 compile unit，会被编译成一个 *.o* 文件，然后由链接器进行合并生成 executable。

比如如下两个文件：

~~~c
// hello.c
#include<stdio.h>
int extern_symbol;

int hello_from_c(int param) {
    printf("Hello from C with param %d", param);
}
~~~

~~~c
//main.c
int hello_from_c(int param);
int weak_symbol;
extern int extern_symbol;

int main() {
    hello_from_c(extern_symbol);
}
~~~

可以分别编译为 relocatable 文件：

~~~text
$ gcc -c hello.c
$ gcc -c main.c
~~~

编写一个 compile unit 时可能会用到其他 compile unit 中的代码或数据。比如 *main.c* 中用到了 *hello.c* 中的 `hello_from_c` 函数。这种引用可以通过 `extern` 关键字来进行声明。函数 declaration 可以省略 `extern` 关键字，因为函数不会是 weak symbol，如果没有进行定义，则只可能是外部的。对于 data 来说，最好是明确写出 `extern` 关键字，否则会被认为是 weak symbol。

~~~
$ readelf -s main.o
Symbol table '.symtab' contains 13 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS main.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1 
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    3 
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4 
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    6 
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT    7 
     7: 0000000000000000     0 SECTION LOCAL  DEFAULT    5 
     8: 0000000000000004     4 OBJECT  GLOBAL DEFAULT  COM weak_symbol
     9: 0000000000000000    24 FUNC    GLOBAL DEFAULT    1 main
    10: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND extern_symbol
    11: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _GLOBAL_OFFSET_TABLE_
    12: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND hello_from_c
~~~

Notice that in `weak_symbol` is in COM pseudosection while `extern_symbol` and `hello_from_c` are both in UND pseudosection. 

之后，在进行链接的时候，编译器需要确认所有 symbol 都只有唯一的定义。所有 undefined symbol 也得有定义。这里分两种情况，静态链接和动态链接。

静态链接的内容会被直接加入最终的可执行文件，相应的符号的 section idx 也会从 UDF/COM 变为具体的数值，一般是 .text 或者 .data 段。

~~~
$ gcc main.o hello.o -o main
$ readelf -s main | grep hello_from_c
    62: 000000000000065f    36 FUNC    GLOBAL DEFAULT   14 hello_from_c
~~~

但动态链接的内容则不会：

~~~
$ gcc -shared -fPIC -Wl,-soname,libhello.so.0  -o libfoo.so.0.0.0 hello.o
$ gcc -L . main.c -l:libhello_c.so.0 -o main
$ readelf -s main | grep hello_from_c
     5: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND hello_from_c
    62: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND hello_from_c
~~~

（偏题一下，注意 main.c 得在 -l:libhello_c.so.0 前面，否则跑不通，回一下 ICS 库为什么得放在后面）
（直接 `gcc main.c libhello_c.so.0 -o` 也是可以的）
（动态库是不能静态链接的，参考这个[问题](https://stackoverflow.com/questions/725472/static-link-of-shared-library-function-in-gcc)，但没说清楚为什么。大概的原因可能是，静态库本质就是一些 relocatable 文件的打包，其内部的 .o 文件还有 .rela.text 段，但 .so 文件内部已经没有这个段了，有的是 .rela.dyn，具体先不细究了）

One line in .dynsym, another in .symtab. `strip` can remove .symtab, but can not remove .dynsym, which is neccessary to run the program.

动态链接实际分两个阶段？第一阶段是编译器产生可执行文件时，进行符号检查，并记录需要的动态库和在 load 时需要解析的 symbol。第一阶段后，仍有 symbol 是 undefined，需要进一步解析。

第二阶段是在 load time。（run time linking 暂时不考虑，和 `dlopen` 有关）。这一阶段再次进行符号解析，修改需要重定位的地方。

遗留问题：

Loader 到底是哪个程序？

Runtime linking 是怎么解析符号的？

`file main` 显示出来的也是 shared object，和 .so 文件的区别在于有一个 interpreter /lib64/ld-linux-x86-64.so.2。但尝试用 main 做动态库进行链接失败了

~~~
$ gcc main.c -L. -l:main -o test
/usr/bin/ld: /tmp/ccTIF21s.o: undefined reference to symbol 'hello_from_c'
libhello_c.so.0: error adding symbols: DSO missing from command line
collect2: error: ld returned 1 exit status
~~~

有些 .so 文件貌似可以执行，但 `./libhello_c.so.0 ` 就 segmentation fault 了。

interpreter /lib64/ld-linux-x86-64.so.2 应该就是 dynamic linker，执行他会给出一些提示，“tell the system's program loader to load the helper program from this file.  This helper program loads the shared libraries needed by the program executable, prepares the program
to run, and runs it.”。从这个意义上来说，确实是个 elf 的interpreter。

main 文件不能执行，ldd 会发现找不到动态库：
~~~
$ ldd main
        linux-vdso.so.1 (0x00007ffcf1ced000)
        libhello.so.0 => not found
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fabdabb6000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fabdb1a9000)
~~~

所以 soname 有啥用，，，
~~~
$ cp libhello.so.0.0.0 libhello.so.0
$ ldd main
        linux-vdso.so.1 (0x00007ffe57c92000)
        libhello.so.0 (0x00007f083535e000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f0834f6d000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f0835762000)
$ objdump -p libhello.so.0.0.0 | grep SONAME
  SONAME               libhello.so.0
$ patchelf --print-soname libhello.so.0.0.0 
libhello.so.0
~~~

编译出新版本的 glibc，既有 libc.so.6，也有 ldd 和 ld-2.31.so，这两个文件是否有版本依赖？

~~~
$ ~/glibc/bin/ldd main
        not a dynamic executable
~~~

是 libc 版本太新了吗，，，

最后是动态库的搜索路径：https://unix.stackexchange.com/questions/22926/where-do-executables-look-for-shared-objects-at-runtime

## ABI
动态链接和静态链接在链接意义上没有太大的不同吧。两个 relocatable 文件想要链接在一起，都需要相同的 ABI。
ABI consists of
	Data layout, which defines how datatype instances are laid out in memory.
	The calling convention, which controls how functions’ arguments are passed and return values are retrieved
	Runtime and Standard library
	Type Metadata, holds information about the types
	Ownership decisions
	Mangling, using which compiler uniquely identifies the names of identifiers across the source.
动态链接更希望有一个稳定的 API，这样 release 的程序对环境的依赖就小很多。对于静态链接，build release 的时候你都静态链接了为啥不一起重新编译一遍。

For C code, every major platform as a de facto standard ABI which is followed by most tools.
For C++ code, the ABI is much more complex than for C. However, for x86 and x86-64, two major platforms, the Itanium ABI is used on a wide variety of platforms, which creates at least a somewhat interoperable environment. In particular, that ABI's rules for name mangling (i.e. how to represent namespace-qualified names and function overload sets as flat strings) are well-known and have widespread tooling support.
C 的 ABI 是稳定的，C++ 至少在 x64 上很稳定，我猜 arm 上也会还好？
C 的 ABI 稳定，但在同一平台不同的系统上也会不一样。比如 Unix 系列用 System V AMD64 ABI，RDI, RSI… 传参。Windows 用  Microsoft x64 calling convention，RCX, RDX, R8, R9 被拿来传参。
传参是一方面，此外还有地址对齐等等。比如 call 之前确保地址 16 字节对齐。

Rust has no intend to provide a stable ABI. 对于 rust 来说，也不太需要。因为 Rust 内部不会用动态链接（Rust 会依赖一些 C 的动态库，但 Rust 编写的包通常不会作为动态库）。
Rust 的函数调用会用栈传参，比起 C 的好处是显然的，能直接传一个结构体而不是必须传指针。但是对于逆向来说就很头疼了。需要去 fix calling convention。这里貌似有个相关的 [ida script](https://github.com/cha5126568/rust-reversing-helper)。

下一个问题，如果需要调用一个外部的函数，需要什么。需要知道在哪个库里，函数名是什么。先从 Rust 调用 C/C++ 开始考虑。
Rust is currently unable to call directly into a C++ library。
那就先考虑 C 吧，，，
库的话应该还是用 soname 可以来确定？静态库咋整？函数的话，在 C 中没有 namespace，也没有 class/obj method 就不需要考虑 demangle 的问题。找函数名还是很方便的。

https://unix.stackexchange.com/questions/22926/where-do-executables-look-for-shared-objects-at-runtime








## Symbol Resolving

## Useful Commands & Files & Env Vars


### `ld`

### `ld.so`

### `ldd`
### `libc.so.6`

### LD_PRELOAD

### LD_LIBRARY_PATH