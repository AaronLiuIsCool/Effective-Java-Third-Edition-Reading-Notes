#Chapter 5. Generics

SINCE Java 5, generics have been a part of the language. Before generics, you had to cast every object you read from a 
collection. If someone accidentally inserted an object of the wrong type, casts could fail at runtime. With generics, 
you tell the compiler what types of objects are permitted in each collection. The compiler inserts casts for you 
automatically and tells you at compile time if you try to insert an object of the wrong type. This results in programs 
that are both safer and clearer, but these benefits, which are not limited to collections, come at a price. This chapter 
tells you how to maximize the benefits and minimize the complications.

## ITEM 26: DON’T USE RAW TYPES
First, a few terms. A class or interface whose declaration has one or more type parameters is a generic class or interface [JLS, 8.1.2, 9.1.2]. 
For example, the List interface has a single type parameter, E, representing its element type. The full name of the 
interface is List<E> (read “list of E”), but people often call it List for short. Generic classes and interfaces are 
collectively known as generic types.

Each generic type defines a set of parameterized types, which consist of the class or interface name followed by an 
angle-bracketed list of actual type parameters corresponding to the generic type’s formal type parameters 
[JLS, 4.4, 4.5]. For example, List<String> (read “list of string”) is a parameterized type representing a list whose 
elements are of type String. (String is the actual type parameter corresponding to the formal type parameter E.)

Finally, each generic type defines a raw type, which is the name of the generic type used without any accompanying type 
parameters [JLS, 4.8]. For example, the raw type corresponding to List<E> is List. Raw types behave as if all of the 
generic type information were erased from the type declaration. They exist primarily for compatibility with pre-generics 
code.

Before generics were added to Java, this would have been an exemplary collection declaration. As of Java 9, it is still 
legal, but far from exemplary:
```aidl
// Raw collection type - don't do this!
// My stamp collection. Contains only Stamp instances.
private final Collection stamps = ... ;
```

If you use this declaration today and then accidentally put a coin into your stamp collection, the erroneous insertion 
compiles and runs without error (though the compiler does emit a vague warning):
```aidl
// Erroneous insertion of coin into stamp collection
stamps.add(new Coin( ... )); // Emits "unchecked call" warning
```
You don’t get an error until you try to retrieve the coin from the stamp collection:
```aidl
// Raw iterator type - don't do this!
for (Iterator i = stamps.iterator(); i.hasNext(); )
    Stamp stamp = (Stamp) i.next(); // Throws ClassCastException
        stamp.cancel();
```

As mentioned throughout this book, it pays to discover errors as soon as possible after they are made, ideally at compile 
time. In this case, you don’t discover the error until runtime, long after it has happened, and in code that may be 
distant from the code containing the error. Once you see the ClassCastException, you have to search through the codebase 
looking for the method invocation that put the coin into the stamp collection. The compiler can’t help you, because it 
can’t understand the comment that says, “Contains only Stamp instances.”

With generics, the type declaration contains the information, not the comment:
```aidl
// Parameterized collection type - typesafe
private final Collection<Stamp> stamps = ... ;
```

From this declaration, the compiler knows that stamps should contain only Stamp instances and guarantees it to be true, 
assuming your entire codebase compiles without emitting (or suppressing; see Item 27) any warnings. When stamps is 
declared with a parameterized type declaration, the erroneous insertion generates a compile-time error message that 
tells you exactly what is wrong:
```aidl
Test.java:9: error: incompatible types: Coin cannot be converted
to Stamp
    c.add(new Coin());
```
{Aaron notes: Above is an important design.}

The compiler inserts invisible casts for you when retrieving elements from collections and guarantees that they won’t 
fail (assuming, again, that all of your code did not generate or suppress any compiler warnings). While the prospect of 
accidentally inserting a coin into a stamp collection may appear far-fetched, the problem is real. For example, it is 
easy to imagine putting a BigInteger into a collection that is supposed to contain only BigDecimal instances.

As noted earlier, it is legal to use raw types (generic types without their type parameters), 
but you should never do it. {Aaron notes: Above is an important design.}
<b>If you use raw types, you lose all the safety and expressiveness benefits of generics. </b> Given that you shouldn’t use them, 
why did the language designers permit raw types in the first place? For compatibility. Java was about to enter its second 
decade when generics were added, and there was an enormous amount of code in existence that did not use generics. It was 
deemed critical that all of this code remain legal and interoperate with newer code that does use generics. It had to be 
legal to pass instances of parameterized types to methods that were designed for use with raw types, and vice versa. This 
requirement, known as migration compatibility, drove the decisions to support raw types and to implement generics using 
erasure (Item 28).

