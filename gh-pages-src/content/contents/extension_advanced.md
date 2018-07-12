---
title: "Extending the Schema (advanced)"
date: 2018-04-03T10:58:39-07:00
draft: true
weight: 80
---

Internally, we use an augmented schema with some additional proprietary columns and measurement tables.

This section examines our process for developing an extensible schema mechanism, and describes how
the core schema can be augmented to fit custom requirements. Note that this process is quite involved, and we recommend
using [tags]({{< ref "tags.md" >}}) for most use cases instead of a customized schema.

Thanks to Leslie Lam for all her work on making persistence generic, and for writing the original form
of this document.

## How to Extend the Schema
An augmented schema will need to have a custom version of `CorePersistenceUtils`. 
This requires several major changes to the `persistence`, `arbiters`, and `collectors` 
packages.

  + Generate Slick tables to represent your new custom Warp Schema.
  + Extend `CoreQueries`, and `CorePersistenceUtils`.
  + Create `Core` and `Internal` implementations of `AbstractQueries` and `AbstractPersistenceUtils`
  + Define `Core` and `Internal` implementations of each of the Arbiters
  + Define `Core` and `Internal` implementations of each of the Collectors

In general, the Core classes/traits inherit from the Abstract traits, and augmented Internal classes/traits inherit 
from Core. This linear inheritance model helps decrease code duplication, as the Internal classes will only need to 
override methods that use custom augmented internal-only columns/tables.

