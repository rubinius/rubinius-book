# Measuring


The configuration options for metrics are accessible using the `-Xhelp` command line option:

         system.metrics.interval: Number of milliseconds between aggregation of VM metrics
                                  default: 10000
           system.metrics.target: Location to send metrics every interval: 'statsd', path
                                  default: none
    system.metrics.statsd.server: The [host:]port of the StatsD server
                                  default: localhost:8125
    system.metrics.statsd.prefix: Prefix for StatsD metric names
                                  default: host.$nodename.$pid.app.rbx
