---
layout: post
title: Notes on C++ Rvalue reference.
published: true
category:
  - C++
  - perfect forwarding
---

### L values
* lvalue is any object that persists beyond a single expression.
* An lvalue is an expression that refers to a memory location and allows us to take the address of that memory location via the `&` operator.
* The lvalue can be implicitly converted to an rvalue. eg `int a = b;`
* A function call is an lvalue if and only if the result type is a reference.


### R values
* rvalues are temperories that exist for the duration of the expression.
* rvalues cannot be converted into a lvalue.

### Const correctness
* `Type&` binds to modifiable lvalues. It can’t bind to const lvalues, as that would violate const correctness. It can’t bind to modifiable rvalues, as that would be extremely dangerous.
* `const Type&` binds to everything: modifiable lvalues, const lvalues, modifiable rvalues, and const rvalues. 
* A reference bound to an rvalue is itself an lvalue. (As only a const reference can be bound to an rvalue, it will be a const lvalue.)
{% highlight cpp linenos %}
    /* const reference, so accepts rvalues as well. 
    void observe(const string& str)
    {
        /* 
         * Here, str is lvalue, even if observe is called
         * with a temperory
         */
         const char *c =  &str[1];
    }

    observe("blog");
{% endhighlight %}
* `NOTE: lvalueness vs rvalueness is a property of expressions, not objects.`


### R value references
* lvalue references are decalred with `&`.
* rvalue reference are declared with `&&`.
* rvalue reference can bind to an rvalue. This is used to support *[move semantics](https://ritsz.github.io/Cpp-Move-Sematics/)*.


### Perfect forwarding problem
* Let's say we need to create a wrapper function that needs to work with template. Let's also assume `func` accepts parameters by reference.
{% highlight cpp linenos %}
    template <typename T1, typename T2>
    void wrapper(T1 e1, T2 e2) 
    {
        func(e1, e2);
    }
{% endhighlight %}
* In the above snippet, func modifies the data `e1` and `e2` which it took as reference. But the `wrapper` itself, took parameters by value, so the changes made by `func` to `e1` and `e2` are not reflected back to the caller of `wrapper`.

{% highlight cpp linenos %}
    template <typename T1, typename T2>
    void wrapper(T1& e1, T2& e2) 
    {
        func(e1, e2);
    }
{% endhighlight %}
* The above code snippet solves our old problem, but now we cannot bind Rvalues to this wrapper. (Temp variables can't be used.)
* We can't use const references as that wouldn't let us modify anything.

### Universal reference and Reference collapsing rule.
* pre-C++11, reference to a reference wasn't allowed. Now that they are allowed, they follow *reference collapsing rule*
>1. A& & becomes A&
>2. A& && becomes A&
>3. A&& & becomes A&
>4. A&& && becomes A&&
* Also Rvalue references in a type deducing context, like in the following code snippet, they don't act like Rvalue reference but take up special meanings
{% highlight cpp linenos %}
    template<typename T>
    void foo(T&&);
{% endhighlight %}

>T depends on whether foo() is called with Lvalue or Rvalue
1. If it's an lvalue of type `U`, `T` is deduced to `U&`
2. If it's an rvalue, `T` is deduced to `U`.
{% highlight cpp linenos %}
    func(4);    // 4 is an rvalue; T is deduced to int
    func(d);    // d is lvalue; T is deduced to int&
{% endhighlight %}

### Perfect forwarding solution
{% highlight cpp linenos %}
    template<class T>
    T&& forward(typename std::remove_reference<T>::type& t) noexcept
    {
        return static_cast<T&&>(t);
    } 
    template<typename T1>
    void wrapper(T1&& e1)
    {
        func(forward<T1>(e1));
    }
{% endhighlight %}
* If `wrapper` gets called with a lvalue, it becomes `wrapper(int& && e1)`. That is `T1` is deduced to `int&`.
{% highlight cpp linenos %}
    int& && forward(int& t) noexcept 
    {
      return static_cast<int& &&>(t);
    }
{% endhighlight %}
Applying reference collapsing rules:
{% highlight cpp linenos %}
    wrapper(int& e1);
    int& forward(int& t) noexcept
    {
        return static_cast<int&>(t);
    }
{% endhighlight %}
Argument is passed onto `func()` through two levels of indirection, both by old-fashioned lvalue reference. 
* If `wrapper` gets called with rvalue, it becomes `wrapper(int&& e1)`, that is `T1` becomes `int`.
{% highlight cpp linenos %}
    wrapper(int&& e1);
    int&& forward(int& t) noexcept
    {
        return static_cast<int&&>(t);
    }
{% endhighlight %}
* Implement `make_unique` which accepts one argument. Note: Multiple arguments can be handles using variadic templates.
{% highlight cpp linenos %}
    template<typename T, typename Args>
    std::unique_ptr<T> make_unique(Args&& args)
    {
        return unique_ptr<T>( new T( std::forward<Args>(args) ) );
    }
    class Base
    {
        int m_iA;
    public:
        Base(int& a) : m_iA(a) { cout << "lvalue" << endl; }
        Base(int&& c) : m_iA(c) { cout << "rvalue" << endl; }
    };
{% endhighlight %}
Using the universal reference and reference collapsing rule, `make_unique` correctly forwards the arguments to the constructor of `Base`.

### References
1. [ACCU: Lvalues and Rvalues](https://accu.org/index.php/journals/227)
2. [MSDN: R Value references](https://blogs.msdn.microsoft.com/vcblog/2009/02/03/rvalue-references-c0x-features-in-vc10-part-2/)
3. [Perfect forwarding and universal references in C++](https://eli.thegreenplace.net/2014/perfect-forwarding-and-universal-references-in-c/)
