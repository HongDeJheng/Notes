# C programming notes
## size_t, ssize_t, loff_t
* size_t
     ```C
      typedef __kernel_size_t size_t
     ```
    * for sparc 64 bit : `unsigned long`
    * for sparc 32 bit : `unsigned int`
* ssize_t 
    ```C
     typedef __kernel_ssize_t ssize_t
    ```
    * for sparc 64 bit : `long`
    * for sparc 32 bit : `int`
    * ssize_t = `signed` size_t
* loff_t 
    ```C
     typedef __kernel_loff_t loff_t
    ```
    * `long long`

## offsetof, container_of
* offsetof 
    * `<linux/stddef.h>`
    * ```C
      #define offsetof(TYPE, MEMBER) ( (size_t) &((TYPE *)0)->MEMBER )
      ```
    * Get the offset of a structure member (distance from member to struct head)
* container_of
    * `<linux/kernel.h>`
    * ```C
      container_of(ptr, type, member) 
      { 
          const typeof( (type *)0->member ) *__mptr = ptr;  
          (type *) ( (char *)__mptr - offsetof(type, member) );
          
      }
      ```
    * Get the head address of struct base on any member

## \_\_attribute__
* Is a GCC extension, which can make compiler to do some special process or checking while compiling
* Three category : `Function Attribute`, `Variable Attribute`, and `Type Attribute`
* Syntax : `__attribute__ ((attriibute-list))`
* For more `attribute-list` details, see the website below
    * [`Function Attributes`](https://gcc.gnu.org/onlinedocs/gcc/Function-Attributes.html)
    * [`Variable Attributes`](https://gcc.gnu.org/onlinedocs/gcc/Variable-Attributes.html#Variable-Attributes)
    * [`Type Attributes`](https://gcc.gnu.org/onlinedocs/gcc/Type-Attributes.html#Type-Attributes)
    * https://winddoing.github.io/post/12087.html
    * https://www.jollen.org/blog/2006/10/_gcc___attribute.html

## __user, __kernel, __safe, __force, __iomem
```C
# define __bitwise   __attribute__( (bitwise) )
# define __user      __attribute__( ( noderef, address_space(1) ) )
# define __kernel    __attribute__( ( address_space(0) ) )
# define __safe      __attribute__( (safe) )
# define __force     __attribute__( (force) )
# define __nocast    __attribute__( (nocast) )
# define __iomem     __attribute__( ( noderef, address_space(2) ) )
```

* `__bitwise`
    * Check whether the variable is in the same endianness or not
    * e.g. `typedef int __bitwise snd_device_type_t;`
* `__user`
    * This annotation notes that the pointer is a `user-space address` that __cannot be directly dereferenced__.
    * For normal compilation, `__user` has no effect, but it can be used by external checking software to find __misuse__ of `user-space address`.
* `__kernel`
    * The pointer is a `kernel-space address`
* `__iomem`
    * The pointer is a `device-space (I/O-space) address`
* `__safe`
    * Check whether the variable is `NULL` or not
* `__force`
    * Force to do type casting
* `__nocast`
    * The variable that pass into a function should be exactly the same to definition
    * e.g. `ktrace_alloc(int nentries, unsigned int __nocast sleep)` 
    sleep should be `unsigned int`
* Notes:
    * `noderef` : pointer address
    * `address_space(n)`
        * `address_space(0)` : Kernel Space
        * `address_space(1)` : User Space
        * `address_space(2)` : I/O Space
        * `address_space(3)` : CPU Space
        * `address_space(4)` : RCU
    
* [Reference](https://www.twblogs.net/a/5c45e178bd9eee35b21eefb9)    

## copy_from_user, copy_to_user
### `copy_from_user`
```C
unsigned long copy_from_user(void *to, const void *from, unsigned long n)
```
* `to` : destination - Kernal Space
* `from` : source - User Space
* `n` : bytes of data to be copied
* return value : 
    * success : 0
    * fail : bytes that failed (e.g. total `n` bytes, only success `10`, return `n - 10`)
### `copy_to_user`
```C
unsigned long copy_to_user(void *to, const void *from, unsigned long n)
```
* `to` : destination - User Space
* `from` : source - Kernel Space
* `n` : bytse of data to be copied
* return value : 
    * success : 0
    * fail : bytes that failed (e.g. total `n` bytes, only success `10`, return `n - 10`)

## `extern`, `static`, `inline`, `const`, `global`
### Compiling, linking and foward declaration
* While compiling, compiler only see the source file that it is going to compile, compiler is not able to see other related (linked/dependent) source file or library.
* Thus, for each source file, we need to have declaration of any function or variable (full definition or only type declaration).
* Schematic diagram
  ```
  +---------------------------------------------------------------+
  |                                                               |
  |   source code             object code            executable   |
  |                                                               |
  |                                                               |
  |  +----------+             +----------+                        |
  |  |          |   compile   |          |                        |
  |  |  main.c  +------------->  main.o  +-+                      |
  |  |          |             |          | |        +-----------+ |
  |  +----------+             +----------+ |  link  |           | |
  |                                        +-------->   a.out   | |
  |  +----------+             +----------+ |        |           | |     
  |  |          |   compile   |          | |        +-----------+ |   
  |  |  test.c  +------------->  test.o  +-+                      |   
  |  |          |             |          |                        |
  |  +----------+             +----------+                        |
  |                                                               |
  +---------------------------------------------------------------+
  ```
* Notes: header file won't be compiled alone, it will be `#include` into other source code (like paste on), and then compile with the source code.

### `extern`
* `extern` means __declare__ but no __definition__
* When we need to use variables defined by external source file, we need to declare the variable as `extern` with the `type` of referenced variable.
  * e.g.
      * test.c
      ```C
      int a = 10;
      ```
      * main.c
      ```C
      extern int a;
  
      int main()
      {
          printf("a = %d\n", a);    // a = 10
          a++;
          printf("a = %d\n", a);    // a = 11
          a = 100;
          printf("a = %d\n", a);    // a = 100
      }
      ```
  * `extern` can only access global variable
  * The variable with `extern` must have the same type as the referenced variable.
  * `extern` variable cannot be initialized
* We can also use `extern` with header file
    * e.g.
        * test.h
          ```C
          extern int a;
          ```
        * test.c
          ```C
          int a = 10;
          ```
        * main.c
          ```C
          #include "test.h"
          
          int main()
          {
              printf("a = %d\n", a);    // a = 10
              a++;
              printf("a = %d\n", a);    // a = 11
              a = 100;
              printf("a = %d\n", a);    // a = 100
          }
          ```
  * In fact, the global variables that appear in header file basically are `extern` variable.
  * If we declare global variables that are not `extern` or `static`, and `#include` it to source file, we might get `multiple definition` error.
* `extern` variable in function
    * ```C
      int test(){
          extern int a;
          ...
      }
      
      int main()
      {
          a = 30;    // Error!
          ...
      }
      ```
    * If we use `extern` in function, the scope of the variable is same as function's local variable.

### `static`
* In _C programming_, `static` has two main effect : 
    1. `static` global variable and function:
        * Variable cannot be accessed by external file, and also won't affect other file's namespace.
        * Internal linkage
    2. `static` local variable : 
        * Expand the lifetime of variable as same as global variable.
        * For whole program, unlike local variable in function that we can have plenty of variables with the same name, only __one__ variable with that name exists.