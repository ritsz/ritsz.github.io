---
layout: post
title: Notes on Design Patterns - 2
published: true
category:
  - Design Patterns
---

### Observer Pattern
* Defines a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically.
* This is also known as *Publisher-Subscriber* model.
* Publisher holds the data/event that has to be observed, while Subscriber is the class interested in the event or change in state of data.
{% highlight cpp linenos %}
    class IObservable
    {
    public:
        void Add(shared_ptr<IObserver>) = 0;
        void Remove(shared_ptr<IObserver>) = 0;
        /* Calls the Update() method on subscriber */*
       void  Notify() = 0;
    private:
        std::vector<shared_ptr<IObserver>> m_subscribers;
    };

    class IObserver
    {
    public:
        /* Uses the passed IObservable object to check new state */*
        Update(IObservable*) = 0;
    };
{% endhighlight %}
* Publisher will have many *has-a* relationsship with the Subscriber. 

### Decorator Pattern
