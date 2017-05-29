---
layout: post
published: true
title: Abstract Factory Design Pattern
---
The Abstract factory pattern is extremely useful among all design patterns. It can be used to solve various problems in dependency injections. In this post, we’ll have a look at two example of abstract factory design pattern.

In the first example we’ll use the simple Foo class. The second example extends the solution provided in my adapter design pattern blog post. We will refactor the solution to make it more DI friendly.

### What problem does it solve
The most common problem that you can use abstract factory pattern to solve is to address the problem of creating objects at run time based on user’s input.

### Example
Assume for an example that you have a class that has two dependencies. One of the dependency is not known until run time.

When using dependency injections you cannot instantiate/wire up such class. Why? because one of the dependency is not known until run time. DI constructs all it’s dependencies at application start up stage. If any dependency are not known at this stage it cannot be wired up using DI.

Following code illustrates this problem.

{% highlight C# 
public interface IFoo {}
 
public class Foo : IFoo
{
    public Foo(IDependency dependency, string data) { }
}

%}

The Foo class has an dependency on the IDependency interface and the data parameter. The data parameter is not known until run-time. This class cannot be constructed using dependency injection as described above. What is the solution?

### Solution
Abstract factory – any place where you need a run-time value to construct a particular dependency. Abstract Factory is very useful.

```C#
public interface IFooFactory
{
    IFoo Create(string data);
}
 
public class FooFactory : IFooFactory
{
    private readonly IDependency _dependency;
         
    public FooFactory(IDependency dependency)
    {
          _dependency = dependency;
    }
 
    public IFoo Create(string data)
    {
        return new Foo(_dependency, data);
    }
}
```

As seen above we have introduced the FooFactory which is now responsible for creating an instance of Foo. It also takes a dependency on IDependency interface which is known before run time. The factory exposes a create method, this method can be used by the consumer to create an instance of foo at run time.

In DI container, we would register both the IDependency and IFooFactory. All consumers where we needed an instance of IFoo. We can now simply take a dependency on IFooFactory by requesting it through the constructor injection.

Example of consumer using IFooFactory.

```C#
public class Consumer
{
    private readonly IFooFactory _fooFactory;
    public Consumer(IFooFactory fooFactory)
    {
         _fooFactory = fooFactory;
    }
 
    public void DoSomething(string userInput)
    {
         var foo = _fooFactory.Create(userInput);
         // use your foo.
    }
}
```

### Business Example
The foo example used above is quite simple. Let’s have a look at a business example.

This example will extend the solution given in my post on adapter pattern. The adapter design pattern solution 
is not DI friendly. This was intentional to get the point across without making the solution more complex.

### Problem
The problem we’ll solve using the abstract pattern in this example, is when there are multiple implementation of 
one interface. And the user has to decide at run time which implementation to use.

Consider the following code taken from adapter pattern post.

```C#
public static void Main(string[] args)
{
       IConfiguration configuration = new Configuration();
       IPaymentGatewayAdapter paymentGatewayAdapter;
 
       if (configuration.IsActivePaymentGateway(PaymentGateway.ExpressPaymentGateway))
           paymentGatewayAdapter = new ExpressPaymentGatewayAdaptee();
       else
       {
           paymentGatewayAdapter = new OnlinePaymentGatewayAdaptee();
       }
 
       ICreditCardRepository creditCardRepository = new CreditCardRepository();
       IPaymentService paymentService = new PaymentService(creditCardRepository, paymentGatewayAdapter);
 
       decimal amount = 20;
       int userId = 20312;
       paymentService.MakePayment(amount, userId);
}
```
The above code should not be too difficult to understand. It creates an concentrate implementation of 
IPaymentGatewayAdapter based on the payment gateway provided in the configuration file. It then passes 
the IPaymentGatewayAdapter to the IPaymentService which uses it to make payments. The IPaymentGatewayAdapter 
has two implementations ExpressPaymentGatewayAdaptee and OnlinePaymentGatewayAdaptee.

Previously, our company controlled which payment gateway the customer can use to pay for products. 
But, the requirements has since changed, now the business has decided it would be good to provide 
our customer with the option to select which gateway they prefer paying through.

As seen in the code example above, the IPaymentGatewayAdapter is initialized based on configurations. 
To accommodate the new requirements the configuration class has to go.

