# Logging

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
