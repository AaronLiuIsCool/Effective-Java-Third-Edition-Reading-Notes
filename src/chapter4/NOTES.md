#Chapter 4. Classes and Interfaces

CLASSES and interfaces lie at the heart of the Java programming language. 
They are its basic units of abstraction. The language provides many powerful elements that 
you can use to design classes and interfaces. 
This chapter contains guidelines to help you make the best use of these elements so that your 
classes and interfaces are usable, robust, and flexible.

##ITEM 15: MINIMIZE THE ACCESSIBILITY OF CLASSES AND MEMBERS

The single most important factor that distinguishes a well-designed component from a poorly designed one is the degree 
to which the component hides its internal data and other implementation details from other components. 
A well-designed component hides all its implementation details, cleanly separating its API from its implementation. 
Components then communicate only through their APIs and are oblivious to each others’ inner workings. 
This concept, known as information hiding or encapsulation, is a fundamental tenet of software design.

Information hiding is important for many reasons, most of which stem from the fact that <b>it decouples the components that
comprise a system, allowing them to be developed, tested, optimized, used, understood, and modified in isolation. 
This speeds up system development because components can be developed in parallel.</b> It eases the burden of maintenance 
because components can be understood more quickly and debugged or replaced with little fear of harming other components. 
While information hiding does not, in and of itself, cause good performance, it enables effective performance tuning: 
once a system is complete and profiling has determined which components are causing performance problems (Item 67), 
those components can be optimized without affecting the correctness of others. Information hiding increases software 
reuse because components that aren’t tightly coupled often prove useful in other contexts besides the ones for which 
they were developed. Finally, information hiding decreases the risk in building large systems because individual 
components may prove successful even if the system does not.

Java has many facilities to aid in information hiding. The access control mechanism [JLS, 6.6] specifies the 
accessibility of classes, interfaces, and members. The accessibility of an entity is determined by the location of its 
declaration and by which, if any, of the access modifiers (private, protected, and public) is present on the declaration.
Proper use of these modifiers is essential to information hiding.

The rule of thumb is simple: make each class or member as inaccessible as possible. In other words, use the lowest
possible access level consistent with the proper functioning of the software that you are writing.

For top-level (non-nested) classes and interfaces, there are only two possible access levels: package-private and public.
If you declare a top-level class or interface with the public modifier, it will be public; otherwise, it will be 
package-private. If a top-level class or interface can be made package-private, it should be. By making it package-private,
you make it part of the implementation rather than the exported API, and you can modify it, replace it, or eliminate it 
in a subsequent release without fear of harming existing clients. <b>If you make it public, you are obligated to support it
forever to maintain compatibility.</b> 
<b>{Aaron's Notes: Normally tight coupling can easily start with a new rather than using abstract factory, factory pattern
or other design pattern. As if you work in a scrum teams and there are hundreds of internal of external scrum teams like 
this. The monolith eval tight coupling code structure built automatically. Each engineer is responsible for their codes
structure, then the system could be loose couple as much as possible.} </b>

If a package-private top-level class or interface is used by only one class, consider making the top-level class a private
static nested class of the sole class that uses it (Item 24). This reduces its accessibility from all the classes in its
package to the one class that uses it. But it is far more important to reduce the accessibility of a gratuitously public class than of a package-private top-level class: the public class is part of the package’s API, while the package-private top-level class is already part of its implementation.

For members (fields, methods, nested classes, and nested interfaces), there are four possible access levels, listed here
in order of increasing accessibility:

• <b>private—The member is accessible only from the top-level class where it is declared.</b>

• <b>package-private—The member is accessible from any class in the package where it is declared. 
Technically known as default access, this is the access level you get if no access modifier is specified 
(except for interface members, which are public by default). </b>

• <b>protected—The member is accessible from subclasses of the class where it is declared 
(subject to a few restrictions [JLS, 6.6.2]) and from any class in the package where it is declared.</b>

