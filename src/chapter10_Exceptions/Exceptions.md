# Chapter 10. Exceptions

WHEN used to best advantage, exceptions can improve a program’s readability, reliability, and maintainability. When 
used improperly, they can have the opposite effect. This chapter provides guidelines for using exceptions effectively.

## ITEM 69: USE EXCEPTIONS ONLY FOR EXCEPTIONAL CONDITIONS

Someday, if you are unlucky, you may stumble across a piece of code that looks something like this:

```aidl
// Horrible abuse of exceptions. Don't ever do this!
try {
    int i = 0;
    while(true)
        range[i++].climb();
} catch (ArrayIndexOutOfBoundsException e) {

}
```

What does this code do? It’s not at all obvious from inspection, and that’s reason enough not to use it (Item 67). It 
turns out to be a horribly ill-conceived idiom for looping through the elements of an array. The infinite loop 
terminates by throwing, catching, and ignoring an ArrayIndexOutOfBoundsException when it attempts to access the 
first array element outside the bounds of the array. It’s supposed to be equivalent to the standard idiom for looping 
through an array, which is instantly recognizable to any Java programmer:
```aidl
for (Mountain m : range)
    m.climb();
```
So why would anyone use the exception-based loop in preference to the tried and true? It’s a misguided attempt to 
improve performance based on the faulty reasoning that, since the VM checks the bounds of all array accesses, the 
normal loop termination test—hidden by the compiler but still present in the for-each loop—is redundant and should 
be avoided. There are three things wrong with this reasoning:

1. • Because exceptions are designed for exceptional circumstances, there is little incentive for JVM implementors 
to make them as fast as explicit tests.

2. • Placing code inside a try-catch block inhibits certain optimizations that JVM implementations might 
otherwise perform.

3. • The standard idiom for looping through an array doesn’t necessarily result in redundant checks. Many JVM 
implementations optimize them away.
{Aaron notes: Above is an important design.}

In fact, the exception-based idiom is far slower than the standard one. On my machine, the exception-based idiom 
is about twice as slow as the standard one for arrays of one hundred elements.

Not only does the exception-based loop obfuscate the purpose of the code and reduce its performance, but it’s 
not guaranteed to work. If there is a bug in the loop, the use of exceptions for flow control can mask the bug, 
greatly complicating the debugging process. Suppose the computation in the body of the loop invokes a method 
that performs an out-of-bounds access to some unrelated array. If a reasonable loop idiom were used, the bug 
would generate an uncaught exception, resulting in immediate thread termination with a full stack trace. If 
the misguided exception-based loop were used, the bug-related exception would be caught and misinterpreted 
as a normal loop termination.

The moral of this story is simple: <b>Exceptions are, as their name implies, to be used only for exceptional 
conditions;</b> they should never be used for ordinary control flow. More generally, use standard, easily recognizable 
idioms in preference to overly clever techniques that purport to offer better performance. Even if the performance 
advantage is real, it may not remain in the face of steadily improving platform implementations. The subtle bugs 
and maintenance headaches that come from overly clever techniques, however, are sure to remain.

This principle also has implications for API design. <b>A well-designed API must not force its clients to use 
exceptions for ordinary control flow.</b> A class with a “state-dependent” method that can be invoked only under certain 
unpredictable conditions should generally have a separate “state-testing” method indicating whether it is appropriate 
to invoke the state-dependent method. For example, the Iterator interface has the state-dependent method next and 
the corresponding state-testing method hasNext. This enables the standard idiom for iterating over a collection 
with a traditional for loop (as well as the for-each loop, where the hasNext method is used internally):
```aidl
for (Iterator<Foo> i = collection.iterator(); i.hasNext(); ) {
    Foo foo = i.next();
    ...
}
```
{Aaron notes: Above is an important design.}

If Iterator lacked the hasNext method, clients would be forced to do this instead:
```aidl
// Do not use this hideous code for iteration over a collection!
try {
    Iterator<Foo> i = collection.iterator();
    while(true) {
        Foo foo = i.next();
        ...
    }
} catch (NoSuchElementException e) {
}
```
This should look very familiar after the array iteration example that began this item. In addition to being wordy 
and misleading, the exception-based loop is likely to perform poorly and can mask bugs in unrelated parts of the 
system.

An alternative to providing a separate state-testing method is to have the state-dependent method return an empty 
optional (Item 55) or a distinguished value such as null if it cannot perform the desired computation.

Here are some guidelines to help you choose between a state-testing method and an optional or distinguished 
return value. If an object is to be accessed concurrently without external synchronization or is subject to 
externally induced state transitions, you must use an optional or distinguished return value, as the object’s state 
could change in the interval between the invocation of a state-testing method and its state-dependent method. 
Performance concerns may dictate that an optional or distinguished return value be used if a separate state-testing 
method would duplicate the work of the state-dependent method. All other things being equal, a state-testing 
method is mildly preferable to a distinguished return value. It offers slightly better readability, and incorrect 
use may be easier to detect: if you forget to call a state-testing method, the state-dependent method will throw 
an exception, making the bug obvious; if you forget to check for a distinguished return value, the bug may be 
subtle. This is not an issue for optional return values.

