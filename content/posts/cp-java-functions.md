+++
title = "Competitive programming with Java: Helper functions"
date = 2017-11-25
author = "Mehdi Cheracher"
tags = ["java", "algorithms"]
keywords = ["java", "java-core", "cp", "competitive programming" , "algorithms"]
description = "An overview of the standard library utility functions used in Competitive programming"
showFullContent = false
+++

Today we are going to explore more helper functions provided by the Java standard library, and specifically functions that deals with `Arrays`, and `Collections`.

```java
Arrays.sort(T[] arr)
```
there are a lot of overrides to this method, it works with almost all primitive types, and all generic types, it has a version in which you specify a `Comparator` Object to determine the ordering.

**Note**: *Note : when using sort in Java you should be careful about what you are sorting, if the elements you are sorting are primitive values (int, long … ) the sort method will use quick sort algorithm to sort your elements which has an O(n\*n) worst time complexity, i.e given the right test case could get you a TLE . the best solution for this is to use Objects instead of primitives (Integer, Long…) so the sort method will use a variation of merge sort with a worst time complexity of O(nLog(n)) . or shuffle the array before sorting it (not recommended)*.
  
```java
Arrays.binarySearch(T[] arr, T key)
```

Implements the binary search algorithm and returns the index of the key if found, else it returns the negative value of the position where it should be inserted , or more formally `(-(index – 1))`.

If the elements are not sorted the behavior of the function is undefined .
```java
Arrays.copyOfRange(T[] arr, int from, int to)
```

Creates a new array with length `(to – from  + 1)` and contains all the elements from the original array in the range `(form – to)`.
```java
Arrays.toString(T[] arr)
```

Creates a pleasant String representation of the given array, which is basically a join of all it’s element by a `,` character. if the elements in the array are Objects their toString() method is invoked to get the String representation of each element.
```java
Arrays.fill(T[] arr, T value)
```
Fills the given array with the given value.

```java
Arrays.parallelPrefix(T[] arr, BinaryOperator operation)
```
Constructs a prefix array of the given one (in place) where all the elements are computed based on the passed operator and the last element , if we take the example of the array

`arr = [1, 3, 4, 2]` and we call the function with `Arrays.parallelPrefix(arr, (a, b) -> a + b)`.

this will create this array `[1, 4, 8, 10]`. all what this is doing is creating a cumulative sum array which is quite popular when solving competitive programming contest, it’s just done with less code, also this function is very efficient when using large arrays.

```java
Collections.copy(List dest ,list src)
```
Copies all the elements in the src list to the dest list.

```java
Collections.reverse(List list)
```

Reverses the order of all the elements in the list,

Example : `list = [1, 2, 3]` after the function call `list = [3, 2, 1]`

```java
Collections.shuffle(List list)
```
Randomly shuffles the elements in the list.

```java
Collections.rotate(List dest ,int distance)
```

Rotates the elements of the list by the given distance, the element at index `i` will be the element previously at index `(i – distance + size) % size` after the rotation.

Well that’s most of what you need in a typical Competitive Programming contest, there are more methods i didn’t cover just because i never used them in competitive programming, the github [repo](https://github.com/chermehdi/competitive-programming-java) will contain code demos of the given functions and more.

play with the code, and see you next time.
