---
title: "Big O notation"
date: 2019-04-07T20:27:04+01:00
---

In this post you'll learn how to use big O notation to compare the performance of different algorithms.<!--more-->

## Measuring the running time of an algorithm

There are two ways to measure the running time of an algorithm:

1. Experimental analysis
2. Theoretical analysis

Experimental analysis involves running a program and measuring how long it takes to complete. The problem with this approach is that programs run non-deterministically and at different speeds depending on the hardware, compiler, programming language, and other factors.

The alternative is theoretical analysis. One approach to theoretical analysis is to approximate the running time by counting the number of steps an algorithm takes for a given input.

## Calculating algorithm steps

Steps are a unit that measure how long an operation takes to complete on a hypothetical computer.

The rules for this hypothetical computer are:

1. Simple operations take 1 step
2. Loops and subroutines are made up of simple operations
3. Memory is unlimited and memory access takes 0 steps

Simple operations can be arithmetic operations like `*`, comparison operations like `==`, assignment operations like `=`, and statements like `return`. They all take 1 step.

Loops and subroutines are made up of the simple operations performed by them. If a loop performs 3 operations each iteration, then n loops would take a total of 3n steps.

The final rule is that memory is unlimited and access takes 0 steps. This makes analysis simpler by ignoring the fact that memory access times vary on real-world machines.

With these rules you can calculate the total number of steps a program will take to run on the hypothetical computer.

For example, the following `sum` function performs 3 simple operations: `=`, `return`, and `+`. So the function takes 3 steps to complete:

```c
int sum(int a, int b) {
  int temp = a + b;
  return temp;
}
```

You can express the total number of steps an algorithm takes using a mathematical function. A mathematical function expresses the output for a given input. In the case of the `sum` function, the number of steps is always 3. So a mathematical function for the total steps taken by the algorithm for an input of n is f(n) = 3.

The following `exp` function performs 1 assignment operation and 2 operations for each loop, where the number of loops is based on the input `e`. So the total number of steps is 2e + 1, which can be expressed as f(n) = 2n + 1:

```c
int exp(int base, int e) {
  int total = 1;
  while(e) {
    total *= base;
    e--;
  }
}
```

_Note: It can be tricky to determine what a simple operation is. Is `*=` 1 operation, or 2 operations? Generally it's OK to choose one or the other, as long as you're consistent._

What about an algorithm where the number of operations depends on the state of the input? For example, an algorithm that returns either the index of a value if it exists in an array, or `-1` if it does not. If the value being searched for is the first element in the array then the function will return after 1 loop, but if the value doesn't exist in the array the algorithm will loop over each element in the array. How do you decide which case to analyze?

## Analyzing the worst-case

I like to think of myself as a realistic optimist, but when I analyze algorithms I'm usually a pessimist. Why? Consider the cases you can analyze:

1. Best-case
2. Average-case
3. Worst-case

The best-case can be misleading. An algorithm might run quickly in the best-case, but take an impossible amount of time in the average-case.

The average-case is intuitively the best choice, but it's difficult to calculate. You might need to use probability theory and make subjective assumptions about the algorithms input.

The worst-case is both easy to determine and useful in practice. The worst-case is an upper-bounds on the algorithms running time, which guarantees the algorithm will finish on time.

Using the worst-case you can analyze algorithms like the `find_index` function below. In the worst-case, `find_index` takes 3n + 2 steps (f(n) = 3n + 2), because in the worst-case the value (`val`) doesn't exist in the array (`n`) and the function will need to loop through the array n times. The loop performs 3 operations each loop, hence 3n:

```c
int find_index(int arr[], int n, int val) {
  for(int i = 0; i < n; i++) {
    if(arr[i] == val) {
      return i;
    }
  }

  return -1;
}
```

Expressing algorithms as mathematical functions is a good way to see how the algorithm handles different size inputs, but the detail of the functions can make it difficult to compare them. For example consider the following functions:

f(n) = 3n + 1

f(n) = 16n + 3

f(n) = 2n + 1230

These functions look quite different, but they grow at roughly the same rate as the input (n) gets large. When you measure algorithms, you normally care about how well it runs as the input gets large.

## Using Big O notation

Big O notation is a way of classifying how quickly mathematical functions grow as their input gets large.

Big O works by removing clutter from functions to focus on the terms that have the biggest impact on the growth of the function. In big O notation f(n) = 5n + 42 and f(n) = 2n can both be written as O(n), pronounced "order of n".

The clutter that I'm talking about are the constant factors and lower-order terms in an expression. Constant factors are the constant values that a term is multiplied by. For example in the term 2n, 2 is a constant factor of n.

If all terms in an expression have the same variable, the leading term is the term with the highest exponent, and the lower-order terms are all other terms in the expression. For example, in the expression 2n³ + n² + 2n + 1, n³ is the leading term, all other terms are lower-order terms.