While you shouldn’t use raw types such as List, it is fine to use types that are parameterized to allow insertion of 
arbitrary objects, such as List<Object>. Just what is the difference between the raw type List and the parameterized 
type List<Object>? Loosely speaking, the former has opted out of the generic type system, while the latter has explicitly 
told the compiler that it is capable of holding objects of any type. While you can pass a List<String> to a parameter of 
type List, you can’t pass it to a parameter of type List<Object>. There are sub-typing rules for generics, and 
List<String> is a subtype of the raw type List, but not of the parameterized type List<Object> (Item 28). As a 
consequence, you lose type safety if you use a raw type such as List, but not if you use a parameterized type such as
List<Object>.

To make this concrete, consider the following program:
```aidl
// Fails at runtime - unsafeAdd method uses a raw type (List)!
public static void main(String[] args) {
    List<String> strings = new ArrayList<>();
    unsafeAdd(strings, Integer.valueOf(42));
    String s = strings.get(0); // Has compiler-generated cast
}

private static void unsafeAdd(List list, Object o) {
    list.add(o);
}
```
This program compiles, but because it uses the raw type List, you get a warning:
```aidl
Test.java:10: warning: [unchecked] unchecked call to add(E) as a
member of the raw type List
    list.add(o);
```

And indeed, if you run the program, you get a ClassCastException when the program tries to cast the result of the 
invocation strings.get(0), which is an Integer, to a String. This is a compiler-generated cast, so it’s normally 
guaranteed to succeed, but in this case we ignored a compiler warning and paid the price.
```aidl
Test.java:5: error: incompatible types: List<String> cannot be
converted to List<Object>
    unsafeAdd(strings, Integer.valueOf(42));
```
{Aaron notes: Above is an important design.}

You might be tempted to use a raw type for a collection whose element type is unknown and doesn’t matter. For example, 
suppose you want to write a method that takes two sets and returns the number of elements they have in common. Here’s 
how you might write such a method if you were new to generics:
```aidl
// Use of raw type for unknown element type - don't do this!
static int numElementsInCommon(Set s1, Set s2) {
    int result = 0;
    for (Object o1 : s1)
        if (s2.contains(o1))
            result++;
    return result;
}
```
This method works but it uses raw types, which are dangerous. The safe alternative is to use unbounded wildcard types. 
If you want to use a generic type but you don’t know or care what the actual type parameter is, you can use a question 
mark instead. For example, the unbounded wildcard type for the generic type Set<E> is Set<?> (read “set of some type”).
{Aaron notes: Above is an important design.}
It is the most general parameterized Set type, capable of holding any set. Here is how the numElementsInCommon 
declaration looks with unbounded wildcard types:
```aidl
// Uses unbounded wildcard type - typesafe and flexible
static int numElementsInCommon(Set<?> s1, Set<?> s2) { ... }
```
{Aaron notes: Above is an important design.}

What is the difference between the unbounded wildcard type Set<?> and the raw type Set? Does the question mark really 
buy you anything? Not to belabor the point, but the wildcard type is safe and the raw type isn’t. You can put any element 
into a collection with a raw type, easily corrupting the collection’s type invariant 
(as demonstrated by the unsafeAdd method on page 119); you can’t put any element (other than null) into a Collection<?>. 
Attempting to do so will generate a compile-time error message like this:
```aidl
WildCard.java:13: error: incompatible types: String cannot be
converted to CAP#1
    c.add("verboten");
          ^
  where CAP#1 is a fresh type-variable:
    CAP#1 extends Object from capture of ?
```

Admittedly this error message leaves something to be desired, but the compiler has done its job, preventing you from 
corrupting the collection’s type invariant, whatever its element type may be. Not only can’t you put any element (other 
than null) into a Collection<?>, but you can’t assume anything about the type of the objects that you get out. If these 
restrictions are unacceptable, you can use generic methods (Item 30) or bounded wildcard types (Item 31).

There are a few minor exceptions to the rule that you should not use raw types. <b>You must use raw types in class literals.</b>
The specification does not permit the use of parameterized types (though it does permit array types and primitive types)
[JLS, 15.8.2]. In other words, List.class, String[].class, and int.class are all legal, but List<String>.class and 
List<?>.class are not.

