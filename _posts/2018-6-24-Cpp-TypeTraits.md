---
layout: post
title: Notes on C++ Type traits.
published: true
category:
  - C++
  - traits
---

* Traits are like *if-else-then* on types.

* Traits allow us to make compile time decisions based on types.
{% highlight cpp %}
Think of a trait as a small object whose main purpose is to carry 
information used by another object or algorithm to determine 
"policy" or "implementation details". - Bjarne Stroustrup
{% endhighlight %}

* Example, let's say we have an API that works only on pointers.
  How do we test during compile time, that the input we are getting is a pointer type?

A generic template `CheckPointer` defined on `T` that has a `static boolean isPointer` that is `false`. 

{% highlight cpp linenos %}
    template <typename T>
    struct CheckPointer
    {
        static const bool isPointer = false;
    };
{% endhighlight %}

Specialize this template on `T*` and make `isPointer` `true` for `T*`. This was any pointer type matches this specialization better than the generic `T`. Thus for any pointer type, `CheckPointer::isPointer` will be `true`.

{% highlight cpp linenos %}
    template <typename T>
    struct CheckPointer <T*>
    {
        static const bool isPointer = true;
    };
    CheckPointer<int>::isPointer;   // false
    CheckPointer<int*>::isPointer;  // true
{% endhighlight %}

NOTE: We added a partial specialization T*, so that it matches all pointer types

* We can also use explicit template specialization to make decisions on type.
{% highlight cpp linenos %}
    template <typename T>
    struct is_void<T>
    {
        static const bool value = false;    
    }

    /* Explicit specialization so keyword typename T not required */
    template <>
    struct is_void <void>
    {
        static const bool value = true;
    };
{% endhighlight %}

### Using constexpr inside struct

* Use `constexpr` instead within the `struct` to represent the value.
  We can define an `integral_constant` type.
{% highlight cpp linenos %}
    /* The type T and the value v are both templates */
    template <typename T, T v>
    struct integral_constant<T>
    {
        static constexpr T value = v;   /* Set the variable value */
        typedef T value_type;           /* Type of the parameter */
        typedef integral_constant<T,v> type;
        constexpr operator T() { return v; }    
    } 
{% endhighlight %}

* And then use the `integral_constant` to define specific types as well.
{% highlight cpp linenos %}
    using true_type = my_integral_constant<bool, true>;
    using false_type = my_integral_constant<bool, false>;
{% endhighlight %}

* Check if the `class` are same type:
{% highlight cpp linenos %}
    template <class T, class U>
    struct is_same : false_type {};

    template <class T>
    struct is_same<T,T> : true_type {};
{% endhighlight %}

* Say we have a collection, updation of which depends on the data type of elements. We can define a trait that specializes for the specific scenarios. We can also make the process of updating the elements generic and specialize them for special cases.
{% highlight cpp linenos %}
    template <class _type>
    struct UpdateAllowed 
    {
        static constexpr bool value = true;
    };

    template<>
    struct UpdateAllowed<long> 
    {
        static constexpr bool value = false;
    };

    template <class _type>
    struct UpdateType 
    {
        static _type IfDataExists(const _type& ci, const _type& pi)
        {
            return ci;
        }
    };

    template <>
    struct UpdateType<int>
    {
        static int IfDataExists( const int& ci, const int& pi)
        {
            /* Update if new value is larger */
            if  ( ci == 0 )
                return pi;

            if ( ci > pi )
                return ci;

            return pi;
        }
    };

    template <class _type>
    class Cache
    {
    public:
        typedef _type data_type;
        vector<data_type> map;
        void Update(size_t index, const data_type&);
    };

    template <class _type>
    void Cache<_type>::Update(size_t index, const data_type& newData)
    {
        if ( index >= map.size() )
        {
            map.push_back(newData);
        }
        else
        {
            if ( !UpdateAllowed<_type>::value )
            {
                cout << "No updates allowed" << endl;
            }
            else
            {
                map[index] = UpdateType<_type>::IfDataExists(newData, map[index]);
            }
        }
    }
{% endhighlight %}

### References
1. [Traits: The else-if-then of Types](https://erdani.com/publications/traits.html)
2. [An introduction to C++ Traits](https://accu.org/index.php/journals/442)
