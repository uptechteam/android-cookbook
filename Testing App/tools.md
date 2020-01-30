**The Legacy Code Dilemma**<br/>
*When we change code, we should have tests in place. To put tests in place, we often have to change code.*

**Navigation:**<br/>
\- [Automated refactoring](#automated-refactoring)<br/>
\- [Mock objects](#mock-objects)<br/>
\- [Unit tests harness](#unit-tests-harness)<br/>
\- [General testing harness](#general-tests-harness)<br/>

## Automated refactoring
Choose your refactoring tools with care.

Here is an example of refactoring tool from **JetBrains**:
> [Automatic Android Code Smell Refactoring Tool](https://plugins.jetbrains.com/plugin/12468-automatic-android-code-smell-refactoring-tool) This plugin is a tool that provides the developer with the ability to automatically detect and 
refactor Android-specific code smells in Android Studio. You can handle these code smells individually by IntelliJ inspections, or you can run the "Detect/Refactor Code Smells..." action across the entire project. 

When you have a tool that does refactorings for you, it's tempting to believe that you don't have to write tests for the code you are about to refactor.
In some cases, that is true. If your tool performs safe refactorings and you go from one automated refactoring to another without doing any other editing,
you can assume that your edits haven't changed behavior. 

However, this isn't always the case.

```Java
public class Example {

  private int alpha = 0;

  private int getValue() {
    alpha++;
    return 12;
  }

  public void doSomething() {
    int v = getValue();
    int total = 0;

    for (int n = 0; n < 10; n++) {
      total += v;
    }
  }

}
```

We can use tool to remove the `v` variable from `doSomething`. After the refactoring, the code looks like this:
```Java
public class Example {

  private int alpha = 0;

  private int getValue() {
    alpha++;
    return 12;
  }

  public void doSomething() {
    int total = 0;

    for (int n = 0; n < 10; n++) {
      total += getValue();
    }
  }

}
```
See the problem? The variable was removed, but now the value of `alpha` is incremented 10 times rather than 1. This change clearly didn't preserve behavior.
<br/>It is a good idea to have tests around your code before you start to use automated refactoring. You *can* do some automated refactoring without tests, but you have to know what the tool is checking and what it isn't.

## Mock objects

// TODO

## Unit tests harness

// TODO

## General testing harness

// TODO
