---
layout: post
title: Notes on Design Patterns
published: true
category:
  - Design Patterns
---

### Policy classes
* A policy defines a class interface or a class template interface. For example, say the policy is to create an object.
* Policy classes are classes that implement this policy
{% highlight cpp linenos %}
    template<class T>
    struct NewCreator
    {
        static T* Create()
        {
            return new T;
        }
    };
    template<class T>
    struct MallocCreator
    {
        static T* Create()
        {
            void *buf = std::malloc(sizeof(T));
            // Placement new operator, uses the provided buffer
            return new(buf) T; 
        }
    };
{% endhighlight %}
* The class that uses the policies is called a host class.
{% highlight cpp linenos %}
    template<class Policy>
    class Manager : public Policy
    {
    } ;

    typedef Manager< NewCreator<MyClass> > MyClassCreator;
{% endhighlight %}
* By design, host classes allow configuration of functionalityby using template derivation.
* This is similar to Stratergy pattern, except the choice happens in compile time (we decide which policy to use) as opposed to run-time (we decide which stratergy to use on run-time).