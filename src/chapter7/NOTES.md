#Chapter 7. Lambdas and Streams

In Java 8, functional interfaces, lambdas, and method references were added to make it easier to create function objects. 
The streams API was added in tandem with these language changes to provide library support for processing sequences of data 
elements. In this chapter, we discuss how to make best use of these facilities.

## ITEM 42: PREFER LAMBDAS TO ANONYMOUS CLASSES

Historically, interfaces (or, rarely, abstract classes) with a single abstract method were used as function types. Their 
instances, known as function objects, represent functions or actions. Since JDK 1.1 was released in 1997, the primary 
means of creating a function object was the anonymous class (Item 24). Here’s a code snippet to sort a list of strings 
in order of length, using an anonymous class to create the sort’s comparison function (which imposes the sort order):

```aidl
// Anonymous class instance as a function object - obsolete!
Collections.sort(words, new Comparator<String>() {
    public int compare(String s1, String s2) {
        return Integer.compare(s1.length(), s2.length());
    }
});
```

Anonymous classes were adequate for the classic objected-oriented design patterns requiring function objects, notably 
the Strategy pattern [Gamma95]. The Comparator interface represents an abstract strategy for sorting; the anonymous class 
above is a concrete strategy for sorting strings. The verbosity of anonymous classes, however, made functional programming 
in Java an unappealing prospect.

In Java 8, the language formalized the notion that interfaces with a single abstract method are special and deserve special 
treatment. These interfaces are now known as functional interfaces, and the language allows you to create instances of these 
interfaces using lambda expressions, or lambdas for short. Lambdas are similar in function to anonymous classes, but far 
more concise. Here’s how the code snippet above looks with the anonymous class replaced by a lambda. The boilerplate is 
gone, and the behavior is clearly evident:

```aidl
// Lambda expression as function object (replaces anonymous class)
Collections.sort(words, s1, s2) -> Integer.compare(s1.length(), s2.length()));
```

Note that the types of the lambda (Comparator<String>), of its parameters (s1 and s2, both String), and of its return 
value (int) are not present in the code. The compiler deduces these types from context, using a process known as type 
inference. In some cases, the compiler won’t be able to determine the types, and you’ll have to specify them. The rules 
for type inference are complex: they take up an entire chapter in the JLS [JLS, 18]. Few programmers understand these 
rules in detail, but that’s OK. <b>Omit the types of all lambda parameters unless their presence makes your program clearer. </b>
{Aaron notes: Above is an important design.}
If the compiler generates an error telling you it can’t infer the type of a lambda parameter, then specify it. Sometimes 
you may have to cast the return value or the entire lambda expression, but this is rare.

One caveat should be added concerning type inference. Item 26 tells you not to use raw types, Item 29 tells you to favor 
generic types, and Item 30 tells you to favor generic methods. This advice is doubly important when you’re using lambdas, 
because the compiler obtains most of the type information that allows it to perform type inference from generics. If you 
don’t provide this information, the compiler will be unable to do type inference, and you’ll have to specify types manually 
in your lambdas, which will greatly increase their verbosity. By way of example, the code snippet above won’t compile if 
the variable words is declared to be of the raw type List instead of the parameterized type List<String>.

Incidentally, the comparator in the snippet can be made even more succinct if a comparator construction method is used in 
place of a lambda (Items 14. 43):

```aidl
Collections.sort(words, comparingInt(String::length));
```
{Aaron notes: Above is an important design.}


In fact, the snippet can be made still shorter by taking advantage of the sort method that was added to the List interface in Java 8:

```aidl
words.sort(comparingInt(String::length));
```
{Aaron notes: Above is an important design.}
The addition of lambdas to the language makes it practical to use function objects where it would not previously have 
made sense. For example, consider the Operation enum type in Item 34. Because each enum required different behavior for 
its apply method, we used constant-specific class bodies and overrode the apply method in each enum constant. To refresh 
your memory, here is the code:

