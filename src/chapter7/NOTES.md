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

Now that Java has lambdas, best practices for writing APIs have changed considerably. For example, the Template Method 
pattern [Gamma95], wherein a subclass overrides a primitive method to specialize the behavior of its superclass, is far 
less attractive. The modern alternative is to provide a static factory or constructor that accepts a function object to 
achieve the same effect. More generally, you’ll be writing more constructors and methods that take function objects as 
parameters. Choosing the right functional parameter type demands care.

Consider LinkedHashMap. You can use this class as a cache by overriding its protected removeEldestEntry method, which is invoked by put each time a new key is added to the map. When this method returns true, the map removes its eldest entry, which is passed to the method. The following override allows the map to grow to one hundred entries and then deletes the eldest entry each time a new key is added, maintaining the hundred most recent entries:

```aidl
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
   return size() > 100;
}
```
This interface would work fine, but you shouldn’t use it, because you don’t need to declare a new interface for this 
purpose. The java.util.function package provides a large collection of standard functional interfaces for your use. 
<b> If one of the standard functional interfaces does the job, you should generally use it in preference to a purpose-built 
functional interface. </b> This will make your API easier to learn, by reducing its conceptual surface area, and will provide 
significant interoperability benefits, as many of the standard functional interfaces provide useful default methods. 
{Aaron notes: Above is an important design.}
The Predicate interface, for instance, provides methods to combine predicates. In the case of our LinkedHashMap example, 
the standard BiPredicate<Map<K,V>, Map.Entry<K,V>> interface should be used in preference to a custom 
EldestEntryRemovalFunction interface.

There are forty-three interfaces in java.util.Function. You can’t be expected to remember them all, but if you remember 
six basic interfaces, you can derive the rest when you need them. The basic interfaces operate on object reference types. 
The Operator interfaces represent functions whose result and argument types are the same. The Predicate interface represents 
a function that takes an argument and returns a boolean. The Function interface represents a function whose argument and 
return types differ. The Supplier interface represents a function that takes no arguments and returns (or “supplies”) a 
value. Finally, Consumer represents a function that takes an argument and returns nothing, essentially consuming its 
argument. The six basic functional interfaces are summarized below:

Interface               ||               Function Signature               ||               Example

UnaryOperator<T>                         T apply(T t)                                      String::toLowerCase

BinaryOperator<T>                        T apply(T t1, T t2)                               BigInteger::add

Predicate<T>                             boolean test(T t)                                 Collection::isEmpty

Function<T,R>                            R apply(T t)                                      Arrays::asList                           

Supplier<T>                              T get()                                           Instant::now

Consumer<T>                              void accept(T t)                                  System.out::println

There are also three variants of each of the six basic interfaces to operate on the primitive types int, long, and double. 
Their names are derived from the basic interfaces by prefixing them with a primitive type. So, for example, a predicate 
that takes an int is an IntPredicate, and a binary operator that takes two long values and returns a long is a 
LongBinaryOperator. None of these variant types is parameterized except for the Function variants, which are parameterized 
by return type. For example, LongFunction<int[]> takes a long and returns an int[].

There are nine additional variants of the Function interface, for use when the result type is primitive. The source and 
result types always differ, because a function from a type to itself is a UnaryOperator. If both the source and result 
types are primitive, prefix Function with SrcToResult, for example LongToIntFunction (six variants). If the source is a 
primitive and the result is an object reference, prefix Function with <Src>ToObj, for example DoubleToObjFunction (three variants).

There are two-argument versions of the three basic functional interfaces for which it makes sense to have them: 
BiPredicate<T,U>, BiFunction<T,U,R>, and BiConsumer<T,U>. There are also BiFunction variants returning the three relevant 
primitive types: ToIntBiFunction<T,U>, ToLongBiFunction<T,U>, and ToDoubleBiFunction<T,U>. There are two-argument variants 
of Consumer that take one object reference and one primitive type: ObjDoubleConsumer<T>, ObjIntConsumer<T>, and 
ObjLongConsumer<T>. In total, there are nine two-argument versions of the basic interfaces.

Finally, there is the BooleanSupplier interface, a variant of Supplier that returns boolean values. This is the only explicit 
mention of the boolean type in any of the standard functional interface names, but boolean return values are supported via 
Predicate and its four variant forms. The BooleanSupplier interface and the forty-two interfaces described in the previous 
paragraphs account for all forty-three standard functional interfaces. Admittedly, this is a lot to swallow, and not terribly 
orthogonal. On the other hand, the bulk of the functional interfaces that you’ll need have been written for you and their 
names are regular enough that you shouldn’t have too much trouble coming up with one when you need it.

