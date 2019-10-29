+++
title = "Design patterns: Event bus"
date = 2018-11-04
author = "Mehdi Cheracher"
tags = ["java", "desing-patterns", "core-java"]
keywords = ["java", "desing-patterns", "core-java"]
description = "An introduction to the Event bus design pattern, with implementation details"
showFullContent = false
+++

## Motivation

Imagine having a large scale application containing a lot of components interacting with each other, and you want a way to make your components communicate, while maintaining loose coupling and separation of concerns principles, the Event Bus pattern can be a good solution for your problem.

The idea of an Event bus is actually quite similar to the Bus studied in Networking (Bus Topology). you have some kind of pipeline and computers connected to it and whenever one of them sends a message it’s dispatched to all of the others, and then they decide if they want to consume the given message, or just discard it.

At a component level it’s quite similar, the computers are your application components and the message is the event or the data you want to communicate, the pipeline is your EventBus object.

## Implementation
There is no *correct* way to implement an Event Bus, i’m going to give a peak at two approaches here, finding other approaches is left as an exercise to the reader.

### First pattern

This one is kind of classic as it relays on defining your `EventBus` interface *(to force a given contract)* and implementing it the way your want, and define a `Subscribable` *(another contract)* to handle `Event` *(and yet another contract)* consumption. *(see code below)*.

```java
/**
 * interface describing a generic event, and it's associated meta data, it's this what's going to
 * get sent in the bus to be dispatched to intrested Subscribers
 *
 * @author chermehdi
 */
public interface Event<T> {

  /**
   * @returns the stored data associated with the event
   */
  T getData();
}
```

```java
import java.util.Set;

/**
 * Description of a generic subscriber
 *
 * @author chermehdi
 */
public interface Subscribable {

  /**
   * Consume the events dispatched by the bus, events passed as parameter are can only be of type
   * declared by the supports() Set
   */
  void handle(Event<?> event);

  /**
   * describes the set of classes the subscribable object intends to handle
   */
  Set<Class<?>> supports();
}
```

```java
/**
 * Description of the contract of a generic EventBus implementation, the library contains two main
 * version, Sync and Async event bus implementations, if you want to provide your own implementation
 * and stay compliant with the components of the library just implement this contract
 *
 * @author chermehdi
 */
public interface EventBus {

  /**
   * registers a new subscribable to this EventBus instance
   */
  void register(Subscribable subscribable);

  /**
   * send the given event in this EventBus implementation to be consumed by interested subscribers
   */
  void dispatch(Event<?> event);

  /**
   * get the list of all the subscribers associated with this EventBus instance
   */
  List<Subscribable> getSubscribers();
}
```

The `Subscribable` declares a method to handle a given type of objects, and also what type of objects it supports by defining the `supports` method.

The Event Bus implementation holds a List of all the Subscribables and notify all of them each time a new event comes to the EventBus `dispatch` method.

Opting for this solution gives you compile time checking of the passed Subscribables, and also it’s more Object Oriented way of doing it, no reflection magic needed, and as you can see it can be easy to implement.

The downside is that contract forcing thing, you always need a new class to handle a type of event, which might not be a problem at first, but as your project grows you’re going to find it a little bit repetitive to create a class just to handle simple logic, such as logging, or some metrics...

### Second pattern
This pattern is inspired from `Guava’s` implementation, the Event Bus implementation looks much simpler and easier to use. for every event consumer you can just annotate a given method with `@Subscribe` and pass it an object of the type you want to consume *(a single object/parameter)* and you can register it as a message consumer by just calling
`eventBus.register(objectContainingTheMethod);`
to produce a new `Event` all you have to do is call `eventBus.post(SomeObject)` and all the interested consumers will be notified .
What happens if no consumer is found for a given object, well .. nothing really in `Guava’s` implementation they call them `DeadEvents` in my implementation the call to post is just ignored.

```java
import java.lang.reflect.Method;
import java.util.Arrays;
import java.util.List;
import java.util.Map;
import java.util.Vector;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;

/**
 * Simple implementation demonstrating how a guava EventBus works generally, without all the noise
 * of special cases handling, and special guava collections
 *
 * @author chermehdi
 */
public class EventBus {

  private Map<Class<?>, List<Invocation>> invocations;

  private String name;

  public EventBus(String name) {
    this.name = name;
    invocations = new ConcurrentHashMap<>();
  }

  public void post(Object object) {
    Class<?> clazz = object.getClass();
    if (invocations.containsKey(clazz)) {
      invocations.get(clazz).forEach(invocation -> invocation.invoke(object));
    }
  }

  public void register(Object object) {
    Class<?> currentClass = object.getClass();
    // we try to navigate the object tree back to object ot see if
    // there is any annotated @Subscribe classes
    while (currentClass != null) {
      List<Method> subscribeMethods = findSubscriptionMethods(currentClass);
      for (Method method : subscribeMethods) {
        // we know for sure that it has only one parameter
        Class<?> type = method.getParameterTypes()[0];
        if (invocations.containsKey(type)) {
          invocations.get(type).add(new Invocation(method, object));
        } else {
          List<Invocation> temp = new Vector<>();
          temp.add(new Invocation(method, object));
          invocations.put(type, temp);
        }
      }
      currentClass = currentClass.getSuperclass();
    }
  }

  private List<Method> findSubscriptionMethods(Class<?> type) {
    List<Method> subscribeMethods = Arrays.stream(type.getDeclaredMethods())
        .filter(method -> method.isAnnotationPresent(Subscribe.class))
        .collect(Collectors.toList());
    checkSubscriberMethods(subscribeMethods);
    return subscribeMethods;
  }

  private void checkSubscriberMethods(List<Method> subscribeMethods) {
    boolean hasMoreThanOneParameter = subscribeMethods.stream()
        .anyMatch(method -> method.getParameterCount() != 1);
    if (hasMoreThanOneParameter) {
      throw new IllegalArgumentException(
          "Method annotated with @Susbscribe has more than one parameter");
    }
  }

  public Map<Class<?>, List<Invocation>> getInvocations() {
    return invocations;
  }

  public String getName() {
    return name;
  }
}
```

You can see that opting for this solution requires less work from your part, nothing prevents you from naming your handler methods **intention-revealing** names rather than a general handle. and you can define all your consumers on the same class you just need to pass an different event type for each method.

## Conclusion

Implementing an Event Bus pattern can be beneficial for your code base as it help loose coupling your classes and promotes a publish-subscribe pattern, it also helps components interact without being aware of each other.

Which implementation to follow is a matter of taste and requirements.

*Note*: code can be found in the [repo](http://github.com/chermehdi/event-bus)