```aidl
// Enum type with constant-specific class bodies & data (Item 34)
public enum Operation {

    PLUS("+") {
        public double apply(double x, double y) { return x + y; }
    },

    MINUS("-") {
        public double apply(double x, double y) { return x - y; }
    },

    TIMES("*") {
        public double apply(double x, double y) { return x * y; }
    },

    DIVIDE("/") {
        public double apply(double x, double y) { return x / y; }
    };

    private final String symbol;

    Operation(String symbol) { this.symbol = symbol; }
    @Override public String toString() { return symbol; }
    public abstract double apply(double x, double y);
}
```
{Aaron notes: Above is an important design.}

Item 34 says that enum instance fields are preferable to constant-specific class bodies. Lambdas make it easy to implement 
constant-specific behavior using the former instead of the latter. Merely pass a lambda implementing each enum constant’s 
behavior to its constructor. The constructor stores the lambda in an instance field, and the apply method forwards 
invocations to the lambda. The resulting code is simpler and clearer than the original version:

```aidl
// Enum with function object fields & constant-specific behavior

public enum Operation {

    PLUS  ("+", (x, y) -> x + y),

    MINUS ("-", (x, y) -> x - y),

    TIMES ("*", (x, y) -> x * y),

    DIVIDE("/", (x, y) -> x / y);

    private final String symbol;
    private final DoubleBinaryOperator op;

    Operation(String symbol, DoubleBinaryOperator op) {

        this.symbol = symbol;

        this.op = op;

    }

    @Override public String toString() { return symbol; }

    public double apply(double x, double y) {
        return op.applyAsDouble(x, y);
    }
}
```

Note that we’re using the DoubleBinaryOperator interface for the lambdas that represent the enum constant’s behavior. This 
is one of the many predefined functional interfaces in java.util.function (Item 44). It represents a function that takes two 
double arguments and returns a double result.

Looking at the lambda-based Operation enum, you might think constant-specific method bodies have outlived their usefulness, 
but this is not the case. Unlike methods and classes, <b>lambdas lack names and documentation; if a computation isn’t 
self-explanatory, or exceeds a few lines, don’t put it in a lambda.</b>
{Aaron notes: Above is an important design.}
One line is ideal for a lambda, and three lines is a 
reasonable maximum. If you violate this rule, it can cause serious harm to the readability of your programs. If a lambda 
is long or difficult to read, either find a way to simplify it or refactor your program to eliminate it. 
{Aaron notes: Above is an important design.}
Also, the 
arguments passed to enum constructors are evaluated in a static context. Thus, lambdas in enum constructors can’t access 
instance members of the enum. Constant-specific class bodies are still the way to go if an enum type has constant-specific 
behavior that is difficult to understand, that can’t be implemented in a few lines, or that requires access to instance 
fields or methods.

Likewise, you might think that anonymous classes are obsolete in the era of lambdas. This is closer to the truth, but there 
are a few things you can do with anonymous classes that you can’t do with lambdas. Lambdas are limited to functional interfaces. 
If you want to create an instance of an abstract class, you can do it with an anonymous class, but not a lambda. 
{Aaron notes: Above is an important design.}
Similarly, 
you can use anonymous classes to create instances of interfaces with multiple abstract methods. 
{Aaron notes: Above is an important design.}
Finally, a lambda cannot 
obtain a reference to itself. In a lambda, the this keyword refers to the enclosing instance, which is typically what you 
want. In an anonymous class, the this keyword refers to the anonymous class instance. If you need access to the function 
object from within its body, then you must use an anonymous class.
{Aaron notes: Above is an important design.}

Lambdas share with anonymous classes the property that you can’t reliably serialize and deserialize them across 
implementations. <b>Therefore, you should rarely, if ever, serialize a lambda (or an anonymous class instance).</b> If you have 
a function object that you want to make serializable, such as a Comparator, use an instance of a private static nested 
class (Item 24).
{Aaron notes: Above is an important design.}

### In summary, as of Java 8, lambdas are by far the best way to represent small function objects. <b> Don’t use anonymous classes for function objects unless you have to create instances of types that aren’t functional interfaces.</b> Also, remember that lambdas make it so easy to represent small function objects that it opens the door to functional programming techniques that were not previously practical in Java.

