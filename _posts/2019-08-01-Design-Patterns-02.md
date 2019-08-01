---
layout: post
title: Notes on Synchronization and Concurrency
published: true
category:
  - Operating System
  - Concurrency
---

### Memory Reordering
* A processor is allowed to reorder instruction (memory interations), as long it never changes the execution of a single-threaded program.
* Processor is allowed to delay the effect of a store past any load from a different location.
{% highlight cpp linenos %}
    // Thread 1
    mov [PTR_A], 1
    mov rax, [PTR_B]

    // Thread 2
    mov [PTR_B], 1
    mov rab, [PTR_A]
{% endhighlight %}
* Assuming the above code is running parallely on 2 processors, depending on execution flow, the expectation would be either or both of `rax` and `rbx` to be one.
* Since processor is allowed to reorder independent memory interations, there is a possibility that both `rax` and `rbx` are zero.
{% highlight cpp linenos %}
    // Thread 1 - Reordered
    mov rax, [PTR_B]
    mov [PTR_A], 1

    // Thread 2 - Reordered
    mov rab, [PTR_A]
    mov [PTR_B], 1
{% endhighlight %}