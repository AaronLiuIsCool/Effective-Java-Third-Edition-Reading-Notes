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

If you’re new to streams, it can be difficult to get the hang of them. Merely expressing your computation as a stream pipeline can be hard. When you succeed, your program will run, but you may realize little if any benefit. Streams isn’t just an API, it’s a paradigm based on functional programming. In order to obtain the expressiveness, speed, and in some cases parallelizability that streams have to offer, you have to adopt the paradigm as well as the API.

The most important part of the streams paradigm is to structure your computation as a sequence of transformations where the result of each stage is as close as possible to a pure function of the result of the previous stage. A pure function is one whose result depends only on its input: it does not depend on any mutable state, nor does it update any state. In order to achieve this, any function objects that you pass into stream operations, both intermediate and terminal, should be free of side-effects.

Occasionally, you may see streams code that looks like this snippet, which builds a frequency table of the words in a text file:
```aidl
// Uses the streams API but not the paradigm--Don't do this!
Map<String, Long> freq = new HashMap<>();
try (Stream<String> words = new Scanner(file).tokens()) {
    words.forEach(word -> {
        freq.merge(word.toLowerCase(), 1L, Long::sum);
    });
}
```

What’s wrong with this code? After all, it uses streams, lambdas, and method references, and gets the right answer. Simply 
put, it’s not streams code at all; it’s iterative code masquerading as streams code. It derives no benefits from the streams 
API, and it’s (a bit) longer, harder to read, and less maintainable than the corresponding iterative code. The problem 
stems from the fact that this code is doing all its work in a terminal forEach operation, using a lambda that mutates 
external state (the frequency table). A forEach operation that does anything more than present the result of the 
computation performed by a stream is a “bad smell in code,” as is a lambda that mutates state. So how should this code look?
```aidl
// Proper use of streams to initialize a frequency table
Map<String, Long> freq;
try (Stream<String> words = new Scanner(file).tokens()) {

    freq = words.collect(groupingBy(String::toLowerCase, counting()));
}
```
This snippet does the same thing as the previous one but makes proper use of the streams API. It’s shorter and clearer. 
So why would anyone write it the other way? Because it uses tools they’re already familiar with. Java programmers know 
how to use for-each loops, and the forEach terminal operation is similar. But the forEach operation is among the least 
powerful of the terminal operations and the least stream-friendly. It’s explicitly iterative, and hence not amenable to 
parallelization. <b>The forEach operation should be used only to report the result of a stream computation, not to perform 
the computation. </b> Occasionally, it makes sense to use forEach for some other purpose, such as adding the results of a 
stream computation to a preexisting collection. 
{Aaron notes: Above is an important design.}

The improved code uses a collector, which is a new concept that you have to learn in order to use streams. The Collectors API is intimidating: it has thirty-nine methods, some of which have as many as five type parameters. The good news is that you can derive most of the benefit from this API without delving into its full complexity. For starters, you can ignore the Collector interface and think of a collector as an opaque object that encapsulates a reduction strategy. In this context, reduction means combining the elements of a stream into a single object. The object produced by a collector is typically a collection (which accounts for the name collector).

The collectors for gathering the elements of a stream into a true Collection are straightforward. There are three such collectors: toList(), toSet(), and toCollection(collectionFactory). They return, respectively, a set, a list, and a programmer-specified collection type. Armed with this knowledge, we can write a stream pipeline to extract a top-ten list from our frequency table.
```aidl
// Pipeline to get a top-ten list of words from a frequency table

List<String> topTen = freq.keySet().stream()
    .sorted(comparing(freq::get).reversed())
    .limit(10)
    .collect(toList());
```
Note that we haven’t qualified the toList method with its class, Collectors. <b>It is customary and wise to statically 
import all members of Collectors because it makes stream pipelines more readable. </b>

