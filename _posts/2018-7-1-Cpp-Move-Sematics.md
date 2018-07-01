---
layout: post
title: Notes on C++ move semantics
published: true
category:
  - C++
  - move semantics
---

### Eliminating copy
* Move sematics are needed in situations where a transfer of ownership is required.
* The building blocks of `move` is [rvalue references](https://ritsz.github.io/Cpp-RValue-references/).
* Instead of copying data from caller to callee, `move` helps in transfering the reference (Rvalue reference in `move`'s case).
* `move` accepts lvalue or rvalue, and returns an rvalue reference *without* triggering a copy constructor.

{% highlight cpp linenos %}
    template <class T>
    typename remove_reference<T>::type&& 
    move(T&& a)
    {
        return a;
    }
{% endhighlight %}

### Move constructor and move assignment
* Move constructors are called when construction or assignment happens with a rvalue or a moved value.
{% highlight cpp linenos %}
    class Derived : public Base
    {
        std::vector<int> vec;
        std::string name;
    public:
        // move semantics
        Derived(Derived&& x)              // rvalues bind here
            : Base(std::move(x)), 
              vec(std::move(x.vec)),
              name(std::move(x.name)) 
        { 
        }
        Derived& operator=(Derived&& x)   // rvalues bind here
        {
            Base::operator=(std::move(x));
            vec  = std::move(x.vec);
            name = std::move(x.name);
            return *this;
        }
    };
{% endhighlight %}
* The argument `x` is treated as an lvalue internal to the move functions, even though it is declared as an rvalue reference parameter. That's why it is necessary to say `move(x)` instead of just `x` when passing down to the base class. This is a key safety feature of move semantics designed to prevent accidently moving twice from some named variable. All moves occur only from rvalues, or with an explicit cast to rvalue such as using `std::move`. 
* **_If you have a name for the variable, it is an lvalue._**. In the following example, `arg` is an lvalue, and hence calls the copy constructor. To call the move constructor, `std::move` needs to be called explicitly. *Why?* Because, `arg` is still in scope after constructor call. If it was handled as a rvalue, move constructor will be called, `arg` is pilfered, but is still accessible.
{% highlight cpp linenos %}
    void foo(X&& arg)
    {
        X anotherX = arg; // calls X(X const & rhs)
        // arg is still in scope
        X yetAnotherX = bar() // calls X(X&& rhs)
        // bar() is not in scope as it was a temperory
    }
{% endhighlight %}
* `x` is not Rvalue inside the move constructor/operator=. It's an lvalue. So just calling `Base(x)` would call the copy constructor. 
* Calling `Base(std::move(x))` explicitly calls `Base`'s move constructor. So the constructor of `Base` sees an rvalue reference and moves the members that are part of `Base` out of `x`. *The other members of `x` are not yet moved, and can be moved in `Derived`'s move constructor.
{% highlight cpp linenos %}
    class Base
    {
        int id;
    public:
        Base(Base&& x)
        : id(std::move(x.i))
        {
        }
        Base& operator=(Base&& x)
        {
            id = std::move(x.i);
            return *this;
        }
    };
{% endhighlight %}