Most of the standard functional interfaces exist only to provide support for primitive types. <b>Don’t be tempted to use 
basic functional interfaces with boxed primitives instead of primitive functional interfaces. </b>While it works, it 
violates the advice of Item 61, “prefer primitive types to boxed primitives.” The performance consequences of using boxed 
primitives for bulk operations can be deadly. "We need an example here."
{Aaron notes: Above is an important design.}

Now you know that you should typically use standard functional interfaces in preference to writing your own. But when 
should you write your own? Of course you need to write your own if none of the standard ones does what you need, for 
example if you require a predicate that takes three parameters, or one that throws a checked exception. But there are 
times you should write your own functional interface even when one of the standard ones is structurally identical.

Consider our old friend Comparator<T>, which is structurally identical to the ToIntBiFunction<T,T> interface. Even if the 
latter interface had existed when the former was added to the libraries, it would have been wrong to use it. There are 
several reasons that Comparator deserves its own interface. 

1. First, its name provides excellent documentation every time it is used in an API, and it’s used a lot. 

2. Second, the Comparator interface has strong requirements on what constitutes a valid instance, which comprise its 
general contract. By implementing the interface, you are pledging to adhere to its contract. 

3. Third, the interface is heavily outfitted with useful default methods to transform and combine comparators.
{Aaron notes: Above is an important design.}

You should seriously consider writing a purpose-built functional interface in preference to using a standard one if you 
need a functional interface that shares one or more of the following characteristics with Comparator:

• It will be commonly used and could benefit from a descriptive name.

• It has a strong contract associated with it.

• It would benefit from custom default methods.
{Aaron notes: Above is an important design.}

If you elect to write your own functional interface, remember that it’s an interface and hence should be designed with 
great care (Item 21).

Notice that the EldestEntryRemovalFunction interface (page 199) is labeled with the @FunctionalInterface annotation. This 
annotation type is similar in spirit to @Override. It is a statement of programmer intent that serves three purposes: 

1. it tells readers of the class and its documentation that the interface was designed to enable lambdas; 
2. it keeps you honest because the interface won’t compile unless it has exactly one abstract method; and 
3. it prevents maintainers from accidentally adding abstract methods to the interface as it evolves. 
Always annotate your functional interfaces with the @FunctionalInterface annotation.
{Aaron notes: Above is an important design.}

A final point should be made concerning the use of functional interfaces in APIs. Do not provide a method with multiple 
overloadings that take different functional interfaces in the same argument position if it could create a possible 
ambiguity in the client. This is not just a theoretical problem. The submit method of ExecutorService can take either a 
Callable<T> or a Runnable, and it is possible to write a client program that requires a cast to indicate the correct 
overloading (Item 52). The easiest way to avoid this problem is not to write overloadings that take different functional 
interfaces in the same argument position. This is a special case of the advice in Item 52, “use overloading judiciously.”

### In summary, now that Java has lambdas, it is imperative that you design your APIs with lambdas in mind. Accept functional interface types on input and return them on output. It is generally best to use the standard interfaces provided in java.util.function.Function, but keep your eyes open for the relatively rare cases where you would be better off writing your own functional interface.

## ITEM 45: USE STREAMS JUDICIOUSLY

The streams API was added in Java 8 to ease the task of performing bulk operations, sequentially or in parallel. This API provides two key abstractions: the stream, which represents a finite or infinite sequence of data elements, and the stream pipeline, which represents a multistage computation on these elements. The elements in a stream can come from anywhere. Common sources include collections, arrays, files, regular expression pattern matchers, pseudorandom number generators, and other streams. The data elements in a stream can be object references or primitive values. Three primitive types are supported: int, long, and double.

A stream pipeline consists of a source stream followed by zero or more intermediate operations and one terminal operation. Each intermediate operation transforms the stream in some way, such as mapping each element to a function of that element or filtering out all elements that do not satisfy some condition. Intermediate operations all transform one stream into another, whose element type may be the same as the input stream or different from it. The terminal operation performs a final computation on the stream resulting from the last intermediate operation, such as storing its elements into a collection, returning a certain element, or printing all of its elements.

