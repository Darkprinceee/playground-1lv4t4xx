So we've already talked a little bit about the improvements in Java 9 to two features that were introduced in Java 8: Streams and Collectors. But what about one of our other Java 8 friends: `Optional`. At Iteratrlearning we're real fans of the `Optional` type. We've found that, if used well, it can make your code more explicit and reduce the scope for bugs.

Of course nothing is perfect. `Optional` in Java 8 was missing a few features that have been improved upon in Java 9 and that's what we'll be talking about here: `stream()`, `ifPresentOrElse()` and `or()`.

# stream
If you've been using Java 8's Stream API in conjunction with the Optional class then you might have hit a situation where you want to replace a stream of Optionals with values that are present. For example, let's suppose you've got a set of settings that may be set by a user. You've implemented a lookupSettingByName() method that returns an Optional if the configuration setting has been set by the user.

```java
List<Setting> settings = SETTING_NAMES
    .stream()
    .map(Main::lookupSetting)
    .filter(Optional::isPresent)
    .map(Optional::get)
    .collect(toList());
```

In this example we're combining two optional methods to achieve our goal. We've used a `filter` on the `isPresent()` method in order to remove empty Optionals. We then unboxed the optionals that we knew had a value in with the `get()` method call.

This is a functional solution, but we can streamline things in Java 9. Here a method on Optional has been added that returns a `Stream` called, funnily enough, `stream()`. That will give us a stream with an element in if the Optional has one, or empty otherwise. Let's see how our code would look with this approach:

```java runnable
// { autofold
import java.util.List;
import java.util.Optional;

import static java.util.stream.Collectors.toList;

public class Main {
    private static final List<String> SETTING_NAMES = List.of(
        "file.separator", "java.vendor", "java.version",
        "omnisharp.NewLinesForBracesInTypes", "omnisharp.NewLinesForBracesInMethods");

    public static void main(String[] args) {
        System.setProperty("omnisharp.NewLinesForBracesInTypes", Boolean.FALSE.toString());
// }

List<Setting> settings = SETTING_NAMES
    .stream()
    .map(Main::lookupSetting)
    // try replacing with filter and map from above, then removing filter line and see what happens
    .flatMap(Optional::stream)
    .collect(toList());
System.out.println(settings);

//{ autofold
    }

    private static Optional<Setting> lookupSetting(String name) {
        String value = System.getProperty(name);
        return value != null ? Optional.of(new Setting(name, value)) : Optional.empty();
    }
}

class Setting {
    private final String name;
    private final String value;

    Setting(String name, String value) {
        this.name = name;
        this.value = value;
    }

    @Override
    public String toString() {
        return String.format("Setting(%s: %s)", name, value);
    }
}
//}
```

This new addition also means that it is simpler to integrate Optional with APIs expecting to work with Streams.

# ifPresentOrElse

The new `ifPresentOrElse` method encodes a common pattern where you want to perform an action if an Optional value is present, or a different action if it's absent. In order to understand it better we'll take an example of someone trying to checkin to an airline flight and see how you would write this code both using null checks and also the `Optional` type.

Our user provides us with a booking reference, which we lookup their booking with. If we have a booking associated with that reference then we can display the check-in page to them, otherwise we'll just display a page explaining that their booking record is missing.

If our `lookupBooking` method were to return null in order to indicate that the booking is missing then our code might look like the following:

```java
Booking booking = lookupBooking(bookingReference);
if (booking != null) {
    displayCheckIn(booking);
} else {
    displayMissingBookingPage();
}
```

Now in Java 8 we could have refactored `lookupBooking` to return an Optional value, which gave us an explicit indication that a booking may not be looked up. It also helps the user to think about the distinction between the booking being present and absent, rather than simply hoping that they have a null check. The simplest refactor for this code would have been to the following:

```java
Optional<Booking> booking = lookupBooking(bookingReference);
if (booking.isPresent()) {
    displayCheckIn(booking.get());
} else {
    displayMissingBookingPage();
}
```

Now this pattern of taking an `Optional` and calling `isPresent()` and `get()` isn't a particularly idiomatic use. In fact it basically leaves us with similar code to the null checked version. Ideally we want to be able to call a method on the `Optional` that is appropriate for our use case. Effectively moving to a tell-don't-ask style of coding. `Optional` from Java 8 has an `ifPresent` method that will invoke its callback if the value inside the `Optional` is present, for example:

```java
lookupBooking(bookingReference)
    .ifPresent(Main::displayCheckIn);
```

Unfortunately it doesn't meet our needs here because it won't handle the case where the value is absent and we want to display our missing booking page. This is the use case that Java 9's Optional addresses. We could refactor our original code as follows:

```java runnable
// { autofold
import java.time.LocalDate;
import java.time.Month;
import java.util.List;
import java.util.Optional;

public class Main {
    private static List<Booking> bookings = List.of(
        new Booking("CrownTowers2019-06-10", "Crown Towers", LocalDate.of(2019, Month.JUNE, 10)),
        new Booking("FlamingoMotel2019-07-20", "Flamingo Motel", LocalDate.of(2019, Month.JULY, 20)),
        new Booking("BallsbridgeHotel2019-08-30", "Ballsbridge Hotel", LocalDate.of(2019, Month.AUGUST, 30)));

    public static void main(String[] args) {
// }

String bookingReference = "FlamingoMotel2019-07-20";
lookupBooking(bookingReference)
    .ifPresentOrElse(
        Main::displayCheckIn,
        Main::displayMissingBookingPage);

//{ autofold
    }

    private static Optional<Booking> lookupBooking(String reference) {
        return bookings.stream().filter(it -> reference.equals(it.getReference())).findAny();
    }

    private static void displayCheckIn(Booking booking) {
        System.out.println(String.format("Check-in at %s: %s", booking.getHotel(), booking.getCheckIn()));
    }

    private static void displayMissingBookingPage() {
        System.out.println("No information for requested booking reference");
    }
}

class Booking {
    private final String reference;
    private final String hotel;
    private final LocalDate checkIn;

    Booking(String reference, String hotel, LocalDate checkIn) {
        this.reference = reference;
        this.hotel = hotel;
        this.checkIn = checkIn;
    }

    String getReference() {
        return reference;
    }

    String getHotel() {
        return hotel;
    }

    LocalDate getCheckIn() {
        return checkIn;
    }
}
//}
```

# or
Another method that has been added to `Optional` in Java 9 is the succinctly named `or()` method. This method takes a function that creates an Optional as an argument. If the object that it gets invoked upon has a value present then it is returned, otherwise the function is invoked and its result returned.

This is particularly useful when you have a couple of methods that all return optionals and you want to return the first one that is present. Let's suppose that we want to lookup information about client's using a company identifier, such as their company number. Firstly we want to check our existing client datastore and see if the company is in there. If it isn't we want to create a new client by looking up the information about the company from Companies House. Now it might be the case that the provided id is a typo from a user and couldn't be looked up at all. If we suppose that our methods just return null in order to indicate that the value is missing then we might write the following code:

```java
Client client = findClient(companyId);
if (client == null) {
    client = lookupCompanyDetails(companyId);
}
// client could still be null
```

Now if someone refactors `findClient()` to return an Optional, we can use the orElseGet() method from Java 8 which will only call our `lookupCompanyDetails()` method if the Optional is absent. For example:

```java
Client client = 
    findClient(companyId)
        .orElseGet(() -> lookupCompanyDetails(companyId));
```

Unfortunately this still doesn't model our use case correctly. Since the companyId may not correspond to an actual company identifier the `lookupCompanyDetails()` method can still fail. If we're going down the route of modelling failure using the `Optional` type then we should also make that method return an `Optional`.

This is where the new `or()` method comes into play. We get an `Optional<Client>` back. If we could lookup the client in our database then that would be the value present, if it was a valid new company then that would be returned, if it was a typo then the `Optional` box would be empty. Et voila:

```java runnable
// { autofold
import java.util.List;
import java.util.Optional;

public class Main {
    private static List<Client> internalClients = List.of(
        new Client(124, "Walmart"),
        new Client(125, "Shell"));
    private static List<Client> externalClients = List.of(
        new Client(136, "Volkswagen"),
        new Client(137, "Toyota"),
        new Client(138, "Verizon"));

    public static void main(String[] args) {
// }

long companyId = 136;
Optional<Client> client =
    findClient(companyId)
        .or(() -> lookupCompanyDetails(companyId));
client.ifPresent(System.out::println);

//{ autofold
    }

    private static Optional<Client> findClient(long id) {
        return internalClients.stream().filter(it -> id == it.getId()).findAny();
    }

    private static Optional<Client> lookupCompanyDetails(long id) {
        return externalClients.stream().filter(it -> id == it.getId()).findAny();
    }
}

class Client {
    private final long id;
    private final String name;

    Client(long id, String name) {
        this.id = id;
        this.name = name;
    }

    long getId() {
        return id;
    }

    String getName() {
        return name;
    }

    @Override
    public String toString() {
        return String.format("Client(id: %d, name: %s)", id, name);
    }
}
//}
```

# get
There is an outstanding proposal to deprecate `Optional.get()` and rename it to something else. The full details of this proposal can be read [here](http://mail.openjdk.java.net/pipermail/core-libs-dev/2016-April/040484.html). Despite this proposal having merit it is still not in Java (latest version: 12) as of today.

# Conclusions
We've talked about Java 8's `Optional` class is being improved in Java 9 with a series of targeted method that each fill a missing use case. Unfortunately, the primitive specialised `Optional` classes, such as `OptionalInt` aren't getting all the same love - they have had `stream()` and `ifPresentOrElse()` added, but not `or()`. They have always been missing some of the methods from `Optional` and are increasingly looking like poor cousins.

If you are interested in an intensive Java course for your team, check out our [Java Software Development Bootcamp](http://iteratrlearning.com/javabootcamp) or [Modern Development with Java](http://iteratrlearning.com/java8course).

# Notes
This playground is based on IteratrLearning's article [Optional Improvements in Java 9](http://iteratrlearning.com/java9/2016/09/05/java9-optional.html)
