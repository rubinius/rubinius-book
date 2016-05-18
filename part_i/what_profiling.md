# Profiling

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
