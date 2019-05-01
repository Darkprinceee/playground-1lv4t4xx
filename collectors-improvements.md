Collectors is an addition to Java 8 which let you specify data processing queries by aggregating the elements of a `Stream` into various containers such as `Map`, `List` and `Set`. For example, you can create a Map of the sum of expenses for each year as follows using the `groupingBy` and `summingLong` collector from the `Collectors` class. In the rest of this article we will assume static imports when referring to static method from the `Collectors` class.

```java
Map<Integer, Long> expensesByYear = expenses
    .stream()
    .collect(groupingBy(Expense::getYear,  
        summingLong(Expense::getAmount)
    ));
```

So what's new in Java 9? In Java 9, two new collectors are added to the Collectors utility class:
- `Collectors.filtering`
- `Collectors.flatMapping`

For the rest of article we will assume the following `Expense` and `Tag` class definitions:

```java
class Expense {
	private final long amount;
	private final int year;
	private final List<Tag> tags;

	Expense(long amount, int year, List<Tag> tags) {
		this.amount = amount;
		this.year = year;
		this.tags = tags;
	}

	long getAmount() {
		return amount;
	}

	int getYear() {
		return year;
	}

	List<Tag> getTags() {
		return tags;
	}
}

enum Tag {
	FOOD, ENTERTAINMENT, TRAVEL, UTILITY
}
```

# filtering
Let's revisit the example above and say you now need to build a `Map` of the list of expenses for each year but only for expenses that are higher than _GBP1000_.

You already know how to generate a Map of the list of expenses for each year as follows:

```java
Map<Integer, List<Expense>> listOfExpensesByYear = expenses
    .stream()
    .collect(groupingBy(Expense::getYear));
```

So you could add a filter to the streams as follows:

```java
Map<Integer, List<Expense>> listOfExpensesByYear = expenses
    .stream()
    .filter(expense -> expense.getAmount() > 1_000)
    .collect(groupingBy(Expense::getYear));
```

Unfortunately, this means that if all expenses' amount for a certain year are below £1000, the resulting Map would not contain an entry for that year (i.e. no key and no value).

Instead you can use the `filtering` collector as follows which would preserve the year in the resulting Map and produce an empty list:

```java runnable
// { autofold
import java.util.List;
import java.util.Map;

import static java.util.stream.Collectors.filtering;
import static java.util.stream.Collectors.groupingBy;
import static java.util.stream.Collectors.summingLong;
import static java.util.stream.Collectors.toList;

public class Main {
	public static void main(String[] args) {
		List<Expense> expenses = List.of(
			new Expense(500, 2016, List.of(Tag.FOOD, Tag.ENTERTAINMENT)),
			new Expense(1_500, 2016, List.of(Tag.UTILITY)),
			new Expense(700, 2015, List.of(Tag.TRAVEL, Tag.ENTERTAINMENT)));
// }

Map<Integer, List<Expense>> listOfExpensesByYear = expenses
    .stream()
    .collect(groupingBy(Expense::getYear,
        filtering(expense -> expense.getAmount() > 1_000, toList())
    ));
System.out.println(listOfExpensesByYear);

//{ autofold
	}
}

class Expense {
	private final long amount;
	private final int year;
	private final List<Tag> tags;

	Expense(long amount, int year, List<Tag> tags) {
		this.amount = amount;
		this.year = year;
		this.tags = tags;
	}

	long getAmount() {
		return amount;
	}

	int getYear() {
		return year;
	}

	List<Tag> getTags() {
		return tags;
	}

	@Override
	public String toString() {
		return String.format("Expense(amount: %d, year: %d, tags: %s)", amount, year, tags);
	}
}

enum Tag {
	FOOD, ENTERTAINMENT, TRAVEL, UTILITY
}
//}
```

# flatMapping
The `flatMapping` collector is the bigger brother of the `mapping` collector. Let's say you need to produce a map of year with a set of tags from the expenses for each year. In other words, you need to produce `Map<Integer, Set<Tag>>`.

A first attempt may look as follows:

```java
expenses
    .stream()
    .collect(groupingBy(Expense::getYear,
        mapping(Expense::getTags, toSet())
    ));
```

Unfortunately this query returns a `Map<Integer, Set<List<Tag>>>`

By using `flatMapping`, you can flatten the intermediate Lists into a single container. The `flatMapping` collector takes two arguments:
1. A function from one element to a Stream of elements
2. A downstream collector to collect the single flattened stream into a container

With this knowledge you can solve the query as follows:

```java runnable
// { autofold
import java.util.List;
import java.util.Map;
import java.util.Set;

import static java.util.stream.Collectors.flatMapping;
import static java.util.stream.Collectors.groupingBy;
import static java.util.stream.Collectors.mapping;
import static java.util.stream.Collectors.toSet;

public class Main {
	public static void main(String[] args) {
		List<Expense> expenses = List.of(
			new Expense(500, 2016, List.of(Tag.FOOD, Tag.ENTERTAINMENT)),
			new Expense(1_500, 2016, List.of(Tag.UTILITY)),
			new Expense(700, 2015, List.of(Tag.TRAVEL, Tag.ENTERTAINMENT)));
// }

Map<Integer, Set<Tag>> tagsByYear = expenses
    .stream()
    .collect(groupingBy(Expense::getYear,
        flatMapping(expense -> expense.getTags().stream(), toSet())
    ));
System.out.println(tagsByYear);

//{ autofold
	}
}

class Expense {
	private final long amount;
	private final int year;
	private final List<Tag> tags;

	Expense(long amount, int year, List<Tag> tags) {
		this.amount = amount;
		this.year = year;
		this.tags = tags;
	}

	long getAmount() {
		return amount;
	}

	int getYear() {
		return year;
	}

	List<Tag> getTags() {
		return tags;
	}

	@Override
	public String toString() {
		return String.format("Expense(amount: %d, year: %d, tags: %s)", amount, year, tags);
	}
}

enum Tag {
	FOOD, ENTERTAINMENT, TRAVEL, UTILITY
}
//}
```

Note that the `flatMapping` collector is related to the `flatMap` method from the Stream API. The `flatMap` method takes a function producing a Stream of zero or more elements for each element in the input Stream. The result is then flattened in a single stream.

# More collectors
If you are interested about more collectors, take a look at the [Eclipse Collections](https://www.eclipse.org/collections/) ([Eclipse Collections Cheat Sheet](https://github.com/eclipse/eclipse-collections/blob/master/docs/guide.md#-eclipse-collections-cheat-sheet)) which provides additional collectors such as `zip`, `zipWithIndex` and `chunk`. You can also always write your own custom collector by implementing the `Collector` interface.

If you are interested in an intensive Java course for your team, check out our [Java Software Development Bootcamp](http://iteratrlearning.com/javabootcamp) or [Modern Development with Java](http://iteratrlearning.com/java8course).

# Notes
This playground is based on IteratrLearning's article [Collectors Improvements in Java 9](http://iteratrlearning.com/java9/2016/08/16/java9-collectors.html)
