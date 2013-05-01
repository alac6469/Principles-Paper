Bobby Crimi, Alex Ackerman, Richard Green
=========================================

SID-7
-----

  In 2008, there was a great deal of discussion regarding the use of arrays in Scala 2.7.  To the non-expert eye, it would appear that they were simply the same as Java arrays.  However, this is simply not the case, and if used incorrectly, they can negatively impact performance.  The key issues with Scala 2.7 stemmed from the fact that ideally, Scala arrays would have the same representation as in Java so that the data could be used interchangeably.  Unfortunately, there are a number of reasons why the 2.7 arrays were unable to do this in light of Java’s particularly low-level representation. First off, Java has multiple array representations which is redundant and can cause a certain amount of ambiguity.  Also, while 2.7 technically did have constructors for arrays of a generic type, these constructors caused some issues, such as the fact that various data types would not function as well as using uninitialized arrays.  Lastly, there were very few effective methods for array indexing and manipulation.  This paper will continue to examine these issues, potential fixes, and what was actually implemented in subsequent releases of Scala in order to avoid some of these problems.
  
  
Take the following code as an example:   
  
~~~~~
  class ArrayWrapper[A](length: Int) {
  private val array = new Array[A](length)
  def apply(x: Int) = array(x)
  def update(x: Int, value: A) = array(x) = value
  override def toString(): String = array.toString
}
~~~~~

One would expect it to simply wrap an array, and provide access to the default array methods and array.toString. When you use it though, unexpected behavior happens. For example, look at the code below:

~~~~~~
scala> val a = new ArrayWrapper[Int](5)
a: ArrayWrapper[Int] = Array(null, null, null, null, null)
~~~~~~

The val a should be initialized to Array(0, 0, 0, 0, 0), as that is what you get when you initialize a regular Array[Int]. But when the compiler initializes the array inside the wrapper, the array has been converted from array[Int] to an array of java.lang.Integer, which is an object. And the default value of an object is null, not 0. So val a contains nulls. 

Now observe the following code:

~~~~~
scala> val x = a(0)
x: Int = 0
 
scala> x.toString
res0: java.lang.String = 0

scala> a(0).toString
java.lang.NullPointerException
       at .<init>(<console>:7)
       at .<clinit>(<console>)
       at RequestResult$.<init>(<console>:3)
       at RequestResult$.<clinit>(<console>)
       at RequestResult$result(<console>)
       at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
       at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:39)
       at sun.reflect.DelegatingMethodAccessorImpl.i...
~~~~~

"Ouch! NullPointerException! This is an unpleasant surprise." says Matt Malone. The val x is initially seen to be 0, and there is no problem when converting this to a string. However, when a(0) is converted to a string, we get a NullPointerException. The reason for these ambiguities is because in Scala 2.7, when arrays are implemented with parameterized types, the arrays are actually Java objects. So, when running different methods on the array, possible interferences can occur that will produce odd results that could be almost impossible to debug. These are but examples of the unobvious behaviours associated with Scala 2.7 the boxed/unboxed array implementation.

  Throughout the Scala Improvement Process, there were a few ways that were presented as possibilities for solving the aforementioned issues with arrays in Scala 2.7.  One such way was to create two different implementations of an array.  In one implementation, there would be the Java representation for interoperation.  This implementation would have all of the same traits as an array in Java and have a quick runtime.  The other implementation would be the Scala representation for use in the collection hierarchy.  One advantage of this second implementation would be that it would have all of the easy-to-use methods for such things as indexing and manipulation.  In an ideal world, these two implementations should be interchangeable, with the first one having the performance boost, and the second one being more flexible.  However, issues arise when thinking about what would occur with this method when very large pieces of code are being used.  Because a developer would need to determine a specific implementation of the array to use, discrepancies will most likely occur when people who are working on the same code choose to use opposite implementations.  This could become very complicated and problematic as the various pieces of code from different programmers, and different implementations, are brought together and tried to be compiled. For example, consider the following code:

Another potential solution that was discussed involved wrapping Java-like arrays in Scala arrays.  Therefore, the native arrays would go through a conversion that would essentially make them a part of Scala’s collection hierarchy.  By doing this, the uncertainty that arises from having two separate implementations is avoided.  In this context, native arrays would have the same implementation as new Scala arrays; however, they would still have the same performance capabilities as Java Arrays. However, the same thing had been done with conversions between Scala's String and RichString, and many problems arose. Take for example the following code:

~~~~~
"abc".reverse.reverse == "abc"
"abc" != "abc".reverse.reverse       
~~~~~

"The problem here was that the reverse method was inherited from class Seq where it was defined to return another Seq. Since strings are not sequences, the only feasible type reverse could return when called on a String was RichString. But then the equals method on Strings which is inherited from Java would not recognize that a String could be equal to a RichString," says Martin Odersky. We can see that the same problems will only occur in conversions between an Array and a similar RichArray type. So, this is not a feasible solution.

In the Scala 2.8 version of Scala Collections, there is a new collections framework which “accompanies collection traits such as Seq with implementation traits that abstract over the representation of the collection.” This allows the base trait to inherit its essential operations from the abstract trait that is instantiated depending on the representation type. Using this new framework, Scala arrays would be able to get the speed of the Java arrays without sacrificing its place in the collection hierarchy.

Scala arrays are integrated into new framework via two implicit conversions. The first maps an Array of type T to an object of type ArrayOps, which will allow programmers to call any Seq method on an array. The returns of these methods will then yield arrays instead of ArrayOps values. And since the intermediate step, converting to ArrayOps, is so brief, modern VM's can reduce the calling overhead to essentially zero. 

        
>If the programmer wants to convert an array into a Seq, another implicit conversion is preformed: the conversion   >between an array and a WrappedArray. WrappedArrays “are mutable vectors that implement all vector operations in    >terms of a given Java array”. The operations on a WrappedArray then return a WrappedArray, instead of an Array     >like ArrayOps does. And then there is an implicit conversion between WrappedArray and Array. Both ArrayOps and     >WrappedArray inherit from ArrayLike, to avoid the duplication of common code, like operations. >(http://docs.scala-lang.org/sips/completed/scala-2-8-arrays.html)
        

Given the two implicit conversions, there is a balancing as to which is called. The conversion to ArrayOps has precedence, but in some cases an array needs to be converted to a sequence, instead of a sequence method just being called on it. In that case, the WrappedArray conversion is used. This precedence is defined by the policy that: “When comparing two different applicable alternatives of an overloaded method or of an implicit, each method gets one point for having more specific arguments, and another point for being defined in a proper subclass. An alternative 'wins' over another if it gets a greater number of points in these two comparisons.” As ArrayOps is placed in the Predef object, and WrappedArray is in the LowPriorityImplicits class inherited from Predef, WrappedArray will only be used if it's definitely needed.
