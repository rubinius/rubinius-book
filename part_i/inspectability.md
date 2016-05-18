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

1. [Logs](#logs)
2. [Diagnostics](#diagnostics)
3. [Metrics](#metrics)
4. [Time Profiles](#time-profiles)
5. [Analysis](#analysis)

Each of these are described in the next sections.

## Logs

Logging tells us, at a very high level, what the system is doing when running our program.
This includes information about how the system was invoked (the command line options), what components of the system are running, when a thread is created or completes running, and notable system conditions or events that may be warnings or errors.

The logging facility must be simple because it needs to be available across the lifetime of the system. It needs to be running as early as possible and until the last few instructions before the system finally exits.

Here is an example of what you may see in a Rubinius log file:

    May 18 10:26:39 [10618] command line: bin/rbx -e puts "Hi, logging world!"
    May 18 10:26:39 [10618] node info: Shirai-MacBook.local Darwin Kernel Version 15.4.0: Fri Feb 26 22:08:05 PST 2016; root:xnu-3248.40.184~3/RELEASE_X86_64
    May 18 10:26:39 [10618] start thread: ruby.main, 7564705, 0x504000
    May 18 10:26:39 [10618] process info: brianshirai rbx 10618 3.30.c28 2.2.2 2016-05-16 1551ff5c 3.6.2 JIT disabled
    May 18 10:26:39 [10618] loading CodeDB: /source/rubinius/rubinius/runtime/core
    May 18 10:26:41 [10618] exit thread: ruby.main 1.274358s
    May 18 10:26:41 [10618] exit process: 10618 0 1.307920s

By default, Rubinius simply writes log entries to a particular file, but you can configure other destinations.

On OS X, the default log file is `$TMPDIR/rbx-<username>.log`. On Unix/Linux systems, the default log file is `/tmp/rbx-<username>.log`.

The configuration options for logging are accessible using the `-Xhelp` command line option:

             system.log: Logging facility to use: 'syslog', 'console', or path
                         default: $TMPDIR/$PROGRAM_NAME-$USER.log
       system.log.limit: The maximum size of the log file before log rotation is performed
                         default: 10485760
    system.log.archives: The number of prior logs that will be saved as '$(system.log).N.Z' zip files
                         default: 5
      system.log.access: Permissions on the log file
                         default: 384
       system.log.level: Logging level: fatal, error, warn, info, debug
                         default: warn

## Diagnostics

In Rubinius, we use the term "diagnostics" to describe a variety of measures of system components. For example, diagnostics for the garbage collector include how many bytes are reserved by the memory regions and how many of those reserved bytes are being used by program objects.

Since most diagnostics are numbers, it may be confusing what the difference is between diagnostics and metrics. The main distinction is that diagnostics are more likely to include _state_ about a component and metrics are more likely to be a _measure of the work performed_ by the component.

Another difference is that diagnostics are typically emitted in response to some event (e.g. when the garbage collector cycle runs), while metrics are a regular metronome of time-series data. Metrics are described in detail in the section on Measuring.

In Rubinius, the diagnostics facility has two main parts: 1. various components track diagnostics information over time, and 2. a separate thread runs that periodically receives requests from the system components to report diagnostics data. The thread converts the data to JSON and writes it to the diagnostics target.

Other components, like the metrics and profiler, can use the diagnostics reporting mechanism to emit their data as well. This makes it possible to consume information about the system in a uniform manner, extracting various pieces of information from a [concatenated stream of JSON](https://en.wikipedia.org/wiki/JSON_Streaming).

The configuration options for diagnostics are accessible using the `-Xhelp` command line option:

    system.diagnostics.target: Location to send diagnostics: host:port, path
                               default: none

## Metrics

The configuration options for metrics are accessible using the `-Xhelp` command line option:

         system.metrics.interval: Number of milliseconds between aggregation of VM metrics
                                  default: 10000
           system.metrics.target: Location to send metrics every interval: 'statsd', path
                                  default: none
    system.metrics.statsd.server: The [host:]port of the StatsD server
                                  default: localhost:8125
    system.metrics.statsd.prefix: Prefix for StatsD metric names
                                  default: host.$nodename.$pid.app.rbx

## Time Profiles

Profiling is a form of diagnostic inquiry focused on a particular resource, the CPU. We want to discover what code is consuming the most CPU time. Knowing this can help us improve the efficiency of our program.

With metrics, the numbers are primary and the particular component being measured is generalized. For example, we look at the number of bytes written, but value of individual bytes is ignored. Every byte is treated like a general byte. We don't count the number of `b`s written.

In contrast, we are most concerned about the particular code being executed when profiling. We are measuring the time a piece of code runs, and the measurement is important for ordering the results, but identifying the pieces of code using the most time is the goal.

The configuration options for profiling are accessible using the `-Xhelp` command line option:

        system.profiler.target: Location to send profiler output: 'diagnostics', path
                                default: none
      system.profiler.interval: Report profiler results every N samples
                                default: 10000
    system.profiler.subprocess: Enable profiling in subprocesses created by fork

When the `-Xsystem.profiler.target=diagnostics` option is used, the diagnostics target must also be set. For example:

    rbx -Xsystem.diagnostics.target='./diagnistics-$PID.json' -Xsystem.profiler.target=diagnostics some_script.rb

## Analysis