You can ignore constant factors and lower-order terms because the leading term always determines how output grows as the input gets large. You can see this by looking at a graph of linear functions f(n) = n, f(n) = 2n, and f(n) = 3n, and a quadratic function f(n) = n². The linear functions barely grow in comparison to the quadratic function.

![Graph showing relative growth of liner and quadratic functions](/images/big-o/big-o-relative-growth.svg)

Since big O is a mathematical concept that's been borrowed by computer scientists, it has a formal definition:

f(n) = O(g(n)) as n→∞ iff |f(n)| ≤ Kg(n) for all n ≥ n₀.

A less precise (but easier-to-understand) definition is that:

> For any two functions that take n as inputs: f and g. f can be written as O(g), if any value of f is less than or equal to g multiplied by some constant (K) for every value of n when n is greater than or equal to some value (n₀).

For example, take the function f(n) = 2x² + 400. f can be written as O(n²) because there is a function g(n) = n² that when multiplied by a constant K (for example 4) is always bigger than f after a certain value (n₀).

![Graph showing two functions growth rates](/images/big-o/big-o-formal.svg)

Note that using this definition of big O you could describe the function f(n) = 2x² + 400 as O(n²), O(n³), or O(n⁴). In practice, you should use the lowest-order term that satisfies the definition.

## Common classes in Big O

Big O creates classes of algorithm by ignoring the details, which makes it easy to compare algorithms to each other.

The common classes have their own terminology. For example an O(n) algorithm is said to run in linear time. You should learn the terminology for the most common classes:

| Big O    | Name        |
| -------- | ----------- |
| O(1)     | constant    |
| O(log n) | logarithmic |
| O(n)     | linear      |
| O(n²)    | quadratic   |
| O(2ⁿ)    | exponential |

You can see how the classes compare in the following graph.

![Graph showing growth rate of common classes](/images/big-o/big-o-graph.svg)

To make this more concrete, take a look at some examples.

### O(1)

O(1) is constant time. No matter the size of the input the algorithm will take the same amount of time to complete. For example, a `sum` function:

```c
int sum(int a, int b) {
  return a + b;
}
```

O(1) solutions are the best, although they aren't possible for most problems.

### O(log n)

O(log n) is a logarithmic function. Logarithmic functions grow slowly because they halve the amount of data they work with on each iteration.

| Input       | Steps (rounded up) |
| ----------- | ------------------ |
| 100         | 7                  |
| 1,000       | 10                 |
| 10,000      | 13                 |
| 100,000     | 16                 |
| 1,000,000   | 20                 |
| 100,000,000 | 27                 |

Even at 100,000,000 input elements, a logarithmic function barely breaks a sweat at 27 steps!

An example of O(log n) is [binary search](https://en.wikipedia.org/wiki/Binary_search_algorithm), which is an algorithm that searches for a value in a sorted array.

### O(n)

Linear functions grow linearly with `n`. For example an algorithm to calculate the exponent of a number:

```c
int exp(int base, int e) {
  int total = 1;
  while(e) {
    total *= base;
    e--;
  }
}
```

Linear algorithms are pretty good and can handle large inputs.

### O(n²)

Quadratic functions grow quickly, so O(n²) algorithms run slowly. An example is an algorithm that checks for duplicates in an array by looping over each element, and then looping over each of the previous element to check for matches:

```c
bool contains_duplicates(int arr[], int n) {
  for (int i = 0; i < n; i++) {
    for (int j = 0; j < i; j++) {
      if (arr[j] == arr[i] && i != j) {
        return true;
      }
    }
  }
  return false;
}
```

Nested loops are a sign that your algorithm is probably quadratic.

### O(2ⁿ)

Exponential functions grow even faster than quadratic functions, so O(2ⁿ) algorithms are incredibly slow. An example of an exponential O(2ⁿ) algorithm is a recursive algorithm to find the nth term of the fibonacci sequence:

```c
int fib(int n) {
  if (n <= 1) {
    return n;
  }
  return fib(n - 1) + fib(n - 2);
}
```

## Downsides of worst-case Big O analysis

Although worst-case big O is the standard for algorithm analysis there are a couple of problems with it:

1. Big O ignores constants
2. The worst-case can be rare

First, big O ignores constants and constants are sometimes large. They can make a difference if the input is small, and it can be difficult to compare algorithms of the same class because the constants are hidden.

Secondly, the worst-case can be rare. A good example of this is the sorting algorithm quicksort, which has a worst-case runtime of O(n²), but an average-case of O(n log n). quicksort is often used regardless of its worst-case because the worst-case is so rare in practice.

Despite its downsides, worst-case big O is the most common method of analyzing algorithms, and I'll use it throughout this series.

## Conclusion

Measuring algorithms experimentally is difficult, so instead you can measure them theoretically. You do this by counting the steps an algorithm will take to complete, and expressing the growth rate of the steps using big O notation.

I hope you enjoyed this post. If you have any questions, please leave a comment.
