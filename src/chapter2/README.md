
#Chapter 2. Creating and Destroying Objects

##ITEM 1: CONSIDER STATIC FACTORY METHODS INSTEAD OF CONSTRUCTORS

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


##ITEM 2: CONSIDER A BUILDER WHEN FACED WITH MANY CONSTRUCTOR PARAMETERS

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

##ITEM 3: ENFORCE THE SINGLETON PROPERTY WITH A PRIVATE CONSTRUCTOR OR AN ENUM TYPE

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

##ITEM 5: PREFER DEPENDENCY INJECTION TO HARDWIRING RESOURCES
Static utility classes and singletons are inappropriate for classes whose behavior is parameterized by an underlying resource.
What is required is the ability to support multiple instances of the class (in our example, SpellChecker), each of which uses the resource desired by the client (in our example, the dictionary). 
A simple pattern that satisfies this requirement is to pass the resource into the constructor when creating a new instance. This is one form of dependency injection: the dictionary is a dependency of the spell checker and is injected into the spell checker when it is created. 


##ITEM 6: AVOID CREATING UNNECESSARY OBJECTS

##ITEM 7: ELIMINATE OBSOLETE OBJECT REFERENCES

##ITEM 8: AVOID FINALIZERS AND CLEANERS

##ITEM 9: PREFER TRY-WITH-RESOURCES TO TRY-FINALLY