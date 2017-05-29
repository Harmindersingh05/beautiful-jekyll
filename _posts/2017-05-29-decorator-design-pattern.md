---
layout: post
published: true
title: Decorator Design Pattern
subtitle: >-
  The decorator design pattern is one of the structural pattern originally
  introduced in the Gang of Four design patterns book.
---
The decorator design pattern is one of the structural pattern originally introduced in the Gang of Four design patterns book.

>It’s intent is to attach additional responsibilities to an object dynamically. Decorators provide a
flexible alternative to subclassing for extending functionality

Let’s go over this pattern with some code examples. For demonstration, consider the following product repository interface and concrete implementation.

```C#
public interface IProductRepository
{
     Product Get(string code);
     Product Save(Product product);
}
```

```C#
public class ProductRepository : IProductRepository
{
     private readonly IDbContext _dbContext;
 
     public ProductRepository(IDbContext dbContext)
     {
         _dbContext = dbContext;
     }

     public Product Get(string code)
     {
         var product = _dbContext
             .Products
             .SingleOrDefault(m => m.Code == code);
 
         return product;
     }
 
     public Product Save(Product product)
     {
         _dbContext.Products.Add(product);
         _dbContext.SaveChanges();
 
         return product;
     }
}
```

Nothing too completed there. The product repository has two methods Get(string code) which returns product that matches the code provided from the database. And the Save() method, which saves a product to the database.

Let’s say that you want to add caching to the product repository so that you don’t have to query the product from database every time. There are three ways to achieve this, one of course is to add caching into the body of the Get method.

### Solution 1 – Adding caching to Get(string code) method

```C#
public Product Get(string code)
{
       ObjectCache cache = MemoryCache.Default;
       if (cache.Contains(code))
           return (Product)cache;
 
       var product = _dbContext
           .Products
           .SingleOrDefault(m => m.Code == code);
 
       if (product == null) return null;
 
       //add product to cache and ensure it is removed from the cache after 1 minute.
       var cacheItemPolicy = new CacheItemPolicy();
       cacheItemPolicy.AbsoluteExpiration = new DateTimeOffset(DateTime.Now.AddMinutes(1));           
       cache.Add(code, product, cacheItemPolicy);
 
       return product;
}
```

The above code should not be too difficult to understand. MemoryCache is used to cache the product. First, we check whether the product being requested is cached. If yes, the product is returned else the database is queried and if product exists it is added to the cache.

### What is wrong with this solution.

There are few things wrong with this solution. First, this solution violates the single responsibility principle of SOLID. The product repository has two responsibility. Caching and retrieving products from database. Mixing responsibility reduces meaningful test-ability and creates extra noise in the code.

Further to this, what if the client wants to choose at run-time whether to query product from cache or go to the database? With current implementation this is not possible. We could modify the Product Get(string code) method to take an extra parameter Get(string code, bool queryCached).

But, modifying the get product method could potentially break other clients that use this method. On another note, modifying existing classes violates the Open/closed principle.

>Software entities (classes, modules, functions, etc.) should be open for extension, but closed for modification.

So how can we achieve this without violating the Open/closed principle?




