Stream pipelines are evaluated lazily: evaluation doesn’t start until the terminal operation is invoked, and data elements that aren’t required in order to complete the terminal operation are never computed. This lazy evaluation is what makes it possible to work with infinite streams. Note that a stream pipeline without a terminal operation is a silent no-op, so don’t forget to include one.

The streams API is fluent: it is designed to allow all of the calls that comprise a pipeline to be chained into a single expression. In fact, multiple pipelines can be chained together into a single expression.

By default, stream pipelines run sequentially. Making a pipeline execute in parallel is as simple as invoking the parallel method on any stream in the pipeline, but it is seldom appropriate to do so (Item 48).

The streams API is sufficiently versatile that practically any computation can be performed using streams, but just because you can doesn’t mean you should. When used appropriately, streams can make programs shorter and clearer; when used inappropriately, they can make programs difficult to read and maintain. There are no hard and fast rules for when to use streams, but there are heuristics.

Consider the following program, which reads the words from a dictionary file and prints all the anagram groups whose size meets a user-specified minimum. Recall that two words are anagrams if they consist of the same letters in a different order. The program reads each word from a user-specified dictionary file and places the words into a map. The map key is the word with its letters alphabetized, so the key for "staple" is "aelpst", and the key for "petals" is also "aelpst": the two words are anagrams, and all anagrams share the same alphabetized form (or alphagram, as it is sometimes known). The map value is a list containing all of the words that share an alphabetized form. After the dictionary has been processed, each list is a complete anagram group. The program then iterates through the map’s values() view and prints each list whose size meets the threshold:

```aidl
// Prints all large anagram groups in a dictionary iteratively

public class Anagrams {

    public static void main(String[] args) throws IOException {
        File dictionary = new File(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);

        Map<String, Set<String>> groups = new HashMap<>();
        try (Scanner s = new Scanner(dictionary)) {
            while (s.hasNext()) {
                String word = s.next();
                groups.computeIfAbsent(alphabetize(word), (unused) -> new TreeSet<>()).add(word);
            }
        }

        for (Set<String> group : groups.values())
            if (group.size() >= minGroupSize)
                System.out.println(group.size() + ": " + group);
    }

    private static String alphabetize(String s) {
        char[] a = s.toCharArray();
        Arrays.sort(a);
        return new String(a);
    }
}
```
{Aaron notes: Above is an important design.}
One step in this program is worthy of note. The insertion of each word into the map, which is shown in bold, uses the 
computeIfAbsent method, which was added in Java 8. This method looks up a key in the map: If the key is present, the 
method simply returns the value associated with it. If not, the method computes a value by applying the given function 
object to the key, associates this value with the key, and returns the computed value. The computeIfAbsent method simplifies 
the implementation of maps that associate multiple values with each key.

Now consider the following program, which solves the same problem, but makes heavy use of streams. Note that the entire 
program, with the exception of the code that opens the dictionary file, is contained in a single expression. The only 
reason the dictionary is opened in a separate expression is to allow the use of the try-with-resources statement, which 
ensures that the dictionary file is closed:

```aidl
// Overuse of streams - don't do this!

public class Anagrams {

  public static void main(String[] args) throws IOException {
    Path dictionary = Paths.get(args[0]);
    int minGroupSize = Integer.parseInt(args[1]);

      try (Stream<String> words = Files.lines(dictionary)) {
        words.collect(groupingBy(word -> word.chars().sorted()
          .collect(StringBuilder::new,
            (sb, c) -> sb.append((char) c),
            StringBuilder::append).toString()))
            .values().stream()
            .filter(group -> group.size() >= minGroupSize)
            .map(group -> group.size() + ": " + group)
            .forEach(System.out::println);
        }
    }
}
```

If you find this code hard to read, don’t worry; you’re not alone. It is shorter, but it is also less readable, especially 
to programmers who are not experts in the use of streams. <b>Overusing streams makes programs hard to read and maintain.</b>
{Aaron notes: Above is an important design.}

