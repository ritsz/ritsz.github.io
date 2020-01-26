---
layout: post
title: Notes on SOLID principles
published: true
category:
  - Design Patterns
---

### Single Responsibility Principle
* A class should be responsible for only one thing.
* Very precise names for small classes is better than generic names for large classes.
* _A class should have one and only one reason to change_
{% highlight cpp linenos %}
    class Rectangle
    {
        int length, breadth;
    public:
        long area();
        void Draw();
    };
{% endhighlight %}
* The above class has two responsibilities, calculate the area and draw the graphical representation of the rectangle in the GUI. This violates Single Responsibility. Refactor different responsibilities to different classes.
{% highlight cpp linenos %}
    class Rectangle
    {
        int length, breadth;
    public:
        long area();
    };

    class GraphicalRectangle
    {
    public:
        void Draw(const Rectangle& object);
    };
{% endhighlight %}

### Open/Closed Principle
* Software entities (classes, modules, functions) should be open to extension but closed for modification.
* This is achieved using inheritence and interfaces that enable classes to polymorphically substitute for each other.
{% highlight cpp linenos %}
class Payment
{
public:
    virtual void pay() = 0;
};

class BankingPayment : public Payment
{
public:
    virtual void pay(); /* Bank payment logic */
};

class GooglePayment : public Payment
{
public:
    virtual void pay(); /* GooglePay logic */
};

class PaymentFactory
{
    static Payment* GetPaymentMethod(string requestType)
    {
        if (requestType == "gpay")
            return new GooglePayment();
        else
            return new BankingPayment(); 
    }
};
{% endhighlight %}

### Liskov Substitution Principle
* Let ϕ(x) be a property provable about objects x of type T.
Then ϕ(y) should be true for objects y of type S, where S is a subtype of T.
* Objects in a program should be replaceable with instances of their subtypes without altering the correctness of that program.
* Subclasses should be substitutable for their base classes.
* In order to be substitutable, the contract of the base class must be honored by the derived class.
* If the subclass has to reimplement all/almost-all interfaces in order to be substitutable, the inheritance is generally wrong then.

### Interface Segregation Principle
* No client should be forced to depend on methods it does not use.
* Do not add additional functionality to an existing interface by adding new methods.
* If you have a class that has several clients, rather than loading the class with all the methods that the clients need, create specific interfaces for each client and multiply inherit them into the class.
* Following is an example where the class has implemented too many interfaces for too many clients. _Changing one interface results in all clients recompiling_.
![Bad LSP]({{ site.url }}/images/Bad-Interface-Segregation.PNG)

* Following is an example of good substitution principle. The class is _an aggregate of interfaces that 
![Good LSP]({{ site.url }}/images/Good-Interface-Segregation.PNG)


### Dependency Inversion Principle
* High level modules should not depend on low level modules. Both should depend on abstractions.
* Abstractions should not depend on details. Details should depend on abstractions.
* Never depend on anything concrete. Depend on abstractions.
* This helps in changing the implementation, without altering the high level code.
![DIP]({{ site.url }}/images/Dependency-Inversion-Principle.PNG)