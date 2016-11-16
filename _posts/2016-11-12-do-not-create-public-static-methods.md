---
layout: post
title: Do not create public static methods
---

The usage of **public static** methods of *\~Utils* classes is very popular. Those methods are great if being a part of the library. Therefore, you shouldn't create them unless you're [developing another StringUtils class](http://http://eclipsesource.com/blogs/2013/11/06/get-rid-of-your-stringutils/) or something similar.

## Why the temptation is so strong?

You could be convinced you need to use the **public static** method, because:

*   your method doesn't operate on any of the instance variables (remember [cohesion](https://books.google.pl/books?id=_i6bDeoCQzsC&lpg=PP1&pg=PA140)?);
*   your method is a *util* - you will use it in many places to do a simple thing;
*   you have a method that doesn't fit in well into a single Entity nor a Value Object.

But you really shouldn't create a **public static** method to do it. Why?

## Example

Imagine that you're developing a price calculator. There are two use cases to implement.

### Use Case 1: Multiplication

*As a client (`PriceFacade`), I want to have `PriceCalculatingService` being able to multiply list of prices, so I can display it.*

The multiplication of prices is a feature tied directly to the prices itself. It is easily implemented in the class wrapping the collection of prices.

<div class="title">PriceFacade.java - client code</div>

``` java
public class PriceFacade {
    private final PriceCalculatingService priceCalculatingService;

    public PriceFacade(PriceCalculatingService priceCalculatingService) {
        this.priceCalculatingService = priceCalculatingService;
    }

    public Price multiply(Prices prices) {
        return priceCalculatingService.multiply(prices);
    }
}
```

`PriceCalculatingService` is delegating the multiplication logic to the `Prices`, which do the math:

<div class="title">PriceCalculatingService.java - application service</div>

```` java
public class PriceCalculatingService {
    public Price multiply(Prices prices) {
        return prices.multiplyAll();
    }
}
````

<div class="title">Prices.java - prices' collection wrapper</div>

```` java
public class Prices {
    private final Collection<Price> prices;

    // constructor, etc.

    public Price multiplyAll() {
            return prices.stream()
                         .reduce(Price::multiply)
                         .get();
    }
````

### Use Case 2: Rounding

*As a client (`PriceFacade`), I want to have `PriceCalculatingService` being able to **properly round** multiplied list of prices, so I can display it better.*

That's the tricky part. Obviously, `Price` should be able to round itself, just as it can multiply itself. However, what does *properly* mean? Should `Price` know what are the proper rounding settings in the current context? I don't suppose so. We have to come up with another class to hold that information.

## Static method solution

We create a `PriceRoundingService` - a *util* class, that contains a single **public static** method:

```` java
public class PriceRoundingService {
    public static Price round(Price price) {
        return price.round(2, RoundingMode.HALF_UP);
    }
}
````

and use it in `PriceCalculatingService` by importing it:

```` java
import com.github.mateuszrasinski.blog.staticmethod.service.PriceRoundingService;

public class PriceCalculatingService {
    public Price multiply(Prices prices) {
        Price product = prices.multiplyAll();
        return PriceRoundingService.round(product);
    }
}
````

The code works. What's wrong then?

### Testing

You can't (or shouldn't have to) mock static method calls. From now on, every time you would like to test any code that uses `PriceRoundingService` you have to rely on its implementation and either *know* what it does or *use* it in the assertion section of the test.

```` groovy
class PriceCalculatingServiceTest extends Specification {

    private PriceCalculatingService priceCalculatingService = new PriceCalculatingService()

    def 'should multiply two prices'() {
        given:
            Price price1 = new Price(1.50, Currency.getInstance('PLN'))
            Price price2 = new Price(1.75, Currency.getInstance('PLN'))
            Prices prices = new Prices([price1, price2])
        when:
            Price product = priceCalculatingService.multiply(prices)
        then:
            product.value == (price1.value * price2.value).setScale(2, RoundingMode.HALF_UP) // <1> 
            product == PriceRoundingService.round(
                    new Price((price1.value * price2.value), price1.getCurrency())
            ) // <2>
    }

    // <1> Repeated implementantion
    // <2> Static method used in the assertion section
}
````

### Changes

Let's assume, that you need to dynamically change the scale of the price's rounding. With **static** method you can't use any instance variable (you can't inject dependency), nor you can extend it. All you can do, is to introduce a new parameter to the method.

