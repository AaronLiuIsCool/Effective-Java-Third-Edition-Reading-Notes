
# Chapter 2. Creating and Destroying Objects

## ITEM 1: CONSIDER STATIC FACTORY METHODS INSTEAD OF CONSTRUCTORS

Pros: 
1. One advantage of static factory methods is that, unlike constructors, they have names.
2. A second advantage of static factory methods is that, unlike constructors, they are not required to create a new object each time they’re invoked. 
3. A third advantage of static factory methods is that, unlike constructors, they can return an object of any subtype of their return type. 
4. A fourth advantage of static factories is that the class of the returned object can vary from call to call as a function of the input parameters. 
5. A fifth advantage of static factories is that the class of the returned object need not exist when the class containing the method is written. 
Cons:
1. The main limitation of providing only static factory methods is that classes without public or protected constructors can NOT be sub-classed. 
2. A second shortcoming of static factory methods is that they are hard for programmers to find. 
    1. Possible solution: document tool such as Javadoc tool. 
    2. Common naming conventions. 
        1. from - a type-conversion method that takes a single parameter and returns a corresponding instance of this type. For example: Date d = Date.from(instant);
        2. of - An aggregation method that takes multiple parameters and returns an instance of this type that incorporates them. For example: Set<Rank> faceCards = EnumSet.of(JACK, QUEEN, KING);
        3. valueOf - A more verbose alternative to from and of For example: BigInteger prime = BigInteger.valueOf(Integer.MAX_VALUE);
        4. instance or newInstance - Like instance or getInstance,
        5. getType
        6. newType
        7. type = A concise alternative to getType and newType, for example: List<Complaint> litany = Collections.list(legacyLitany);

In short, static factory methods and public //TODO finish all

## ITEM 2: CONSIDER A BUILDER WHEN FACED WITH MANY CONSTRUCTOR PARAMETERS

 In short, the telescoping constructor pattern works, but it is hard to write client code when there are many parameters, and harder still to read it. 

// Builder Pattern
```
public final class NutritionFact {
	private final int servingSize;
	private final int servings;
	private final int calories;
	private final int fat;
	private final int sodium;
	private final int carbohydrate;
	//etc…

	public static class Builder {
		//Required parameters
	    private final int servingSize;
	    private final int servings;
	    
	    //Optional parameters - initialized to default values.
	    private int calories = 0;
	    private int fat = 0;
	    private int sodium = 0;
	    private int carbohydrate = 0;
	    
	    public Builder(int servingSize, int servings) {
	        this.servingSize = servingSize;
	        this.servings = servings;
	    }
	    
	    public Builder calories(int val) { calories = val; return this; }
	    public Builder fat(int val) { fat = val; return this; }
	    public Builder sodium(int val) { sodium = val; return this; }
	    public Builder carbohydrate(int val) { carbohydrate = val; return this; }
	    
	    public NutritionFacts build() { return new NutritionFacts(this); }
	    
	    private NutritionFacts(Builder builder) {
	        servingSize  = builder.servingSize;
            servings     = builder.servings;
            calories     = builder.calories;
            fat          = builder.fat;
            sodium       = builder.sodium;
            carbohydrate = builder.carbohydrate;
	    }
	}
}
```
The NutritionFact class is an immutable class, and all parameter default values are in one places.
The builder's setter methods return the builder itself so that invocations can be chained. As below,
```$xslt
NutritionFact sprite = new NutritionFact.Builder(240, 8)
    .calories(100).far(20).sodium(30).build();
```
This client code is easy to write and, more importantly, easy to read. 
The Builder pattern simulates named optional parameters as found in Python and Scala.

The Builder pattern is well suited to class hierarchies. 
Use a parallel hierarchy of builders, each nested in the corresponding class. 
Abstract classes have abstract builders; concrete classes have concrete builders. 
For example, consider an abstract class at the root of a hierarchy representing various kinds of pizza:
```$xslt
// Builder pattern for class hierarchies
public abstract class Pizza {

   public enum Topping { HAM, MUSHROOM, ONION, PEPPER, SAUSAGE }

   final Set<Topping> toppings;

   abstract static class Builder<T extends Builder<T>> {

      EnumSet<Topping> toppings = EnumSet.noneOf(Topping.class);

      public T addTopping(Topping topping) {

         toppings.add(Objects.requireNonNull(topping));

         return self();

      }

      abstract Pizza build();

      // Subclasses must override this method to return "this"

      protected abstract T self();

   }

   Pizza(Builder<?> builder) {

      toppings = builder.toppings.clone(); // See Item  50

   }
}
```
Note that Pizza.Builder is a generic type with a recursive type parameter (Item 30). 
This, along with the abstract self method, allows method chaining to work properly in subclasses, without the need for casts. 
This workaround for the fact that Java lacks a self type is known as the simulated self-type idiom.

