# Logging

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