• <b>public—The member is accessible from anywhere.</b>

After carefully designing your class’s public API, your reflex should be to make all other members private. Only if
another class in the same package really needs to access a member should you remove the private modifier, making the 
member package-private. If you find yourself doing this often, you should reexamine the design of your system to see 
if another decomposition might yield classes that are better decoupled from one another. That said, both private and 
package-private members are part of a class’s implementation and do not normally impact its exported API. These fields 
can, however, “leak” into the exported API if the class implements Serializable (Items 86 and 87).

For members of public classes, a huge increase in accessibility occurs when the access level goes from package-private
to protected. A protected member is part of the class’s exported API and must be supported forever. Also, a protected
member of an exported class represents a public commitment to an implementation detail (Item 19). The need for protected
members should be relatively rare.

There is a key rule that restricts your ability to reduce the accessibility of methods. If a method overrides a superclass
method, it cannot have a more restrictive access level in the subclass than in the superclass [JLS, 8.4.8.3]. This is
necessary to ensure that an instance of the subclass is usable anywhere that an instance of the superclass is usable
(the Liskov substitution principle, see Item 15). If you violate this rule, the compiler will generate an error message
when you try to compile the subclass. A special case of this rule is that if a class implements an interface, all of the
class methods that are in the interface must be declared public in the class.

To facilitate testing your code, you may be tempted to make a class, interface, or member more accessible than otherwise
necessary. This is fine up to a point. <b>It is acceptable to make a private member of a public class package-private in
order to test it, but it is not acceptable to raise the accessibility any higher. In other words, it is not acceptable
to make a class, interface, or member a part of a pack-age’s exported API to facilitate testing. Luckily, it isn’t
necessary either because tests can be made to run as part of the package being tested, thus gaining access to its
package-private elements. </b>