A second exception to the rule concerns the instanceof operator. Because generic type information is erased at runtime, 
it is illegal to use the instanceof operator on parameterized types other than unbounded wildcard types. The use of 
unbounded wildcard types in place of raw types does not affect the behavior of the instanceof operator in any way. In 
this case, the angle brackets and question marks are just noise. <b>This is the preferred way to use the instanceof operator 
with generic types:</b>

```aidl
// Legitimate use of raw type - instanceof operator
if (o instanceof Set) {       // Raw type
    Set<?> s = (Set<?>) o;    // Wildcard type
    ...
}
```

### In summary, using raw types can lead to exceptions at runtime, so don’t use them. They are provided only for compatibility and interoperability with legacy code that predates the introduction of generics. As a quick review, Set<Object> is a parameterized type representing a set that can contain objects of any type, Set<?> is a wildcard type representing a set that can contain only objects of some unknown type, and Set is a raw type, which opts out of the generic type system. The first two are safe, and the last is not.

For quick reference, the terms introduced in this item (and a few introduced later in this chapter) are summarized in 
the following table:
!===================================================================================!
Term                                Example                         Item

Parameterized type                  List<String>                    Item 26

Actual type parameter               String                          Item 26

Generic type                        List<E>                         Items 26, 29

Formal type parameter               E                               Item 26

Unbounded wildcard type             List<?>                         Item 26

Raw type                            List                            Item 26

Bounded type parameter              <E extends Number>              Item 29

Recursive type bound                <T extends Comparable<T>>       Item 30

Bounded wildcard type               List<? extends Number>          Item 31

Generic method                      static <E> List<E> asList(E[] a)Item 30

Type token                          String.class                    Item 33
!===================================================================================!
{Aaron notes: Above is an important design.}

## ITEM 27: ELIMINATE UNCHECKED WARNINGS
When you program with generics, you will see many compiler warnings: unchecked cast warnings, unchecked method invocation
warnings, unchecked parameterized vararg type warnings, and unchecked conversion warnings. The more experience you acquire
with generics, the fewer warnings you’ll get, but don’t expect newly written code to compile cleanly.

Many unchecked warnings are easy to eliminate. For example, suppose you accidentally write this declaration:
```aidl
Set<Lark> exaltation = new HashSet();
```
The compiler will gently remind you what you did wrong:
```aidl
Venery.java:4: warning: [unchecked] unchecked conversion
        Set<Lark> exaltation = new HashSet();
                               ^
  required: Set<Lark>
  found:    HashSet
```

You can then make the indicated correction, causing the warning to disappear. Note that you don’t actually have to specify
the type parameter, merely to indicate that it’s present with the diamond operator (<>), introduced in Java 7. The compiler
will then infer the correct actual type parameter (in this case, Lark):

```aidl
Set<Lark> exaltation = new HashSet<>();
```
Some warnings will be much more difficult to eliminate. This chapter is filled with examples of such warnings. When you 
get warnings that require some thought, persevere! <b>Eliminate every unchecked warning that you can.</b> If you eliminate all 
warnings, you are assured that your code is typesafe, which is a very good thing. It means that you won’t get a 
ClassCastException at runtime, and it increases your confidence that your program will behave as you intended.

<b>If you can’t eliminate a warning, but you can prove that the code that provoked the warning is typesafe, then 
(and only then) suppress the warning with an @SuppressWarnings("unchecked") annotation.</b>

If you suppress warnings without first proving that the code is typesafe, you are giving yourself a false sense of security. 
The code may compile without emitting any warnings, but it can still throw a ClassCastException at runtime. If, however, 
you ignore unchecked warnings that you know to be safe (instead of suppressing them), you won’t notice when a new warning 
crops up that represents a real problem. The new warning will get lost amidst all the false alarms that you didn’t silence.

The SuppressWarnings annotation can be used on any declaration, from an individual local variable declaration to an 
entire class. Always use the SuppressWarnings annotation on the smallest scope possible. Typically this will be a variable 
declaration or a very short method or constructor. Never use SuppressWarnings on an entire class. Doing so could mask 
critical warnings.

