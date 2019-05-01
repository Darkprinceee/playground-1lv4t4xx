Java 8 introduced `CompletableFuture<T>` as an enhancement to `Future<T>`. It is a new class which lets you express the flow of information from different tasks using a callback driven style. A `CompletableFuture` is used for defining computations on singular events, which is a different use case than computations on streams of events (e.g. `Observable` using RxJava). In this article, you will learn about the problem with timeouts in Java 8's CompletableFuture and the improvements that Java 9 brings.

# Combining two services
For the purpose of this article, let's say you'd like to combine the result of two services over the network:
1. A best price finder for a flight route
2. An exchange service that converts USD to GBP

Both of these services will introduce a certain delay before responding back with a result. This is due to the costs of network communication with the service.

You could solve this problem by making use of CompletableFuture as follows (by default a CompletableFuture uses the [common thread pool](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html) but this can be parametrised with an `Executor` using an overload of `supplyAsync`):

```java runnable
// { autofold
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Random;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ThreadLocalRandom;
import java.util.function.Supplier;

public class Main {
    private static FlightRoutePriceFinder flightRoutePriceFinder = new FlightRoutePriceFinder();
    private static ExchangeService exchangeService = new ExchangeService();

    public static void main(String[] args) throws ExecutionException, InterruptedException {
// }

BigDecimal amount =
    CompletableFuture.supplyAsync(() -> flightRoutePriceFinder.bestFor(AirportCode.LCY, AirportCode.JFK))
        .thenCombine(CompletableFuture.supplyAsync(() -> exchangeService.rateFor(Currency.GBP)),
            Main::convert)
        .get();
System.out.printf("The price is %s %s", amount, Currency.GBP);

//{ autofold
    }

    private static BigDecimal convert(BigDecimal price, BigDecimal rate) {
        return Utils.decimal(price.multiply(rate));
    }
}

class FlightRoutePriceFinder {
    BigDecimal bestFor(AirportCode departure, AirportCode destination) {
        return bestPriceWithDelay(departure, destination);
    }

    private BigDecimal bestPriceWithDelay(AirportCode departure, AirportCode destination) {
        return new Delay<BigDecimal>().random()
            .then(() -> {
                double price = 10 * Utils.randomChar(departure.getName()) + Utils.randomChar(destination.getName());
                return Utils.decimal(price);
            });
    }
}

enum AirportCode {
    LCY("London City Airport"), JFK("John F. Kennedy International Airport");

    private final String name;

    AirportCode(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }
}

enum Currency {
    USD(1.0), GBP(0.769375);

    public final BigDecimal rate;

    Currency(Double rate) {
        this.rate = new BigDecimal(rate.toString());
    }
}

class ExchangeService {
    BigDecimal rateFor(Currency currency) {
        return rateWithDelayFor(currency);
    }

    private BigDecimal rateWithDelayFor(Currency currency) {
        return new Delay<BigDecimal>().random()
            .then(() -> currency.rate);
    }
}

class Delay<T> {
    private static final int MIN_DELAY_IN_MS = 750;
    private static final int MAX_DELAY_IN_MS = 1000;

    Delay<T> random() {
        int delayInMs = ThreadLocalRandom.current().nextInt(MIN_DELAY_IN_MS, MAX_DELAY_IN_MS);
        try {
            Thread.sleep(delayInMs);
        } catch (InterruptedException exception) {
            throw new RuntimeException(exception);
        }
        return this;
    }

    T then(Supplier<T> supplier) {
        return supplier.get();
    }
}

class Utils {
    private static final int SCALE = 2;

    static char randomChar(String value) {
        int randomInt = new Random().nextInt(value.length());
        return value.charAt(randomInt);
    }

    static BigDecimal decimal(double value) {
        var bigDecimal = new BigDecimal(value);
        bigDecimal = bigDecimal.setScale(SCALE, RoundingMode.HALF_UP);
        return bigDecimal;
    }

    static BigDecimal decimal(BigDecimal value) {
        return decimal(value.doubleValue());
    }
}
//}
```

In the code above, the method `convert` takes the two BigDecimal results from `flightRoutePriceFinder.bestFor` and `exchangeService.rateFor` and calculates the final amount. You can refer to the [javadoc](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html) for a more detailed description of the methods available using `CompletableFuture`.

# Timeout mechanism
However, there are a few problems associated with this code. First, `get()` is a blocking call. This means that the main thread will have to wait until the result is ready before it can progress. Ideally, you'd like the main thread to do other useful work while the result is calculated in the background. Second, the main thread could be blocking indefinitely because there isn't a timeout specified. What if one of the service is overloaded and doesn't respond? To add a timeout mechanism, you can use the other version of `get` inherited from Future which throws a `TimeoutException` when the overall pipeline takes longer than a certain amount of time to return the result:

