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
import java.util.Arrays;
import java.util.List;

public class Main {
    public static void main(String[] args) {
// }

var list = List.of("Hello", "World", "from", "Java");
System.out.println(list);
var set = Set.of("Hello", "World", "from", "Java", "World");
System.out.println(set);

//{ autofold
    }
}
//}
```

# Notes
This playground is based on IteratrLearning's article [Collection Factory Methods in Java 9](http://iteratrlearning.com/java9/2016/11/09/java9-collection-factory-methods.html)

Similar playground by Gowgi: [Java 9 Collection Improvements](https://tech.io/playgrounds/3384/java-9-collection-improvements)
