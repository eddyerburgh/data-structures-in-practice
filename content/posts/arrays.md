---
Title: "Arrays"
Summary: Learn what arrays are, why they're useful, and how they are implemented in C and Ruby.
date: 2019-05-18T12:27:04+01:00
---

In order to master data structures you need a solid understanding of the basics. In this post you'll learn what arrays are, why they are useful, and how they are implemented in C and Ruby.<!--more-->

## What is an array?

An array is a data structure that holds a collection of elements.

For example, this is an array in the Ruby programming language:

```ruby
arr = ['a','b','c','d','e']
```

Generally, an array is **a contiguous piece of memory**, where each element exists one after the other.

You can imagine an array as a list of boxes containing a value and a number (an _index_) to identify the box.

![An array](/images/arrays/array.svg)

Array elements are accessed by their index number. Normally arrays are 0 indexed, meaning the first element is at index `0`:

```ruby
arr[0] #=> "a"
arr[1] #=> "b"
```

Arrays provide **random access** to data. Theoretically, it takes the same amount of time to access an element at index `1` as it does to access an element at index `1000`. That's an O(1) operation in [Big O notation](../big-o).

There are two common array implementations in programming languages: static arrays, and dynamic arrays. Static arrays cannot grow in size once they have been defined, whereas dynamic arrays can.

_Note: static arrays shouldn't be confused with C arrays defined with the `static` keyword. In this post, I use static to mean an array that can't increase in size once defined._

C arrays are static. To define an array in C, you provide the data type that the array will hold (for example `int`) and the size of the array:

```c
int arr[5];
```

Once defined, you can access and set array data using an index:

```c
arr[0] = 0;
```

But you can't increase the number of elements that the array can hold.

In contrast, dynamic arrays grow to accommodate extra elements. Dynamic arrays are often implemented as a class with methods to operate on the array, like the `push` method:

```ruby
arr = ['a','b']

arr.push('c') #=> ["a", "b", "c"]
```

Arrays are a fundamental data structure, and almost every language implements them, but why should you use them?

## Why use arrays?