```java
BigDecimal amount =
    CompletableFuture.supplyAsync(() -> flightRoutePriceFinder.bestFor(AirportCode.LCY, AirportCode.JFK))
        .thenCombine(CompletableFuture.supplyAsync(() -> exchangeService.rateFor(Currency.GBP)),
            Main::convert)
        .get(1, TimeUnit.SECONDS);
System.out.printf("The price is %f %s", amount, Currency.GBP);
```

Unfortunately this code is still blocking and prevents the main thread from doing useful work in the meantime! To tackle this issue, you can refactor the above code to use `thenAccept` and provide a callback which is executed when the result is finally available:

```java
CompletableFuture.supplyAsync(() -> flightRoutePriceFinder.bestFor(AirportCode.LCY, AirportCode.JFK))
    .thenCombine(CompletableFuture.supplyAsync(() -> exchangeService.rateFor(Currency.GBP)),
        Main::convert)
    .thenAccept(amount -> System.out.printf("The price is %f %s", amount, Currency.GBP));
```

However, using this approach we lost the timeout functionality! Ideally we'd like to specify a timeout using a non-blocking method. Unfortunately there isn't a built-in elegant support to solve this problem in Java 8. Solutions available include using `acceptEither` or `applyToEither` together with the `CompletableFuture` you are waiting the result for and another `CompletableFuture` which wraps up a `ScheduledThreadpoolExecutor` that throws a `TimeoutException` after a certain time:

```java runnable
// { autofold
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Random;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;
import java.util.function.Supplier;

public class Main {
    private static FlightRoutePriceFinder flightRoutePriceFinder = new FlightRoutePriceFinder();
    private static ExchangeService exchangeService = new ExchangeService();
    private static ScheduledExecutorService delayer = Executors.newScheduledThreadPool(1);

    public static void main(String[] args) throws InterruptedException {
// }

CompletableFuture.supplyAsync(() -> flightRoutePriceFinder.bestFor(AirportCode.LCY, AirportCode.JFK))
    .thenCombine(CompletableFuture.supplyAsync(() -> exchangeService.rateFor(Currency.GBP)),
        Main::convert)
    .acceptEither(
        // try decreasing timeout to 1 TimeUnit.MILLISECONDS and see what happens
        timeoutAfter(1, TimeUnit.SECONDS),
        amount -> System.out.printf("The price is %s %s", amount, Currency.GBP));

//{ autofold
        delayer.awaitTermination(1, TimeUnit.SECONDS);
        delayer.shutdown();
    }

    private static <T> CompletableFuture<T> timeoutAfter(long timeout, TimeUnit unit) {
        var future = new CompletableFuture<T>();
        delayer.schedule(() -> future.completeExceptionally(new TimeoutException()), timeout, unit);
        return future;
    }

    private static BigDecimal convert(BigDecimal price, BigDecimal rate) {
        return Utils.decimal(price.multiply(rate));
    }
}

class FlightRoutePriceFinder {
    BigDecimal bestFor(AirportCode departure, AirportCode destination) {
        return bestPriceWithDelay(departure, destination);
    }

    private BigDecimal bestPriceWithDelay(AirportCode departure, AirportCode destination) {
        return new Delay<BigDecimal>().random()
            .then(() -> {
                double price = 10 * Utils.randomChar(departure.getName()) + Utils.randomChar(destination.getName());
                return Utils.decimal(price);
            });
    }
}

enum AirportCode {
    LCY("London City Airport"), JFK("John F. Kennedy International Airport");

    private final String name;

    AirportCode(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }
}

enum Currency {
    USD(1.0), GBP(0.769375);

    public final BigDecimal rate;

    Currency(Double rate) {
        this.rate = new BigDecimal(rate.toString());
    }
}

class ExchangeService {
    BigDecimal rateFor(Currency currency) {
        return rateWithDelayFor(currency);
    }

    private BigDecimal rateWithDelayFor(Currency currency) {
        return new Delay<BigDecimal>().random()
            .then(() -> currency.rate);
    }
}

class Delay<T> {
    private static final int MIN_DELAY_IN_MS = 750;
    private static final int MAX_DELAY_IN_MS = 1000;

    Delay<T> random() {
        int delayInMs = ThreadLocalRandom.current().nextInt(MIN_DELAY_IN_MS, MAX_DELAY_IN_MS);
        try {
            Thread.sleep(delayInMs);
        } catch (InterruptedException exception) {
            throw new RuntimeException(exception);
        }
        return this;
    }

    T then(Supplier<T> supplier) {
        return supplier.get();
    }
}

class Utils {
    private static final int SCALE = 2;

    static char randomChar(String value) {
        int randomInt = new Random().nextInt(value.length());
        return value.charAt(randomInt);
    }

    static BigDecimal decimal(double value) {
        var bigDecimal = new BigDecimal(value);
        bigDecimal = bigDecimal.setScale(SCALE, RoundingMode.HALF_UP);
        return bigDecimal;
    }

    static BigDecimal decimal(BigDecimal value) {
        return decimal(value.doubleValue());
    }
}
//}
```

