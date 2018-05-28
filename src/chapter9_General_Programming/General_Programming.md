# Chapter 9. General Programming

THIS chapter is devoted to the nuts and bolts of the language. It discusses local variables, control structures, libraries, 
data types, and two extralinguistic facilities: reflection and native methods. Finally, it discusses optimization and 
naming conventions.

## ITEM 57: MINIMIZE THE SCOPE OF LOCAL VARIABLES

This item is similar in nature to Item 15, “Minimize the accessibility of classes and members.” By minimizing the scope 
of local variables, you increase the readability and maintainability of your code and reduce the likelihood of error.

Older programming languages, such as C, mandated that local variables must be declared at the head of a block, and some 
programmers continue to do this out of habit. It’s a habit worth breaking. As a gentle reminder, Java lets you declare 
variables anywhere a statement is legal (as does C, since C99).

<b>The most powerful technique for minimizing the scope of a local variable is to declare it where it is first used.</b>
If a variable is declared before it is used, it’s just clutter—one more thing to distract the reader who is trying to 
figure out what the program does. By the time the variable is used, the reader might not remember the variable’s type or 
initial value.

Declaring a local variable prematurely can cause its scope not only to begin too early but also to end too late. The 
scope of a local variable extends from the point where it is declared to the end of the enclosing block. If a variable 
is declared outside of the block in which it is used, it remains visible after the program exits that block. If a 
variable is used accidentally before or after its region of intended use, the consequences can be disastrous.

<b>Nearly every local variable declaration should contain an initializer.</b> If you don’t yet have enough information to 
initialize a variable sensibly, you should postpone the declaration until you do. One exception to this rule concerns 
try-catch statements. If a variable is initialized to an expression whose evaluation can throw a checked exception, the 
variable must be initialized inside a try block (unless the enclosing method can propagate the exception). If the value 
must be used outside of the try block, then it must be declared before the try block, where it cannot yet be “sensibly 
initialized.” For an example, see page 283.
{Aaron notes: Above is an important design.}

Loops present a special opportunity to minimize the scope of variables. The for loop, in both its traditional and 
for-each forms, allows you to declare loop variables, limiting their scope to the exact region where they’re needed. 
(This region consists of the body of the loop and the code in parentheses between the for keyword and the body.) 
Therefore, <b>prefer for loops to while loops</b>, assuming the contents of the loop variable aren’t needed after the 
loop terminates.

For example, here is the preferred idiom for iterating over a collection (Item 58):

```aidl
// Preferred idiom for iterating over a collection or array
for (Element e : c) {
    ... // Do Something with e
}
```

If you need access to the iterator, perhaps to call its remove method, the preferred idiom uses a traditional for loop 
in place of the for-each loop:

```aidl
// Idiom for iterating when you need the iterator
for (Iterator<Element> i = c.iterator(); i.hasNext(); ) {
    Element e = i.next();
    ... // Do something with e and i
}
```

To see why these for loops are preferable to a while loop, consider the following code fragment, which contains two 
while loops and one bug:

```aidl
Iterator<Element> i = c.iterator();

while (i.hasNext()) {
    doSomething(i.next());
}

...

Iterator<Element> i2 = c2.iterator();
while (i.hasNext()) {             // BUG!
    doSomethingElse(i2.next());
}
```

The second loop contains a copy-and-paste error: it initializes a new loop variable, i2, but uses the old one, i, which 
is, unfortunately, still in scope. The resulting code compiles without error and runs without throwing an exception, but 
it does the wrong thing. Instead of iterating over c2, the second loop terminates immediately, giving the false impression
that c2 is empty. Because the program errs silently, the error can remain undetected for a long time.

If a similar copy-and-paste error were made in conjunction with either of the for loops (for-each or traditional), the 
resulting code wouldn’t even compile. The element (or iterator) variable from the first loop would not be in scope in 
the second loop. Here’s how it looks with the traditional for loop:

```aidl
for (Iterator<Element> i = c.iterator(); i.hasNext(); ) {
    Element e = i.next();
    ... // Do something with e and i
}

...

// Compile-time error - cannot find symbol i
for (Iterator<Element> i2 = c2.iterator(); i.hasNext(); ) {
    Element e2 = i2.next();
    ... // Do something with e2 and i2
}
```

Moreover, if you use a for loop, it’s much less likely that you’ll make the copy-and-paste error because there’s no 
incentive to use different variable names in the two loops. The loops are completely independent, so there’s no harm in 
reusing the element (or iterator) variable name. In fact, it’s often stylish to do so.

The for loop has one more advantage over the while loop: it is shorter, which enhances readability.

Here is another loop idiom that minimizes the scope of local variables:

```aidl
for (int i = 0, n = expensiveComputation(); i < n; i++) {
    ... // Do something with i;
}
```

The important thing to notice about this idiom is that it has two loop variables, i and n, both of which have exactly 
the right scope. The second variable, n, is used to store the limit of the first, thus avoiding the cost of a redundant 
computation in every iteration. As a rule, you should use this idiom if the loop test involves a method invocation that 
is guaranteed to return the same result on each iteration.
{Aaron notes: Above is an important design.} It's important in concurrent programming. 

### A final technique to minimize the scope of local variables is to <b>keep methods small and focused</b>. If you combine two activities in the same method, local variables relevant to one activity may be in the scope of the code performing the other activity. To prevent this from happening, simply separate the method into two: one for each activity.


## ITEM 58: PREFER FOR-EACH LOOPS TO TRADITIONAL FOR LOOPS

## ITEM 59: KNOW AND USE THE LIBRARIES

## ITEM 60: AVOID FLOAT AND DOUBLE IF EXACT ANSWERS ARE REQUIRED

## ITEM 61: PREFER PRIMITIVE TYPES TO BOXED PRIMITIVES

## ITEM 62: AVOID STRINGS WHERE OTHER TYPES ARE MORE APPROPRIATE

## ITEM 63: BEWARE THE PERFORMANCE OF STRING CONCATENATION

## ITEM 64: REFER TO OBJECTS BY THEIR INTERFACES

## ITEM 65: PREFER INTERFACES TO REFLECTION

## ITEM 66: USE NATIVE METHODS JUDICIOUSLY

## ITEM 67: OPTIMIZE JUDICIOUSLY

## ITEM 68: ADHERE TO GENERALLY ACCEPTED NAMING CONVENTIONS