Luckily, there is a happy medium. The following program solves the same problem, using streams without overusing them. 
The result is a program that’s both shorter and clearer than the original:
```aidl
// Tasteful use of streams enhances clarity and conciseness

public class Anagrams {

   public static void main(String[] args) throws IOException {
      Path dictionary = Paths.get(args[0]);
      int minGroupSize = Integer.parseInt(args[1]);

      try (Stream<String> words = Files.lines(dictionary)) {
         words.collect(groupingBy(word -> alphabetize(word)))
           .values().stream()
           .filter(group -> group.size() >= minGroupSize)
           .forEach(g -> System.out.println(g.size() + ": " + g));
      }
   }

   // alphabetize method is the same as in original version
}
```
{Aaron notes: Above is an important design.}
Even if you have little previous exposure to streams, this program is not hard to understand. It opens the dictionary file 
in a try-with-resources block, obtaining a stream consisting of all the lines in the file. The stream variable is named 
words to suggest that each element in the stream is a word. The pipeline on this stream has no intermediate operations; 
its terminal operation collects all the words into a map that groups the words by their alphabetized form (Item 46). This 
is exactly the same map that was constructed in both previous versions of the program. Then a new Stream<List<String>> is 
opened on the values() view of the map. The elements in this stream are, of course, the anagram groups. The stream is 
filtered so that all of the groups whose size is less than minGroupSize are ignored, and finally, the remaining groups 
are printed by the terminal operation forEach.
{Aaron notes: Above is an important design.}

Note that the lambda parameter names were chosen carefully. The parameter g should really be named group, but the 
resulting line of code would be too wide for the book. <b>In the absence of explicit types, careful naming of lambda 
parameters is essential to the readability of stream pipelines. </b>

Note also that word alphabetization is done in a separate alphabetize method. This enhances readability by providing a 
name for the operation and keeping implementation details out of the main program. Using helper methods is even more 
important for readability in stream pipelines than in iterative code because pipelines lack explicit type information 
and named temporary variables.

The alphabetize method could have been reimplemented to use streams, but a stream-based alphabetize method would have been 
less clear, more difficult to write correctly, and probably slower. These deficiencies result from Java’s lack of support 
for primitive char streams (which is not to imply that Java should have supported char streams; it would have been 
infeasible to do so). To demonstrate the hazards of processing char values with streams, consider the following code:
```aidl
"Hello world!".chars().forEach(System.out::print);
```
You might expect it to print Hello world!, but if you run it, you’ll find that it prints 721011081081113211911111410810033. 
This happens because the elements of the stream returned by "Hello world!".chars() are not char values but int values, 
so the int overloading of print is invoked. It is admittedly confusing that a method named chars returns a stream of int 
values. You could fix the program by using a cast to force the invocation of the correct overloading:
```aidl
"Hello world!".chars().forEach(x -> System.out.print((char) x));
```
but ideally you should refrain from using streams to process char values.

When you start using streams, you may feel the urge to convert all your loops into streams, but resist the urge. While it 
may be possible, it will likely harm the readability and maintainability of your code base. As a rule, even moderately 
complex tasks are best accomplished using some combination of streams and iteration, as illustrated by the Anagrams 
programs above. So <b>refactor existing code to use streams and use them in new code only where it makes sense to do so.</b>

As shown in the programs in this item, stream pipelines express repeated computation using function objects (typically 
lambdas or method references), while iterative code expresses repeated computation using code blocks. There are some 
things <b>you can do from code blocks that you can’t do from function objects:</b>

• 1. From a code block, you can read or modify any local variable in scope; from a lambda, you can only read final or 
effectively final variables [JLS 4.12.4], and you can’t modify any local variables.

• 2. From a code block, you can return from the enclosing method, break or continue an enclosing loop, or throw any 
checked exception that this method is declared to throw; from a lambda you can do none of these things.

If a computation is best expressed using these techniques, then it’s probably not a good match for streams. Conversely, 
streams make it very easy to do some things:

• 1. Uniformly transform sequences of elements

• 2. Filter sequences of elements

• 3. Combine sequences of elements using a single operation (for example to add them, concatenate them, or compute their minimum)

• 4. Accumulate sequences of elements into a collection, perhaps grouping them by some common attribute

• 5. Search a sequence of elements for an element satisfying some criterion
If a computation is best expressed using these techniques, then it is a good candidate for streams.
{Aaron notes: Above is an important design.}

One thing that is hard to do with streams is to access corresponding elements from multiple stages of a pipeline simultaneously: 
once you map a value to some other value, the original value is lost. One workaround is to map each value to a pair object 
containing the original value and the new value, but this is not a satisfying solution, especially if the pair objects are 
required for multiple stages of a pipeline. The resulting code is messy and verbose, which defeats a primary purpose of 
streams. When it is applicable, a better workaround is to invert the mapping when you need access to the earlier-stage value.
{Aaron notes: Above is an important design.}