## Slick
We use [Slick](http://slick.lightbend.com/) to generate classes that allow us to interact with the database using 
nice Scala collection-like functions. However, there are several caveats that make generic persistence difficult.
(Spoiler: we solved all our problems using [type classes](#winning-the-war-type-classes)).

#### Problem 1: Case to Case Inheritance
Slick generates case classes that represent a row in a table. e.g. `case class BuildRow(...)`. If we wish to 
override the return type of methods defined in `AbstractQueries` with more specific types, we need these case classes
to inherit from each other. 
[However, case-to-case inheritance is prohibited in Scala.](http://www.scala-lang.org/old/node/4962.html#comment-20411)
In order to circumvent this, we create wrapper classes for all of the core case classes. We then define 
implicit conversions between the case classes and the wrappers, thus allowing us to use them interchangeably and still
adhere to our linear inheritance model. The wrapper classes are then extended to create their custom augmented forms.

{{< highlight scala "linenos=" >}}
class BuildRowWrapper(...) extends BuildRowLike
case class BuildRow(...) extends BuildRowWrapper(...)
implicit def BuildRowWrapper2BuildRow(x: BuildRowWrapper): BuildRow = ...
implicit def BuildRow2BuildRowWrapper(x: BuildRow): BuildRowWrapper = ...
{{< /highlight >}}

Fortunately, all of this can be generated automatically by extending Slick's [source code generator](https://github.com/slick/slick-codegen-example).

#### Problem 2: Slick Queries are [Invariant](https://docs.scala-lang.org/tour/variances.html)
{{< highlight scala "linenos=" >}}
sealed abstract class Query[+E, U, C[_]]
{{< /highlight >}}

The first type parameter `+E` (which is covariant) corresponds to the autogenerated Table class (which extends `profile.api.Table[U]`). The 
second type parameter `U` corresponds to the autogenerated case class representing the Row. 
The third type parameter `C[_]` is the container type
for the query results (usually `Seq`). Unfortunately, because `U` is declared as invariant, `Query[A, B', C]` is not 
considered a subclass of `Query[A, B, C]`, even if `B'` is a subclass of `B`. This is problematic because several of 
methods are defined with return types of `Query`. If we use these return types in the abstract definitions, then we 
cannot override them with more specific types in the Core/Internal implementations.

Our workaround is using `DBIO` instead of `Query` when we need covariance. This means that we 
must use for-comprehensions when dealing with the results, instead of directly using collection-like methods.

## The Database Inheritance Model
While subclassing and inheritance does not really apply to databases... we're going to do it anyway. The Internal 
database can be considered a "subclass" of the Core database, as it completely encompasses the Core database, adding 
columns and tables as necessary.

There are essentially three "layers" to our model that occur everywhere: an abstract layer, a Core 
implementation layer, and an augmented Internal implementation layer, with each layer inheriting from the one above.

Our database model is as follows:

* `com.workday.warp.persistence.model.TablesLike.scala`:

    A trait `TablesLike` containing all of the supertraits describing the required tables in the Core Schema. e.g.
{{< highlight scala "linenos=" >}}
trait BuildRowLike {
	val idBuild: Int
	val year: Int
	val week: Int
	val buildNumber: Int
	val firstTested: java.sql.Timestamp
	val lastTested: java.sql.Timestamp
}
{{< /highlight >}}

    This trait will also contain the type class definitions for each `RowLike`
    (See: [Type Classes](#winning-the-war-type-classes))
     
* `com.workday.warp.persistence.model.Tables.scala`:

     This file contains Slick's autogenerated representation of the Core schema. It also contains the implicit 
     objects defining the type class for each Row.
     
* `com.workday.warp.persistence.model.internal.Tables.scala`:

    This file contains Slick's autogenerated representation of the Internal schema. It will look the same as the Core 
    `Tables.scala` except that the case classes defined in this file will inherit from their corresponding `Wrapper` 
    classes defined in the Core `Tables.scala`, which allows linear inheritance. e.g.
{{< highlight scala "linenos=" >}}
case class BuildRow(override val idBuild: Int,
					override val year: Int, override val week: Int,
					override val buildNumber: Int,
					confidenceLevel: String,
					override val firstTested: java.sql.Timestamp,
					override val lastTested: java.sql.Timestamp)
	 extends BuildRowWrapper(idBuild,year,week,buildNumber,firstTested,lastTested)
{{< /highlight >}}
  Notice how all of the overlapping columns have an `override` operator, while the new column `confidenceLevel` does
     not.
**There is one important limitation: the columns must be defined in the same order as the original columns, otherwise the 
    parameter list in the `extends` clause will be incorrect.** You can insert new columns, but you may not reorder the
    existing ones.
  
The database model is defined as a trait, so it cannot be used directly. Instead, we have defined a separate `Tables`
object in `com.workday.warp.persistence.(internal).Tables.scala` which mixes in the correct `model.Tables.scala` 
trait, defines the correct Slick `JdbcProfile`, and provides other convenience functions. In order to access these classes, 
please import this object. 
 
A corresponding `TablesLike` object has also been created in `com.workday.warp.persistence.TablesLike.scala` which
allows importing of the `TablesLike` traits.

The reason for creating accessor objects is because if the types are mixed in, then Scala will infer
[path-dependent types](http://danielwestheide.com/blog/2013/02/13/the-neophytes-guide-to-scala-part-13-path-dependent-types.html),
resulting in type mismatch errors.
  
## Queries, PersistenceUtils, and PersistenceAware
#### AbstractQueries.scala
This file defines all of the queries that Warp Core supports. These queries ideally should not be called directly 
through client code, but rather through the corresponding `PersistenceUtils`. In addition, you'll notice that all of 
the return types are defined in terms of `TablesLike`.

#### PersistenceAware.scala
This is a trait that can be mixed in to enforce the definition of some `PersistenceUtils`. It defines a 
protected trait `AbstractPersistenceUtils` which defines utility functions for interacting with the database. Notice 
that `initUtils()` is called in the body. `initUtils()` is defined as an abstract function which should be overridden
to initialize the database (create the schema, insert seed data, etc.). This ensures that the queries will not throw 
exceptions.

Generally, we define a companion object which contains the initialization logic in its body. Because 
companion objects are lazily initialized, we override `initUtils()` by calling something defined in the object.

`AbstractPersistenceUtils` and its subclasses are protected traits in order to prevent clients from 
creating a `PersistenceUtils` without initializing the correct schema first. As a result, `PersistenceUtils` can now 
only be accessed by classes or traits that mix in some form of `PersistenceAware`.

Originally this was achieved by initializing the schema when `PersistenceUtils` was initialized, but because 
`InternalPersistenceUtils` inherits from `CorePersistenceUtils`, this caused `initSchema()` to be called multiple 
times, resulting in the Core schema being loaded instead of the Internal schema. With the overridden `initUtils()` 
function, we ensure that the correct `initSchema()` is called and only called once.

#### CorePersistenceAware.scala and InternalPersistenceAware.scala
These files implement the Core and Internal versions of `PersistenceUtils`, respectively. `InternalPersistenceAware` is
not part of our open-source package. Note that 
`InternalPersistenceAware` extends `CorePersistenceAware`, allowing `InternalPersistenceUtils` to subclass 
`CorePersistenceUtils`. This is important because it allows us to easily create internal versions of classes (i.e. arbiters, 
collectors) by simply extending the core version and additionally mixing in `InternalPersistenceAware`.

#### Initializing the Schema with Flyway Migrations
The `initSchema()` function defined in the `PersistenceUtils` companion objects use 
[flyway](https://github.com/flyway/flyway) for database migrations. 

Internally, we use a completely different set of migrations.
Any schema migrator can mix in the trait `MigrateSchemaLike` and call the `migrate` method:
{{< highlight scala "linenos=" >}}
trait MigrateSchemaLike extends PersistenceAware {

  /**
    * Applies migrations to bring our schema up to date.
    * 
    * Only supported for MySQL, not H2.
    *
    * Recursively scans `locations` to look for unapplied migration scripts.
    * If `locations` is empty, we'll allow flyway to use its default locations.
    *
    * The type of each entry is determined by its prefix:
    *   - no prefix (or "classpath:" prefix) indicates a location on the classpath
    *   - a "filesystem:" prefix points to a directory on the file system
    *
    * @param locations locations we'll recursively scan for migration scripts.
    */
  def migrate(locations: Seq[String] = Seq.empty): Unit = { ...  }
}
{{< /highlight >}}

The migration to create the Core schema lives in the default flyway directory `db/migration/`.

#### A Failed Attempt at Generifying Persistence: Structural Types
[Structural Types](https://twitter.github.io/scala_school/advanced-types.html#structural), or "Duck Typing", defines
a type based on its interface. This allows you to pass in objects to a function as long as it satisfies the defined 
structure.
{{< highlight scala "linenos=" >}}
type TestExecutionRowLikeType = {
  val idTestExecution: Int
  val idTestDefinition: Int
  ...
}
{{< /highlight >}}

This defines a `TestExecutionRowLikeType` to be any object that contains the corresponding values. Therefore, all of our 
classes and case classes satisfy this structural type. After defining all of these types, we can define our functions as
follows:
{{< highlight scala "linenos=" >}}
def writeTestExecutionQuery(row: TestExecutionRowLikeType): DBIO[TestExecutionRowLike]
{{< /highlight >}}

Satisfyingly clean! We then define augmented Internal versions of the structural types in order to match on the type parameter
in the internal implementations.
{{< highlight scala "linenos=" >}}
val row: InternalTestExecutionRowLikeType = maybeRow match {
  case r: InternalTestExecutionRowLikeType => r
  case _ => throw new Exception // Alternatively: build a InternalTestExecutionRow with default values.
}
{{< /highlight >}}

Everything compiles and tests pass! Except for one insidious compiler warning:
```
a pattern match on a refinement type is unchecked
      case r: InternalTestExecutionRowLikeType => r
```

Defeated by type erasure! The issue here is that structural types are
[erased at runtime](https://stackoverflow.com/questions/1988181/pattern-matching-structural-types-in-scala), so the
match statement actually becomes something like:
{{< highlight scala "linenos=" >}}
val row = maybeRow match {
  case r: Object => r
  case _ => throw new Exception // Alternatively: build a InternalTestExecutionRow with default values.
}
{{< /highlight >}}

which will always succeed. Of course, this is not ideal; the code only works if the user passes in the correct 
`TestExecutionRow` type.

#### Winning the War: Type Classes
Our final (and successful!) attempt was type classes.
To quote the [official Scala team blog](https://blog.scalac.io/2017/04/19/typeclasses-in-scala.html):

> Type classes are a powerful and flexible concept that adds ad-hoc polymorphism to Scala.

Our solution uses type classes to achieve the same behavior has structural typing, but it also allows us 
to match on the parameter type. (With the added benefit of avoiding the overhead of reflection!)

Type classes allow us to define a contract which objects must satisfy. In our case it's the same as the structural 
type.

{{< highlight scala "linenos=" >}}
/** Type Class for TestExecutionRowLike **/
trait TestExecutionRowLikeType[T] {
  def idTestExecution(row: T): Int
  def idTestDefinition(row: T): Int
  def idBuild(row: T): Int
  def passed(row: T): Boolean
  def responseTime(row: T): Double
  def responseTimeRequirement(row: T): Double
  def startTime(row: T): java.sql.Timestamp
  def endTime(row: T): java.sql.Timestamp
}

/** Prove that TestExecutionRow belongs to the type class of TestExecutionRowLikeType **/
implicit object TestExecutionRowTypeClassObject extends TestExecutionRowLikeType[TestExecutionRow] {
  def idTestExecution(row: TestExecutionRow): Int = row.idTestExecution
  def idTestDefinition(row: TestExecutionRow): Int = row.idTestDefinition
  def idBuild(row: TestExecutionRow): Int = row.idBuild
  def passed(row: TestExecutionRow): Boolean = row.passed
  def responseTime(row: TestExecutionRow): Double = row.responseTime
  def responseTimeRequirement(row: TestExecutionRow): Double = row.responseTimeRequirement
  def startTime(row: TestExecutionRow): java.sql.Timestamp = row.startTime
  def endTime(row: TestExecutionRow): java.sql.Timestamp = row.endTime
  def tenant(row: TestExecutionRow): String = row.tenant
  def teamcityBuildId(row: TestExecutionRow): Int = row.teamcityBuildId
  def teamcityAgentInstance(row: TestExecutionRow): Int = row.teamcityAgentInstance
}
  
/** Implicit conversion from type class to TestExecutionRow **/
implicit def TestExecutionRowFromTypeClass[T: TestExecutionRowLikeType](x: T): TestExecutionRow =
  TestExecutionRow(implicitly[TestExecutionRowLikeType[T]].idTestExecution(x),
              implicitly[TestExecutionRowLikeType[T]].idTestDefinition(x),
              implicitly[TestExecutionRowLikeType[T]].idBuild(x),
              implicitly[TestExecutionRowLikeType[T]].passed(x), 
              implicitly[TestExecutionRowLikeType[T]].responseTime(x),
              implicitly[TestExecutionRowLikeType[T]].responseTimeRequirement(x),
              implicitly[TestExecutionRowLikeType[T]].startTime(x),
              implicitly[TestExecutionRowLikeType[T]].endTime(x))
  
override def writeTestExecutionQuery[T: TestExecutionRowLikeType](row: T): DBIO[TestExecutionRowLike]
{{< /highlight >}}

You'll notice that the type class `TestExecutionRowLikeType` takes some generic `T` which must define functions that return the column values. 
We then simply need to define these functions in an implicit object. This provides implicit evidence that `TestExecutionRow` is 
`TestExecutionRowLikeType`. As a result, when given some object of `TestExecutionRowLikeType`, we can use `implicitly` to 
access the relevant columns. Of course, this is rather unwieldy, so we provide *another* implicit conversion that 
converts from the type class to the case class. Keep in mind that all this boilerplate is generated using the Slick code generator.

It's important to note that the context bounds are actually syntactic sugar for implicit parameters:
{{< highlight scala "linenos=" >}}
// This function expands to
def writeTestExecutionQuery[T: TestExecutionRowLikeType](row: T): DBIO[TestExecutionRowLike]
  
// This function
def writeTestExecutionQuery[T](row: T)(implicit ev: TestExecutionRowLikeType[T]): DBIO[TestExecutionRowLike]
{{< /highlight >}}
When you call this function, Scala will look for implicit evidence that satisfies `TestExecutionRowLikeType[T]`. 
Consequently, this means that our implicit objects (e.g. `TestExecutionRowTypeClassObject`) need to be in scope. In 
general, these will be in scope as long as you import `Tables.RowTypeClasses._` or `TablesLike.RowTypeClasses._`. 
We've included a helpful `ImplicitNotFound` message that will be printed at compile time if Scala cannot find the 
correct implicit, such as:
```
Could not find an implicit value for evidence of type class TestExecutionRowLikeType[TestExecutionRow].
You might pass an (implicit ev: TestExecutionRowLikeType[TestExecutionRow]) parameter to your method or import Tables.RowTypeClasses._
```

The best part is that all of these type classes and implicit objects can be generated automatically with Slick's 
source code generator! Hurrah! We may have lost many battles, but we have won the war! In addition, we never have 
to lie to the compiler with `asInstanceOf`, which is keeping with our Workday core value of integrity :)

To truly understand what's happening with type classes, I highly recommend reading this series of blog posts by Aakash 
N S:
* [Type Classes in Scala](http://aakashns.github.io/type-class.html)
* [Better Type Class Pt. 1](http://aakashns.github.io/better-type-class.html)
* [Better Type Class Pt. 2](http://aakashns.github.io/better-type-class-2.html)

#### An Aside about Identifiers
Several of our queries look up rows based on some set of primary keys. In our augmented internal schema, this is usually the 
`methodSignature` and the `confidenceLevel` or the `idTestDefinition` and the `confidenceLevel`. In our open-source package, however,
there is no `confidenceLevel`, and rows are identified by either the `methodSignature` or the `confidenceLevel`. In 
order to define a common abstract function that these queries can implement, we need to combine these parameters and 
create functions that read from an `Identifier` (similar to a composite key).

We also achieved this with type classes.
{{< highlight scala "linenos=" >}}
trait IdentifierType[T] {
  def methodSignature(identifier: T): String
  def idTestDefinition(identifier: T): Int
}
{{< /highlight >}}

We then define a `CoreIdentifier` which implements this type class and an `InternalIdentifier` which includes the 
additional `confidenceLevel` field. The handling of the `Identifiers` is similar to how we handle the different `Row`
type classes, where the augmented internal implementations match on the `IdentifierType`.

## Summary
If you have a unique use case, it is possible to use WARP with a customized schema, however a simpler approach is 
using [tags]({{< ref "tags.md" >}}) for recording additional metadata.

1. Traits and type classes are autogenerated into `model.TablesLike.scala`
2. Table classes, case classes, and implicit type class objects are autogenerated into `model.Tables.scala`
3. Use the `TablesLike` and `Tables` objects defined in `com.workday.warp.persistence` to access the generated traits
4. Mix in some form of `PersistenceAware` and use the provided `this.persistenceUtils` in order to access the 
    persistence functions
5. Make sure you import the correct `Tables._`. Also don't forget `Tables.RowClassTypes._` to bring the implicit 
    objects in scope


This tweet by [Aakash N S](https://twitter.com/aakashns/status/587687158032412672) essentially summarizes the main 
lesson learned from this endeavor:

> As a rule of thumb, the answer to every question of the form “Can I do XYZ in #Scala?” is
“Yeah, totally man! Just use implicits.”