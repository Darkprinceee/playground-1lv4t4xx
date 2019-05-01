At IteratrLearning we love teaching Java courses, and find that a lot of the attendees enjoy coding in Java. One of the things that can irritate about Java, however, is that it's often overly verbose or ceremonious when it comes to doing the most basic of things. One of the ways that Java 9 helps solve this problem is by providing ways of instantatiating collections from specific values in a very abbreviated way.

# Motivation
Let's start by looking at the problem that this is trying to solve, by instantiating a list with a few String values:

```java
List<String> values = new ArrayList<>();
values.add("Hello");
values.add("World");
values.add("from");
values.add("Java");
```

Let's face it, this is pretty bulky for such a simple and common thing to do. We admit that this isn't the only way to to instantiate a List though. `Arrays.asList()` has been around since before Java 5, and it originally took just an array. In Java 5, it was converted to accept varargs and is in common use.

```java
List<String> values = Arrays.asList("Hello", "World", "from", "Java");
```

Let's be honest - this is a pretty significant improvement. The `List` returned by `Arrays.asList()` is a little strange however. If you try to add an element to the List then it'll throw an `UnsupportedOperationException`. That's ok, you might think - it's a `List` that cannot be mutated! Not so fast. It's actually a list that wraps an array. So the `set()` operation will modify the backing array, and in fact if you hold onto the array that is wrapped then it can be modified. The following example prints “[World]”:

```java runnable
// { autofold
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

public class Main {
    public static void main(String[] args) {
// }

String[] hello = { "Hello" };
List<String> values = Arrays.asList(hello);
hello[0] = "World";
System.out.println(values);

//{ autofold
    }
}
//}
```

If you want to create a `Set` or a `Queue` then you are out of luck as well - there's no `Arrays.asSet()`. The normal way of solving this problem is to use the Collection constructor overload:

```java
Set<String> values = new HashSet<>(Arrays.asList("Hello", "World", "from", "Java"));
```

Again, fairly verbose.

# Collection Literals
Some programming languages offer a feature to solve this problem by adding Collection Literals to the programming language. This gives you some syntax that instantiates a collection from some specific values. Here's an example in Groovy:

```groovy
def values = ["Hello", "World", "from", "Java"] as Set
```

Now adding new language features to the Java language is an expensive process. It takes a lot of engineering time and each language feature has huge opportunity cost. That is to say that you could spend that engineering time improving other language features and quite possibly get a bigger win!

# Collection Factories
An alternative is to provide some methods that construct collections from values and using a var-args constructor to make them syntactically shorter like collection literals. This is the choice that was made in Java 9, so we can do the following:

```java runnable
// { autofold
import java.util.List;
import java.util.Set;

public class Main {
    public static void main(String[] args) {
// }

var list = List.of("List", "of", "elements");
System.out.println(list);
var set = Set.of("Set", "of", "elements");
System.out.println(set);

//{ autofold
    }
}
//}
```

The way that var-args APIs are implemented in Java is basically syntactic sugar around passing an array as the final parameter to the method, which would mean that these methods have to bear the overhead of instantiating an array. To avoid the performance cost of this operation Java 9 has 10 overloads with a fixed number of elements. When you call the methods you won't even notice the difference without looking at the overloads in your IDE. The downside of this approach is that more code needs to be maintained by JVM engineers and also the API itself is cluttered. If Java had a more efficient way of allocating var-args parameters that would be nice. A man can dream, a man can dream...

# Maps
Maps have also had factory methods added. They work a little differently as Maps have keys and values rather than a single type of element. For up to 10 entries Maps have overloaded constructors that take pairs of keys and values. For example we could build a map of various cities and their populations (according to google in October 2016) as follow:

```java
var cities = Map.of("Brussels", 1_139_000, "Cardiff", 341_000);
```

The var-args case for Map is a little bit harder, you need to have both keys and values, but in Java, methods can't have two var-args parameters. So the general case is handled by taking a var-args method of `Map.Entry<K, V>` objects and adding a static `entry()` method that constructs them. For example:

```java runnable
// { autofold
import java.util.Map;

import static java.util.Map.entry;

public class Main {
    public static void main(String[] args) {
// }

var cities = Map.ofEntries(entry("Brussels", 1_139_000), entry("Cardiff", 341_000));
System.out.println(cities);

//{ autofold
    }
}
//}
```

# Safety
The goal here isn't just to reduce verbosity though, it is to also reduce the scope for programmer errors. All the collections added in recent years have banned the use of nulls as elements within the collections and these collections follow suit. This helps reduce the scope for bugs around referring to null values in collections and also simplifies the internal implementation.

A bigger difference when compared to most collections in the JDK is that these collections are Immutable. Immutability reduces the scope for bugs by removing the ability for one part of an application to cause bugs by modifying state that another component is relying on. In the case of these factory methods no new interfaces were introduced as part of this process. So they will expose an `add()` method that will throw an `UnsupportedOperationException()`. This is not ideal, but adding a separate hierarchy of Immutable collection interfaces would have massively bloated the scope of this small API change.

A similar, but distinct feature that Java already has are the collection views returned by Collections.unmodifiableList(), unmodifiableSet(), and unmodifiableMap(). They have similar behaviour to the immutable collections in that they throw an `UnsupportedOperationException` when you try to modify them directly. They still aren't the same though, since the unmodifiable views wrap a reference to an existing collection. If that existing collection is modified then that change is visible to readers of the unmodifiable view. However, immutable collections can never be changed at all.

Another safety concept used here is the compile time type-safety of `Map.ofEntries()`. An alternative implementation, for instance, was to have the `ofEntries()` factory method take an `Object...` varargs parameter. This would have been intended to take alternative keys and values. This implementation raises the possibility of new kinds of errors, such as using the wrong type for a key or a value, or having an odd number of elements. These can only be caught at runtime. The `Map.ofEntries(Entry...)` approach is compile-time type-safe, at the cost of boxing each key and value into an `Entry` object.

The final safety feature is the randomized iteration order of the immutable `Set` elements and `Map` keys. `HashSet` and `HashMap` iteration order has always been unspecified, but fairly stable, leading to code having inadvertent dependencies on that order. This causes things to break when the iteration order changes, which occasionally happens. The new Set/Map collections change their iteration order from run to run, hopefully flushing out order dependencies earlier in test or development.

# Conclusions
Collection literals are an appealing language feature, adding some easier syntax to perform a common operation. However, the cost-benefit tradeoff is much better for Collection factory methods that are simply a library change. They give us most of the benefits without any language changes. The addition of these factory methods in Java 9 provides Immutable collections that ban the use of null values as elements.

If you enjoy watching conference talks there's also a good video by Stuart Marks from the Java Core Libraries team on [youtube](https://www.youtube.com/watch?v=LgR9ByD1dEw).

If you are interested in an intensive Java course for your team, check out our [Java Software Development Bootcamp](http://iteratrlearning.com/javabootcamp) or [Modern Development with Java](http://iteratrlearning.com/java8course).

# Notes
This playground is based on IteratrLearning's article [Collection Factory Methods in Java 9](http://iteratrlearning.com/java9/2016/11/09/java9-collection-factory-methods.html)

Similar playground by Gowgi: [Java 9 Collection Improvements](https://tech.io/playgrounds/3384/java-9-collection-improvements)
