---
layout: post
title: Notes on Monad Pattern
published: true
category:
  - Monads
  - Functional Programming
---

* A function can only read what is supplied to it in its arguments and the only way it can have an effect on the world is through the values it returns.
* 
### Monad
* A monad is created by defining a type constructor `M` and two operations, `bind` and `return` (where `return` is often also called `unit`):
1. The unary `return` operation takes a value from a plain type `a` and puts it into a container using the constructor, creating monadic value with type `M a`
2. The binary bind operation `>>=` takes as its arguments a monadic value with type `M a` and a function `(a → M b)` that can transform the value.
    2.1 The bind operator unwraps the plain value with type a embedded in its input monadic value with type M a, and feeds it to the function.
    2.2 The function then creates a new monadic value, with type M b, that can be fed to the next bind operators composed in the pipeline.
* Compose a sequence of function calls (the **_pipeline_**) with several bind operators chained together in an expression
* In Rust, `from_str` function returns an `Option<T>` type (`Option<int>` in the following example.)
{% highlight cpp linenos %}
    fn main()
    {
        from_str::<int>("4");           // valid
        from_str::<int>("Potatoes");    // invalid
    }
{% endhighlight %}
* `Option<int>` has 2 constructors. `Some(x)` and `None`.
{% highlight cpp linenos %}
    enum Option<T> 
    { 
        None, 
        Some(T) 
    }
{% endhighlight %}
* `Option` can be unwrapped using `match`, `.expect()` or `.unwrap()`.
{% highlight cpp linenos %}
    fn main () {
        // Create an Option containing the value 1.
        let a_monad: Option<int> = Some(1);
        // Extract and branch based on result.
        let value_from_match = match a_monad {
            Some(x) => x,
            None    => 0i // A fallback value.
        };
    }
{% endhighlight %}

### Composition
* The power of monad is realized when using composition (or a pipeline).
{% highlight cpp linenos %}
    fn sqrt(value: f64) -> Option<f64> {
        match value.sqrt() {
            x if x.is_normal() => Some(x),
            _                  => None
        }
    }
    fn double(value: f64) -> f64 {
        value * 2.
    }
{% endhighlight %}
* `map` provides a way to apply a function of the signature `|T| -> U` to an `Option<T>`, returning an `Option<U>`, for example `double` above can be used with `map` as it has a signature of `|T| -> U`. This is a functor interface, like `fmap` in Haskell
* `and_then` allows you to apply a `|T| -> Option<U>` function to an `Option<T>`, returning an `Option<U>`. This allows for functions which may return no value, like `sqrt()`, to be applied, or functions the return `Option` themselves. This call corresponds to `bind` in Haskell and_then theoretical Monad definitions. 

### References
1. [Functors, Applicatives, And Monads In Pictures](http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html)
2. [Option Monads in Rust](https://hoverbear.org/2014/08/12/option-monads-in-rust/
)
3. [You Could Have Invented Monads!](http://blog.sigfpe.com/2006/08/you-could-have-invented-monads-and.html)