### In summary, exceptions are designed for exceptional conditions. Don’t use them for ordinary control flow, and don’t write APIs that force others to do so.

### ITEM 70: USE CHECKED EXCEPTIONS FOR RECOVERABLE CONDITIONS AND RUNTIME EXCEPTIONS FOR PROGRAMMING ERRORS

Java provides three kinds of throwables: checked exceptions, runtime exceptions, and errors. There is some confusion 
among programmers as to when it is appropriate to use each kind of throwable. While the decision is not always 
clear-cut, there are some general rules that provide strong guidance.

The cardinal rule in deciding whether to use a checked or an unchecked exception is this: use checked exceptions for 
conditions from which the caller can reasonably be expected to recover. By throwing a checked exception, you force 
the caller to handle the exception in a catch clause or to propagate it outward. Each checked exception that a method 
is declared to throw is therefore a potent indication to the API user that the associated condition is a possible 
outcome of invoking the method.
{Aaron notes: Above is an important design.}

By confronting the user with a checked exception, the API designer presents a mandate to recover from the condition. 
The user can disregard the mandate by catching the exception and ignoring it, but this is usually a bad idea (Item 77).

There are two kinds of unchecked throwables: runtime exceptions and errors. They are identical in their behavior: 
both are throwables that needn’t, and generally shouldn’t, be caught. If a program throws an unchecked exception or an 
error, it is generally the case that recovery is impossible and continued execution would do more harm than good. If a 
program does not catch such a throwable, it will cause the current thread to halt with an appropriate error message.

Use runtime exceptions to indicate programming errors. The great majority of runtime exceptions indicate precondition 
violations. A precondition violation is simply a failure by the client of an API to adhere to the contract established 
by the API specification. For example, the contract for array access specifies that the array index must be between 
zero and the array length minus one, inclusive. ArrayIndexOutOfBoundsException indicates that this precondition was 
violated.

One problem with this advice is that it is not always clear whether you’re dealing with a recoverable conditions or a 
programming error. For example, consider the case of resource exhaustion, which can be caused by a programming error 
such as allocating an unreasonably large array, or by a genuine shortage of resources. If resource exhaustion is caused 
by a temporary shortage or by temporarily heightened demand, the condition may well be recoverable. It is a matter 
of judgment on the part of the API designer whether a given instance of resource exhaustion is likely to allow for 
recovery. If you believe a condition is likely to allow for recovery, use a checked exception; if not, use a runtime 
exception. If it isn’t clear whether recovery is possible, you’re probably better off using an unchecked exception, 
for reasons discussed in Item 71.

While the Java Language Specification does not require it, there is a strong convention that errors are reserved for 
use by the JVM to indicate resource deficiencies, invariant failures, or other conditions that make it impossible to 
continue execution. Given the almost universal acceptance of this convention, it’s best not to implement any new 
Error subclasses. Therefore, all of the unchecked throwables you implement should subclass RuntimeException (directly 
or indirectly). Not only shouldn’t you define Error subclasses, but with the exception of AssertionError, you 
shouldn’t throw them either.

It is possible to define a throwable that is not a subclass of Exception, RuntimeException, or Error. The JLS doesn’t 
address such throwables directly but specifies implicitly that they behave as ordinary checked exceptions (which are 
subclasses of Exception but not RuntimeException). So when should you use such a beast? In a word, never. They have 
no benefits over ordinary checked exceptions and would serve merely to confuse the user of your API.

API designers often forget that exceptions are full-fledged objects on which arbitrary methods can be defined. The 
primary use of such methods is to provide code that catches the exception with additional information concerning 
the condition that caused the exception to be thrown. In the absence of such methods, programmers have been known 
to parse the string representation of an exception to ferret out additional information. This is extremely bad 
practice (Item 12). Throwable classes seldom specify the details of their string representations, so string 
representations can differ from implementation to implementation and release to release. Therefore, code that parses 
the string representation of an exception is likely to be nonportable and fragile.

Because checked exceptions generally indicate recoverable conditions, it’s especially important for them to provide 
methods that furnish information to help the caller recover from the exceptional condition. For example, suppose a 
checked exception is thrown when an attempt to make a purchase with a gift card fails due to insufficient funds. 
The exception should provide an accessor method to query the amount of the shortfall. This will enable the caller 
to relay the amount to the shopper. See Item 75 for more on this topic.

### To summarize, throw checked exceptions for recoverable conditions and unchecked exceptions for programming errors. When in doubt, throw unchecked exceptions. Don’t define any throwables that are neither checked exceptions nor runtime exceptions. Provide methods on your checked exceptions to aid in recovery.

### ITEM 71: AVOID UNNECESSARY USE OF CHECKED EXCEPTIONS

### ITEM 72: FAVOR THE USE OF STANDARD EXCEPTIONS

### ITEM 73: THROW EXCEPTIONS APPROPRIATE TO THE ABSTRACTION

### ITEM 74: DOCUMENT ALL EXCEPTIONS THROWN BY EACH METHOD

### ITEM 75: INCLUDE FAILURE-CAPTURE INFORMATION IN DETAIL MESSAGES

### ITEM 76: STRIVE FOR FAILURE ATOMICITY

### ITEM 77: DON’T IGNORE EXCEPTIONS