Random memory access is the most compelling reason to use an array. Algorithms like [binary search](https://en.wikipedia.org/wiki/Binary_search_algorithm) use random access to speed up tasks by orders of magnitude.

Although access is fast, other array operations are expensive. Take a look at the worst case complexity table for array operations:

| Operation | Worst case |
| --------- | ---------- |
| Access    | O(1)       |
| Search    | O(n)       |
| Insertion | O(n)       |
| Deletion  | O(n)       |

Other data structures like stacks, and hash-tables offer improved search, insertion, and deletion times. But random access is so damn powerful that many of the other data structures are built using arrays.

## How are arrays implemented?

Because arrays are often a language feature, you need to look at the level below the language to see how they are implemented.

In this section you'll learn how static arrays are implemented by looking at the assembly code generated from a C array, and then how dynamic arrays are implemented by looking at the code of a Ruby interpreter.

### Implementing static arrays in C

C is a compiled language. In order to execute a C program, you must first convert it to machine code by running it through a compiler.

Arrays in C contain _homogenous data_, where each element is of the same data type:

```c
int arr[5];

arr[0] = 1;
```

A C compiler uses the data type and size of an array to allocate space in memory. For example, on my Mac an `int` is 4 bytes, So an array that can hold 5 `int` values requires 20 bytes of memory.

![An array showing byte allocation](/images/arrays/array-with-bytes.svg)

When C is compiled to machine code, array elements are accessed by their memory address, which is calculated using the element index and data type. You can see how this works by looking at the assembly code generated from a C program.

_Note: assembly code is human-readable source code that's very close to actual machine code._

Take the following C code that declares an array `arr` with space for 100 `int` elements:

_Note: `extern` is used to make the assembly easier to understand. `extern` tells the compiler that the memory for `arr` is allocated elsewhere._

```c
extern int arr[100];

arr[0] = 1;
arr[1] = 2;
arr[99] = 3;
```

The base memory address of `arr` is represented as `arr(%rip)` in the assembly code. So the following code moves a value of 1 (`$1`) to the array base memory address:

```S
movl	$1, _arr(%rip)
```

Which is the equivalent of:

```c
arr[0] = 1;
```

The assembly accesses array elements at different indexes by adding an _offset_ to the array base address, calculated as `index * data_type_bytes`. For example, index `1` is `4` bytes from the base address, because the index (`1`) multiplied by the number of bytes of the data type (`4`) is `4`:

```S
movl	$2, _arr+4(%rip)
```

And index `99` is 396 bytes (`99 * 4`) from the base address:

```S
movl	$3, _arr+396(%rip)
```

As you can see, arrays in C translate very closely to assembly code.

Because C arrays are static and can't grow in size, they can be difficult to work with. Dynamic arrays solve this problem.

### Implementing dynamic arrays in Ruby

Ruby is an interpreted language. Instead of being compiled to machine code, Ruby programs are run by another program known as an interpreter. There are many different Ruby interpreters, but I'll show you how Ruby arrays are implemented in the original Ruby interpreter written in C (CRuby).

<!-- prettier-ignore-start -->
_Note: The code examples are from CRuby v1\_0, which uses legacy C syntax. The current CRuby array implementation contains optimizations that make it more difficult to explain, but it works in a similar way._
<!-- prettier-ignore-end -->

Unlike C arrays, Ruby arrays are implemented as a class with methods for operating on the array. For example the `push` method, which adds new elements to an array:

```ruby
arr = [0,1]

arr.push(2) #=> [0, 1, 2]
```

Ruby arrays can also handle different data types:

```ruby
arr = ['str', 1, nil]
```

So how is this dynamic array implemented?

The basic idea is to create a wrapper object to manage the array data. The wrapper handles access to the array elements, and reallocates memory if the array needs to increase in size.

In CRuby an array is defined in a C struct—`RArray`. `RArray` contains a `len` value, which is the current number of elements in the array (known as the _length_). It also contains a `capa` value, which is the maximum capacity of the array, and a `ptr` which is a pointer (reference) to a contiguous chunk of memory.

```c
struct RArray {
    struct RBasic basic;
    UINT len, capa;
    VALUE *ptr;
};
```

The memory that `ptr` points to contains a contiguous chunk of memory with space for `capa` number of elements.

![An RArray object](/images/arrays/ruby-array.svg)

Notice that the `ptr` data type is `VALUE`:

```c
struct RArray {
    // ..
    VALUE *ptr;
};
```

`VALUE` represents any valid Ruby object, either as a pointer or as the value itself. This is how Ruby handles multiple data types in the same array.

Ruby achieves random access to array elements by using the pointer (`ptr`) and accessing the element at the offset. You can see this in the CRuby function that accesses an element at a given offset (`ary_entry`):

```c
ary_entry(ary, offset)
    struct RArray *ary;
    int offset;
{
    // ..

    return ary->ptr[offset];
}
```

To learn how Ruby dynamically increases the array capacity, you can look at the implementation of the `<<` (append) method. `<<` adds a new element to the end of an array:

```ruby
arr = []
arr << 0 #=> [0]
arr << 1 #=> [0,1]
```

Internally, Ruby uses the function `ary_push` to implement the `<<` method.

`ary_push` calls a helper function `ary_store` to store an element (`item`) at the index of the arrays current length (`ary->len`). This has the effect of appending the element to the array, because the last item in a Ruby array is always at index `length - 1`:

```c
ary_push(ary, item)
    struct RArray *ary;
    VALUE item;
{
    ary_store(ary, ary->len, item);
    return (VALUE) ary;
}
```

The `ary_store` function is where the magic happens. It attempts to add a new element (`val`) to `ary -> ptr` at the specified index, but will reallocate memory if needed.

`ary_store` first checks that the index for the new array element (`idx`) can fit in the currently assigned memory (by checking the `ary -> capa` value). If it can't, then `ary_store` allocates more memory using `REALLOC_N` (which uses `realloc` internally):

_Note: `realloc` expands the current area of memory if possible. If there isn't enough free memory following the existing area `realloc` allocates a new memory block, copies the old memory area to the newly allocated memory, and frees the old memory area_

```c
void ary_store(ary, idx, val)
    struct RArray *ary;
    int idx;
    VALUE val;
{
  if (idx >= ary->capa) {
    ary->capa = idx + ARY_DEFAULT_SIZE;
    REALLOC_N(ary->ptr, VALUE, ary->capa);
  }

 // ..
}
```

`ary_store` then updates the `ary -> len` value, and adds the new item to the array `ptr` at the index (`idx`):

```c
void ary_store(ary, idx, val)
    struct RArray *ary;
    int idx;
    VALUE val;
{
  // ..

  if (idx >= ary->len) {
    ary->len = idx + 1;
  }
  ary->ptr[idx] = val;
}
```

The full function looks like this:

```c
void ary_store(ary, idx, val)
    struct RArray *ary;
    int idx;
    VALUE val;
{
  if (idx >= ary->capa) {
    ary->capa = idx + ARY_DEFAULT_SIZE;
    REALLOC_N(ary->ptr, VALUE, ary->capa);
  }

  if (idx >= ary->len) {
    ary->len = idx + 1;
  }
  ary->ptr[idx] = val;
}
```

You might have noticed that insertion to a CRuby array takes a different amount of time depending on whether the array is at capacity or not. Adding a new element takes constant time (O(1)) if the array has capacity for new elements, but if the array does not have capacity it will take O(n) time to allocate new memory and copy over existing values. Instead of using worst-case analysis, we can use a technique called amortized analysis to get a better idea of the cost of an insertion operation.

### Amortized analysis

Amortized analysis is a way of analyzing the run time of an operation on average over many operations, rather than the worst case of a single operation. Amortized analysis is useful when looking at data structures with operations that occasionally require an expensive operation, like allocating memory, but normally take much less time.

One method of calculating the amortized time is the aggregate method. You analyze how long k operations take in total, and then divide by k.

For the CRuby array implementation in this blog post, the array capacity begins at 16, and the reallocation adds 16 extra capacity to the array each time it reallocates memory. The amortized cost of this works out to O(n), which is the same as the worse case insertion operation. This means the original CRuby dynamic array wasn't efficient for insertion operations.

In 2009 CRuby improved performance by doubling the array capacity each time memory is reallocated. This improvement means that the total cost of k operations divided by k is equal to a constant time. So the amortized cost of insertion in the new CRuby implementation is O(1). (for a rigorous explanation of how this works, see these [lecture notes on amortized analysis](https://www2.cs.duke.edu/courses/fall17/compsci330/lecture17note.pdf))

## Conclusion

Arrays are a fundamental data structure that can be either dynamic or static. Although static arrays are easier to implement in a language, dynamic arrays are easier to use from a programmers perspective.

Future posts in this series will explore how arrays are used to implement more complicated data structures.

I hope you enjoyed this post. If you have any questions, please leave a comment.
