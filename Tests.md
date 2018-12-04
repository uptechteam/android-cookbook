# Definitions
Types of objects used in tests:

A **dummy** object is passed around but never used (e.g. parameters in method);  
A **fake** object has a simplified implementation;  
A **stub** class is a partial implementation for an interface or class;  
A **mock** object is a dummy implementation for an interface or a class in which you define the output of certain method calls.  

# Types of tests
## Unit tests
Unit testing is the practice of testing small pieces of code, typically individual functions, alone and isolated. If your test uses some external resource, like the network or a database, it’s not a unit test.

Unit tests should be fairly simple to write. A unit tests should essentially just give the function that’s tested some inputs, and then check what the function outputs is correct. In practice this can vary, because if your code is poorly designed, writing unit tests can be difficult. Because of that, unit testing is the only testing method which also helps you write better code – code that’s hard to unit test usually has poor design.

In a sense, unit testing is the backbone. You can use unit tests to help design your code and keep it as a safety net when doing changes, and the same methods you use for unit testing are also applicable to the other types of testing. All the other test types are also constructed from similar pieces as unit tests, they are just more complex and less precise.

Unit tests are also great for preventing regressions – bugs that occur repeatedly. Many times there’s been a particularly troublesome piece of code which just keeps breaking no matter how many times I fix it. By adding unit tests to check for those specific bugs, you can easily prevent situations like that. You can also use integration tests or functional tests for regression testing, but unit tests are much more useful because they are very specific, which makes it easy to pinpoint and then fix the problem.

When should you use unit testing? Ideally all the time, by applying test-driven development. A good set of unit tests do not only prevent bugs, but also improve your code design, and make sure you can later refactor your code without everything completely breaking apart.  

**Examples:**
* a method that parses a complex JSON response;
* method that maps one entity into another;
* method that merges 2 streams into one and combines the items from those streams;
* method that calculates a price based on business rules.
## Integration tests
As the name suggests, in integration testing the idea is to test how parts of the system work together – the integration of the parts. Integration tests are similar to unit tests, but there’s one big difference: while unit tests are isolated from other components, integration tests are not. For example, a unit test for database access code would not talk to a real database, but an integration test would.

Integration testing is mainly useful for situations where unit testing is not enough. Sometimes you need to have tests to verify that two separate systems – like a database and your app – work together correctly, and that calls for an integration test. As a result, when validating integration test results, you could for example validate a database related test by querying the database to check the database state is correct.

Integration tests are often slower than unit tests because of the added complexity. They also might need some set up or configuration, such as the setting up of a test database. This makes writing and maintaining them harder than unit tests, so you should focus on unit tests unless you absolutely need an integration test.

You should have fewer integration tests than unit tests. You should mainly use them if you need to test two separate systems together, or if a piece of code is too complex to unit test. But in the latter case, I would recommend fixing the code so it’s easy to unit test instead.

Integration tests can usually be written with the same tools as unit tests.

**Examples:**
* requesting data from network and saving it to the database;
* [Android components](https://developer.android.com/training/testing/integration-testing);
* purchasing flow;
* game flow;
* full login flow;
* basically anything that you can describe with the "flow" word.

## Functional tests
Functional testing is a software testing process used within software development in which software is tested to ensure that it conforms with all requirements. Functional testing is a way of checking software to ensure that it has all the required functionality that's specified within its functional requirements.

Functional testing is primarily is used to verify that a piece of software is providing the same output as required by the end-user or business. Typically, functional testing involves evaluating and comparing each software function with the business requirements. Software is tested by providing it with some related input so that the output can be evaluated to see how it conforms, relates or varies compared to its base requirements. Moreover, functional testing also checks the software for usability, such as by ensuring that the navigational functions are working as required.

Some functional testing techniques include smoke testing, white box testing, black box testing, unit testing and user acceptance testing.

# Android tools for testing
## JUnit


## Espresso
* IdlingResource

## Robolectric


# Resources
* https://developer.android.com/studio/test/
* https://developer.android.com/training/testing/fundamentals
* https://developer.android.com/training/testing/unit-testing/local-unit-tests
* https://developer.android.com/training/testing/espresso/
* https://www.youtube.com/watch?v=pK7W5npkhho
