# C programming
## size_t, ssize_t, loff_t
* size_t (`typedef __kernel_size_t size_t`)
    * for sparc 64 bit : `unsigned long`
    * for sparc 32 bit : `unsigned int`
* ssize_t (`typedef __kernel_ssize_t ssize_t`)
    * for sparc 64 bit : `long`
    * for sparc 32 bit : `int`
    * ssize_t = `signed` size_t
* loff_t (`typedef __kernel_loff_t loff_t`)
    * `long long`

## offsetof, container_of
* offsetof 
    * `<linux/stddef.h>`
    * `#define offsetof(TYPE, MEMBER) ( (size_t) &((TYPE *)0)->MEMBER )`
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