```C#
public static void Main(string[] args)
 {             
       IPaymentGatewayAdapter paymentGatewayAdapter;
       var paymentGateway = Console.ReadLine();
 
       if (paymentGateway.Equals("ExpressPaymentGatewayAdaptee"))
       {
           paymentGatewayAdapter = new ExpressPaymentGatewayAdaptee();
       }
       else if (paymentGateway.Equals("OnlinePaymentGatewayAdaptee"))
       {
           paymentGatewayAdapter = new OnlinePaymentGatewayAdaptee();
       }
             
       ICreditCardRepository creditCardRepository = new CreditCardRepository();
       IPaymentService paymentService = new PaymentService(creditCardRepository, paymentGatewayAdapter);
 
       decimal amount = 20;
       int userId = 20312;
       paymentService.MakePayment(amount, userId);
}
```
In the code above the configuration file has been removed and we are initialing the 
PaymentGatewayAdapter based on the user’s input.

But, in any web application for example, where DI container will most likely be used to construct dependencies. 
This solution will not work. That is because DI constructs all dependencies at application start up 
stage and the user’s input is not available at this stage.

### Solution

You guessed it, abstract factory. Let’s refactor the above code to make it more DI friendly using a abstract factory.

```C#
public static void Main(string[] args)
{
     IPaymentGatewayFactory paymentGatewayFactory = new PaymentGatewayFactory();
     ICreditCardRepository creditCardRepository = new CreditCardRepository();
     IPaymentService paymentService = new PaymentService(creditCardRepository, paymentGatewayFactory);
 
     decimal amount = 20;
     int userId = 20312;
     string gatewayName = "ExpressPaymentGatewayAdaptee";
     paymentService.MakePayment(gatewayName,amount, userId);
}
```
In the above code, we have introduced IPaymentGatewayFactory, modified the MakePayment method and updated 
the PaymentService class to depend on the IPaymentGatewayFactory. The IPaymentGatewayFactory factory is 
responsible for initialing the IPaymentGatewayAdapter based on the customer’s selection. The customer’s 
selection is provided through the new parameter on MakePayment method.

PaymentGatewayFactory.cs

```C#
public class PaymentGatewayFactory : IPaymentGatewayFactory
{
     Dictionary<string, Type> _paymentGateways;
 
     public PaymentGatewayFactory()
     {
         LoadPaymentGatewayTypes();
     }
 
     public IPaymentGatewayAdapter CreatePaymentGateway(string gatewayName)
     {
         IPaymentGatewayAdapter paymentGateway = null;
 
         try
         {               
             var type = _paymentGateways.ContainsKey(gatewayName) ? _paymentGateways[gatewayName] : null;
             paymentGateway = Activator.CreateInstance(type) as IPaymentGatewayAdapter;
         }
         catch (Exception ex)
         {
             throw ex;
         }
                        
        return paymentGateway;
     }
       
     private void LoadPaymentGatewayTypes()
     {
         _paymentGateways = new Dictionary<string, Type>();
 
         Type[] typesInAssembly = Assembly.GetExecutingAssembly().GetTypes();
         var paymentGatewayTypes = typesInAssembly
             .Where(type => type.GetInterface(typeof(IPaymentGatewayAdapter).ToString()) != null);
 
         foreach (Type type in paymentGatewayTypes)
         {
             _paymentGateways.Add(type.Name.ToLower(), type);
         }
}
```

The PaymentGatewayFactory uses reflection to find all concrete classes of type IPaymentGatewayAdapter based on 
the customer’s selection.  We could have used If else statements to select the correct implementation. 
But that would violate the Open/Closed Principle of SOLID. As adding a new PaymentGateway will require 
modification to the factory. With the current implementation, reflection will automatically pick any new 
class that implements IPaymentGatewayAdapter.

Re-factored PaymentService.cs

```C#
public class PaymentService : IPaymentService
{
     private readonly ICreditCardRepository _creditCardRepository;
     private readonly IPaymentGatewayFactory _paymentGatewayFactory;
 
     public PaymentService(ICreditCardRepository creditCardRepository, IPaymentGatewayFactory paymentGatewayFactory)
     {
         _creditCardRepository = creditCardRepository;
         _paymentGatewayFactory = paymentGatewayFactory;
     }
 
     public void MakePayment(string gatewayName, decimal amount, int userId)
     {
         var creditCardDetails = _creditCardRepository.GetCreditCard(userId);
         var paymentGateway = _paymentGatewayFactory.CreatePaymentGateway(gatewayName);
         paymentGateway.ProcessPayment(amount, creditCardDetails);
     }
}
```
As demonstrated in the two examples above, the abstract factory pattern fits perfectly when you 
have to construct a implementation based on run-time values.






