```` java
public class PriceRoundingService {
    public static Price round(Price price, RoundingSettings roundingSettings) {
        return price.round(roundingSettings.getScale(), roundingSettings.getMode());
    }
}
````

That makes your client code has to provide the information about rounding. The problem how to *properly* round the price is now also the client's responsibility. Not nice.

```` java
public class PriceCalculatingService {

    private final RoundingSettings roundingSettings;
    // how to initialize roundingSettings? maybe statically, again...

    public Price multiply(Prices prices) {
        Price product = prices.multiplyAll();
        return PriceRoundingService.round(product, roundingSettings);
    }
}
````

As if that weren't bad enough, the new parameter has also broken the tests that were using the static method. The following line is not compiling anymore:

```` groovy
product == PriceRoundingService.round(new Price((price1.value * price2.value), price1.getCurrency()))
````

We would have to pass a `RoundingSettings` instance as the second argument of the `PriceRoundingService.round()` method. We'd just have to *know* how to use it. Again…​

## Object-oriented solution

We create nearly the same `PriceRoundingService`. The only change is that `round(Price)` is **not** static:

```` java
public class PriceRoundingService {
    public Price round(Price price) {
        return price.round(2, RoundingMode.HALF_UP);
    }
}
````

The usage is more demanding to initialize, but it states the needed dependency clearly in the constructor:

```` java
public class PriceCalculatingService {
    private final PriceRoundingService priceRoundingService;

    public PriceCalculatingService(PriceRoundingService priceRoundingService) {
        Assert.notNull(priceRoundingService);
        this.priceRoundingService = priceRoundingService;
    }

    public Price multiply(Prices prices) {
        Price product = prices.multiplyAll();
        return priceRoundingService.round(product);
    }
}
````

The code works, too. It has even lines. What's so great about it, then?

### Testing

We can stub the call to the external service easily, so we can focus on the single responsibility of the class. Each class can be tested independently. Additionally, if there is a need to test the interactions with the external class, it can be mocked.

```` groovy
class PriceCalculatingServiceTest extends Specification {

    private PriceCalculatingService priceCalculatingService

    void setup() {
        PriceRoundingService priceRoundingServiceMock = Stub(PriceRoundingService) {
            round(_ as Price) >> { Price price -> return price }
        }
        priceCalculatingService = new PriceCalculatingService(priceRoundingServiceMock)
    }

    def 'should multiply two prices'() {
        given:
            Price price1 = new Price(1.50, Currency.getInstance('PLN'))
            Price price2 = new Price(1.75, Currency.getInstance('PLN'))
            Prices prices = new Prices([price1, price2])
        when:
            Price product = priceCalculatingService.multiply(prices)
        then:
            product.value == price1.value * price2.value
    }

    // ...
}
````

### Changes

With OO design we can simply implement changes to the service, as we did in the client code - by injecting the dependency:

```` java
public class PriceRoundingService {

    private final RoundingSettings roundingSettings;

    public PriceRoundingService(RoundingSettings roundingSettings) {
        this.roundingSettings = roundingSettings;
    }

    public Price round(Price price) {
        return price.round(roundingSettings.getScale(), roundingSettings.getMode());
    }
}
````

And that's it. We don't have to change the client code, because the contract of the `round(Price)` method hasn't changed and we don't have to change the tests - because we've stubbed the service call. One reason to change made us change one class's code. That's how it supposed to be.

## Summary

*-Helper* or *-Utils* classes are perfectly fine to patch a common class's bad design or lack of features. Good examples are: StringUtils (from Apache or Spring) or ~~DateUtils~~ (use `java.time` package!). The `String` class is heavily used in different ways and it's missing a lot of features. You can't just add a feature to the `String`, so in order to do something more with it, public static methods from the `StringUtils` are used. That's fine.

What is definitely not fine is creating *-Utils* class in your own project to use with your own classes. Instead of fixing the bad design - it would worsen it. There are better ways to write reusable code, but they may need a little more effort upfront.
