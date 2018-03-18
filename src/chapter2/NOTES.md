
#Chapter 2. Creating and Destroying Objects

ITEM 1: CONSIDER STATIC FACTORY METHODS INSTEAD OF CONSTRUCTORS

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

In short, static factory methods and public 


ITEM 2: CONSIDER A BUILDER WHEN FACED WITH MANY CONSTRUCTOR PARAMETERS

 In short, the telescoping constructor pattern works, but it is hard to write client code when there are many parameters, and harder still to read it. 

// Builder Pattern
```
public class NutritionFact {
	private final int servingSize;
	private final int servings;
	private final int calories;
	private final int fat;
	private final int sodium;
	private final int carbohydrate;
	//etc…

	public static class Builder {
		//Required parameters
	
	}
}
```






* ITEM 3: ENFORCE THE SINGLETON PROPERTY WITH A PRIVATE CONSTRUCTOR OR AN ENUM TYPE
* 




* ITEM 1: CONSIDER STATIC FACTORY METHODS INSTEAD OF CONSTRUCTORS