The only tricky part of this code is the comparator that we pass to sorted, comparing(freq::get).reversed(). The comparing 
method is a comparator construction method (Item 14) that takes a key extraction function. The function takes a word, and 
the “extraction” is actually a table lookup: the bound method reference freq::get looks up the word in the frequency table 
and returns the number of times the word appears in the file. Finally, we call reversed on the comparator, so we’re sorting 
the words from most frequent to least frequent. Then it’s a simple matter to limit the stream to ten words and collect them 
into a list.

The previous code snippets use Scanner’s stream method to get a stream over the scanner. This method was added in Java 9. If you’re using an earlier release, you can translate the scanner, which implements Iterator, into a stream using an adapter similar to the one in Item 47 (streamOf(Iterable<E>)).

So what about the other thirty-six methods in Collectors? Most of them exist to let you collect streams into maps, which is far more complicated than collecting them into true collections. Each stream element is associated with a key and a value, and multiple stream elements can be associated with the same key.

The simplest map collector is toMap(keyMapper, valueMapper), which takes two functions, one of which maps a stream element to a key, the other, to a value. We used this collector in our fromString implementation in Item 34 to make a map from the string form of an enum to the enum itself:

```aidl
// Using a toMap collector to make a map from string to enum
private static final Map<String, Operation> stringToEnum =

    Stream.of(values()).collect( toMap(Object::toString, e -> e) );
```
This simple form of toMap is perfect if each element in the stream maps to a unique key. If multiple stream elements map 
to the same key, the pipeline will terminate with an IllegalStateException.
{Aaron notes: Above is an important design.}

The more complicated forms of toMap, as well as the groupingBy method, give you various ways to provide strategies for 
dealing with such collisions. One way is to provide the toMap method with a merge function in addition to its key and value 
mappers. The merge function is a BinaryOperator<V>, where V is the value type of the map. Any additional values associated 
with a key are combined with the existing value using the merge function, so, for example, if the merge function is 
multiplication, you end up with a value that is the product of all the values associated with the key by the value mapper.

The three-argument form of toMap is also useful to make a map from a key to a chosen element associated with that key. 
For example, suppose we have a stream of record albums by various artists, and we want a map from recording artist to 
best-selling album. This collector will do the job.

```aidl
// Collector to generate a map from key to chosen element for key

Map<Artist, Album> topHits = albums.collect( <b>toMap(Album::artist, a->a, maxBy(comparing(Album::sales)))</b> );
```
{Aaron notes: Above is an important design.}

Note that the comparator uses the static factory method maxBy, which is statically imported from BinaryOperator. This 
method converts a Comparator<T> into a BinaryOperator<T> that computes the maximum implied by the specified comparator. 
In this case, the comparator is returned by the comparator construction method comparing, which takes the key extractor 
function Album::sales. This may seem a bit convoluted, but the code reads nicely. Loosely speaking, it says, “convert 
the stream of albums to a map, mapping each artist to the album that has the best album by sales.” This is surprisingly 
close to the problem statement.

Another use of the three-argument form of toMap is to produce a collector that imposes a last-write-wins policy when 
there are collisions. For many streams, the results will be nondeterministic, but if all the values that may be associated 
with a key by the mapping functions are identical, or if they are all acceptable, this collector’s s behavior may be just 
what you want:
```aidl
// Collector to impose last-write-wins policy
toMap(keyMapper, valueMapper, (v1, v2) -> v2)
```

The third and final version of toMap takes a fourth argument, which is a map factory, for use when you want to specify a 
particular map implementation such as an EnumMap or a TreeMap.

There are also variant forms of the first three versions of toMap, named toConcurrentMap, that run efficiently in parallel 
and produce ConcurrentHashMap instances.

In addition to the toMap method, the Collectors API provides the groupingBy method, which returns collectors to produce 
maps that group elements into categories based on a classifier function. The classifier function takes an element and 
returns the category into which it falls. This category serves as the element’s map key. The simplest version of the 
groupingBy method takes only a classifier and returns a map whose values are lists of all the elements in each category. 
This is the collector that we used in the Anagram program in Item 45 to generate a map from alphabetized word to a list 
of the words sharing the alphabetization:
```
words.collect(groupingBy(word -> alphabetize(word)))
```

