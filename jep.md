Summary
-------

Add default methods to the Comparable interface to make it easier to use
Comparables in boolean expressions.

Goals
-----

Introduce more conveinent, and memorable mechanisms than
`foo.compareTo(bar) $op 0` (where `$op` means `>`,`<`,`>=`,`<=`,`==`)
for comparing the order of two instances of `Comprarble<T>`.


Success Metrics
---------------

While it may not be clear how to measure some of these metrics, the
right success metrics are:

* Fewer mistakes made when developers write conditional expressions.
* High adoption of the newly introduced default methods.
* Easier comprehension of code (a.k.a readability.)

Motivation
----------

The contract of `compareTo(T o)` can be difficult for new Java developers
to internalize. A common mistake when implementing the method is to
reverse the meaning of a negative and positive return value. For example:

```java
public int compareTo(Earthquake other) {
    return other.getMagnitude() - this.getMagnitude();
}
```

Bugs of this nature are most frequently identified when writing and
running unit tests and realizing that the sort order is backwards for
this type.

Similarly, Java developers frequently have to stop and think when using
a `Comparable` in boolean conditions what it means for some instance
`a` of the class compared to some other instance `b` is `> 0`. Novice
programmers spend a good amount of time looking at constructs like.

```java
if (earthquakeA.compareTo(recordEarthquake) > 0) {
    ...
}
```

And thinking, "Is that true when the `recordEarthquake` is bigger or
smaller than `earthquakeA`?

It is very difficult to *prove* that this confusion exists, but many
engineers (in informal discussions on the topic) have admitted that
they have tripped up on these problems.

Description
-----------

The proposal is to introduce 5 new default methods on the `Comparable`
interface which would aid in correctly writing and reading boolean
expressions with types which implement `Comparable`.

```java
public interface Comparable<T> {

    default boolean isAtLeast(T o) {
        return this.compareTo(o) >= 0;
    }

    default boolean isAtMost(T o) {
        return this.compareTo(o) <= 0;
    }

    default boolean isEquivalentTo(T o) {
        return this.compareTo(o) == 0;
    }

    default boolean isGreaterThan(T o) {
        return this.compareTo(o) > 0;
    }

    default boolean isLessThan(T o) {
        return this.compareTo(o) < 0;
    }

}
```

Alternatives
------------

### Different Names

Several different proposals for the names of these methods have been considered:

#### isGreaterThanOrEqualTo and isLessThanOrEqualTo

Instead of `isAtLeast` and `isAtMost`, `isGreaterThanOrEqualTo` and
`isLessThanOrEqualTo` are *strongly* perferred by many engineers
because those method names match how they read the corresponding
operators `>=` and `<=`.

The main arguments against these methods are:

* They are verbose.
* They are not consistent with the popular
  [Truth](https://github.com/google/truth) library.
  
Given that these methods are so *strongly* preferred by a very large
minority of engineers, we may want to consider adding them _in
addition_ to the `isAtLeast` and `isAtMost` methods.

#### isEquvalentAccordingToCompareTo

As an alternative to `isEquivalentTo`,
`isEquivalentAccordingToCompareTo` is verbose, but leaves no doubt in
the readers mind as to whether or not the implementation of equals()
on the class is consulted directly in the implementation.

The decision was to make it clear in the javadoc that `isEquivalentTo`
relies only on the implementation of `compareTo`.

#### Other alternative names

Some other names considered instead of `isEquivalentTo` were:

* `comparesEqualTo` - Implies it might use `Object.equals`
* `isEqualTo` - Implies it might use `Object.equals`
* `isOrderedEquivalentTo` - Verbose
* `isSameAs` - Implies it might use `==` (reference equality)
* `isSameValueAs` - Implies it might use `==` (reference equality)

And instead of the other methods, methods of the form:

* `comparesGreaterThan` - Not quite as easy to read.
* `greaterThan` - A little less fluent.
* `gt` - Too brief.

### Access methods through a wrapper around Comparable

This is easiest to explain with an example implementation.

```java
public final class Comparables {
   /**
   * Returns a {@link WrappedComparable} which can be used for comparisons.
   *
   * @param value the comparable to use for comparisons
   */
  public static <T extends Comparable<? super T>> WrappedComparable<T> is(T value) {
    return new WrappedComparable<T>(value);
  }

  /** Wraps a {@link Comparable} instance for use in comparisons. */
  public static final class WrappedComparable<T extends Comparable<? super T>> {
    private final T value;

    private WrappedComparable(T value) {
      this.value = checkNotNull(value);
    }

    /** Tests whether the {@link Comparable} is equivalent to toCompare. */
    public boolean equivalentTo(T toCompare) {
      return value.compareTo(toCompare) == 0;
    }

    /** Tests whether the {@link Comparable} is greater than toCompare. */
    public boolean greaterThan(T toCompare) {
      return value.compareTo(toCompare) > 0;
    }

    /** Tests whether the {@link Comparable} is less than toCompare. */
    public boolean lessThan(T toCompare) {
      return value.compareTo(toCompare) < 0;
    }

    /** Tests whether the {@link Comparable} is less than or equal to toCompare. */
    public boolean atMost(T toCompare) {
      return value.compareTo(toCompare) <= 0;
    }

    /** Tests whether the {@link Comparable} is greater than or equal to toCompare. */
    public boolean atLeast(T toCompare) {
      return value.compareTo(toCompare) >= 0;
    }
  }
}
```

The `is` method could then be staticaly imported and used in conditions like this:

```java
if (is(earthquakeA).greaterThan(recordEarthquake)) {
    ...
}
```

While this approach accomplishes the goal of making the boolean
condition more readible, it is not nearly as discoverable as having
the methods exist directly on the `Comparable` interface and requires
an additional dependency for any project wishing to create readable
boolean expresions.

Testing
-------

Unit tests should be sufficient for a simple change like this one.

Risks and Assumptions
---------------------

There are a few risks and assumptions worth mentioning:

### Existing methods

There may be classes which implement `Comparable<T>` and already have
methods with these 5 names but with ambiguous signatures.

It seems like class authors would have to change their code before
switching to whichever JDK this change is realeased in.

Maybe, if we change the design to a subclass of the `Comparable<T>`
interface, then class authors could choose to opt-in their types to
these new methods. But, the obvious drawback of such a design is that
developers would always have to check that the `Comparable` they were
dealing with was actually aslo a `FluentComparable` (or whatever name
we gave the subclass.)

### Operator overloading

If the JDK were enhanced to allow operator overloading, this proposal
*should* become obsolete. For example:

```java
if (earthquakeA > recordEarthquake) {
    ...
}
```

would be the very easiest syntax to understand.

One thing which would be espeically challenging in the case of
Comparables is to figure out what operator to use instead of this
proposal's `isEquivalentTo(T o)` method.