If you find yourself using the SuppressWarnings annotation on a method or constructor that’s more than one line long, 
you may be able to move it onto a local variable declaration. You may have to declare a new local variable, but it’s worth it. 
For example, consider this toArray method, which comes from ArrayList:
```aidl
public <T> T[] toArray(T[] a) {
    if (a.length < size)
       return (T[]) Arrays.copyOf(elements, size, a.getClass());
    System.arraycopy(elements, 0, a, 0, size);
    if (a.length > size)
       a[size] = null;
    return a;
}
```
If you compile ArrayList, the method generates this warning:
```aidl
ArrayList.java:305: warning: [unchecked] unchecked cast
       return (T[]) Arrays.copyOf(elements, size, a.getClass());
                                 ^
  required: T[]
  found:    Object[]
```
It is illegal to put a SuppressWarnings annotation on the return statement, because it isn’t a declaration [JLS, 9.7]. 
You might be tempted to put the annotation on the entire method, but don’t. Instead, declare a local variable to hold 
the return value and annotate its declaration, like so:
{Aaron notes: Above is an important design.}

```aidl
// Adding local variable to reduce scope of @SuppressWarnings
public <T> T[] toArray(T[] a) {
    if (a.length < size) {
        // This cast is correct because the array we're creating
        // is of the same type as the one passed in, which is T[].
        @SuppressWarnings("unchecked") T[] result =
            (T[]) Arrays.copyOf(elements, size, a.getClass());
        return result;
    }
    System.arraycopy(elements, 0, a, 0, size);
    if (a.length > size)
        a[size] = null;
    return a;
}
```
The resulting method compiles cleanly and minimizes the scope in which unchecked warnings are suppressed.
<b>Every time you use a @SuppressWarnings("unchecked") annotation, add a comment saying why it is safe to do so. </b>
{Aaron notes: Above is an important design.}
This will help others understand the code, and more importantly, it will decrease the odds that someone will modify the 
code so as to make the computation unsafe. If you find it hard to write such a comment, keep thinking. You may end up 
figuring out that the unchecked operation isn’t safe after all.

### In summary, unchecked warnings are important. Don’t ignore them. Every unchecked warning represents the potential for a ClassCastException at runtime. Do your best to eliminate these warnings. If you can’t eliminate an unchecked warning and you can prove that the code that provoked it is typesafe, suppress the warning with a @SuppressWarnings("unchecked") annotation in the narrowest possible scope. Record the rationale for your decision to suppress the warning in a comment.

## ITEM 28: PREFER LISTS TO ARRAYS
Arrays differ from generic types in two important ways. First, arrays are covariant. This scary-sounding word means
simply that if Sub is a subtype of Super, then the array type Sub[] is a subtype of the array type Super[]. Generics,
by contrast, are invariant: for any two distinct types Type1 and Type2, List<Type1> is neither a subtype nor a supertype
of List<Type2> [JLS, 4.10; Naftalin07, 2.5]. You might think this means that generics are deficient, but arguably it is
arrays that are deficient. This code fragment is legal:

```aidl
// Fails at runtime!
Object[] objectArray = new Long[1];
objectArray[0] = "I don't fit in"; // Throws ArrayStoreException (runtime exception)
```
but this one is not:
```aidl
// Won't compile!
List<Object> ol = new ArrayList<Long>(); // Incompatible types (complie error)
ol.add("I don't fit in");
```
Either way you can’t put a String into a Long container, but with an array you find out that you’ve made a mistake at 
runtime; with a list, you find out at compile time. Of course, you’d rather find out at compile time.
{Aaron notes: Above is an important design.}

The second major difference between arrays and generics is that arrays are reified [JLS, 4.7]. This means that arrays 
know and enforce their element type at runtime. As noted earlier, if you try to put a String into an array of Long, 
you’ll get an ArrayStoreException. Generics, by contrast, are implemented by erasure [JLS, 4.6]. This means that they 
enforce their type constraints only at compile time and discard (or erase) their element type information at runtime. 
Erasure is what allowed generic types to interoperate freely with legacy code that didn’t use generics (Item 26), 
ensuring a smooth transition to generics in Java 5.

Because of these fundamental differences, arrays and generics do not mix well. For example, it is illegal to create 
an array of a generic type, a parameterized type, or a type parameter. Therefore, none of these array creation 
expressions are legal: new List<E>[], new List<String>[], new E[]. All will result in generic array creation errors at 
compile time.

Why is it illegal to create a generic array? Because it isn’t typesafe. If it were legal, casts generated by the compiler
in an otherwise correct program could fail at runtime with a ClassCastException. This would violate the fundamental
guarantee provided by the generic type system.
{Aaron notes: Above is an important design.}