If you want groupingBy to return a collector that produces a map with values other than lists, you can specify a downstream 
collector in addition to a classifier. A downstream collector produces a value from a stream containing all the elements 
in a category. The simplest use of this parameter is to pass toSet(), which results in a map whose values are sets of 
elements rather than lists.

Alternatively, you can pass toCollection(collectionFactory), which lets you create the collections into which each category 
of elements is placed. This gives you the flexibility to choose any collection type you want. Another simple use of the 
two-argument form of groupingBy is to pass counting() as the downstream collector. This results in a map that associates 
each category with the number of elements in the category, rather than a collection containing the elements. That’s what 
you saw in the frequency table example at the beginning of this item:
```aidl
Map<String, Long> freq = words.collect(groupingBy(String::toLowerCase, counting()));
```

The third version of groupingBy lets you specify a map factory in addition to a downstream collector. Note that this 
method violates the standard telescoping argument list pattern: the mapFactory parameter precedes, rather than follows, 
the downStream parameter. This version of groupingBy gives you control over the containing map as well as the contained 
collections, so, for example, you can specify a collector that returns a TreeMap whose values are TreeSets.

The groupingByConcurrent method provides variants of all three overloadings of groupingBy. These variants run efficiently 
in parallel and produce ConcurrentHashMap instances. There is also a rarely used relative of groupingBy called partitioningBy. 
In lieu of a classifier method, it takes a predicate and returns a map whose key is a Boolean. There are two overloadings 
of this method, one of which takes a downstream collector in addition to a predicate.

The collectors returned by the counting method are intended only for use as downstream collectors. The same functionality 
is available directly on Stream, via the count method, so <b>there is never a reason to say collect(counting())</b>. There are 
fifteen more Collectors methods with this property. They include the nine methods whose names begin with summing, averaging, 
and summarizing (whose functionality is available on the corresponding primitive stream types). They also include all 
overloadings of the reducing method, and the filtering, mapping, flatMapping, and collectingAndThen methods. Most 
programmers can safely ignore the majority of these methods. From a design perspective, these collectors represent an 
attempt to partially duplicate the functionality of streams in collectors so that downstream collectors can act as 
“ministreams.”
{Aaron notes: Above is an important design.}

There are three Collectors methods we have yet to mention. Though they are in Collectors, they don’t involve collections. 
The first two are minBy and maxBy, which take a comparator and return the minimum or maximum element in the stream as 
determined by the comparator. They are minor generalizations of the min and max methods in the Stream interface and are 
the collector analogues of the binary operators returned by the like-named methods in BinaryOperator. Recall that we used 
BinaryOperator.maxBy in our best-selling album example.

The final Collectors method is joining, which operates only on streams of CharSequence instances such as strings. In its 
parameterless form, it returns a collector that simply concatenates the elements. Its one argument form takes a single 
CharSequence parameter named delimiter and returns a collector that joins the stream elements, inserting the delimiter 
between adjacent elements. If you pass in a comma as the delimiter, the collector returns a comma-separated values string 
(but beware that the string will be ambiguous if any of the elements in the stream contain commas). The three argument form 
takes a prefix and suffix in addition to the delimiter. The resulting collector generates strings like the ones that you 
get when you print a collection, for example [came, saw, conquered].

### In summary, the essence of programming stream pipelines is side-effect-free function objects. This applies to all of the 
### many function objects passed to streams and related objects. The terminal operation forEach should only be used to report 
### the result of a computation performed by a stream, not to perform the computation. In order to use streams properly, you 
### have to know about collectors. The most important collector factories are <b>toList, toSet, toMap, groupingBy, and joining.</b>
{Aaron notes: Above is an important design.}

## ITEM 47: PREFER COLLECTION TO STREAM AS A RETURN TYPE

## ITEM 48: USE CAUTION WHEN MAKING STREAMS PARALLEL