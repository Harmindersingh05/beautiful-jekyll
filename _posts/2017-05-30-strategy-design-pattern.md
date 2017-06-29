---
layout: post
published: true
title: Strategy Design Pattern
---

The strategy pattern is one of my favorite design pattern. It has made my life substantially easier on a bunch of occasions.  You can use strategy design patterns to simplify code with lot of if-else or switch statements. It can be very useful in situations where you check some condition and then run validation logic that is specific to a thing or a category. The strategy pattern can help you clean up the code by converting the if-else or switch statements into strategy objects.

### Example

To illustrate strategy design pattern in code. We’ll create a simple application to validate the physical address of an account, before creating the account in the database. For the sake of our simple application, we will assume that the following address  fields are required for the given country.

* New Zealand – Suburb
* Australia – Suburb, State, PostalCode
* United States – State, PostalCode

We’ll create two solutions one without design pattern and one with the design pattern, to demonstrate the effectiveness of the strategy design pattern.

We will be working with the following domain classes.

<pre><code class="language-csharp">
public class Account
{
      public string FirstName { get; set; }
      public string LastName { get; set; }
      public Address PhysicalAddress { get; set; }
}

public class Address
{   
      [Required]
      public string Street { get; set; }       
      public string Suburb { get; set; }        
      public string State { get; set; }
      [Required]
      public string City { get; set; }     
      [Required]
      public Country Country { get; set; }
      public string PostalCode { get; set; }
}
</code></pre>   

We will assume that entity framework will handle the validation for fields marked with the required attribute. As these fields are common across all countries.

<pre><code class="language-csharp">
public enum Country
{
      NewZealand,
      Australia,
      UnitedStatesOfAmerica
}
</code></pre>    

<pre><code class="language-csharp">
public class AccountRepository : IAccountRepository
{
      public void Save(Account account)
      {
            //implementation omitted 
      }
}

public class AccountService : IAccountService
{
      private readonly IAccountRepository _accountRepository;
 
      public AccountService(IAccountRepository accountRepository)
      {
          _accountRepository = accountRepository;
      }
 
      public Response SaveAccountDetails(Account account)
      {
          var response = new Response { ResponseStatus = ResponseStatus.Success };
          //TODO: Validate address      
 
           _accountRepository.Save(account);
          return response;
      }      
}
</code></pre>   

The domain classes shouldn’t be difficult to understand. For simplicity, some objects have been removed from this post. 

As seen in the above code, we have to complete the SaveAccountDetails function, this function will perform the country specific address validation. We can implement the validation in two ways, using an strategy design pattern by breaking up the country specific validation into separate strategy objects or without using the design pattern.

First we’ll implement the validation without using the design pattern and then refactor it using the strategy design pattern.

### Solution without strategy design pattern.

<pre><code class="language-csharp">
public class AccountService : IAccountService
{
    private readonly IAccountRepository _accountRepository;

    public AccountService(IAccountRepository accountRepository)
    {
        _accountRepository = accountRepository;
    }

    public Response SaveAccountDetails(Account account)
    {
        var response = new Response { ResponseStatus = ResponseStatus.Success };

        //validate address by country
        switch (account.PhysicalAddress.Country)
        {
            case Country.NewZealand:
                if (string.IsNullOrWhiteSpace(account.PhysicalAddress.Suburb))
                    response.ErrorMessages.Add(response.CreateErrorMessage("Suburb", "Suburb is required."));
                break;
            case Country.Australia:
                if (string.IsNullOrWhiteSpace(account.PhysicalAddress.Suburb))
                    response.ErrorMessages.Add(response.CreateErrorMessage("Suburb", "Suburb is required."));

                if (string.IsNullOrWhiteSpace(account.PhysicalAddress.State))
                    response.ErrorMessages.Add(response.CreateErrorMessage("State", "State is required."));

                if (string.IsNullOrWhiteSpace(account.PhysicalAddress.PostalCode))
                    response.ErrorMessages.Add(response.CreateErrorMessage("PostalCode", "PostalCode is required."));

                break;
            case Country.UnitedStatesOfAmerica:
                if (string.IsNullOrWhiteSpace(account.PhysicalAddress.State))
                    response.ErrorMessages.Add(response.CreateErrorMessage("State", "State is required."));

                if (string.IsNullOrWhiteSpace(account.PhysicalAddress.PostalCode))
                    response.ErrorMessages.Add(response.CreateErrorMessage("PostalCode", "PostalCode is required."));
                break;
        }

        if (response.ErrorMessages.Any())
        {
            response.ResponseStatus = ResponseStatus.Error;
            return response;
        }

        _accountRepository.Save(account);
        return response;
    }
}
</code></pre>   

If you are not familiar with strategy design pattern. The above implementation might look perfectly fine to you. We are performing the country specific validation using switch statements.

It is fine to assume that in future we may introduce new country and with it more country specific validation. The current validation implementation does not take care of the new country. We’d have to manually extend the switch statement to account for the new country.

The modification to the the account service class will violate the Open close principle of SOLID: a class is open for extensions but closed for modification. Further to this, adding more switch statements and if statement will make our account service class more noisy and hard to maintenance. Just imagine adding two more countries in the account service class, it’ll become maintenance nightmare. It is not a good practice to modify our account service class just to accommodate the addition of a new country.

