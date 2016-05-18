# Inspectability: Understanding Program Behavior

When we write a program, we have a specific expectation for the behavior we want from the program when it runs. But these expectations are almost whimsical. They are almost pure imagination. Nothing requires them, as symbols in a file or diagrams or whatever, to _behave_ in any particular way. It is only when we run our program that this pure imagination becomes real.

This fact of the difference between programs as written and programs as run does not depend on any characteristic of the particular programming language. A common fallacy is that a program in a statically typed language is more specific than a program in a dynamically typed language. Even the statically typed program must be run through the type-checker before assertions about the program are real.

Another fallacy is that interactive systems do not suffer from this divide between a program as written and a program as run. In fact, it does not matter how small we make the unit of execution, there is still some piece of code as expressed in symbols or diagrams, and then that piece of code running. The feedback may be much quicker if we are typing a line of code at a time into a REPL and it being run. But there are still two distinct states, and a single line of code in a high-level language can represent extensive, complex behavior.

So neither programming language characteristics nor the size of a unit of execution can bridge the gap between our expecations for our program and the program's actual behavior. We need something more. What we need is an _inspectable system_.

## An Inspectable System

An inspectable system is one that is designed to provide system information in a usable and efficient manner. Inspectability is a feature that is weaved into every component of a system. It cannot be added as an afterthought.

DTrace is an example of an inspectable system. It is comprised of two parts: the DTrace subsystem and the application probes that provide an interface between the application and the DTrace subsystem.

The DTrace subsytem includes a kernel component that executes a very specifically constrained language to gather application runtime data. The application probes are added to provide the DTrace system with specific access to application data.

DTrace separates monitoring the system from the system’s functionality, yet still provides sufficient flexibility to gather relevant information in a specific context.

Three principles guided the design of DTrace: 1. zero cost when not running, 2. stability and security (or it would not be allowed in production), and 3. reasonable cost when running (ie not just turning on a firehose that requires expensive post processing, but rather emiting only the specifically desired information and possibly doing initial data processing at the collection site.)

The principles behind the DTrace architecture guided the design of Rubinius as an inspectable system.

## A Hierarchy of Information

If the cost of collecting, transferring, and storing logs were negligible, all we would need for an inspectable system are logs.

A typical log entry contains: 1. a timestamp, 2. some categorization of the event, and 3. some description. Given enough log entries, we can understand exactly what an application is doing at a partiular point in time.

From the log entries, we can derive metrics: periodicity, duration, frequency, and other measures. To add monitoring, we could layer some sort of alerting system on these measures we derive from the logs.

We could also derive sequence (in what order events occurred), adjacency (which events occurred together), and direct causality (which events state they triggered other events). With this information, we can analyze the system behavior at a later time.

However, the cost of a log with enough detail to support an inspectable system is very high. We include efficiency in our definition of an inspectable system because no one will use a system that doesn't do the work they need in a reasonable amount of time.

To make the system efficient, we need a way to get the information we want while doing less work to get it. We can achieve this by separating different kinds of information.

In Rubinius, we separate the information mostly by ways we interact with a running program:

1. Logging
2. Diagnostics
3. Metrics
4. Profiling
5. Analysis

Each of these are described in the next sections.
