---
title : "A brief overview of the JVM"
date : 2016-12-25
author : "Mehdi Cheracher"
cover : ""
tags : ["java", "jvm"]
keywords : ["java", "jvm"]
description : "A brief description of some of the components of the Java virtual machine"
showFullContent : false
---

Everyone knows about the Java programming language, but fewer people know what makes it so popular. One of the reasons is what’s going to be described here today: the JVM.

The JVM primary role is to convert java code to machine code (passing through bytecode of course) , it provides a runTime environment where the bytecode gets interpreted and executed. 

A few things to note:

- **The Java Virtual Machine is a specification**: meaning it’s just a formal description of a required  implementation. ([HotSpot](https://www.oracle.com/technetwork/java/hotspotfaq-138619.html), [GraalVM](https://graalvm.org) and [Zulu](https://www.azul.com/downloads/zulu-community/?&architecture=x86-64-bit&package=jdk) are pretty popular implementations).
- **The Java Virtual Machine is Platform Independent**: obviously because it acts as a layer between the operating system and the code.

The implementation of the Java Virtual Machine is called a Java Run Time environment *(JRE)* and when you execute your code (running the `java` command) a new Instance of the Java Virtual Machine is created.

{{< image src="/img/jvm-overview.gif" alt="Overview of the JVM" position="center" style="border-radius: 8px;" >}}

The figure above shows an the different parts (subsystems) of the JVM Specification,
we will briefly describe few of them later on.

One must know that the JVM performs a lot of tasks to execute your Java application, but essentially 
it boils down to this 4 steps:

1. Loading the Code .
2. Verifying the Code .
3. Executing the Code.
4. Providing a Runtime Environment.

### Description of JVM components

- **Class Loader**: It’s the component responsible of loading classes into memory and make them known to the runtime.
- **Method/Class Area** :It stores per-class structures such as the runtime constant pool, field and method data ..., it is initialized per JVM startup.
- **The Heap**: It's a data area, where all object are stored, and it's accessible by all the threads created by the JVM.
- **The Stack**: A Java Virtual Machine stack is analogous to the stack of a conventional language such as C . it holds local variables and partial results, and plays a part in method invocation and return.
When a new Thread is created and new Java Stack is associated with it’s creation , so every thread has his own stack.
- **Pc Registers**: It contains the address of the Java virtual machine instruction currently being executed.
- **Native Method Stacks**: It contains all the native methods used in the application.
- **The Execution Engine**: The component responsible of running your code, either via the interpreter or via JIT.

That's the overview of the components, see you in the next one.