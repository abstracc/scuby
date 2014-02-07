## PRESENTATION

Scuby is a thin integration layer between Scala and JRuby. It aims to provide DSLs for Scala and JRuby to ease interoperability. The original inspiration came from a couple of blog entries published by Daniel Spiewak:

[JRuby Interop DSL in Scala](http://www.codecommit.com/blog/ruby/jruby-interop-dsl-in-scala)

[Integrating Scala into JRuby](http://www.codecommit.com/blog/ruby/integrating-scala-into-jruby)

The idea behind this polyglot architecture came about based on the [Fractal Programming](http://olabini.com/blog/2008/06/fractal-programming/) post by Ola Bini, in which he talks about dividing a program in 3 layers: the static layer for performance-critical functionality and those functions in which type safety is paramount, a dynamic layer where you create a DSL, and a DSL layer where the bulk of the business logic resides.

Scuby was first publicly presented at RubyConf 2009, in the talk _Ruby is from Mars, Functional Languages are from Venus: Integrating Ruby with Erlang, Scala or F#_ by Angela O.K. Wright.

The project has been sponsored in part by [Abstra.cc](http://www.abstra.cc) where we use it for our internal development.

## ADDING SCUBY TO YOUR PROJECT

To add Scuby to a Maven project, just add the following dependency to your build.sbt:

```
libraryDependencies += "com.tecnoguru" %% "scuby" % "0.2.2"
```

or if you use Maven, add this to your pom.xml:

```xml
<dependency>
  <groupId>com.tecnoguru</groupId>
  <artifactId>scuby_2.10</artifactId>
  <version>0.2.2</version>
</dependency>
```

If you use SBT, Gradle, etc.., please convert this to your preferred syntax.

## DEPENDENCIES

The Scuby pom.xml file includes dependencies on `org.scala-lang.scala-library 2.10.3` and `org.jruby.jruby-complete 1.7.9`. If you don't use Maven or something that understands Maven pom.xml files, you will need to download the corresponding jars and place them in your CLASSPATH.

At the moment Scuby is based on JRuby 1.7.9 and Scala 2.10.3, even though it makes almost no use (yet) of the new Java interoperability features introduced with JRuby 1.4. These should slowly find their way into Scuby as time permits.

## USAGE

At this point the Scala -> Ruby part is partly done, while the Ruby -> Scala part is still in the planning stages, although it's become less necessary with the JRuby changes in 1.6.x and the other projects mentioned in the _RELATED PROJECTS_ section. Here are some examples of the things you can do today. Assume you have the following Ruby code in a file called `test.rb` somewhere in your CLASSPATH (this file is part of the Scuby unit tests and can be found in `src/test/resources/test.rb`):

```ruby
module Core
  class Person
    attr_accessor :firstname, :lastname
    def initialize (firstname, lastname)
      @firstname = firstname
      @lastname = lastname
   end

    def fullname
      "#{firstname} #{lastname}"
    end

    def get_label
      javax.swing.JLabel.new(fullname)
    end
  end

  module BackEnd
   def self.get_people
      # Get data from the backend and return an Array of Person
    end

    def self.get_data
      { :people => get_people, :other_data => get_other_data }
    end

    def self.get_person(name)
      # Get a person's data from the backend and return a Person object
    end

    def self.get_other_data
      # Get some other data that is needed for the app
    end
  end
end
```


Here are some examples using Scuby:

```scala
import com.tecnoguru.scuby._
import JRuby._

object Main {
  def main(args: Array[String]) = { 
    // Require a Ruby file from the classpath
    require("example") 

    // Eval a Ruby statement discarding the return value
    evalIgnore("import Core")

    // Eval a Ruby statement that returns a Ruby object
    val array = evalRuby("[]")

    // or that returns a Scala/Java object or primitive
    val array2 = eval[Int]("[1, 2, 3].length")
    val array2: Int = eval("[1, 2, 3].length")

    // Create a Ruby object
    val array3 = new RubyObject('Array)

    // Easily call is_a? and respond_to?
    array3 isA_? 'Array // true
    array3 isA_? 'Hash  // false

    array3 respondTo_? 'length // true
    array3 respondTo_? 'foo    //false

    // Create a proxy object for the Ruby BackEnd class
    val backEnd = RubyClass('BackEnd)

    // Create an instance of the Person class
    val person = new RubyObject('Person, "Zaphod", "Beeblebrox")
    val person2 = RubyClass('Person) ! ('new, "Ford", "Prefect")

    // Call a method on a Ruby object (in this case, the Ruby class),
    // passing in parameters, and get back another Ruby object
    val zaphod = backEnd ! ('get_person, "Zaphod")

    // Call a Ruby method with no parameters
    val data = backEnd ! 'get_data

    // Ruby method chaining
    val length = backEnd ! 'get_people ! 'length

    // Access to a Ruby hash or array (i.e., something that implements the
    // [] method]). Returns an AnyRef (i.e. a java.lang.Object). Also, creating
    // a Ruby Symbol using %
    val people = data(%('people))
    val zaphod2 = people(0)

    // Multidimensional hashes or arrays (i.e., data["parm1"]["parm2"])
    val ford = data(%('people), 1)

    // Modify/add an element to a Collection (or anything that implements []=)
    people(2) = RubyClass('Person) ! ('new, "Arthur", "Dent")

    // Call a Ruby method which returns a Java object, in a type-safe way
    val label = person.send[JLabel]('get_label)

    // This is one way of chaining method calls that include array access. This
    // is kind of ugly but necessary because of the strong typing, since
    // Array/Hash access returns an AnyRef.
    // Ruby equivalent:
    // label = backEnd.get_people[:people][0].get_label
    // contents = form.components[:startables].widget.peer  # Returns a JComponent
    val label = (backEnd ! 'get_data ! ('[], %('people)) ! ('[], 0)).send[JLabel]('get_label)
    // OR
    val zaphod3 = (backEnd ! 'get_data(%'people, 0)).asInstanceOf[RubyObj]
    val label = zaphod3.get_label

    // You can also use the as[T] method to wrap a RubyObj with a
    // dynamically-generated proxy for a trait, to get all the statically-typed
    // goodness on the Scala side. The proxy also takes care of the CamelCase to
    // snake_case conversion. I have tested the basic calls and case conversion,
    // plus the attribute assignments. More testing is needed to make sure
    // there are no more edge cases
    trait Person {
      def firstname: String
      def firstname_=(f: String): Unit
      def lastname: String
      def lastname_=(l: String): Unit
      def fullname: String
      def getLabel: JLabel
    }

    val zaphod:Person = new RubyObject('Person, "Zaphod", "Beeblebrox").as[Person]
    zaphod.firstname = "The Zeeb"
    println(zaphod.fullname)
    val label = zaphod.getLabel
 }
}
```

This is the gist of it. Basically we can create Ruby objects, call methods on them and evaluate Ruby code. Scuby has its own `RubyObj` class that wraps a JRuby  `RubyObject` and does its magic. It also has its own `RubyClass` class that  represents a Ruby class.


## RELATED PROJECTS

[jruby-scala](http://rubygems.org/gems/jruby-scala): Allows you to use Ruby Procs as Scala functions, including Scala traits into Ruby modules, and more.

[jruby-scala-collections](https://github.com/RubyAndScala/jruby-scala-collections): Eases the pain of passing JRuby and Scala collections back and forth


## NEXT STEPS AND IDEAS

On the Scala side

* Use typeclasses to remove all the explicit wrapping of org.jruby.RubyObject's into com.tecnoguru.scuby.RubyObj's

* Transparently wrap Ruby collections in Scala collections, so you can use them with `foreach`, `for`, `map`, `foldLeft`, `foldRight`, etc... (see [jruby-scala-collections](https://github.com/arturaz/jruby-scala-collections))

* Create traits that mimic the standard Ruby object hierarchy, at least for Object, Class and Module, and either allow the traits passed in to as[T] to wrap them, or automatically extend the traits depending on the Ruby object's type

* Create a Java-friendly API so Scuby can be more-easily used from Java or other JVM languages

* Optimization. There are probably a lot of things that can be optimized by calling the internal JRuby APIs

* More test cases for `as[T]` and perhaps simplify the implementation

* More testing in general

On the JRuby side (although lots of it is handled automatically by JRuby).

* Create a gem for loading into JRuby (See [jruby-scala](http://rubygems.org/gems/jruby-scala) and [jruby-scala-collections](https://github.com/RubyAndScala/jruby-scala-collections))

* FunctionN -> block conversion, so you could pass in a Scala function to any Ruby method that expects a block. (See [jruby-scala](http://rubygems.org/gems/jruby-scala))

* Wrapping the Scala collections with their Ruby/JRuby/Java equivalents. (See [jruby-scala-collections](https://github.com/RubyAndScala/jruby-scala-collections))

* Add an `Object#to_scala` method which wraps the Ruby Object in a Scuby `RubyObj`. Ideally the wrapping should be done automatically but I'm not totally sure if that's possible. This will probably not be necessary once I correctly use typeclasses.

* Create a scala top-level function, so you can do, i.e., `scala.mutable.List` instead of `Java::ScalaMutable::List`

## COLLABORATING

As usual on GitHub: fork, pull, modify, *create tests*, commit, push, pull request. You can also find me on Twitter as [@thedoc](https://twitter.com/thedoc)