Here are two concrete subclasses of Pizza, one of which represents a standard New-York-style pizza, the other a calzone. The former has a required size parameter, while the latter lets you specify whether sauce should be inside or out:
```$xslt
public class NyPizza extends Pizza {
    public enum Size { SMALL, MEDIUM, LARGE }
    private final Size size;

    public static class Builder extends Pizza.Builder<Builder> {

        private final Size size;

        public Builder(Size size) {

            this.size = Objects.requireNonNull(size);

        }

        @Override public NyPizza build() {
            return new NyPizza(this);
        }

        @Override protected Builder self() { return this; }
    }
    
    private NyPizza(Builder builder) {
        super(builder);
        size = builder.size;
    }
}

public class Calzone extends Pizza {
    private final boolean sauceInside;
    
    public static class Builder extends Pizza.Builder<Builder> {

        private boolean sauceInside = false; // Default

        public Builder sauceInside() {
            sauceInside = true;
            return this;

        }

        @Override public Calzone build() {
            return new Calzone(this);
        }
        
        @Override protected Builder self() { return this; }
    }

    private Calzone(Builder builder) {
        super(builder);
        sauceInside = builder.sauceInside;
    }
}
```
Note that the build method in each subclass’s builder is declared to return the correct subclass: the build method of NyPizza.Builder returns NyPizza, while the one in Calzone.Builder returns Calzone. This technique, wherein a subclass method is declared to return a subtype of the return type declared in the super-class, is known as covariant return typing. It allows clients to use these builders without the need for casting.

The client code for these “hierarchical builders” is essentially identical to the code for the simple NutritionFacts builder. The example client code shown next assumes static imports on enum constants for brevity:
```$xslt
NyPizza pizza = new NyPizza.Builder(SMALL)
        .addTopping(SAUSAGE).addTopping(ONION).build();

Calzone calzone = new Calzone.Builder()
        .addTopping(HAM).sauceInside().build();
```
A minor advantage of builders over constructors is that builders can have multiple varargs parameters because each parameter is specified in its own method. Alternatively, builders can aggregate the parameters passed into multiple calls to a method into a single field, as demonstrated in the addTopping method earlier.

The Builder pattern is quite flexible. A single builder can be used repeatedly to build multiple objects. The parameters of the builder can be tweaked between invocations of the build method to vary the objects that are created. A builder can fill in some fields automatically upon object creation, such as a serial number that increases each time an object is created.

The Builder pattern has disadvantages as well. In order to create an object, you must first create its builder. While the cost of creating this builder is unlikely to be noticeable in practice, it could be a problem in performance-critical situations. Also, the Builder pattern is more verbose than the telescoping constructor pattern, so it should be used only if there are enough parameters to make it worthwhile, say four or more. But keep in mind that you may want to add more parameters in the future. But if you start out with constructors or static factories and switch to a builder when the class evolves to the point where the number of parameters gets out of hand, the obsolete constructors or static factories will stick out like a sore thumb. Therefore, it’s often better to start with a builder in the first place.

In summary, the Builder pattern is a good choice when designing classes whose constructors or static factories would have more than a handful of parameters, especially if many of the parameters are optional or of identical type. Client code is much easier to read and write with builders than with telescoping constructors, and builders are much safer than JavaBeans.

## ITEM 3: ENFORCE THE SINGLETON PROPERTY WITH A PRIVATE CONSTRUCTOR OR AN ENUM TYPE

The main advantage of the public field approach is that the API makes it clear that the class is a singleton: the public static field is final, so it will always contain the same object reference. The second advantage is that it’s simpler.

One advantage of the static factory approach is that it gives you the flexibility to change your mind about whether the class is a singleton without changing its API. The factory method returns the sole instance, but it could be modified to return, say, a separate instance for each thread that invokes it. A second advantage is that you can write a generic singleton factory if your application requires it (Item 30). A final advantage of using a static factory is that a method reference can be used as a supplier, for example Elvis::instance is a Supplier<Elvis>. Unless one of these advantages is relevant, the public field approach is preferable.

