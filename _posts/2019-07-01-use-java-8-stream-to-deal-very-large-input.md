---
layout: single
classes: wide
title:  "Use Java 8 streams for handling very large data"
date:   2019-08-11 20:00 -0500
tags:
  - Java
  - Java8
  - data
  - IO
  - Streams
  - search
  - filter
  - Reactive
  - Long process
 
categories: Java tricks
excerpt: "How would you deal with infinite of very large data in Java 8+ without breaking the CPU? Let's see the way I would do it."
comments: true
---

## Understanding the scenario

 Not so long, I saw a problem that says something like

 > Based on a very large list of `Integer` values as input, try to find all missing `Integer` values having in count that you have only left 2 Gbs of RAM.

Firstly, it intrigued me the fact that I had to have in count 2 Gbs of RAM, but probably that was just to bear in mind the size of Integer as a data structure and how many of them could I create before depleting the memory (Memory Overflow). To calculate the maximum amount of Integers we should be able to create, I used the following constant:

```java
private static final int MAX_CAPACITY = BigInteger.valueOf(2)
            .multiply(BigInteger.valueOf(1073741824))
            .divide(BigInteger.valueOf(Integer.BYTES))
            .intValue()
```

- `Integer.BYTES` returns the amount of memory in bytes each `Integer` variable will occupy, which depends on the [Java Platform][1]. In my Java 8 (64 bits architecture) it is 32 bits, which is 4 bytes.
- 1024 bytes is 1 Kb, multiplied by 1024 again is the amount of Mb, which multiplied by 1024 is the amount of Gbs, which is [1,073,741,824](https://www.gbmb.org/gb-to-mb)
- I multiplied that number by 2 because to represent the maximum of 2 Gb to occupy. Afterwards I divided it by `Integer.BYTES` to determinate the amount of `Integer` values I can affore.
- I had to use `BigInteger` to be able to make such division correctly.

In order to find the missing `Integer` values that are not in the input list I had to implement a function called `findMissingIntegers`, so I can print them later

```java
final List<Integer> result = findMissingIntegers(Arrays.asList(
        2, 4, 6, 1, 14
));

result.forEach(System.out::println);
```

At the end I expect that the variable `result` contained a list of all those `Integer` values I wanted to work with.

## Calculate a very large input and output using a List in Java

 (Eventually bad solution)
By instinct the first that will come to most developer's mind is to obtain a final definitive response, probably boxed into a List.
My version of that solution was something like

```java
    static List findMissingIntegers(List<Integer> inputIntegers) {
        List result = new ArrayList<Integer>(MAX_CAPACITY);                                     #1

        Collections.sort(inputIntegers);                                                        #2

        Integer previous = Integer.MIN_VALUE;
        Integer foundInts = 0;
        for (Integer i : inputIntegers) {
            if (i - previous > 1) {                                                             #3
                addNumbersRangeToList(result, previous + 1, i - 1);                             #4

                foundInts += i - previous;                                                      #5
                if (foundInts >= MAX_CAPACITY) {                                                #6
                    System.out.printf("Maximum arrived %d of Integers%n", foundInts);
                    return result;                                                              #7
                }
            }
            previous=i;
        }

        final int MAX_NUMBERS_LEFT_TO_ADD = MAX_CAPACITY - foundInts;                           #8

        addNumbersRangeToList(                                                                  #9
            result, previous + 1,
            Math.max(previous + MAX_NUMBERS_LEFT_TO_ADD, Integer.MAX_VALUE)
        );

        return result;                                                                          #10
    }

    private static void addNumbersRangeToList(List<Integer> list, Integer start, Integer end) {
        if (start == end) {
            list.add(start);
        } else {
            IntStream.rangeClosed(start, end).forEach(list::add);
        }
    }
```

1. This is the amount of RAM that needs to be allocated.
2. I used the the `java.util.Arrays.sort` [that guarantees since Java 7 a O(n log n) performance][2], which is much better than iterating over n elements and search if that element is contained in the input array, which will be unordered and is prone to actually have a BigO(n) complexity to find if an element is contained.
3. As the input is ordered, it eases the searching of an Integer `i` in the closed range `[Integer.MIN_VALUE, Integer.MAX_VALUE]`, which is where the input array is contained. We can see check chunks of data, based on the difference of the previous value gotten from the input to the current one. For instance, if the first value of the input is `-2`, then we can add easly from `Integer.MAX_MIN` to `-1` to the list of missing Integers, which saves a big amount of calculation for sure.
4. Add in one step all the numbers not found from the last one found form the input, to the next one following. It saves us from checking one by one.
5. The variable to count the amount of Integers increases based on the number of numbers we just added.
6. The previous checking was important to verify we are not surpassing the allowed amount of memory for allocating these `Integer` values.
7. Ends the process if we have reached that maximum amount of allowed memory to use for storing the final Integer values.
8. Calculates the maximum amount of numbers that is possible to add.
9. Adds in one step all numbers after the one found to the maximum allowed number in Java for `Integer`.
10. Returns the result containing all the missing `Integer` values, based on the input.

Result: When you run it it will break for sure, which makes it an awful solution.

There is a problem with this solution: the consumer will have to wait for all the numbers in the result to be calculated, so it can start processing them.
A better approach is to be reactive to the discovery of every value of the result, which means that for every number I found I should pass it right away to its consumer. This way the consumer can work on it in parallel. This is better for the memory, because we will not have to allocate too much: every entry of the result that is produced is right away consumed; so it becomes disposable for the JVM and removed by the garbage collector when necessary.

## Calculate a very large input and output using a Stream in Java

In order to process an input that can be very long and provide a response in a responsive way, a Java stream seems for most of the cases
the best contendent. Lets repicture the previous solution into an easier and more effective one:

```java
    static Stream<Integer> findMissingIntegers(List<Integer> inputIntegers) {
        Collections.sort(inputIntegers);                                                #1
        return IntStream.rangeClosed(Integer.MIN_VALUE, Integer.MAX_VALUE)              #2
                .boxed()                                                                #3
                .filter(x -> Collections.binarySearch(inputIntegers, x) < 0)            #4
                .limit(MAX_CAPACITY);                                                   #5
    }
```

Which changes a bit the way we call to this function:

```java
findMissingIntegers(Arrays.asList(
        2, 4, 6, 1, 14
)).forEach(System.out::println);

```

Let’s dive into the new version of `findMissingIntegers` to see how it works:

1. We need to sort the input array to ease the search using the [Binary Search algorithm](https://en.wikipedia.org/wiki/Binary_search_algorithm). As mentioned before, [it guarantees since Java 7 a O(n log n) performance][2], which will set the bar of an initial delay.
2. It creates a `Stream` for all valid `Integer` values. The interesting part of it is that they will be native values, i.e. `int` instead of `Object` values,
i.e. `Integer`;
3. This boxing is optional, in case you rather using the `Object` value. That is why the return value is a `Stream<Integer>`, instead of an `IntStream`. I personally, prefer the `IntStream` alternative, which skips me of an additional wrapping for this particular solution. Try removing this line and change the return value to `IntStream` and it shall work as well.
4. Does the searching for every single `Integer` value inside `inputIntegers`. Thanks to _#1_ we have them ordered and ready to have a `O(log n)` with
a binary search, which represent a great performance improvement. You will notice it during the runtime.
5. This will guarantee that we have not produced more than _n_ results. But as they are consumed right after they are found there are no compromise with the memory usage. Therefore, we can remove this instruction if wanted.

This final solution has a complexity of `O(log n)` and actually works fluently.

## Conclusion

We have noticed a huge improvement between processing a very long input with a Java Stream, instead of doing it sequentially. To do so, bear in mind you should not collect the final result of the stream, i.e. `.collect(Collectors.toList());`, otherwise it will work sequentially like the first solution.
As this strategy using a `Stream` is reactive, you are actually being able to print while you are calculating. This way, the time consuming of the printing is not added to the one that calculates the missing numbers. Instead, they are done in parallel. All this without making the code more complex. Actually, it is easier to understand that the first option.

It is sad that returning a `Stream` instead of a boxed result is not a thing you can do in traditional client-server architecture: The server needs to have a final response, so that it can be returned to the user. While this later solution requires the client to be watching every output in real time. That happens because connections are synchronous and have a timeout.  
Having this in mind, [Reactive Programming][3] solutions like [RxJava](https://github.com/ReactiveX/RxJava) were born and adopted by frameworks to ease asynchronous and event-driven communication between services.

I would recommend watching [Real-world Reactive Programming in Java: The Definitive Guide by Erwin de Gier](https://www.youtube.com/watch?v=fXWywFRwHOk) and [Reactive Spring by Josh Long](https://www.youtube.com/watch?v=zVNIZXf4BG8), which are good introductions to the usability of this powerful area of modern architectures.

## Read more

[Primitives Data Types][1] in the Official website
How the sorting of collections [in Java uses TimSort][2].
Basics about [Reactive programming][3]

[1]: https://docs.oracle.com/javase/tutorial/java/nutsandbolts/datatypes.html
[2]: https://bugs.openjdk.java.net/browse/JDK-6804124
[3]: https://en.wikipedia.org/wiki/Reactive_programming
