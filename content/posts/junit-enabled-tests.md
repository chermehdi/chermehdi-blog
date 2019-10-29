+++
title = "JUnit 5: Injection enabled tests"
date = 2018-11-11
author = "Mehdi Cheracher"
tags = ["junit", "java", "tests", "junit5", "dependency-injection"]
keywords = ["junit", "java", "tests", "junit5", "dependency-injection"]
description = "an introduction on how to enable dependency injection in your JUnit 5 tests"
showFullContent = false
+++
## Introduction

We all are excited about the new release of JUnit and are happy about the modularisation of the platform and removing the restrictions that last version *(JUnit 4)* enforced.*( test methods must be public and have no arguments, and test classes must be public etc…)*

Today i’ll try and explain how to enable dependency injection in your test methods, say for testing services you need so often and you don’t want to write a @BeforeAll each time you write a test to initialize them.

This is how our end result will look like:

```java
class TestClass {
  
  @Test
  @DisplayName("DemoService should be injected in the method params")
  void testMethod(DemoService service) {
    // do some work with the service
  }
}
```

No need to mention that the idea of dependency injection inside our tests is not something new it is afforded by `spring`’s test utilities and it’s not that hard to implement for your general case, the following code snippet will show how you can create a costume `JUnit4` Runner to enable field injection inside your tests.

```java

import javax.enterprise.inject.se.SeContainer;
import javax.enterprise.inject.se.SeContainerInitializer;
import org.junit.runners.BlockJUnit4ClassRunner;
import org.junit.runners.model.InitializationError;

/**
 * @author chermehdi
 */
public class InjectionRunner extends BlockJUnit4ClassRunner {

  private SeContainer container;

  /**
   * Creates a BlockJUnit4ClassRunner to run {@code klass}
   *
   * @throws InitializationError if the test class is malformed.
   */
  public InjectionRunner(Class<?> klass) throws InitializationError {
    super(klass);
    SeContainerInitializer initializer = SeContainerInitializer.newInstance();
    container = initializer.initialize();
  }

  /**
   * inject the dependencies of the given object
   */
  @Override
  protected Object createTest() throws Exception {
    Object test = super.createTest();
    return container.select(test.getClass()).get();
  }
}
```
The implementation uses CDI to do the injection but that’s just an implementation detail .

and now in your test class you can just do:

```java
import static org.junit.jupiter.api.Assertions.assertEquals;

import javax.inject.Inject;
import org.junit.Test;
import org.junit.runner.RunWith;

/**
 * @author chermehdi
 */
@RunWith(InjectionRunner.class)
public class TestUsingRunner {

  @Inject
  SomeService service;

  @Inject
  SomeOtherService serviceOther;

  @Test
  public void helloTest() {
    assertEquals("i am a service depend", service.getString());
  }

  @Test
  public void helloTestAgain() {
    assertEquals("i am a service", serviceOther.getString());
  }
}
```
This was the old way of doing things, and that was the basics of how spring did its magic using the `@RunWith(SpringJUnit4ClassRunner.class)`.

Now How can we do the same thing but on the method level, in `JUnit4` we couldn’t, but now with the new release it’s quite easy to add support for it.

`JUnit5` comes with `Extensions` and they are classes that extend `JUnit5` to do more custom work .the `Extension` interface is just a marker interface that all extension classes must implement, that been said we can’t do much if we rely on it by itself .

The platform comes with a lot of interfaces and abstract classes that extend the Extension interface and that provide more methods that we can hook into and add our custom logic. To register an extension you just annotate your class with `@ExtendWith(MyExtension.class)`, register it programmatically via `@RegisterExtension` annotated field, or you can do it using the `ServiceLoader` API.

In our case, the platform comes with an interface extending from the Extension interface called `ParameterResolver`, this interface defines methods to hook into the test execution and try and resolves test methods parameters at runtime .
The example here is going to use `Java EE’s` Context and Dependency Injection api *CDI*, and its implementation `jboss.weld`, but a more abstract way is shown in the [repo](https://github.com/chermehdi/junit-di) so make sure to check it out and the demos included for a better understanding of the project.

We will be using a simple maven project structure so we’ll start by adding a dependency on *CDI* in our `pom.xml`

```html
<dependency>
  <groupId>org.jboss.weld.se</groupId>
  <artifactId>weld-se-core</artifactId>
  <version>3.0.1.Final</version>
</dependency>
```
and we will create a beans.xml file in our `test/resources/META-INF` directory *(this is specific to CDI)*

```html
<beans xmlns="http://xmlns.jcp.org/xml/ns/javaee"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/beans_1_1.xsd"
  version="1.1"bean-discovery-mode="all">
</beans>
```
Now we write our service class(es) ... 

```java
public class DemoService {
  public String demo(String value) {
    return value + " is a demo";
  }
}
```

Now all that’s left is to create the extension and hook in our DI logic:

```java
public class InjectionExtension implements ParameterResolver {

  private SeContainer container;

  /**
   * boot the CDI context
   */
  public InjectionExtension() {
    container = SeContainerInitializer.newInstance().initialize();
  }

  /**
   * determines weather we can inject all the parameters specified in the test method
   */
  @Override
  public boolean supportsParameter(ParameterContext parameterContext,
      ExtensionContext extensionContext) throws ParameterResolutionException {
    Method method = (Method) parameterContext.getDeclaringExecutable();
    Class<?>[] types = method.getParameterTypes();
    return Arrays.stream(types).allMatch(type -> container.select(type).isResolvable());
  }

  /**
   * resolve the return the object to be used in the test method
   */
  @Override
  public Object resolveParameter(ParameterContext parameterContext,
      ExtensionContext extensionContext) throws ParameterResolutionException {
    int paramIndex = parameterContext.getIndex();
    Method method = (Method) parameterContext.getDeclaringExecutable();
    Parameter param = method.getParameters()[paramIndex];
    return container.select(param.getType()).get();
  }
}
```
To use the extension all you have to do is:

```java
/**
 * @author chermehdi
 */
@ExtendWith(InjectionExtension.class)
public class TestExtension {

  @Test
  void testExtension(DemoService service) {
    assertNotNull(service);
  }
}
```
#### Explanation
The `supportsParameter` method is called on each parameter found in your test methods, and if it returns true, then resolveParameter is called, if not the test crashes with an error message.

Testing if we can provide the parameter or resolving the parameter is specific to each DI container, you just need to get a hold of the test method and you do the logic necessary to
resolve the parameter.
in this case we create an `SeContainer` and we try and resolve the parameter types, by selecting the type and using `isResolvable` method on the Instance returned .
That was it, make sure you visit the [repo](https://github.com/chermehdi/junit-di) of the project abstracting the idea, it’s called `junit-di`, the repo includes also demos using CDI and a `Map` based dependency injection mechanism. Happy Coding ...