Personal notes:
(This one's example is not good as Initialization Demand Holder (IoDH) in concurrency java) as below:
```$xslt
//Initialization on Demand Holder 
public class Singleton {
    private Singleton() {}
    
    private static class HolderClass() {
        private final static Singleton instance = new Single();
    }
    
    public static Singleton getInstance() {
        return HolderClass.instance;
    }
    
    public static void main(String args[]) {
        Singleton s1, s2;
        s1 = Singleton.getInstance();
        s2 = Singleton.getInstance();
        System.out.println( s1 == s2 );
    }
}
```
编译并运行上述代码，运行结果为：true，即创建的单例对象s1和s2为同一对象。
由于静态单例对象没有作为Singleton的成员变量直接实例化，因此类加载时不会实例化Singleton，第一次调用getInstance()时将加载内部类HolderClass，在该内部类中定义了一个static类型的变量instance，此时会首先初始化这个成员变量，由Java虚拟机来保证其线程安全性，确保该成员变量只能初始化一次。
由于getInstance()方法没有任何线程锁定，因此其性能不会造成任何影响。

通过使用IoDH，我们既可以实现延迟加载，又可以保证线程安全，不影响系统性能，不失为一种最好的Java语言单例模式实现方式（其缺点是与编程语言本身的特性相关，很多面向对象语言不支持IoDH）。

练习

分别使用饿汉式单例、带双重检查锁定机制的懒汉式单例以及IoDH技术实现负载均衡器LoadBalancer。
至此，三种单例类的实现方式我们均已学习完毕，它们分别是饿汉式单例、懒汉式单例以及IoDH。

## ITEM 4: ENFORCE NONINSTANTIABILITY WITH A PRIVATE CONSTRUCTOR

Such utility classes were not designed to be instantiated: an instance would be nonsensical. 
In the absence of explicit constructors, however, the compiler provides a public, parameterless default constructor. 
To a user, this constructor is indistinguishable from any other. 
It is not uncommon to see unintentionally instantiable classes in published APIs.

Attempting to enforce noninstantiability by making a class abstract does not work. 
The class can be subclassed and the subclass instantiated. 
Furthermore, it misleads the user into thinking the class was designed for inheritance (Item 19). 
There is, however, a simple idiom to ensure noninstantiability. 
A default constructor is generated only if a class contains no explicit constructors, so a class can be made noninstantiable by including a private constructor:

```$xslt
// Noninstantiable utility class
public class UtilityClass {
    // Suppress default constructor for noninstantiability
    private UtilityClass() {
        throw new AssertionError();
    }
    ... // Remainder omitted
}
```
As a side effect, this idiom also prevents the class from being subclassed. All constructors must invoke a superclass constructor, explicitly or implicitly, and a subclass would have no accessible superclass constructor to invoke.

## ITEM 5: PREFER DEPENDENCY INJECTION TO HARDWIRING RESOURCES
Static utility classes and singletons are inappropriate for classes whose behavior is parameterized by an underlying resource.
What is required is the ability to support multiple instances of the class (in our example, SpellChecker), each of which uses the resource desired by the client (in our example, the dictionary). 
A simple pattern that satisfies this requirement is to pass the resource into the constructor when creating a new instance. This is one form of dependency injection: the dictionary is a dependency of the spell checker and is injected into the spell checker when it is created. 
```aidl
// Dependency injection provides flexibility and testability

public class SpellChecker {

    private final Lexicon dictionary;

    public SpellChecker(Lexicon dictionary) {

        this.dictionary = Objects.requireNonNull(dictionary);

    }

    public boolean isValid(String word) { ... }

    public List<String> suggestions(String typo) { ... }

}
```
The dependency injection pattern is so simple that many programmers use it for years without knowing it has a name. 
While our spell checker example had only a single resource (the dictionary), dependency injection works with an arbitrary number of resources and arbitrary dependency graphs. 
It preserves immutability (Item 17), so multiple clients can share dependent objects (assuming the clients desire the same underlying resources). 
Dependency injection is equally applicable to constructors, static factories (Item 1), and builders (Item 2).

A useful variant of the pattern is to pass a resource factory to the constructor. 
A factory is an object that can be called repeatedly to create instances of a type. 
Such factories embody the Factory Method pattern [Gamma95]. 
The Supplier<T> interface, introduced in Java 8, is perfect for representing factories. 
Methods that take a Supplier<T> on input should typically constrain the factory’s type parameter using a bounded wildcard type (Item 31) to allow the client to pass in a factory that creates any subtype of a specified type.
For example, here is a method that makes a mosaic using a client-provided factory to produce each tile:
```aidl
Mosaic create(Supplier<? extends Tile> tileFactory) { ... }
```
Although dependency injection greatly improves flexibility and testability, it can clutter up large projects, which typically contain thousands of dependencies. 
This clutter can be all but eliminated by using a dependency injection framework, such as Dagger [Dagger], Guice [Guice], or Spring [Spring]. 
The use of these frameworks is beyond the scope of this book, but note that APIs designed for manual dependency injection are trivially adapted for use by these frameworks.

In summary, do not use a singleton or static utility class to implement a class that depends on one or more underlying resources whose behavior affects that of the class, 
and do not have the class create these resources directly. 
Instead, pass the resources, or factories to create them, into the constructor (or static factory or builder). 
This practice, known as dependency injection, will greatly enhance the flexibility, reusability, and testability of a class.

## ITEM 6: AVOID CREATING UNNECESSARY OBJECTS
It is often appropriate to reuse a single object instead of creating a new functionally equivalent object each time it is needed. 
Reuse can be both faster and more stylish. An object can always be reused if it is immutable.

You can often avoid creating unnecessary objects by using static factory methods (Item 1) in preference to constructors on immutable classes that provide both. 
For example, the factory method Boolean.valueOf(String) is preferable to the constructor Boolean(String), which was deprecated in Java 9. 
The constructor must create a new object each time it’s called, while the factory method is never required to do so and won’t in practice. 
In addition to reusing immutable objects, you can also reuse mutable objects if you know they won’t be modified.

Some object creations are much more expensive than others. 
If you’re going to need such an “expensive object” repeatedly, it may be advisable to cache it for reuse.
Unfortunately, it’s not always obvious when you’re creating such an object. 
Suppose you want to write a method to determine whether a string is a valid Roman numeral. 
Here’s the easiest way to do this using a regular expression:
```aidl
// Performance can be greatly improved!

static boolean isRomanNumeral(String s) {

    return s.matches("^(?=.)M*(C[MD]|D?C{0,3})"

            + "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");

}
```
The problem with this implementation is that it relies on the String.matches method. 
While String.matches is the easiest way to check if a string matches a regular expression, 
it’s not suitable for repeated use in performance-critical situations. 
The problem is that it internally creates a Pattern instance for the regular expression and uses it only once, 
after which it becomes eligible for garbage collection. 
Creating a Pattern instance is expensive because it requires compiling the regular expression into a finite state machine.

To improve the performance, explicitly compile the regular expression into a Pattern instance (which is immutable) as part of class initialization, 
cache it, and reuse the same instance for every invocation of the isRomanNumeral method:
```aidl
// Reusing expensive object for improved performance

public class RomanNumerals {

    private static final Pattern ROMAN = Pattern.compile(

            "^(?=.)M*(C[MD]|D?C{0,3})"

            + "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");

    static boolean isRomanNumeral(String s) {

        return ROMAN.matcher(s).matches();

    }

}
```

The improved version of isRomanNumeral provides significant performance gains if invoked frequently. On my machine, the original version takes 1.1 µs on an 8-character input string, while the improved version takes 0.17 µs, which is 6.5 times faster. Not only is the performance improved, but arguably, so is clarity. Making a static final field for the otherwise invisible Pattern instance allows us to give it a name, which is far more readable than the regular expression itself.

If the class containing the improved version of the isRomanNumeral method is initialized but the method is never invoked, the field ROMAN will be initialized needlessly. It would be possible to eliminate the initialization by lazily initializing the field (Item 83) the first time the isRomanNumeral method is invoked, but this is not recommended. As is often the case with lazy initialization, it would complicate the implementation with no measurable performance improvement (Item 67).

When an object is immutable, it is obvious it can be reused safely, but there are other situations where it is far less obvious, even counterintuitive. Consider the case of adapters [Gamma95], also known as views. An adapter is an object that delegates to a backing object, providing an alternative interface. Because an adapter has no state beyond that of its backing object, there’s no need to create more than one instance of a given adapter to a given object.

For example, the keySet method of the Map interface returns a Set view of the Map object, consisting of all the keys in the map. 
Naively, it would seem that every call to keySet would have to create a new Set instance, but every call to keySet on a given Map object may return the same Set instance. Although the returned Set instance is typically mutable, all of the returned objects are functionally identical: when one of the returned objects changes, so do all the others, because they’re all backed by the same Map instance. While it is largely harmless to create multiple instances of the keySet view object, it is unnecessary and has no benefits.

Another way to create unnecessary objects is autoboxing, which allows the programmer to mix primitive and boxed primitive types, boxing and unboxing automatically as needed. Autoboxing blurs but does not erase the distinction between primitive and boxed primitive types. There are subtle semantic distinctions and not-so-subtle performance differences (Item 61). Consider the following method, which calculates the sum of all the positive int values. To do this, the program has to use long arithmetic because an int is not big enough to hold the sum of all the positive int values:

```aidl
// Hideously slow! Can you spot the object creation?

private static long sum() {

    Long sum = 0L;

    for (long i = 0; i <= Integer.MAX_VALUE; i++)
        sum += i;

    return sum;

}
```
This program gets the right answer, but it is much slower than it should be, due to a one-character typographical error. 
The variable sum is declared as a Long instead of a long, which means that the program constructs about 231 unnecessary Long instances (roughly one for each time the long i is added to the Long sum). 
Changing the declaration of sum from Long to long reduces the runtime from 6.3 seconds to 0.59 seconds on my machine. 
The lesson is clear: prefer primitives to boxed primitives, and watch out for unintentional autoboxing.

This item should not be misconstrued to imply that object creation is expensive and should be avoided. 
On the contrary, the creation and reclamation of small objects whose constructors do little explicit work is cheap, especially on modern JVM implementations. 
Creating additional objects to enhance the clarity, simplicity, or power of a program is generally a good thing.

Conversely, avoiding object creation by maintaining your own object pool is a bad idea unless the objects in the pool are extremely heavyweight. 
The classic example of an object that does justify an object pool is a database connection. 
The cost of establishing the connection is sufficiently high that it makes sense to reuse these objects. 
Generally speaking, however, maintaining your own object pools clutters your code, increases memory footprint, and harms performance.
Modern JVM implementations have highly optimized garbage collectors that easily outperform such object pools on lightweight objects.

The counterpoint to this item is Item 50 on defensive copying. The present item says, 
“Don’t create a new object when you should reuse an existing one,” while Item 50 says, 
“Don’t reuse an existing object when you should create a new one.” 
Note that the penalty for reusing an object when defensive copying is called for is far greater than the penalty for 
needlessly creating a duplicate object. Failing to make defensive copies where required can lead to insidious bugs and 
security holes; creating objects unnecessarily merely affects style and performance.

## ITEM 7: ELIMINATE OBSOLETE OBJECT REFERENCES
If you switched from a language with manual memory management, such as C or C++, to a garbage-collected language such as Java, your job as a programmer was made much easier by the fact that your objects are automatically reclaimed when you’re through with them. It seems almost like magic when you first experience it. It can easily lead to the impression that you don’t have to think about memory management, but this isn’t quite true.

Consider the following simple stack implementation:
```aidl
// Can you spot the "memory leak"?

public class Stack {

    private Object[] elements;

    private int size = 0;

    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        return elements[--size];

    }
    /**
     * Ensure space for at least one more element, roughly
     * doubling the capacity each time the array needs to grow.
     */

    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }

}
```
There’s nothing obviously wrong with this program (but see Item 29 for a generic version). You could test it exhaustively, and it would pass every test with flying colors, but there’s a problem lurking. Loosely speaking, the program has a “memory leak,” which can silently manifest itself as reduced performance due to increased garbage collector activity or increased memory footprint. In extreme cases, such memory leaks can cause disk paging and even program failure with an OutOfMemoryError, but such failures are relatively rare.

So where is the memory leak? If a stack grows and then shrinks, the objects that were popped off the stack will not be garbage collected, even if the program using the stack has no more references to them. This is because the stack maintains obsolete references to these objects. An obsolete reference is simply a reference that will never be dereferenced again. In this case, any references outside of the “active portion” of the element array are obsolete. The active portion consists of the elements whose index is less than size.

Memory leaks in garbage-collected languages (more properly known as unintentional object retentions) are insidious. If an object reference is unintentionally retained, not only is that object excluded from garbage collection, but so too are any objects referenced by that object, and so on. Even if only a few object references are unintentionally retained, many, many objects may be prevented from being garbage collected, with potentially large effects on performance.

The fix for this sort of problem is simple: null out references once they become obsolete. In the case of our Stack class, the reference to an item becomes obsolete as soon as it’s popped off the stack. The corrected version of the pop method looks like this:

```aidl
public Object pop() {
    if (size == 0)
        throw new EmptyStackException();

    Object result = elements[--size];
    elements[size] = null; // Eliminate obsolete reference
    return result;
}
```

An added benefit of nulling out obsolete references is that if they are subsequently dereferenced by mistake, the program will immediately fail with a NullPointerException, rather than quietly doing the wrong thing. It is always beneficial to detect programming errors as quickly as possible.

When programmers are first stung by this problem, they may overcompensate by nulling out every object reference as soon as the program is finished using it. This is neither necessary nor desirable; it clutters up the program unnecessarily. Nulling out object references should be the exception rather than the norm. The best way to eliminate an obsolete reference is to let the variable that contained the reference fall out of scope. This occurs naturally if you define each variable in the narrowest possible scope (Item 57).

So when should you null out a reference? What aspect of the Stack class makes it susceptible to memory leaks? Simply put, it manages its own memory. The storage pool consists of the elements of the elements array (the object reference cells, not the objects themselves). The elements in the active portion of the array (as defined earlier) are allocated, and those in the remainder of the array are free. The garbage collector has no way of knowing this; to the garbage collector, all of the object references in the elements array are equally valid. Only the programmer knows that the inactive portion of the array is unimportant. The programmer effectively communicates this fact to the garbage collector by manually nulling out array elements as soon as they become part of the inactive portion.

Generally speaking, whenever a class manages its own memory, the programmer should be alert for memory leaks. Whenever an element is freed, any object references contained in the element should be nulled out.

### Another common source of memory leaks is caches. Once you put an object reference into a cache, it’s easy to forget that it’s there and leave it in the cache long after it becomes irrelevant. There are several solutions to this problem. If you’re lucky enough to implement a cache for which an entry is relevant exactly so long as there are references to its key outside of the cache, represent the cache as a WeakHashMap; entries will be removed automatically after they become obsolete. Remember that WeakHashMap is useful only if the desired lifetime of cache entries is determined by external references to the key, not the value.

More commonly, the useful lifetime of a cache entry is less well defined, with entries becoming less valuable over time. Under these circumstances, the cache should occasionally be cleansed of entries that have fallen into disuse. This can be done by a background thread (perhaps a ScheduledThreadPoolExecutor) or as a side effect of adding new entries to the cache. The LinkedHashMap class facilitates the latter approach with its removeEldestEntry method. For more sophisticated caches, you may need to use java.lang.ref directly.

A third common source of memory leaks is listeners and other callbacks. If you implement an API where clients register callbacks but don’t deregister them explicitly, they will accumulate unless you take some action. One way to ensure that callbacks are garbage collected promptly is to store only weak references to them, for instance, by storing them only as keys in a WeakHashMap.

In summary, because memory leaks typically do not manifest themselves as obvious failures, they may remain present in a system for years. They are typically discovered only as a result of careful code inspection or with the aid of a debugging tool known as a heap profiler. Therefore, it is very desirable to learn to anticipate problems like this before they occur and prevent them from happening.

## ITEM 8: AVOID FINALIZERS AND CLEANERS

Finalizers are unpredictable, often dangerous, and generally unnecessary. Their use can cause erratic behavior, poor performance, and portability problems. Finalizers have a few valid uses, which we’ll cover later in this item, but as a rule, you should avoid them. As of Java 9, finalizers have been deprecated, but they are still being used by the Java libraries. The Java 9 replacement for finalizers is cleaners. Cleaners are less dangerous than finalizers, but still unpredictable, slow, and generally unnecessary.

C++ programmers are cautioned not to think of finalizers or cleaners as Java’s analogue of C++ destructors. In C++, destructors are the normal way to reclaim the resources associated with an object, a necessary counterpart to constructors. In Java, the garbage collector reclaims the storage associated with an object when it becomes unreachable, requiring no special effort on the part of the programmer. C++ destructors are also used to reclaim other nonmemory resources. In Java, a try-with-resources or try-finally block is used for this purpose (Item 9).

One shortcoming of finalizers and cleaners is that there is no guarantee they’ll be executed promptly [JLS, 12.6]. It can take arbitrarily long between the time that an object becomes unreachable and the time its finalizer or cleaner runs. This means that you should never do anything time-critical in a finalizer or cleaner. For example, it is a grave error to depend on a finalizer or cleaner to close files because open file descriptors are a limited resource. If many files are left open as a result of the system’s tardiness in running finalizers or cleaners, a program may fail because it can no longer open files.

The promptness with which finalizers and cleaners are executed is primarily a function of the garbage collection algorithm, which varies widely across implementations. The behavior of a program that depends on the promptness of finalizer or cleaner execution may likewise vary. It is entirely possible that such a program will run perfectly on the JVM on which you test it and then fail miserably on the one favored by your most important customer.

Tardy finalization is not just a theoretical problem. Providing a finalizer for a class can arbitrarily delay reclamation of its instances. A colleague debugged a long-running GUI application that was mysteriously dying with an OutOfMemoryError. Analysis revealed that at the time of its death, the application had thousands of graphics objects on its finalizer queue just waiting to be finalized and reclaimed. Unfortunately, the finalizer thread was running at a lower priority than another application thread, so objects weren’t getting finalized at the rate they became eligible for finalization. The language specification makes no guarantees as to which thread will execute finalizers, so there is no portable way to prevent this sort of problem other than to refrain from using finalizers. Cleaners are a bit better than finalizers in this regard because class authors have control over their own cleaner threads, but cleaners still run in the background, under the control of the garbage collector, so there can be no guarantee of prompt cleaning.

Not only does the specification provide no guarantee that finalizers or cleaners will run promptly; it provides no guarantee that they’ll run at all. It is entirely possible, even likely, that a program terminates without running them on some objects that are no longer reachable. As a consequence, you should never depend on a finalizer or cleaner to update persistent state. For example, depending on a finalizer or cleaner to release a persistent lock on a shared resource such as a database is a good way to bring your entire distributed system to a grinding halt.

Don’t be seduced by the methods System.gc and System.runFinalization. They may increase the odds of finalizers or cleaners getting executed, but they don’t guarantee it. Two methods once claimed to make this guarantee: System.runFinalizersOnExit and its evil twin, Runtime.runFinalizersOnExit. These methods are fatally flawed and have been deprecated for decades [ThreadStop].

Another problem with finalizers is that an uncaught exception thrown during finalization is ignored, and finalization of that object terminates [JLS, 12.6]. Uncaught exceptions can leave other objects in a corrupt state. If another thread attempts to use such a corrupted object, arbitrary nondeterministic behavior may result. Normally, an uncaught exception will terminate the thread and print a stack trace, but not if it occurs in a finalizer—it won’t even print a warning. Cleaners do not have this problem because a library using a cleaner has control over its thread.

There is a severe performance penalty for using finalizers and cleaners. On my machine, the time to create a simple AutoCloseable object, to close it using try-with-resources, and to have the garbage collector reclaim it is about 12 ns. Using a finalizer instead increases the time to 550 ns. In other words, it is about 50 times slower to create and destroy objects with finalizers. This is primarily because finalizers inhibit efficient garbage collection. Cleaners are comparable in speed to finalizers if you use them to clean all instances of the class (about 500 ns per instance on my machine), but cleaners are much faster if you use them only as a safety net, as discussed below. Under these circumstances, creating, cleaning, and destroying an object takes about 66 ns on my machine, which means you pay a factor of five (not fifty) for the insurance of a safety net if you don’t use it.

Finalizers have a serious security problem: they open your class up to finalizer attacks. The idea behind a finalizer attack is simple: If an exception is thrown from a constructor or its serialization equivalents—the readObject and readResolve methods (Chapter 12)—the finalizer of a malicious subclass can run on the partially constructed object that should have “died on the vine.” This finalizer can record a reference to the object in a static field, preventing it from being garbage collected. Once the malformed object has been recorded, it is a simple matter to invoke arbitrary methods on this object that should never have been allowed to exist in the first place. Throwing an exception from a constructor should be sufficient to prevent an object from coming into existence; in the presence of finalizers, it is not. Such attacks can have dire consequences. Final classes are immune to finalizer attacks because no one can write a malicious subclass of a final class. To protect nonfinal classes from finalizer attacks, write a final finalize method that does nothing.

So what should you do instead of writing a finalizer or cleaner for a class whose objects encapsulate resources that require termination, such as files or threads? Just have your class implement AutoCloseable, and require its clients to invoke the close method on each instance when it is no longer needed, typically using try-with-resources to ensure termination even in the face of exceptions (Item 9). One detail worth mentioning is that the instance must keep track of whether it has been closed: the close method must record in a field that the object is no longer valid, and other methods must check this field and throw an IllegalStateException if they are called after the object has been closed.

So what, if anything, are cleaners and finalizers good for? They have perhaps two legitimate uses. One is to act as a safety net in case the owner of a resource neglects to call its close method. While there’s no guarantee that the cleaner or finalizer will run promptly (or at all), it is better to free the resource late than never if the client fails to do so. If you’re considering writing such a safety-net finalizer, think long and hard about whether the protection is worth the cost. Some Java library classes, such as FileInputStream, FileOutputStream, ThreadPoolExecutor, and java.sql.Connection, have finalizers that serve as safety nets.

A second legitimate use of cleaners concerns objects with native peers. A native peer is a native (non-Java) object to which a normal object delegates via native methods. Because a native peer is not a normal object, the garbage collector doesn’t know about it and can’t reclaim it when its Java peer is reclaimed. A cleaner or finalizer may be an appropriate vehicle for this task, assuming the performance is acceptable and the native peer holds no critical resources. If the performance is unacceptable or the native peer holds resources that must be reclaimed promptly, the class should have a close method, as described earlier.

Cleaners are a bit tricky to use. Below is a simple Room class demonstrating the facility. Let’s assume that rooms must be cleaned before they are reclaimed. The Room class implements AutoCloseable; the fact that its automatic cleaning safety net uses a cleaner is merely an implementation detail. Unlike finalizers, cleaners do not pollute a class’s public API:

```aidl
// An autocloseable class using a cleaner as a safety net

public class Room implements AutoCloseable {

    private static final Cleaner cleaner = Cleaner.create();
    // Resource that requires cleaning. Must not refer to Room!
    private static class State implements Runnable {

        int numJunkPiles; // Number of junk piles in this room

        State(int numJunkPiles) {

            this.numJunkPiles = numJunkPiles;

        }

        // Invoked by close method or cleaner

        @Override public void run() {

            System.out.println("Cleaning room");

            numJunkPiles = 0;

        }

    }

    // The state of this room, shared with our cleanable

    private final State state;

    // Our cleanable. Cleans the room when it’s eligible for gc

    private final Cleaner.Cleanable cleanable;

    public Room(int numJunkPiles) {

        state = new State(numJunkPiles);

        cleanable = cleaner.register(this, state);

    }

    @Override public void close() {

        cleanable.clean();

    }

}
```
The static nested State class holds the resources that are required by the cleaner to clean the room. In this case, it is simply the numJunkPiles field, which represents the amount of mess in the room. More realistically, it might be a final long that contains a pointer to a native peer. State implements Runnable, and its run method is called at most once, by the Cleanable that we get when we register our State instance with our cleaner in the Room constructor. The call to the run method will be triggered by one of two things: Usually it is triggered by a call to Room’s close method calling Cleanable’s clean method. If the client fails to call the close method by the time a Room instance is eligible for garbage collection, the cleaner will (hopefully) call State’s run method.

It is critical that a State instance does not refer to its Room instance. If it did, it would create a circularity that would prevent the Room instance from becoming eligible for garbage collection (and from being automatically cleaned). Therefore, State must be a static nested class because nonstatic nested classes contain references to their enclosing instances (Item 24). It is similarly inadvisable to use a lambda because they can easily capture references to enclosing objects.

As we said earlier, Room’s cleaner is used only as a safety net. If clients surround all Room instantiations in try-with-resource blocks, automatic cleaning will never be required. This well-behaved client demonstrates that behavior:

```aidl
public class Adult {
    public static void main(String[] args) {
        try (Room myRoom = new Room(7)) {
            System.out.println("Goodbye");
        }
    }
}
```
As you’d expect, running the Adult program prints Goodbye, followed by Cleaning room. But what about this ill-behaved program, which never cleans its room?

```aidl
public class Teenager {

    public static void main(String[] args) {
        new Room(99);
        System.out.println("Peace out");
    }
}
```
You might expect it to print Peace out, followed by Cleaning room, but on my machine, it never prints Cleaning room; it just exits. This is the unpredictability we spoke of earlier. The Cleaner spec says, “The behavior of cleaners during System.exit is implementation specific. No guarantees are made relating to whether cleaning actions are invoked or not.” While the spec does not say it, the same holds true for normal program exit. On my machine, adding the line System.gc() to Teenager’s main method is enough to make it print Cleaning room prior to exit, but there’s no guarantee that you’ll see the same behavior on your machine.

In summary, don’t use cleaners, or in releases prior to Java 9, finalizers, except as a safety net or to terminate noncritical native resources. Even then, beware the indeterminacy and performance consequences.

## ITEM 9: PREFER TRY-WITH-RESOURCES TO TRY-FINALLY

The Java libraries include many resources that must be closed manually by invoking a close method. Examples include InputStream, OutputStream, and java.sql.Connection. Closing resources is often overlooked by clients, with predictably dire performance consequences. While many of these resources use finalizers as a safety net, finalizers don’t work very well (Item 8).

Historically, a try-finally statement was the best way to guarantee that a resource would be closed properly, even in the face of an exception or return:

```aidl
// try-finally - No longer the best way to close resources!

static String firstLineOfFile(String path) throws IOException {

    BufferedReader br = new BufferedReader(new FileReader(path));

    try {

        return br.readLine();

    } finally {

        br.close();

    }

}
```
This may not look bad, but it gets worse when you add a second resource:
```aidl
// try-finally is ugly when used with more than one resource!

static void copy(String src, String dst) throws IOException {
    InputStream in = new FileInputStream(src);
    try {
        OutputStream out = new FileOutputStream(dst);
        try {
            byte[] buf = new byte[BUFFER_SIZE];
            int n;
            while ((n = in.read(buf)) >= 0)
                out.write(buf, 0, n);
        } finally {
            out.close();
        }
    } finally {
        in.close();
    }
}
```

It may be hard to believe, but even good programmers got this wrong most of the time. For starters, I got it wrong on page 88 of Java Puzzlers [Bloch05], and no one noticed for years. In fact, two-thirds of the uses of the close method in the Java libraries were wrong in 2007.

Even the correct code for closing resources with try-finally statements, as illustrated in the previous two code examples, has a subtle deficiency. The code in both the try block and the finally block is capable of throwing exceptions. For example, in the firstLineOfFile method, the call to readLine could throw an exception due to a failure in the underlying physical device, and the call to close could then fail for the same reason. Under these circumstances, the second exception completely obliterates the first one. There is no record of the first exception in the exception stack trace, which can greatly complicate debugging in real systems—usually it’s the first exception that you want to see in order to diagnose the problem. While it is possible to write code to suppress the second exception in favor of the first, virtually no one did because it’s just too verbose.

All of these problems were solved in one fell swoop when Java 7 introduced the try-with-resources statement [JLS, 14.20.3]. To be usable with this construct, a resource must implement the AutoCloseable interface, which consists of a single void-returning close method. Many classes and interfaces in the Java libraries and in third-party libraries now implement or extend AutoCloseable. If you write a class that represents a resource that must be closed, your class should implement AutoCloseable too.

Here’s how our first example looks using try-with-resources:
```aidl
// try-with-resources - the the best way to close resources!

static String firstLineOfFile(String path) throws IOException {

    try (BufferedReader br = new BufferedReader(

           new FileReader(path))) {

       return br.readLine();

    }

}
```
And here’s how our second example looks using try-with-resources:
```aidl
// try-with-resources on multiple resources - short and sweet

static void copy(String src, String dst) throws IOException {

    try (InputStream   in = new FileInputStream(src);

         OutputStream out = new FileOutputStream(dst)) {

        byte[] buf = new byte[BUFFER_SIZE];

        int n;

        while ((n = in.read(buf)) >= 0)

            out.write(buf, 0, n);

    }

}
```
Not only are the try-with-resources versions shorter and more readable than the originals, but they provide far better diagnostics. Consider the firstLineOfFile method. If exceptions are thrown by both the readLine call and the (invisible) close, the latter exception is suppressed in favor of the former. In fact, multiple exceptions may be suppressed in order to preserve the exception that you actually want to see. These suppressed exceptions are not merely discarded; they are printed in the stack trace with a notation saying that they were suppressed. You can also access them programmatically with the getSuppressed method, which was added to Throwable in Java 7.

You can put catch clauses on try-with-resources statements, just as you can on regular try-finally statements. 
This allows you to handle exceptions without sullying your code with another layer of nesting. 
As a slightly contrived example, here’s a version our firstLineOfFile method that does not throw exceptions, 
but takes a default value to return if it can’t open the file or read from it:

```aidl
// try-with-resources with a catch clause

static String firstLineOfFile(String path, String defaultVal) {

    try (BufferedReader br = new BufferedReader(new FileReader(path))) {
        return br.readLine();
    } catch (IOException e) {
        return defaultVal;
    }

}
```
The lesson is clear: Always use try-with-resources in preference to try-finally when working with resources that must be closed. 
The resulting code is shorter and clearer, and the exceptions that it generates are more useful. 
The try-with-resources statement makes it easy to write correct code using resources that must be closed, which was practically impossible using try-finally.