Instance fields of public classes should rarely be public (Item 16). If an instance field is non final or is a reference
to a mutable object, then by making it public, you give up the ability to limit the values that can be stored in the
field. {Aaron's note: This is so dangerous!} This means you give up the ability to enforce invariants involving the field. 
Also, you give up the ability to take any action when the field is modified, so classes with public mutable fields are 
NOT generally thread-safe. Even if a field is final and refers to an immutable object, by making it public you give up 
the flexibility to switch to a new internal data representation in which the field does not exist.
{Aaron notes: Above is an important design.}

The same advice applies to static fields, with one exception. You can expose constants via public static final fields,
assuming the constants form an integral part of the abstraction provided by the class. By convention, such fields have
names consisting of capital letters, with words separated by underscores (Item 68). It is critical that these fields
contain either primitive values or references to immutable objects (Item 17). a field containing a reference to a mutable
object has all the disadvantages of a non-final field. While the reference cannot be modified, the referenced object can
be modified—with disastrous results.

Note that a nonzero-length array is always mutable, so it is wrong for a class to have a public static final array field, 
or an accessor that returns such a field. If a class has such a field or accessor, clients will be able to modify the 
contents of the array. This is a frequent source of security holes:

```
// Potential security hole!

public static final Thing[] VALUES = { ... };
```
Beware of the fact that some IDEs generate accessors that return references to private array fields, resulting in exactly
this problem. There are two ways to fix the problem. You can make the public array private and add a public immutable list:

```
private static final Thing[] PRIVATE_VALUES = { ... };

public static final List<Thing> VALUES =

   Collections.unmodifiableList(Arrays.asList(PRIVATE_VALUES));
```
Alternatively, you can make the array private and add a public method that returns a copy of a private array:

```
private static final Thing[] PRIVATE_VALUES = { ... };

public static final Thing[] values() {

    return PRIVATE_VALUES.clone();

}
```
To choose between these alternatives, think about what the client is likely to do with the result. Which return type 
will be more convenient? Which will give better performance?

<b>As of Java 9, there are two additional, implicit access levels introduced as part of the module system. A module is a
grouping of packages, like a package is a grouping of classes. A module may explicitly export some of its packages via
export declarations in its module declaration (which is by convention contained in a source file named module-info.java).
Public and protected members of unexported packages in a module are inaccessible outside the module; within the module,
accessibility is unaffected by export declarations. Using the module system allows you to share classes among packages
within a module without making them visible to the entire world. Public and protected members of public classes in
unexported packages give rise to the two implicit access levels, which are intramodular analogues of the normal public
and protected levels. The need for this kind of sharing is relatively rare and can often be eliminated by rearranging
the classes within your packages.</b>

Unlike the four main access levels, the two module-based levels are largely advisory. If you place a module’s JAR file
on your application’s class path instead of its module path, the packages in the module revert to their non-modular
behavior: all of the public and protected members of the packages’ public classes have their normal accessibility,
regardless of whether the packages are exported by the module [Reinhold, 1.2]. The one place where the newly introduced
access levels are strictly enforced is the JDK itself: the unexported packages in the Java libraries are truly
inaccessible outside of their modules.

Not only is the access protection afforded by modules of limited utility to the typical Java programmer, and largely
advisory in nature; in order to take advantage of it, you must group your packages into modules, make all of their
dependencies explicit in module declarations, rearrange your source tree, and take special actions to accommodate any
access to non-modularized packages from within your modules [Reinhold, 3]. It is too early to say whether modules will
achieve widespread use outside of the JDK itself. In the meantime, it seems best to avoid them unless you have a
compelling need.

To summarize, you should reduce accessibility of program elements as much as possible (within reason). After carefully
designing a minimal public API, you should prevent any stray classes, interfaces, or members from becoming part of the
API. With the exception of public static final fields, which serve as constants, public classes should have no
public fields. Ensure that objects referenced by public static final fields are immutable.
{Aaron notes: Above is an important design.} 

##ITEM 16: IN PUBLIC CLASSES, USE ACCESSOR METHODS, NOT PUBLIC FIELDS

Occasionally, you may be tempted to write degenerate classes that serve no purpose other than to group instance fields:

```
// Degenerate classes like this should not be public!

class Point {
    public double x;
    public double y;
}
```
Because the data fields of such classes are accessed directly, these classes do not offer the benefits of encapsulation 
(Item 15). You can’t change the representation without changing the API, you can’t enforce invariants, and you can’t take
auxiliary action when a field is accessed. Hard-line object-oriented programmers feel that such classes are anathema and
should always be replaced by classes with private fields and public accessor methods (getters) and, for mutable classes,
mutators (setters): {Aaron notes: a lot devs knowing to code like this, but don't know it's behind's reason.}

```
// Encapsulation of data by accessor methods and mutators

class Point {

    private double x;
    private double y;

    public Point(double x, double y) {
        this.x = x;
        this.y = y;
    }

    public double getX() { return x; }
    public double getY() { return y; }

    public void setX(double x) { this.x = x; }
    public void setY(double y) { this.y = y; }

}
```
Certainly, the hard-liners are correct when it comes to public classes: if a class is accessible outside its package,
provide accessor methods to preserve the flexibility to change the class’s internal representation. If a public class
exposes its data fields, all hope of changing its representation is lost because client code can be distributed far and
wide.

However, if a class is package-private or is a private nested class, there is nothing inherently wrong with exposing its
data fields—assuming they do an adequate job of describing the abstraction provided by the class. This approach generates
less visual clutter than the accessor-method approach, both in the class definition and in the client code that uses it.
While the client code is tied to the class’s internal representation, this code is confined to the package containing the
class. If a change in representation becomes desirable, you can make the change without touching any code outside the
package. In the case of a private nested class, the scope of the change is further restricted to the enclosing class.

Several classes in the Java platform libraries violate the advice that public classes should not expose fields directly. 
Prominent examples include the Point and Dimension classes in the java.awt package. Rather than examples to be emulated,
these classes should be regarded as cautionary tales. As described in Item 67, the decision to expose the internals of
the Dimension class resulted in a serious performance problem that is still with us today.

While it’s never a good idea for a public class to expose fields directly, it is less harmful if the fields are immutable.
You can’t change the representation of such a class without changing its API, and you can’t take auxiliary actions when
a field is read, but you can enforce invariants. For example, this class guarantees that each instance represents a valid
time:

```
// Public class with exposed immutable fields - questionable
public final class Time {
    private static final int HOURS_PER_DAY    = 24;
    private static final int MINUTES_PER_HOUR = 60;
    
    public final int hour;
    public final int minute;

    public Time(int hour, int minute) {
        if (hour < 0 || hour >= HOURS_PER_DAY)
           throw new IllegalArgumentException("Hour: " + hour);
        if (minute < 0 || minute >= MINUTES_PER_HOUR)
           throw new IllegalArgumentException("Min: " + minute);
        this.hour = hour;
        this.minute = minute;
    }
    ... // Remainder omitted

}
```
In summary, public classes should never expose mutable fields. It is less harmful, though still questionable, for public classes to expose immutable fields. It is, however, sometimes desirable for package-private or private nested classes to expose fields, whether mutable or immutable.


##ITEM 17: MINIMIZE MUTABILITY
An immutable class is simply a class whose instances cannot be modified. All of the information contained in each 
instance is fixed for the lifetime of the object, so no changes can ever be observed. The Java platform libraries
contain many immutable classes, including String, the boxed primitive classes, and BigInteger and BigDecimal. There are
many good reasons for this: 

### Immutable classes are easier to design, implement, and use than mutable classes. They are less prone to error and are more secure.

To make a class immutable, follow these five rules:
1. Don’t provide methods that modify the object’s state (known as mutators).
2. Ensure that the class can’t be extended. This prevents careless or malicious subclasses from compromising the 
immutable behavior of the class by behaving as if the object’s state has changed. Preventing subclassing is generally 
accomplished by making the class final, but there is an alternative that we’ll discuss later.
3. Make all fields final. This clearly expresses your intent in a manner that is enforced by the system. Also, it is 
necessary to ensure correct behavior if a reference to a newly created instance is passed from one thread to another 
without synchronization, as spelled out in the memory model 
4. Make all fields private. This prevents clients from obtaining access to mutable objects referred to by fields and 
modifying these objects directly. While it is technically permissible for immutable classes to have public final fields
containing primitive values or references to immutable objects, it is not recommended because it precludes changing the
internal representation in a later release (Items 15 and 16).
5. Ensure exclusive access to any mutable components. If your class has any fields that refer to mutable objects, ensure
that clients of the class cannot obtain references to these objects. Never initialize such a field to a client-provided
object reference or return the field from an accessor. Make defensive copies (Item 50) in constructors, accessors, and
readObject methods (Item 88).

Many of the example classes in previous items are immutable. One such class is PhoneNumber in Item 11, which has
accessors for each attribute but no corresponding mutators. Here is a slightly more complex example:
{Aaron notes: Above is an important design.}

```
// Immutable complex number class
public final  class Complex {
    private final  double re;
    private final  double im;

    public Complex(double re, double im) {
        this.re = re;
        this.im = im;

    }
    public double realPart()      { return re; }
    public double imaginaryPart() { return im; }
    public Complex plus(Complex c) {
        return new Complex(re + c.re, im + c.im);

    }
    
    public Complex minus(Complex c) {
        return new Complex(re - c.re, im - c.im);
    }
    
    public Complex times(Complex c) {
        return new Complex(re * c.re - im * c.im,
                           re * c.im + im * c.re);
    }

    public Complex dividedBy(Complex c) {

        double tmp = c.re * c.re + c.im * c.im;
        return new Complex((re * c.re + im * c.im) / tmp,
                           (im * c.re - re * c.im) / tmp);
    }

    @Override public boolean equals(Object o) {
       if (o == this)
           return true;
       if (!(o instanceof Complex))
           return false;
           
       Complex c = (Complex) o;
       // See page 47 to find out why we use compare instead of ==
       return Double.compare(c.re, re) == 0
           && Double.compare(c.im, im) == 0;
    }
    @Override public int hashCode() {
        return 31 * Double.hashCode(re) + Double.hashCode(im);

    }

    @Override public String toString() {
        return "(" + re + " + " + im + "i)";
    }
}
```
This class represents a complex number (a number with both real and imaginary parts). In addition to the standard Object
methods, it provides accessors for the real and imaginary parts and provides the four basic arithmetic operations:
addition, subtraction, multiplication, and division. Notice how the arithmetic operations create and return a new Complex
instance rather than modifying this instance. This pattern is known as the functional approach because methods return
the result of applying a function to their operand, without modifying it. Contrast it to the procedural or imperative
approach in which methods apply a procedure to their operand, causing its state to change. Note that the method names
are prepositions (such as plus) rather than verbs (such as add). This emphasizes the fact that methods don’t change the
values of the objects. <b>The BigInteger and BigDecimal classes did not obey this naming convention, and it led to many
usage errors.</b>

The functional approach may appear unnatural if you’re not familiar with it, but it enables immutability, which has many
advantages. Immutable objects are simple. An immutable object can be in exactly one state, the state in which it was created. If you make sure that all constructors establish class invariants, then it is guaranteed that these invariants will remain true for all time, with no further effort on your part or on the part of the programmer who uses the class. Mutable objects, on the other hand, can have arbitrarily complex state spaces. If the documentation does not provide a precise description of the state transitions performed by mutator methods, it can be difficult or impossible to use a mutable class reliably.

Immutable objects are inherently thread-safe; they require no synchronization. They cannot be corrupted by multiple
threads accessing them concurrently. This is far and away the easiest approach to achieve thread safety. Since no thread
can ever observe any effect of another thread on an immutable object, immutable objects can be shared freely.
Immutable classes should therefore encourage clients to reuse existing instances wherever possible. One easy way to do
this is to provide public static final constants for commonly used values. For example, the Complex class might provide
these constants:
```
public static final Complex ZERO = new Complex(0, 0);
public static final Complex ONE  = new Complex(1, 0);
public static final Complex I    = new Complex(0, 1);
```
This approach can be taken one step further. An immutable class can provide static factories (Item 1) that cache 
frequently requested instances to avoid creating new instances when existing ones would do. All the boxed primitive 
classes and BigInteger do this. Using such static factories causes clients to share instances instead of creating new 
ones, reducing memory footprint and garbage collection costs. Opting for static factories in place of public constructors
when designing a new class gives you the flexibility to add caching later, without modifying clients.
{Aaron notes: Above is an important design.}

A consequence of the fact that immutable objects can be shared freely is that you never have to make defensive copies of
them (Item 50). In fact, you never have to make any copies at all because the copies would be forever equivalent to the
originals. Therefore, you need not and should not provide a clone method or copy constructor (Item 13) on an
immutable class. This was not well understood in the early days of the Java platform, so the String class does have a
copy constructor, but it should rarely, if ever, be used (Item 6).

Not only can you share immutable objects, but they can share their internals. For example, the BigInteger class uses a
sign-magnitude representation internally. The sign is represented by an int, and the magnitude is represented by an int
array. The negate method produces a new BigInteger of like magnitude and opposite sign. It does not need to copy the
array even though it is mutable; the newly created BigInteger points to the same internal array as the original.

Immutable objects make great building blocks for other objects, whether mutable or immutable. It’s much easier to
maintain the invariants of a complex object if you know that its component objects will not change underneath it.
A special case of this principle is that immutable objects make great map keys and set elements: you don’t have to worry
about their values changing once they’re in the map or set, which would destroy the map or set’s invariants.

Immutable objects provide failure atomicity for free (Item 76). Their state never changes, so there is no possibility of
a temporary inconsistency.

The major disadvantage of immutable classes is that they require a separate object for each distinct value. Creating
these objects can be costly, especially if they are large. For example, suppose that you have a million-bit BigInteger
and you want to change its low-order bit:
```
BigInteger moby = ...;
moby = moby.flipBit(0);
```
The flipBit method creates a new BigInteger instance, also a million bits long, that differs from the original in only
one bit. The operation requires time and space proportional to the size of the BigInteger. 
Contrast this to java.util.BitSet. Like BigInteger, BitSet represents an arbitrarily long sequence of bits, but unlike
BigInteger, BitSet is mutable. The BitSet class provides a method that allows you to change the state of a single bit of
a million-bit instance in constant time:
```aidl
BitSet moby = ...;
moby.flip(0);
```
The performance problem is magnified if you perform a multi-step operation that generates a new object at every step, 
eventually discarding all objects except the final result. There are two approaches to coping with this problem. The 
first is to guess which multi-step operations will be commonly required and to provide them as primitives. If a multi-step
operation is provided as a primitive, the immutable class does not have to create a separate object at each step. 
Internally, the immutable class can be arbitrarily clever. For example, BigInteger has a package-private mutable 
“companion class” that it uses to speed up multistep operations such as modular exponentiation. It is much harder to use
the mutable companion class than to use BigInteger, for all of the reasons outlined earlier. Luckily, you don’t have to
use it: the implementors of BigInteger did the hard work for you.
{Aaron notes: Above is an important design.}

The package-private mutable companion class approach works fine if you can accurately predict which complex operations
clients will want to perform on your immutable class. If not, then your best bet is to provide a public mutable companion
class. The main example of this approach in the Java platform libraries is the String class, whose mutable companion is
StringBuilder (and its obsolete predecessor, StringBuffer).

Now that you know how to make an immutable class and you understand the pros and cons of immutability, let’s discuss a
few design alternatives. Recall that to guarantee immutability, a class must not permit itself to be subclassed. This
can be done by making the class final, but there is another, more flexible alternative. Instead of making an immutable
class final, you can make all of its constructors private or package-private and add public static factories in place of
the public constructors (Item 1). To make this concrete, here’s how Complex would look if you took this approach:

```
// Immutable class with static factories instead of constructors

public class Complex {
    private final double re;
    private final double im;

    private Complex(double re, double im) {
        this.re = re;
        this.im = im;

    }

    public static Complex valueOf(double re, double im) {
        return new Complex(re, im);
    }
    ... // Remainder unchanged

}
```
{Aaron notes: Above is an important design.}
This approach is often the best alternative. It is the most flexible because it allows the use of multiple package-private
implementation classes. To its clients that reside outside its package, the immutable class is effectively final because
it is impossible to extend a class that comes from another package and that lacks a public or protected constructor.
Besides allowing the flexibility of multiple implementation classes, this approach makes it possible to tune the
performance of the class in subsequent releases by improving the object-caching capabilities of the static factories.

It was not widely understood that immutable classes had to be effectively final when BigInteger and BigDecimal were written,
so all of their methods may be overridden. Unfortunately, this could not be corrected after the fact while preserving
backward compatibility.{Aaron notes: LOL.} If you write a class whose security depends on the immutability of a BigInteger
or BigDecimal argument from an untrusted client, you must check to see that the argument is a “real” BigInteger or BigDecimal,
rather than an instance of an untrusted subclass. If it is the latter, you must defensively copy it under the assumption
that it might be mutable (Item 50):

```
public static BigInteger safeInstance(BigInteger val) {
    return val.getClass() == BigInteger.class ?
            val : new BigInteger(val.toByteArray());

}
```
The list of rules for immutable classes at the beginning of this item says that no methods may modify the object and
that all its fields must be final. In fact these rules are a bit stronger than necessary and can be relaxed to improve
performance. In truth, no method may produce an externally visible change in the object’s state. However, some immutable
classes have one or more nonfinal fields in which they cache the results of expensive computations the first time they
are needed. If the same value is requested again, the cached value is returned, saving the cost of recalculation. This
trick works precisely because the object is immutable, which guarantees that the computation would yield the same result
if it were repeated.

For example, PhoneNumber’s hashCode method (Item 11, page 53) computes the hash code the first time it’s invoked and
caches it in case it’s invoked again. This technique, an example of lazy initialization (Item 83), is also used by String.
{Aaron notes: Above is an important design.}

One caveat should be added concerning serializability. If you choose to have your immutable class implement Serializable
and it contains one or more fields that refer to mutable objects, you must provide an explicit readObject or readResolve
method, or use the ObjectOutputStream.writeUnshared and ObjectInputStream.readUnshared methods, even if the default 
serialized form is acceptable. Otherwise an attacker could create a mutable instance of your class. This topic is covered
in detail in Item 88.
{Aaron notes: Above is an important design.}

To summarize, resist the urge to write a setter for every getter. Classes should be immutable unless there’s a very good
reason to make them mutable. Immutable classes provide many advantages, and their only disadvantage is the potential for
performance problems under certain circumstances. You should always make small value objects, such as PhoneNumber and
Complex, immutable. 

(There are several classes in the Java platform libraries, such as java.util.Date and java.awt.Point, that should have 
been immutable but aren’t.) You should seriously consider making larger value objects, such as String and BigInteger,
immutable as well. You should provide a public mutable companion class for your immutable class only once you’ve
confirmed that it’s necessary to achieve satisfactory performance (Item 67).
{Aaron notes: Above is an important design.}

There are some classes for which immutability is impractical. If a class cannot be made immutable, limit its mutability
as much as possible. Reducing the number of states in which an object can exist makes it easier to reason about the
object and reduces the likelihood of errors. Therefore, make every field final unless there is a compelling reason to
make it non-final. Combining the advice of this item with that of Item 15, your natural inclination should be to declare
every field private final unless there’s a good reason to do otherwise.

Constructors should create fully initialized objects with all of their invariants established. Don’t provide a public
initialization method separate from the constructor or static factory unless there is a compelling reason to do so.
Similarly, don’t provide a “reinitialize” method that enables an object to be reused as if it had been constructed with
a different initial state. Such methods generally provide little if any performance benefit at the expense of increased
complexity.

The CountDownLatch class exemplifies these principles. It is mutable, but its state space is kept intentionally small.
You create an instance, use it once, and it’s done: once the countdown latch’s count has reached zero, you may not
reuse it.

A final note should be added concerning the Complex class in this item. This example was meant only to illustrate
immutability. It is not an industrial-strength complex number implementation. It uses the standard formulas for complex
multiplication and division, which are not correctly rounded and provide poor semantics for complex NaNs and infinities 
[Kahan91, Smith62, Thomas94].

##ITEM 18: FAVOR COMPOSITION OVER INHERITANCE

##ITEM 19: DESIGN AND DOCUMENT FOR INHERITANCE OR ELSE PROHIBIT IT

##ITEM 20: PREFER INTERFACES TO ABSTRACT CLASSES

##ITEM 21: DESIGN INTERFACES FOR POSTERITY

##ITEM 22: USE INTERFACES ONLY TO DEFINE TYPES

##ITEM 23: PREFER CLASS HIERARCHIES TO TAGGED CLASSES

##ITEM 24: FAVOR STATIC MEMBER CLASSES OVER NONSTATIC

##ITEM 25: LIMIT SOURCE FILES TO A SINGLE TOP-LEVEL CLASS