To make this more concrete, consider the following code fragment:
```aidl
// Why generic array creation is illegal - won't compile!
List<String>[] stringLists = new List<String>[1];  // (1)
List<Integer> intList = List.of(42);               // (2)
Object[] objects = stringLists;                    // (3)
objects[0] = intList;                              // (4)
String s = stringLists[0].get(0);                  // (5)
```

Let’s pretend that line 1, which creates a generic array, is legal. Line 2 creates and initializes a List<Integer> 
containing a single element. Line 3 stores the List<String> array into an Object array variable, which is legal because 
arrays are covariant. Line 4 stores the List<Integer> into the sole element of the Object array, which succeeds because 
generics are implemented by erasure: the runtime type of a List<Integer> instance is simply List, and the runtime type 
of a List<String>[] instance is List[], so this assignment doesn’t generate an ArrayStoreException. Now we’re in trouble. 
We’ve stored a List<Integer> instance into an array that is declared to hold only List<String> instances. In line 5, 
we retrieve the sole element from the sole list in this array. The compiler automatically casts the retrieved element 
to String, but it’s an Integer, so we get a ClassCastException at runtime. In order to prevent this from happening, l
ine 1 (which creates a generic array) must generate a compile-time error.

Types such as E, List<E>, and List<String> are technically known as nonreifiable types [JLS, 4.7]. Intuitively speaking, 
a non-reifiable type is one whose runtime representation contains less information than its compile-time representation. 
Because of erasure, the only parameterized types that are reifiable are unbounded wildcard types such as List<?> and Map<?,?> (Item 26). 
It is legal, though rarely useful, to create arrays of unbounded wildcard types.

The prohibition on generic array creation can be annoying. It means, for example, that it’s not generally possible for a 
generic collection to return an array of its element type (but see Item 33 for a partial solution). It also means that 
you get confusing warnings when using varargs methods (Item 53) in combination with generic types. This is because every 
time you invoke a varargs method, an array is created to hold the varargs parameters. If the element type of this array 
is not reifiable, you get a warning. The SafeVarargs annotation can be used to address this issue (Item 32).

When you get a generic array creation error or an unchecked cast warning on a cast to an array type, the best solution 
is often to use the collection type List<E> in preference to the array type E[]. You might sacrifice some conciseness 
or performance, but in exchange you get better type safety and interoperability

For example, suppose you want to write a Chooser class with a constructor that takes a collection, and a single method 
hat returns an element of the collection chosen at random. Depending on what collection you pass to the constructor, 
you could use a chooser as a game die, a magic 8-ball, or a data source for a Monte Carlo simulation. Here’s a simplistic 
implementation without generics:
```aidl
// Chooser - a class badly in need of generics!
public class Chooser {
    private final Object[] choiceArray;
    public Chooser(Collection choices) {
        choiceArray = choices.toArray();
    }
    
    public Object choose() {
        Random rnd = ThreadLocalRandom.current();
        return choiceArray[rnd.nextInt(choiceArray.length)];
    }

}
```
To use this class, you have to cast the choose method’s return value from Object to the desired type every time you use 
invoke the method, and the cast will fail at runtime if you get the type wrong. Taking the advice of Item 29 to heart, 
we attempt to modify Chooser to make it generic. Changes are shown in boldface:

```aidl
// A first cut at making Chooser generic - won't compile
public class Chooser<T> {
    private final T[] choiceArray;
    
    public Chooser(Collection<T> choices) {
        choiceArray = choices.toArray();
    }
    // choose method unchanged

}
```
If you try to compile this class, you’ll get this error message:
```aidl
Chooser.java:9: error: incompatible types: Object[] cannot be
converted to T[]
        choiceArray = choices.toArray();
                                     ^
  where T is a type-variable:
    T extends Object declared in class Chooser
```
No big deal, you say, I’ll cast the Object array to a T array:
```aidl
choiceArray = (T[]) choices.toArray();
```
This gets rid of the error, but instead you get a warning:
```aidl
Chooser.java:9: warning: [unchecked] unchecked cast
        choiceArray = (T[]) choices.toArray();
                                           ^
  required: T[], found: Object[]
  where T is a type-variable:
T extends Object declared in class Chooser
```
The compiler is telling you that it can’t vouch for the safety of the cast at runtime because the program won’t know what 
type T represents—remember, element type information is erased from generics at runtime. Will the program work? Yes, but 
the compiler can’t prove it. You could prove it to yourself, put the proof in a comment and suppress the warning with an 
annotation, but you’re better off eliminating the cause of warning (Item 27).

