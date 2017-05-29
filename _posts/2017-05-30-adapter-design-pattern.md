---
layout: post
published: true
title: Adapter Design Pattern
---

In software development, the adapter pattern is definitely one of the most used design pattern. It is used to make two incompatible interfaces work together. The definition of the adapter provided in the original Gang of Four book on design patterns states:
>Convert the interface of a class into another interface clients expect. Adapter lets classes work together that couldn’t otherwise because of incompatible interfaces.

### Real life example

One of the real life example of an adapter design pattern is a laptop charger. We have different electric outlets all over the world. The laptop charger from US will not be compatible with the wall outlet in Europe. This is where an adapter is useful, an adapter that is compatible with local power outlet can be used to resolve this incompatibility.

[image here]

### Business example

To illustrate adapter design pattern in business scenario. Lets say the company that you work for has decided to extend their product website by implementing a module to process credit card payments. For now, the business has decided to go with Online Payments Ltd and Express Payments Ltd as the payment gateways. In future they might also decide to go with PayPal. In addition to this, business want to be able to chose which payment gateway should be used at given time.

Both payment gateways provide Api that we can integrate with. The Api endpoints that we are interested in are provided in c#:

```C#
public class OnlinePaymentGateway
{
     public void ProcessCreditCardPayment(CreditCardInfo creditCardInfo)
     {
         //implementation omitted
     }
}
```
```C#
public class ExpressPaymentGateway
   {
       public void MakeCreditCardPayment(CreditCard creditCard)
       {
           //implementation omitted
       }
   }
```
Note how both endpoints have different method signatures that can be used to make payments. In our website we have the following PaymentService class that is responsible for processing credit card payments, using one of the two payment gateways. This service also uses the ICreditCardRepository to retrieve the credit card details for the user making the payment.

```C#
public class PaymentService
{
    private readonly ICreditCardRepository _creditCardRepository;

    public PaymentService(ICreditCardRepository creditCardRepository)
    {
        this._creditCardRepository = creditCardRepository;
    }

    public void MakePayment(decimal amount, int userId)
    {
        var creditCardDetails = _creditCardRepository.GetCreditCard(userId);

        //Make payment using 3rd party payment gateways
    }
}
```
We have to complete the MakePayment function, this function must communicate with the active gateway and process payments. This function can be implemented in two ways, using an adapter design pattern and without it. We’ll first implement it without using the adapter design pattern then refactor it using the adapter pattern.

### Solution without adapter pattern.
```C#
public class PaymentService
{
    private readonly ICreditCardRepository _creditCardRepository;
    private readonly IPaymentGatewayConfig _paymentGatewayConfig;

    public PaymentService(ICreditCardRepository creditCardRepository, IPaymentGatewayConfig paymentGatewayConfig)
    {
        _creditCardRepository = creditCardRepository;
        _paymentGatewayConfig = paymentGatewayConfig;
    }

    public void MakePayment(decimal amount, int userId)
    {
        var creditCardDetails = _creditCardRepository.GetCreditCard(userId);

        //Make payment via 3rd party payment gateways
        if (_paymentGatewayConfig.GetActivePaymentGateway() == PaymentGateway.ExpressPaymentGateway)
        {
            var onlineGateway = new OnlinePaymentGateway();
            onlineGateway.ProcessCreditCardPayment(amount, creditCardDetails);
        }
        else
        {
            var onlineGateway = new ExpressPaymentGateway();
            onlineGateway.MakeCreditCardPayment(amount, creditCardDetails);
        }
    }
}
```
As seen in the code above, We have introduced the IPaymentGatewayConfig, this is used to retrieve the active payment gateway from database. Second, we have integrated the 3rd party payment gateways directly into the MakePayment method.

With this implementation we’re stuck with the two payment gateways. What if we want to add another gateway? In that case we’d need to go in and modify the PaymentService. Even worse, lets say the business wants to add 3 more payment gateways the MakePayment method will become quite noisy.

On a different note: the method also violates the Open close principle of SOLID as it requires modification for further extension of the PaymentService class.

To support more gateways we’ll factor out the payment gateways using an adapter pattern.

### Solution using adapter pattern.
Create a common adapter interface for all payment gateways.

```c#
public interface IPaymentGatewayAdapter
{
     void ProcessPayment(decimal amount, CreditCard creditCardInfo);
}
```

Next we want to update the PaymentService class to depend on this interface.

```C#
public class PaymentService
{
    private readonly ICreditCardRepository _creditCardRepository;
    private readonly IPaymentGatewayAdapter _paymentGatewayAdapter;

    public PaymentService(ICreditCardRepository creditCardRepository,
        IPaymentGatewayAdapter paymentGatewayAdapter)
    {
        _creditCardRepository = creditCardRepository;
        _paymentGatewayAdapter = paymentGatewayAdapter;
    }

    public void MakePayment(decimal amount, int userId)
    {
        var creditCardDetails = _creditCardRepository.GetCreditCard(userId);
        _paymentGatewayAdapter.ProcessPayment(amount, creditCardDetails);
    }
}
```

We’ve completely got rid of all references to the payment gateway API. Now this class has single responsibility that is to only make payments and does not need to know anything about payment gateways. We now have one interface that PaymentService can depend on. The next task is to inject the payment gateway classes using the IPaymentGatewayAdapter. create two classes that will adapt to IPaymentGatewayAdapter.

```C#
public class OnlinePaymentGatewayAdaptee : IPaymentGatewayAdapter
{
    public void ProcessPayment(decimal amount, CreditCard creditCardInfo)
    {
        var onlineGateway = new OnlinePaymentGateway();
        onlineGateway.ProcessCreditCardPayment(amount, creditCardInfo);
    }
}

public class ExpressPaymentGatewayAdaptee : IPaymentGatewayAdapter
{
    public void ProcessPayment(decimal amount, CreditCard creditCardInfo)
    {
        var expressPayment = new ExpressPaymentGateway();
        expressPayment.MakeCreditCardPayment(amount, creditCardInfo);
    }
}
```

Note how the adapter pattern is useful in this situation. It is resolving the incompatibility of the two Interfaces the OnlinePaymentGateway and the ExpressPaymentGateway. This is the job of the adapter pattern, as stated in the description “Adapter lets classes work together that couldn’t otherwise because of incompatible interfaces”.

Using poor man’s dependency injection we can put everything together as follows:

```c#
public class Program
{
    private static void Main(string[] args)
    {
        IPaymentGatewayConfig config = new PaymentGatewayConfig();
        IPaymentGatewayAdapter paymentGatewayAdapter;

        if (config.GetActivePaymentGateway() == PaymentGateway.ExpressPaymentGateway)
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
}
```
In the future if you want to add more payment gateways then all you need to do is create another adapter class for that. Note that we have not used any Inversion of Control containers to inject our dependencies so that we don’t get distracted by technical details. Instead we’ll do it the old way, i.e. by poor man’s dependency injection.

We can further improve the code above by refactoring the creation of the payment gatways into a abstract factory. I will do this in a another blog post on abstract factory design pattern.


























