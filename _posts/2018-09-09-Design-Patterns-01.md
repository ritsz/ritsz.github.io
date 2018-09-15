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
    template<class Policy>
    class Manager : public Policy
    {
    };
    typedef Manager<NewCreator<MyClass>> MyClassCreator;
{% endhighlight %}
* The class that uses the policies is called a host class.The host class derives from the policy class to derive the interface of the policies to be used.
* By design, host classes allow configuration of functionality by using template derivation.
* This is similar to Stratergy pattern, except the choice happens in compile time (we decide which policy to use) as opposed to run-time (we decide which stratergy to use on run-time).
* In the above example, giving `MyClass` as *NewCreator* template argument was redundant since we know what the host class is going to work on. This can be solved using a `template template class`
{% highlight cpp linenos %}
    template< template <class Created> class Policy>
    class MyClassManager : public Policy<MyClass>
    {
    } ;
    typedef MyClassManager<NewCreator> MyClassCreator;
{% endhighlight %}
* class *Created* is a *template template parameter*, specifying that the template that is about to follow accepts template parameters. We use a specialized case of that template parameter for *MyClass*
* In the above example, the policy is still configurable, but the type on which we work is decided now.
* New policies can be added easily (say, a new memory allocator) without changing the host classes implementation. The host class works like a code generation engine.
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


### Strategy Pattern
* Strategy pattern defines a family of algorithms, and makes them interchangeable. This lets the algorithm vary independently from clients that use it.
* The client class uses an interface class for the family of algorithms. This interface class should ideally be an abstract class.
* Strategy patterns eliminate conditional statements. Code with many conditional statements often indicate the need to apply different strategy classes.
* The drawback is that the client needs to be aware of the different strategies before being able to chose one.
* Another drawback is the designing the interface between the client and the strategy classes. The different strategy classes might require different parameters to get initialized. All these parameters need to be sent to the interface, irrrespective of the implementation that will be used. A simple strategy could be using none of the parameters sent during initialization.
* Strategy patterns do via interfaces almost exactly what policy patterns do via inheritance.

{% highlight cpp linenos %}
    class INewlineInterface
    {
        public:
            virtual string LineEnding() = 0;
    };
    class Windows : public class INewlineInterface
    {
        public:
            virtual string LineEnding()
            {
                return "\r\n";
            }
    }
    class Unix : public class INewlineInterface
    {
        public:
            virtual string LineEnding()
            {
                return "\n";
            }
    }
    class DocumentProcessor
    {
        unique_ptr<INewlineInterface> ptrNewLineDelimiter;
        public:
            DocumentProcessor(OSType input)
            {
                if (input == WIN32)
                {
                    ptrNewLineDelimiter.reset( new Windows );
                } else {
                    ptrNewLineDelimiter.reset( new Unix );
                }
            }
    }
{% endhighlight %}
* Another behavior that can be achieved is that `DocumentProcessor` could itself be subclassed and just be an *aggregation of interfaces*, with each subclass using one of the interfaces from the family of algorithms that it chooses to.
{% highlight cpp linenos %}
    class IDocumentFormatter
    {
    };
    class PlainTextFormatter : public class IDocumentFormatter
    {
    };
    class JSONFormatter : public class IDocumentFormatter
    {
    };
    class DocumentProcessor
    {
        unique_ptr<INewlineInterface> ptrNewLineDelimiter;
        unique_ptr<IDocumentFormatter> ptrDocumentFormatter;
    };
{% endhighlight %}