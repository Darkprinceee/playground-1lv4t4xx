We've found, while teaching developers about lambdas and Streams in Java 8, that many developers really appreciate the code that comes out at the other end. It tends to read a lot more like the problem that developers are trying to solve than traditional for loops and, as a side effect, is often shorter to boot. That doesn't mean that the Streams API is perfect or even complete. In this article we will be looking at a few small improvements that were introduced in Java 9 that make it even better.

# ofNullable
The `Stream` interface has a couple of factory methods called of() that allow you to create Streams from pre-specified values - one is an overload for a single value and the other takes a varargs parameter. These are incredibly useful both when you're trying to test out Streams code and also when you want to just instantiate a Stream with a few values. Java 9 adds an `ofNullable()` factory - let's see how you might use it and what it's for.

Let's suppose we're trying to find a location to put some configuration files in a Java application. We want to look at a couple of different properties - e.g. "app.config" and "app.home". Let's write this code in Java 8:

```java
final String configurationDirectory =
    Stream.of("app.config", "app.home", "user.home")
        .flatMap(key -> {
            final String property = System.getProperty(key);
            if (property == null) {
                return Stream.empty();
            }
            else {
                return Stream.of(property);
            }
        })
        .findFirst()
        .orElseThrow(IllegalStateException::new);
```

Hmm, that's a little bit ugly right - What's going on here? Well we're looking up each property in our Stream and using the flatMap operation. We use flatMap here because it allows us to map an element to 0 or 1 elements in the Stream. If we can lookup the system property then we return a Stream containing only it, but if we can't look it up then we return an empty Stream in its place. This results in no element being added into the stream.

Unfortunately what we've ended up with is a fairly large statement style lambda expression with a null check in the middle of the code. One alternative would be to use a ternary operator

```java
final String configurationDirectory =
    Stream.of("app.config", "app.home", "user.home")
        .flatMap(key -> {
            final String property = System.getProperty(key);
            return property == null ? Stream.empty() : Stream.of(property);
        })
        .findFirst()
        .orElseThrow(IllegalStateException::new);
```

Even after this refactor, however, the code reads slightly inelegantly. Java 9's `ofNullable` would allow us to write the same pattern much more succinctly and more readably.

```java runnable
// { autofold
import java.util.stream.Stream;

public class Main {
    public static void main(String[] args) {
// }

final String configurationDirectory =
    Stream.of("app.config", "app.home", "user.home")
        .flatMap(key -> Stream.ofNullable(System.getProperty(key)))
        .findFirst()
        .orElseThrow(IllegalStateException::new);
System.out.println(configurationDirectory);

//{ autofold
    }
}
//}
```

It's worth noting for the sake of completeness that this isn't the only way to solve this problem in code, we could also have mapped over the System.getProperty() function and then filtered out the null values. That is perhaps a more natural way of writing this code for some people, but has the downside of resulting in null values getting into our Stream - something we try to steer clear of.

# takeWhile and dropWhile
We have an application that is processing payments being made on an ecommerce website and we're maintaining a list of all payments in the current day that are sorted from the most expensive down to the cheapest. We have a business requirement to produce a report on every payment that is £500 or greater in value at the end of the day. A natural way of writing this code using Java 8 Streams might be:

```java
final List<Payment> expensivePayments = paymentsByValue
    .stream()
    .filter(payment -> payment.getValue() >= 500)
    .collect(Collectors.toList());
```

Unfortunately the downside of this approach is that if you start processing lots and lots of transactions in a day the `filter` operation gets applied to every transaction in your input list. You know that your input list is sorted by descending value of the transaction, so once you have found a transaction that fails your predicate every transaction after that point can be filtered out. Thankfully Java 9 solves this problem with the addition of a takeWhile operation.

```java runnable
// { autofold
import java.util.List;

import static java.util.stream.Collectors.toList;

public class Main {
    public static void main(String[] args) {
        List<Payment> paymentsByValue = List.of(
            new Payment(900),
            new Payment(700),
            new Payment(500),
            new Payment(300),
            new Payment(100),
            new Payment(0)
        );
// }

final List<Payment> expensivePayments = paymentsByValue
    .stream()
    .takeWhile(payment -> payment.getValue() >= 500)
    .collect(toList());
System.out.println(expensivePayments);

//{ autofold
    }
}

class Payment {
	private final int value;

	Payment(int value) {
		this.value = value;
	}

	int getValue() {
		return value;
	}

	@Override
	public String toString() {
		return String.format("Payment(value: %d)", value);
	}
}
//}
```

While `filter` retains all elements in the Stream that match its predicate, `takeWhile` stops once it has found an element that fails to match. The `dropWhile` operation does the inverse: throws away the elements at the start where the predicate is false.

# Notes
This playground is based on IteratrLearning's article [Stream Improvements in Java 9](http://iteratrlearning.com/java9/2016/08/06/java9-streams.html)

Similar playground by Gowgi: [Java 9 Streams Enhancements](https://tech.io/playgrounds/2513/java-9-streams-enhancements)
