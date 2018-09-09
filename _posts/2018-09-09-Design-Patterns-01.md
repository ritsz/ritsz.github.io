---
layout: post
title: Notes on Design Patterns
published: true
category:
  - Design Patterns
---

### Policy classes
* A policy defines a class interface or a class template interface. For example, say the policy is to create an object.
* Policy classes are classes that implement this policy.
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
* In the above example, giving *MyClass* as *NewCreator* template argument was redundant since we know what the host class is going to work on. This can be solved using a *template template class*
{% highlight cpp linenos %}
    template< template <class Created> class Policy>
    class MyClassManager : public Policy<MyClass>
    {
    } ;
    typedef MyClassManager<NewCreator> MyClassCreator;
{% endhighlight %}
* In the above example, the policy is still configurable, but the type on which we work is decided now.
* New policies can be added easily (say, a new memory allocator). The host class works like a code generation engine.
* Since the host classes are derived from policy classes, someone can use a policy base pointer to refer to host class. This would cause issues during destructor calls unless destructor in policy classes are virtual. But having virtual functions goes against the static nature of this pattern. Instead, we move the destructor of policies to protected. This way only derived classes can call the destructor of the base policy.
{% highlight cpp linenos %}
    template<class T>
    struct NewCreator
    {
        static T* Create()
        {
            return new T;
        }
        protected:
        ~NewCreator() {}
    };
    MyClassCreator myCC;
    NewCreator<MyClass>* pBase = &myCC;
    delete pBase;   //Invalid code as destructor is protected.
{% endhighlight %} 