To eliminate the unchecked cast warning, use a list instead of an array. Here is a version of the Chooser class that 
compiles without error or warning:
```
// List-based Chooser - typesafe
public class Chooser<T> {
    private final List<T> choiceList;

    public Chooser(Collection<T> choices) {
        choiceList = new ArrayList<>(choices);
    }

    public T choose() {
        Random rnd = ThreadLocalRandom.current();
        return choiceList.get(rnd.nextInt(choiceList.size()));
    }
}
```
{Aaron notes: Above is an important design.}
This version is a tad more verbose, and perhaps a tad slower, but it’s worth it for the peace of mind that you won’t 
get a ClassCastException at runtime.

### In summary, arrays and generics have very different type rules. Arrays are covariant and reified; generics are invariant and erased. As a consequence, arrays provide runtime type safety but not compile-time type safety, and vice versa for generics. As a rule, arrays and generics don’t mix well. If you find yourself mixing them and getting compile-time errors or warnings, your first impulse should be to replace the arrays with lists.

## ITEM 29: FAVOR GENERIC TYPES
It is generally not too difficult to parameterize your declarations and make use of the generic types and methods 
provided by the JDK. Writing your own generic types is a bit more difficult, but it’s worth the effort to learn how.

Consider the simple (toy) stack implementation from Item 7:
```aidl
// Object-based collection - a prime candidate for generics
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
        Object result = elements[--size];
        elements[size] = null; // Eliminate obsolete reference
        return result;
    }

    public boolean isEmpty() {
        return size == 0;
    }

    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```
This class should have been parameterized to begin with, but since it wasn’t, we can generify it after the fact. 
In other words, we can parameterize it without harming clients of the original non-parameterized version. As it stands, 
the client has to cast objects that are popped off the stack, and those casts might fail at runtime. The first step in 
generifying a class is to add one or more type parameters to its declaration. In this case there is one type parameter, 
representing the element type of the stack, and the conventional name for this type parameter is E (Item 68).

The next step is to replace all the uses of the type Object with the appropriate type parameter and then try to compile 
the resulting program:
```aidl
// Initial attempt to generify Stack - won't compile!

public class Stack<E> {
    private E[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new E[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(E e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public E pop() {
        if (size == 0)
            throw new EmptyStackException();
        E result = elements[--size];
        elements[size] = null; // Eliminate obsolete reference
        return result;
    }

    ... // no changes in isEmpty or ensureCapacity

}
```
You’ll generally get at least one error or warning, and this class is no exception. Luckily, this class generates only 
one error:
```aidl
Stack.java:8: generic array creation
        elements = new E[DEFAULT_INITIAL_CAPACITY];
```
As explained in Item 28, you can’t create an array of a non-reifiable type, such as E. This problem arises every time 
you write a generic type that is backed by an array. There are two reasonable ways to solve it. The first solution 
directly circumvents the prohibition on generic array creation: create an array of Object and cast it to the generic 
array type. Now in place of an error, the compiler will emit a warning. This usage is legal, but it’s not (in general) 
typesafe:
```aidl
Stack.java:8: warning: [unchecked] unchecked cast
found: Object[], required: E[]
        elements = (E[]) new Object[DEFAULT_INITIAL_CAPACITY];
                       ^
```

The compiler may not be able to prove that your program is typesafe, but you can. You must convince yourself that the 
unchecked cast will not compromise the type safety of the program. The array in question (elements) is stored in a 
private field and never returned to the client or passed to any other method. The only elements stored in the array 
are those passed to the push method, which are of type E, so the unchecked cast can do no harm.