A simple implementation of `timeoutAfter` is as follows where `delayer` is an instance of a `ScheduledThreadPoolExecutor`:

```java
<T> CompletableFuture<T> timeoutAfter(long timeout, TimeUnit unit) {
    var future = new CompletableFuture<T>();
    delayer.schedule(() -> future.completeExceptionally(new TimeoutException()), timeout, unit);
    return future;
}
```

# Java 9 improvements
Java 9's `CompletableFuture` introduces several new methods amongst which are `orTimeout` and `completeOnTimeOut` that provide built-in support for dealing with timeouts.

## orTimeout
The method has the following signature:

```java
public CompletableFuture<T> orTimeout(long timeout, TimeUnit unit)
```

It internally uses a `ScheduledThreadExecutor` and completes the CompletableFuture with a `TimeoutException` after the specified timeout has elapsed. It also returns another CompletableFuture, meaning you can further chain your computation pipeline and deal with the `TimeoutException` by providing a friendly message back. Note that `whenComplete` in the code below could report other exceptional completions that might occur before the timeout occurs.

For example:

```java runnable
// { autofold
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Random;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.TimeUnit;
import java.util.function.Supplier;

public class Main {
    private static FlightRoutePriceFinder flightRoutePriceFinder = new FlightRoutePriceFinder();
    private static ExchangeService exchangeService = new ExchangeService();
    private static ExecutorService executorService = Executors.newFixedThreadPool(1);

    public static void main(String[] args) throws InterruptedException {
// }

CompletableFuture.supplyAsync(() -> flightRoutePriceFinder.bestFor(AirportCode.LCY, AirportCode.JFK), executorService)
    .thenCombine(CompletableFuture.supplyAsync(() -> exchangeService.rateFor(Currency.GBP)),
        Main::convert)
    // try decreasing timeout to 1 TimeUnit.MILLISECONDS and see what happens
    .orTimeout(1, TimeUnit.SECONDS)
    .whenComplete((amount, error) -> {
        if (error == null) {
            System.out.printf("The price is %s %s", amount, Currency.GBP);
        } else {
            System.out.println("Sorry, something unexpected happened: " + error);
        }
    });

//{ autofold
        executorService.awaitTermination(1, TimeUnit.SECONDS);
        executorService.shutdown();
    }

    private static BigDecimal convert(BigDecimal price, BigDecimal rate) {
        return Utils.decimal(price.multiply(rate));
    }
}

class FlightRoutePriceFinder {
    BigDecimal bestFor(AirportCode departure, AirportCode destination) {
        return bestPriceWithDelay(departure, destination);
    }

    private BigDecimal bestPriceWithDelay(AirportCode departure, AirportCode destination) {
        return new Delay<BigDecimal>().random()
            .then(() -> {
                double price = 10 * Utils.randomChar(departure.getName()) + Utils.randomChar(destination.getName());
                return Utils.decimal(price);
            });
    }
}

enum AirportCode {
    LCY("London City Airport"), JFK("John F. Kennedy International Airport");

    private final String name;

    AirportCode(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }
}

enum Currency {
    USD(1.0), GBP(0.769375);

    public final BigDecimal rate;

    Currency(Double rate) {
        this.rate = new BigDecimal(rate.toString());
    }
}

class ExchangeService {
    BigDecimal rateFor(Currency currency) {
        return rateWithDelayFor(currency);
    }

    private BigDecimal rateWithDelayFor(Currency currency) {
        return new Delay<BigDecimal>().random()
            .then(() -> currency.rate);
    }
}

class Delay<T> {
    private static final int MIN_DELAY_IN_MS = 750;
    private static final int MAX_DELAY_IN_MS = 1000;

    Delay<T> random() {
        int delayInMs = ThreadLocalRandom.current().nextInt(MIN_DELAY_IN_MS, MAX_DELAY_IN_MS);
        try {
            Thread.sleep(delayInMs);
        } catch (InterruptedException exception) {
            throw new RuntimeException(exception);
        }
        return this;
    }

    T then(Supplier<T> supplier) {
        return supplier.get();
    }
}

class Utils {
    private static final int SCALE = 2;

    static char randomChar(String value) {
        int randomInt = new Random().nextInt(value.length());
        return value.charAt(randomInt);
    }

    static BigDecimal decimal(double value) {
        var bigDecimal = new BigDecimal(value);
        bigDecimal = bigDecimal.setScale(SCALE, RoundingMode.HALF_UP);
        return bigDecimal;
    }

    static BigDecimal decimal(BigDecimal value) {
        return decimal(value.doubleValue());
    }
}
//}
```