Using the strategy design pattern, we can split the validation code for each countries into separate strategy objects.

### Solution using strategy design pattern.

First we will create a common interface that will be implemented by the strategy objects and used by the account service.

<pre><code class="language-csharp">
public interface IAddressValidationStrategy
{
    Response ValidateAddress(Address address);
 
    bool IsMatch(Country country);
}
</code></pre>  
Next, we have to create strategy objects for each country.

NzAddressValidationStrategy

<pre><code class="language-csharp">
public class NzAddressValidationStrategy : IAddressValidationStrategy
{

    public Response ValidateAddress(Address address)
    {
        var response = new Response { ResponseStatus = ResponseStatus.Success };

        if (string.IsNullOrWhiteSpace(address.Suburb))
            response.ErrorMessages.Add(response.CreateErrorMessage("Suburb", "Suburb is required."));

        if (response.ErrorMessages.Any()) response.ResponseStatus = ResponseStatus.Error;
        return response;
    }

    public bool IsMatch(Country country)
    {
        return country.Equals(Country.NewZealand);
    }
}
</code></pre>  

UsaAddressValidationStrategy

<pre><code class="language-csharp">
public class UsaAddressValidationStrategy : IAddressValidationStrategy
{
    public Response ValidateAddress(Address address)
    {
        var response = new Response {ResponseStatus = ResponseStatus.Success};

        if (string.IsNullOrWhiteSpace(address.State))
            response.ErrorMessages.Add(response.CreateErrorMessage("State", "State is required."));

        if (string.IsNullOrWhiteSpace(address.PostalCode))
            response.ErrorMessages.Add(response.CreateErrorMessage("PostalCode", "PostalCode is required."));

        if (response.ErrorMessages.Any()) response.ResponseStatus = ResponseStatus.Error;
        return response;
    }

    public bool IsMatch(Country country)
    {
        return country.Equals(Country.UnitedStatesOfAmerica);
    }
}
</code></pre> 

AusAddressValidationStrategy

<pre><code class="language-csharp">
public class AusAddressValidationStrategy : IAddressValidationStrategy
{
    public Response ValidateAddress(Address address)
    {
        var response = new Response { ResponseStatus = ResponseStatus.Success };

        if (string.IsNullOrWhiteSpace(address.Suburb))
            response.ErrorMessages.Add(response.CreateErrorMessage("Suburb", "Suburb is required."));

        if (string.IsNullOrWhiteSpace(address.State))
            response.ErrorMessages.Add(response.CreateErrorMessage("State", "State is required."));

        if (string.IsNullOrWhiteSpace(address.PostalCode))
            response.ErrorMessages.Add(response.CreateErrorMessage("PostalCode", "PostalCode is required."));

        if (response.ErrorMessages.Any()) response.ResponseStatus = ResponseStatus.Error;
        return response;
    }

    public bool IsMatch(Country country)
    {
        return country.Equals(Country.Australia);
    }
}
</code></pre> 

<pre>
<code class="language-csharp">
public class AccountService : IAccountService
{
    private readonly List&lt;IAddressValidationStrategy&gt; _addressValidationStrategies;
    private readonly IAccountRepository _accountRepository;

    public AccountServiceWithPattern(List&lt;IAddressValidationStrategy&gt; addressValidationStrategies,
        IAccountRepository accountRepository)
    {
        _addressValidationStrategies = addressValidationStrategies;
        _accountRepository = accountRepository;
    }

    public Response SaveAccountDetails(Account account)
    {
        var response =
            _addressValidationStrategies.First(m => m.IsMatch(account.PhysicalAddress.Country)).
                ValidateAddress(account.PhysicalAddress);

        if (response.ResponseStatus.Equals(ResponseStatus.Success))
            _accountRepository.Save(account);

        return response;
    }
}
</code>
</pre>

As seen in above we have introduced a common interface for our validation and split the country specific validation code into strategy objects. We also maintain a list of validation strategies inside the account service class. The usage of the common validation interface has drastically reduced the validation code inside the SaveAccountDetails function. The improved account service simply locates the first matching validation strategy for the country and performs the validation.

Using poor man’s dependency injection we can put everything together as follows:

<pre><code class="language-csharp">
public class Program
{
    public static void Main(string[] args)
    {
        var addressValidation = new List&lt;IAddressValidationStrategy&gt;
        {
            new AusAddressValidationStrategy(),
            new UsaAddressValidationStrategy(),
            new NzAddressValidationStrategy()
        };

        var accountRepository = new AccountRepository();
        var accountService = new AccountServiceWithPattern(addressValidation, accountRepository);

        var account = new Account
        {
            FirstName = "John",
            LastName = "doe",
            PhysicalAddress = new Address
            {
                Street = "124 Queen Street",
                Suburb = "Ponsonby",
                City = "Auckland",
                Country = Country.NewZealand,
                PostalCode = "1023"
            }
        };

        accountService.SaveAccountDetails(account);
    }
}
</code></pre> 

I am sure you would agree that the account service looks lot more polished. If we have to add a new country and validation, we can simply create a concrete strategy object and implement the IAddressValidationStrategy and include it in the strategy validation list that is passed to our account service.

We no longer have to modify the account service class to add new country specific validation. This also prevents us from violating the open/close principle of SOLID.