Once you’ve proved that an unchecked cast is safe, suppress the warning in as narrow a scope as possible (Item 27). In 
this case, the constructor contains only the unchecked array creation, so it’s appropriate to suppress the warning in 
the entire constructor. With the addition of an annotation to do this, Stack compiles cleanly, and you can use it 
without explicit casts or fear of a ClassCastException:
```aidl
// The elements array will contain only E instances from push(E).
// This is sufficient to ensure type safety, but the runtime
// type of the array won't be E[]; it will always be Object[]!
@SuppressWarnings("unchecked")
public Stack() {
    elements = (E[]) new Object[DEFAULT_INITIAL_CAPACITY];

}
```
The second way to eliminate the generic array creation error in Stack is to change the type of the field elements from 
E[] to Object[]. If you do this, you’ll get a different error:
```aidl
Stack.java:19: incompatible types
found: Object, required: E
        E result = elements[--size];
```
You can change this error into a warning by casting the element retrieved from the array to E, but you will get a warning:
```aidl
Stack.java:19: warning: [unchecked] unchecked cast
found: Object, required: E
        E result = (E) elements[--size];
                               ^
```
Because E is a non-reifiable type, there’s no way the compiler can check the cast at runtime. Again, you can easily prove
to yourself that the unchecked cast is safe, so it’s appropriate to suppress the warning. In line with the advice of Item
27, we suppress the warning only on the assignment that contains the unchecked cast, not on the entire pop method:
```aidl
// Appropriate suppression of unchecked warning
public E pop() {
    if (size == 0)
        throw new EmptyStackException();

    // push requires elements to be of type E, so cast is correct
    @SuppressWarnings("unchecked") E result =
        (E) elements[--size];

    elements[size] = null; // Eliminate obsolete reference
    return result;
}
```

Both techniques for eliminating the generic array creation have their adherents. The first is more readable: the array 
is declared to be of type E[], clearly indicating that it contains only E instances. It is also more concise: in a 
typical generic class, you read from the array at many points in the code; the first technique requires only a single 
cast (where the array is created), while the second requires a separate cast each time an array element is read. Thus, 
the first technique is preferable and more commonly used in practice. It does, however, cause heap pollution (Item 32): 
the runtime type of the array does not match its compile-time type (unless E happens to be Object). This makes some 
programmers sufficiently queasy that they opt for the second technique, though the heap pollution is harmless in this 
situation.
{Aaron notes: Above is an important design.}

The following program demonstrates the use of our generic Stack class. The program prints its command line arguments in 
reverse order and converted to uppercase. No explicit cast is necessary to invoke String’s toUpperCase method on the 
elements popped from the stack, and the automatically generated cast is guaranteed to succeed:
```aidl
// Little program to exercise our generic Stack
public static void main(String[] args) {
    Stack<String> stack = new Stack<>();
    for (String arg : args)
        stack.push(arg);
    while (!stack.isEmpty())
        System.out.println(stack.pop().toUpperCase());
}
```
The foregoing example may appear to contradict Item 28, which encourages the use of lists in preference to arrays. It 
is not always possible or desirable to use lists inside your generic types. Java doesn’t support lists natively, so 
some generic types, such as ArrayList, must be implemented atop arrays. Other generic types, such as HashMap, are 
implemented atop arrays for performance.

The great majority of generic types are like our Stack example in that their type parameters have no restrictions: you 
can create a Stack<Object>, Stack<int[]>, Stack<List<String>>, or Stack of any other object reference type. Note that 
you can’t create a Stack of a primitive type: trying to create a Stack<int> or Stack<double> will result in a 
compile-time error. This is a fundamental limitation of Java’s generic type system. You can work around this restriction
by using boxed primitive types (Item 61).

There are some generic types that restrict the permissible values of their type parameters. For example, consider 
java.util.concurrent.DelayQueue, whose declaration looks like this:
```aidl
class DelayQueue<E extends Delayed> implements BlockingQueue<E>
```
The type parameter list (<E extends Delayed>) requires that the actual type parameter E be a subtype of 
java.util.concurrent.Delayed. This allows the DelayQueue implementation and its clients to take advantage of Delayed 
methods on the elements of a DelayQueue, without the need for explicit casting or the risk of a ClassCastException. 
The type parameter E is known as a bounded type parameter. Note that the subtype relation is defined so that every type 
is a subtype of itself [JLS, 4.10], so it is legal to create a DelayQueue<Delayed>.
{Aaron notes: Above is an important design.}

### In summary, generic types are safer and easier to use than types that require casts in client code. When you design new types, make sure that they can be used without such casts. This will often mean making the types generic. If you have any existing types that should be generic but aren’t, generify them. This will make life easier for new users of these types without breaking existing clients (Item 26).

## ITEM 30: FAVOR GENERIC METHODS

## ITEM 31: USE BOUNDED WILDCARDS TO INCREASE API FLEXIBILITY

## ITEM 32: COMBINE GENERICS AND VARARGS JUDICIOUSLY

## ITEM 33: CONSIDER TYPESAFE HETEROGENEOUS CONTAINERS

