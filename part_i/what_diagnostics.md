# Diagnostics

In Rubinius, we use the term "diagnostics" to describe a variety of measures of system components. For example, diagnostics for the garbage collector include how many bytes are reserved by the memory regions and how many of those reserved bytes are being used by program objects.

Since most diagnostics are numbers, it may be confusing what the difference is between diagnostics and metrics. The main distinction is that diagnostics are more likely to include _state_ about a component and metrics are more likely to be a _measure of the work performed_ by the component.

Another difference is that diagnostics are typically emitted in response to some event (e.g. when the garbage collector cycle runs), while metrics are a regular metronome of time-series data. Metrics are described in detail in the section on Measuring.

In Rubinius, the diagnostics facility has two main parts: 1. various components track diagnostics information over time, and 2. a separate thread runs that periodically receives requests from the system components to report diagnostics data. The thread converts the data to JSON and writes it to the diagnostics target.

Other components, like the metrics and profiler, can use the diagnostics reporting mechanism to emit their data as well. This makes it possible to consume information about the system in a uniform manner, extracting various pieces of information from a [concatenated stream of JSON](https://en.wikipedia.org/wiki/JSON_Streaming).

The configuration options for diagnostics are accessible using the `-Xhelp` command line option:

    system.diagnostics.target: Location to send diagnostics: host:port, path
                               default: none
