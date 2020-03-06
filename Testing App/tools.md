**The Legacy Code Dilemma**<br/>
*When we change code, we should have tests in place. To put tests in place, we often have to change code.*

**Navigation:**<br/>
\- [Automated refactoring](#automated-refactoring)<br/>
\- [Mock objects](#mock-objects)<br/>
\- [Unit tests harness](#unit-tests-harness)<br/>
\- [General testing harness](#general-testing-harness)<br/>

## Automated refactoring
Choose your refactoring tools with care.

Here is an example of refactoring tool from **JetBrains**:
> [Automatic Android Code Smell Refactoring Tool](https://plugins.jetbrains.com/plugin/12468-automatic-android-code-smell-refactoring-tool) This plugin is a tool that provides the developer with the ability to automatically detect and 
refactor Android-specific code smells in Android Studio. You can handle these code smells individually by IntelliJ inspections, or you can run the "Detect/Refactor Code Smells..." action across the entire project. 

When you have a tool that does refactorings for you, it's tempting to believe that you don't have to write tests for the code you are about to refactor.
In some cases, that is true. If your tool performs safe refactorings and you go from one automated refactoring to another without doing any other editing,
you can assume that your edits haven't changed behavior. 

However, this isn't always the case.
<p align="center">
	<kbd>
 		<img src="https://github.com/uptechteam/android-cookbook/blob/chapter/testing-tools/Testing App/assets/example-tools-highlight.png" alt="preview" width="720" height="230"/>
	</kbd>
</p>

This change clearly didn't preserve behavior.
<br/>It is a good idea to have tests around your code before you start to use automated refactoring. You *can* do some automated refactoring without tests, but you have to know what the tool is checking and what it isn't.

## Mock objects
Classes and therefore methods are final by default in Kotlin. Unfortunately, some libraries like Mockito are relying on subclassing which fails in these cases. What are the solutions to this?

- use interfaces
- open the class and methods explicitly for subclassing

**Create Mocks Once**</br>
Recreating mocks before every test is slow and requires the usage of lateinit var. So the variable can be reassigned which can harm the independence of each test.
```kotlin
// Don't 🚫
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class DesignControllerTest_RecreatingMocks {

    private lateinit var dao: DesignDAO
    private lateinit var mapper: DesignMapper
    private lateinit var controller: DesignController

    @BeforeEach
    fun init() {
        dao = mockk()
        mapper = mockk()
        controller = DesignController(dao, mapper)
    }

    // takes 2 s!
    @RepeatedTest(300)
    fun foo() {
        controller.doSomething()
    }
}
```
Instead, create the mock instance once and reset them before or after each test. It’s significantly faster (2 s vs 250 ms in the example) and allows using immutable fields with val.
```kotlin
// Do ✅
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class DesignControllerTest {

    private val dao: DesignDAO = mockk()
    private val mapper: DesignMapper = mockk()
    private val controller = DesignController(dao, mapper)

    @BeforeEach
    fun init() {
        clearAllMocks()
    }

    // takes 250 ms
    @RepeatedTest(300)
    fun foo() {
        controller.doSomething()
    }
}
```
**Handle Classes with State**</br>
The presented create-once-approach for the test fixture and the classes under test only works if they don’t have any state or can be reset easily (like mocks). In other cases, re-creation before each test is inevitable.
```kotlin
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class DesignViewTest {

    private val dao: DesignDAO = mockk()
    private lateinit var view: DesignView // the class under test has state

    @BeforeEach
    fun init() {
        clearAllMocks()
        view = DesignView(dao) // re-creation is required
    }

    @Test
    fun changeButton() {
        assertThat(view.button.caption).isEqualTo("Hi")
        view.changeButton()
        assertThat(view.button.caption).isEqualTo("Guten Tag")
    }
}
```

**Data Classes for Assertions** </br>
**Single Objects** </br>
If possible don’t compare each property for your object with a dedicated assertion. This bloats your code and - even more important - leads to an unclear failure message.
```kotlin
// Don't 🚫
val actualDesign = client.requestDesign(id = 1)

assertThat(actualDesign.id).isEqualTo(2) // ComparisonFailure
assertThat(actualDesign.userId).isEqualTo(9)
assertThat(actualDesign.name).isEqualTo("Cat")
assertThat(actualDesign.dateCreated).isEqualTo(Instant.ofEpochSecond(1518278198))
```
This leads to poor failure messages:
```kotlin
org.junit.ComparisonFailure: expected:<[2]> but was:<[1]>
Expected :2
Actual   :1
```
`Expected: 2. Actual: 1`? What is the semantics of the numbers? Design id or User id? What is the context/the containing class? Hard to say.

Instead, create an instance of the data classes with the expected values and use it directly in a single equality assertion.
```kotlin
// Do ✅
val actualDesign = client.requestDesign(id = 1)

val expectedDesign = Design(id = 2, userId = 9, name = "Cat", dateCreated = Instant.ofEpochSecond(1518278198))
assertThat(actualDesign).isEqualTo(expectedDesign)
```
This way, we get a nice and descriptive failure message:
```kotlin
org.junit.ComparisonFailure: expected:<Design(id=[2], userId=9, name=Cat...> but was:<Design(id=[1], userId=9, name=Cat...>
Expected :Design(id=2, userId=9, name=Cat, dateCreated=2018-02-10T15:56:38Z)
Actual   :Design(id=1, userId=9, name=Cat, dateCreated=2018-02-10T15:56:38Z)
```
We take advantage of Kotlin’s data classes. They implement `equals()` and `toString()` out of the box. So the equals check works and we get a really nice failure message. Moreover, by using named arguments, the code for creating the expected object becomes very readable.

**Group Assertions With `with()`**</br>
If you really want to assert only a few properties of a data class, consider grouping the assertions using the with() function.
```kotlin
with(dao.findDesign(1)) {
    assertThat(name).isEqualTo("cat")
    assertThat(userId).isEqualTo(10)
    assertThat(tags).containsOnly("Cat", "Animal")
}
```
**Helper Functions**</br>
**Use Helper Functions with Default Arguments to Ease Object Creation**</br>
In reality, data structures are complex and nested. Creating those objects again and again in the tests can be cumbersome. In those cases, it’s handy to write a utility function that simplifies the creation of the `data` objects. Kotlin’s default arguments are really nice here as they allow every test to set only the relevant properties and don’t have to care about the remaining ones.
```kotlin
fun createDesign(
    id: Int = 1,
    name: String = "Cat",
    date: Instant = Instant.ofEpochSecond(1518278198),
    tags: Map<Locale, List<Tag>> = mapOf(
        Locale.US to listOf(Tag(value = "$name in English")),
        Locale.GERMANY to listOf(Tag(value = "$name in German"))
    )
) = Design(
    id = id,
    userId = 9,
    name = name,
    fileName = name,
    dateCreated = date,
    dateModified = date,
    tags = tags
)

// Usage
// only set the properties that are relevant for the current test
val testDesign = createDesign()
val testDesign2 = createDesign(id = 1, name = "Fox") 
val testDesign3 = createDesign(id = 1, name = "Fox", tags = mapOf())
```
This leads to concise and readable object creation code.
- Don’t add default arguments to the `data` classes in the production code just to make your tests easier. If they are used only for the tests, they should be located in the test folder. So use helper functions like the one above and set the default arguments there.
- Don’t use `copy()` just to ease object creation. Extensive `copy()` usage is hard to read; especially with nested structures. Prefer the helper functions.
- Locate all helper functions in a single file like `CreationUtils.kt`. This way, we can reuse the functions like lego bricks for each test.

**Helper Extension Functions for Frequently Used Values**</br>
I prefer to use fixed values instead of randomized or changing values. But writing code like `Instant.ofEpochSecond(1525420010L)` again and again is annoying and blows the code.
```kotlin
// Don't 🚫
val date1 = Instant.ofEpochSecond(1L)
val date2 = Instant.ofEpochSecond(2L)
val date3 = Instant.ofEpochSecond(3L)
val uuid1 = UUID.fromString("00000000-000-0000-0000-000000000001");
val uuid2 = UUID.fromString("00000000-000-0000-0000-000000000002");
```
Fortunately, we can write extension functions for frequently used objects like `Instant`, `UUID`, etc.
```kotlin
fun Int.toInstant(): Instant 
    = Instant.ofEpochSecond(this.toLong())

fun Int.toUUID(): UUID 
    = UUID.fromString("00000000-0000-0000-a000-${this.toString().padStart(11, '0')}")

fun String.toObjectId(): ObjectId 
    = ObjectId(this.padStart(24, '0'))
```
The code will become concise while keeping the different values visible:
```kotlin
val date1 = 1.toInstant()
val date2 = 2.toInstant()
val date3 = 3.toInstant()
val uuid1 = 1.toUUID()
val uuid2 = 2.toUUID()
```
The failure messages can be easily traced back to the test code:
```kotlin
org.junit.ComparisonFailure:
Expected :00000000-0000-0000-a000-00000000001
Actual   :00000000-0000-0000-a000-00000000002
```
**Test-Specific Extension Functions**</br>
Extension Functions can be useful to extend an existing library in a natural way. Here is an example: AssertJ’s `isCloseTo()` requires a second `Offset` parameter. This leads to a lot of code duplication, bloated code, and distraction while reading the tests.
```kotlin
// Don't 🚫
assertThat(taxRate1).isCloseTo(0.3f, Offset.offset(0.001f))
assertThat(taxRate2).isCloseTo(0.2f, Offset.offset(0.001f))
assertThat(taxRate3).isCloseTo(0.5f, Offset.offset(0.001f))
```
Instead, define an extension function for `isCloseTo()` which delegates to an invocation with a fixed `Offset`. This method is specific for the current test class and can be placed there.
```kotlin
private fun AbstractFloatAssert<*>.isCloseTo(expected: Float) 
    = this.isCloseTo(expected, Offset.offset(0.001f))
```
This feels like overloading. The essential test logic becomes clearer and the code is still idiomatic as we stick to the fluent pattern.
```kotlin
// Usage:
assertThat(taxRate1).isCloseTo(0.3f)
assertThat(taxRate2).isCloseTo(0.2f)
assertThat(taxRate3).isCloseTo(0.5f)
```
The same technique can be used for all libraries with a fluent API.</br>
**Testcontainers: Reuse a Single Container**</br>
It’s better to create the container once and reuse it for every test. Kotlin’s `object` singletons and lazy initialized properties (by `lazy { }`) are very helpful here.
```kotlin
object MongoContainer {
    val instance by lazy { startMongoContainer() }

    private fun startMongoContainer() = KGenericContainer("mongo:4.0.2").apply {
        withExposedPorts(27017)
        setWaitStrategy(Wait.forListeningPort())
        start()
    }
}

// Usage:
class DesignDAOTest {
    private val container = MongoContainer.instance
    private val dao = DesignDAO(container.host, container.port) // pseudo-code
    @Test
    fun foo() { }
}
```
- This way, we don’t need any test framework integration (like a JUnit5 extension for [Testcontainers](https://www.testcontainers.org/)).
- Don’t forget to clean up the database before each test method using a `@BeforeEach` method.
- Be careful when you are running the [tests in parallel](https://junit.org/junit5/docs/current/user-guide/#writing-tests-parallel-execution). Consider using different “database names” (or “schema” in case of RDBs) for each test class to avoid side-effects.
- Don't use [in-memory DBs](https://phauer.com/2017/dont-use-in-memory-databases-tests-h2/)

## Unit tests harness

// TODO

## General testing harness

// TODO
