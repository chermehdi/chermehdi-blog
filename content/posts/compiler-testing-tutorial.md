+++
title = "Google compile testing tutorial"
date = 2020-05-11
author = "Mehdi Cheracher"
tags = ["java", "annotation-processors", "testing"]
keywords = ["java", "java-core", "testing"]
description = "how to use the google compiler testing library to test your annotation processors"
showFullContent = false
+++

## Introduction

- Today's topic is going to be in the form of a little tutorial on how to use
  the [Google compile testing library](https://github.com/google/compile-testing), it can be used to compile 
  Java source files and make assertions about compilation results (warnings,
  errors, generated files ...), and it integrates very well with annotation
  processors.


## Adding compile testing library to your project

- You can add the library to your test dependency list in `pom.xml` if you are using
  maven or `build.gradle` if you are using gradle.

  ```html
  <dependency>
    <groupId>com.google.testing.compile</groupId>
    <artifactId>compile-testing</artifactId>
    <version>0.17</version>
    <scope>test</scope>
  </dependency>
  ```


## API description

- The library comes with a small and rich API:

  - `com.google.testing.compile.Compiler`: which provides an API for compiling
    Java source files, with providing the ability to customize compilation
    options.
  - `com.google.testing.compile.Compilation`: which represents an Immutable
    object that is the result of a compilation process, and it gives you access
    to the List of `javax.tools.Diagnostics` *Which mainly represents 
    a problem at some position in a given file, could be a warning or an error*
  - `com.google.testing.compile.CompilationSubject`,
    `com.google.testing.compile.JavaFileObjectSubject`: These are [Google Truth](https://github.com/google/truth) subjects for fluent and easier
    assertions over both the `Compilation` and `JavaFileObject` objects.

## Under the hood

- Essentially the library will spare you from writing all the boilerplate
  associated with compiling Java files and dealing with classpath setting,
  resource allocation and freeing. The java files provided are compiled using an
  instance of `javax.tools.JavaCompiler`.
- Compiling a bunch of `JavaSourceFile` instances is done in the following steps:
  - Creates an `com.google.testing.compile.InMemoryJavaFileManager` which is an
    implementation of the `javax.tools.StandardJavaFileManager` required by the
    JavaCompiler tool.

  - Constructs the classpath from a list of given files (if any are specified).
  - Creates an instance of a `javax.tools.JavaCompiler.CompilationTask` which
    describes a future Compilation task that can be further configured before it
    gets started with the `java.util.Locale` that is going to be used when formatting
    diagnostic messages, and the list of annotation processors `javax.annotation.processor.Processor` to be applied
    as part of this compilation operation.
  - Creates a new Compilation object from the results of running the
    `CompilationTask` and returning it so that it can be used for tests.

## Usage example

- Let's start with some simple examples to familiarize ourselfs with the API
  described above. 
- Now imagine we want to test if some Java code generator is actually generating
  compilable Java code:

```java
public interface CodeGenerator {
  String generate(OperationContext context);
}
```

And we want to test weather the code compiles without any errors, this can be
done as follows:

```java
@Test
void testSuccessfullCodeGenerator() {
  Compilation compilation = Compilation.javac()
      .compile(JavaFileObjects.forSourceString(targetClassName, generator.generate(mockContext));

  CompilationSubject.assertThat(compilation).succeeded();
}
```

Now suppose we want to add some annotation processors that will process some
java test files that we have as part of our test resources, and make assertions
on the generated files:

```java
@Test
void testSuccessfullCodeGenerator() {
  SomeAnnotationProcessor pr = new SomeAnnotationProcessor();
  Compilation compilation = Compilation.javac()
      .withProcessors(pr)
      .compile(JavaFileObjects.forResource("TestFile.java"))

  CompilationSubject.assertThat(compilation).succeeded();
  CompilationSubject.assertThat(compilation)
    .generatedSourceFile("TestFileGen.java")
    .hasSourceEquivalentTo("TestFileGenExpected.java");
}
```

The `forResource` method is pretty handy, it will do a classpath lookup on
a file named `TestFile.java` and it will use it as the source for compilation.

In general you have multiple types of resources that you can use to create
a compilation object:
- Java source files via `JavaFileObjects.forResource("SomeJavaFile.java")`.
- Java jar files via `JavaFileObject.forResource("some-jar.jar")`.
- Java Html docs via `JavaFileObject.forResource("some-docs.html")`.
- Java source files in the form of simple Java strings via
  `JavaFileObjects.forString(...)` or `JavaFileObjects.forSourceLines(...)`.

You can also have finer grain assertions over the Compilations with an
annotation processors via the utilities provided by `CompilationSubject` class.

```java
@Test
void testFinerGrainAnnotationProcessorCompilationSubject() {
  SomeAnnotationProcessor pr = new SomeAnnotationProcessor();
  JavaFileObject targetFile = JavaFileObjects.forResource("TestFile.java");
  Compilation compilation = Compilation.javac()
      .withProcessors(pr)
      .compile(targetFile);

  CompilationSubject.assertThat(compilation).failed();
  // Asserting the number of errors
  CompilationSubject.assertThat(compilation)
    .hadErrorCount(2);
  
  // You can fine grain error messages much as you want
  CompilationSubject.assertThat(compilation)
    .hadErrorContainingMatch("Some Error regex[0-9]")
    .inFile(targetFile)
    .onLine(10);
}
```

The library also offers a more thorough way to test the generated files:

```java
@Tes
void testFinerGrainAnnotationProcessorJavaSourceObjectSubject() {
  SomeAnnotationProcessor pr = new SomeAnnotationProcessor();
  JavaFileObject targetFile = JavaFileObjects.forResource("TestFile.java");

  Compilation compilation = Compilation.javac()
      .withProcessors(pr)
      .compile(targetFile);

  CompilationSubject.assertThat(compilation).succeeded();

  ImmutableList<JavaFileObject> generatedFiles = compilation.generatedFiles();
  // mix and match with junit core assertions
  assertEquals(1, generatedFiles.size());
  JavaFileObject generatedFile = generatedFiles.get(0);
  // This is not a line by line checking, but an AST comparison between the two
  // source files, pretty cool 😇
  JavaFileObjectSubject.assertThat(generatedFile)
    .hasSourceEquivalentTo(someFileObject);
}
```

## Conclusion

- Using the Compile testing library is a good idea to cleanup tests that require
  Java source file compilation, The library provides all the basic blocks for
  doing so, and you can extend it for your use case by writing your own Junit
  extensions and rules.

## Resources

- [Google compile testing](https://github.com/google/compile-testing) 
- [Google Truth: assertion library](https://github.com/google/truth) 
- [Java tools
  package](https://docs.oracle.com/en/java/javase/11/docs/api/java.compiler/javax/tools/package-summary.html)