For example, let’s write a program to print the first twenty Mersenne primes. To refresh your memory, a Mersenne number is a 
number of the form 2p − 1. If p is prime, the corresponding Mersenne number may be prime; if so, it’s a Mersenne prime. 
As the initial stream in our pipeline, we want all the prime numbers. Here’s a method to return that (infinite) stream. 
We assume a static import has been used for easy access to the static members of BigInteger:
```aidl
static Stream<BigInteger> primes() {
    return Stream.iterate(TWO, BigInteger::nextProbablePrime);
}
```

The name of the method (primes) is a plural noun describing the elements of the stream. This naming convention is highly 
recommended for all methods that return streams because it enhances the readability of stream pipelines. The method uses 
the static factory Stream.iterate, which takes two parameters: the first element in the stream, and a function to generate 
the next element in the stream from the previous one. Here is the program to print the first twenty Mersenne primes:
```aidl
public static void main(String[] args) {
    primes().map(p -> TWO.pow(p.intValueExact()).subtract(ONE))
        .filter(mersenne -> mersenne.isProbablePrime(50))
        .limit(20)
        .forEach(System.out::println);
}
```

This program is a straightforward encoding of the prose description above: it starts with the primes, computes the 
corresponding Mersenne numbers, filters out all but the primes (the magic number 50 controls the probabilistic primality 
test), limits the resulting stream to twenty elements, and prints them out.

Now suppose that we want to precede each Mersenne prime with its exponent (p). This value is present only in the initial 
stream, so it is inaccessible in the terminal operation, which prints the results. Luckily, it’s easy to compute the 
exponent of a Mersenne number by inverting the mapping that took place in the first intermediate operation. The exponent 
is simply the number of bits in the binary representation, so this terminal operation generates the desired result:
```aidl
.forEach(mp -> System.out.println(mp.bitLength() + ": " + mp));
```

There are plenty of tasks where it is not obvious whether to use streams or iteration. For example, consider the task of 
initializing a new deck of cards. Assume that Card is an immutable value class that encapsulates a Rank and a Suit, both 
of which are enum types. This task is representative of any task that requires computing all the pairs of elements that 
can be chosen from two sets. Mathematicians call this the Cartesian product of the two sets. Here’s an iterative 
implementation with a nested for-each loop that should look very familiar to you:

```aidl
// Iterative Cartesian product computation
private static List<Card> newDeck() {
    List<Card> result = new ArrayList<>();
    for (Suit suit : Suit.values())
        for (Rank rank : Rank.values())
            result.add(new Card(suit, rank));
    return result;
}
```

And here is a stream-based implementation that makes use of the intermediate operation flatMap. This operation maps each 
element in a stream to a stream and then concatenates all of these new streams into a single stream (or flattens them). 
Note that this implementation contains a nested lambda, shown in boldface:
```aidl
// Stream-based Cartesian product computation
private static List<Card> newDeck() {
    return Stream.of(Suit.values())
        .flatMap(suit -> Stream.of(Rank.values())
            .map(rank -> new Card(suit, rank)))
        .collect(toList());
}
```

Which of the two versions of newDeck is better? It boils down to personal preference and the environment in which you’re 
programming. The first version is simpler and perhaps feels more natural. A larger fraction of Java programmers will be 
able to understand and maintain it, but some programmers will feel more comfortable with the second (stream-based) 
version. It’s a bit more concise and not too difficult to understand if you’re reasonably well-versed in streams and 
functional programming. If you’re not sure which version you prefer, the iterative version is probably the safer choice. 
If you prefer the stream version and you believe that other programmers who will work with the code will share your 
preference, then you should use it.

### In summary, some tasks are best accomplished with streams, and others with iteration. Many tasks are best accomplished by combining the two approaches. There are no hard and fast rules for choosing which approach to use for a task, but there are some useful heuristics. In many cases, it will be clear which approach to use; in some cases, it won’t. 
### <b>If you’re not sure whether a task is better served by streams or iteration, try both and see which works better.</b>

## ITEM 46: PREFER SIDE-EFFECT-FREE FUNCTIONS IN STREAMS

## ITEM 47: PREFER COLLECTION TO STREAM AS A RETURN TYPE

## ITEM 48: USE CAUTION WHEN MAKING STREAMS PARALLEL