## completeOnTimeout
The method has the following signature:

```java
public CompletableFuture<T> completeOnTimeout(T value, long timeout, TimeUnit unit)
```

It also uses a `ScheduledThreadExecutor` internally but in contrast to `orTimeout`, it provides a default value in the case that the CompletableFuture pipeline times out. You can conceptually relate this method to `orElse()` using `java.util.Optional`.

```java runnable
// { autofold
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Random;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.TimeUnit;
import java.util.function.Supplier;

public class Main {
    private static final BigDecimal DEFAULT_PRICE = Utils.decimal(500);

    private static FlightRoutePriceFinder flightRoutePriceFinder = new FlightRoutePriceFinder();
    private static ExchangeService exchangeService = new ExchangeService();
    private static ExecutorService executorService = Executors.newFixedThreadPool(1);

    public static void main(String[] args) throws InterruptedException {
// }

CompletableFuture.supplyAsync(() -> flightRoutePriceFinder.bestFor(AirportCode.LCY, AirportCode.JFK), executorService)
    .thenCombine(CompletableFuture.supplyAsync(() -> exchangeService.rateFor(Currency.GBP)),
        Main::convert)
    // try decreasing timeout to 1 TimeUnit.MILLISECONDS and see what happens
    .completeOnTimeout(DEFAULT_PRICE, 1, TimeUnit.SECONDS)
    .thenAccept(amount -> {
        System.out.printf("The price is %s %s", amount, Currency.GBP);
    });

//{ autofold
        executorService.awaitTermination(1, TimeUnit.SECONDS);
        executorService.shutdown();
    }

    private static BigDecimal convert(BigDecimal price, BigDecimal rate) {
        return Utils.decimal(price.multiply(rate));
    }
}

class FlightRoutePriceFinder {
    BigDecimal bestFor(AirportCode departure, AirportCode destination) {
        return bestPriceWithDelay(departure, destination);
    }

    private BigDecimal bestPriceWithDelay(AirportCode departure, AirportCode destination) {
        return new Delay<BigDecimal>().random()
            .then(() -> {
                double price = 10 * Utils.randomChar(departure.getName()) + Utils.randomChar(destination.getName());
                return Utils.decimal(price);
            });
    }
}

enum AirportCode {
    LCY("London City Airport"), JFK("John F. Kennedy International Airport");

    private final String name;

    AirportCode(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }
}

enum Currency {
    USD(1.0), GBP(0.769375);

    public final BigDecimal rate;

    Currency(Double rate) {
        this.rate = new BigDecimal(rate.toString());
    }
}

class ExchangeService {
    BigDecimal rateFor(Currency currency) {
        return rateWithDelayFor(currency);
    }

    private BigDecimal rateWithDelayFor(Currency currency) {
        return new Delay<BigDecimal>().random()
            .then(() -> currency.rate);
    }
}

class Delay<T> {
    private static final int MIN_DELAY_IN_MS = 750;
    private static final int MAX_DELAY_IN_MS = 1000;

    Delay<T> random() {
        int delayInMs = ThreadLocalRandom.current().nextInt(MIN_DELAY_IN_MS, MAX_DELAY_IN_MS);
        try {
            Thread.sleep(delayInMs);
        } catch (InterruptedException exception) {
            throw new RuntimeException(exception);
        }
        return this;
    }

    T then(Supplier<T> supplier) {
        return supplier.get();
    }
}

class Utils {
    private static final int SCALE = 2;

    static char randomChar(String value) {
        int randomInt = new Random().nextInt(value.length());
        return value.charAt(randomInt);
    }

    static BigDecimal decimal(double value) {
        var bigDecimal = new BigDecimal(value);
        bigDecimal = bigDecimal.setScale(SCALE, RoundingMode.HALF_UP);
        return bigDecimal;
    }

    static BigDecimal decimal(BigDecimal value) {
        return decimal(value.doubleValue());
    }
}
//}
```

In the code above, if the services are slow to respond, a default price is provided otherwise the result of combining the two services is returned.

If you are curious about the internal implementation details of these methods, you can view the source of `CompletableFuture` in JDK9 [here](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/CompletableFuture.html).

If you interested to learn more about this asynchronous style of programming check out our [intensive Asynchronous and Reactive Java in-house course](http://iteratrlearning.com/javareactive). For everything Java 8 related, we also teach a popular [two-day Modern Development using Java 8 course](http://iteratrlearning.com/java8course).

# Notes
This playground is based on IteratrLearning's article [Asynchronous timeouts with CompletableFutures in Java 8 and Java 9](http://iteratrlearning.com/java9/2016/09/13/java9-timeouts-completablefutures.html)