## ITEM 43: PREFER METHOD REFERENCES TO LAMBDAS
The primary advantage of lambdas over anonymous classes is that they are more succinct. Java provides a way to generate 
function objects even more succinct than lambdas: method references. Here is a code snippet from a program that maintains 
a map from arbitrary keys to Integer values. If the value is interpreted as a count of the number of instances of the key, 
then the program is a multiset implementation. The function of the code snippet is to associate the number 1 with the key 
if it is not in the map and to increment the associated value if the key is already present:

```aidl
map.merge(key, 1, (count, incr) -> count + incr);
```

Note that this code uses the merge method, which was added to the Map interface in Java 8. If no mapping is present for 
the given key, the method simply inserts the given value; if a mapping is already present, merge applies the given function 
to the current value and the given value and overwrites the current value with the result. This code represents a typical 
use case for the merge method.

The code reads nicely, but there’s still some boilerplate. The parameters count and incr don’t add much value, and they 
take up a fair amount of space. Really, all the lambda tells you is that the function returns the sum of its two arguments. 
As of Java 8, Integer (and all the other boxed numerical primitive types) provides a static method sum that does exactly 
the same thing. We can simply pass a reference to this method and get the same result with less visual clutter:
{Aaron notes: Above is an important design.}
```aidl
map.merge(key, 1, Integer::sum);
```

The more parameters a method has, the more boilerplate you can eliminate with a method reference. In some lambdas, however, 
the parameter names you choose provide useful documentation, making the lambda more readable and maintainable than a method 
reference, even if the lambda is longer.

There’s nothing you can do with a method reference that you can’t also do with a lambda (with one obscure exception—see 
JLS, 9.9-2 if you’re curious). That said, method references usually result in shorter, clearer code. They also give you 
an out if a lambda gets too long or complex: You can extract the code from the lambda into a new method and replace the 
lambda with a reference to that method. You can give the method a good name and document it to your heart’s content.
{Aaron notes: Above is an important design.}

If you’re programming with an IDE, it will offer to replace a lambda with a method reference wherever it can. You should 
usually, but not always, take the IDE up on the offer. Occasionally, a lambda will be more succinct than a method reference. 
This happens most often when the method is in the same class as the lambda. For example, consider this snippet, which is 
presumed to occur in a class named GoshThisClassNameIsHumongous:

```aidl
service.execute(GoshThisClassNameIsHumongous::action);
```

The lambda equivalent looks like this:
```aidl
service.execute(() -> action());
```

The snippet using the method reference is neither shorter nor clearer than the snippet using the lambda, so prefer the 
latter. Along similar lines, the Function interface provides a generic static factory method to return the identity function, 
Function.identity(). It’s typically shorter and cleaner not to use this method but to code the equivalent lambda inline: x -> x.

Many method references refer to static methods, but there are four kinds that do not. Two of them are bound and unbound 
instance method references. In bound references, the receiving object is specified in the method reference. Bound references 
are similar in nature to static references: the function object takes the same arguments as the referenced method. In 
unbound references, the receiving object is specified when the function object is applied, via an additional parameter 
before the method’s declared parameters. Unbound references are often used as mapping and filter functions in stream 
pipelines (Item 45). Finally, there are two kinds of constructor references, for classes and arrays. Constructor 
references serve as factory objects. All five kinds of method references are summarized in the table below:

Method Ref Type             ||                  Example                 ||          Lambda Equivalent

Static                                      Integer::parseInt                   str -> Integer.parseInt(str)

Bound                                      Instant.now()::isAfter               Instant then = Instant.now(); t -> then.isAfter(t)

Unbound                                     String::toLowerCase                 str -> str.toLowerCase()

Class Constructor                           TreeMap<K,V>::new                   () -> new TreeMap<K,V>

Array Constructor                           int[]::new                          len -> new int[len]
{Aaron notes: Above is an important design.}

### In summary, method references often provide a more succinct alternative to lambdas. Where method references are shorter and clearer, use them; where they aren’t, stick with lambdas.

## ITEM 44: FAVOR THE USE OF STANDARD FUNCTIONAL INTERFACES

## ITEM 45: USE STREAMS JUDICIOUSLY

## ITEM 46: PREFER SIDE-EFFECT-FREE FUNCTIONS IN STREAMS

## ITEM 47: PREFER COLLECTION TO STREAM AS A RETURN TYPE

## ITEM 48: USE CAUTION WHEN MAKING STREAMS PARALLEL