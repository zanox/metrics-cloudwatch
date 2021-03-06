CloudWatch integration for codahale metrics
===========================================
This is a [metrics reporter implementation](https://github.com/codahale/metrics/blob/master/metrics-core/src/main/java/com/codahale/metrics/ScheduledReporter.java)
from [codahale metrics](http://metrics.codahale.com/) (v3.x) to [Amazon CloudWatch](http://aws.amazon.com/cloudwatch/).


### Metric submission types ###

These translations have been made to CloudWatch. Generally only the atomic data is submitted so that it can be
predictably aggregated via the CloudWatch API or UI. Codehale Metrics instances are NOT reset on
each CloudWatch report so they retain their original, cumulative functionality. The following`type` is submitted with
each metric as a CloudWatch Dimension.

| Metric    | Type           | sent statistic meaning per interval                                                     |
| --------- | -------------- | --------------------------------------------------------------------------------------- |
| Gauge     | gauge          | current value (if numeric)                                                              |
| Counter   | counterSum     | change in sum since last report                                                         |
| Meter     | meterSum       | change in sum since last report                                                         |
| Histogram | histogramCount | change in samples since last report                                                     |
|           | histogramSet   | CloudWatch StatisticSet based on Snapshot                                               |
| Timer     | timerCount     | change in samples since last report                                                     |
|           | timerSet       | CloudWatch StatisticSet based on Snapshot; sum / 1,000,000 (nanos -> millis)            |

`histogramSum` and `timerSum` do not submit differences per polling interval due to the possible sliding history
mechanics in each of them. Instead all available values are summed and counted to be sent as the simplified CloudWatch
StatisticSet. In this way, multiple submissions at the same time aggregate predictably with the standard CloudWatch UI
functions. As a consequence, any new process using these abstractions when viewed in CloudWatch as *sums* or *samples*
over time will steadily grow until the Codahale Metrics Reservoir decides to eject values: see
[Codahale Metrics: Histograms](http://metrics.codahale.com/manual/core/#histograms). Viewing these metrics as
*averages* in CloudWatch is usually the most sensible indication of represented performance.



Usage
-----


## Maven Dependency (Gradle) ##

    repositories {
        mavenCentral()
    }

    dependencies {
        compile 'com.blacklocus:metrics-cloudwatch:0.3.3'
    }

## Code Integration ##

    new CloudWatchReporter(
            metricRegistry,
            CloudWatchReporterTest.class.getSimpleName(),
            new AmazonCloudWatchAsyncClient()
    ).start(1, TimeUnit.MINUTES);
    // 1 minute lines up with CloudWatch resolution most naturally. Longer intervals could be used,
    // but I'm not sure of the implications.

If you already have a Codahale MetricsRegistry, you only need to give it to a CloudWatchReporter to start submitting
all your existing metrics code to CloudWatch. There are certain symbols which if part of metric names will result
in RuntimeExceptions in the CloudWatchReporter thread. These metrics should be renamed to avoid these symbols
to be used with the CloudWatchReporter.

See the test which generates bogus metrics from two simulated machines (threads):
[CloudWatchReporterTest.java](https://github.com/blacklocus/metrics-cloudwatch/blob/master/src/test/java/com/blacklocus/metrics/CloudWatchReporterTest.java)



Metric Naming
-------------

There is implicit support for CloudWatch Dimensions should you choose to use them. Any un-spaced portions of the metric
name that contain a '=' will be interpreted as CloudWatch dimensions. e.g. "CatCounter dev breed=calico" will result
in a CloudWatch metric with Metric Name "CatCounter dev" and one Dimension  { "breed": "calico" }.

Additionally, CloudWatch does not aggregate metrics over dimensions on custom metrics
([see CloudWatch documentation](http://docs.aws.amazon.com/AmazonCloudWatch/latest/DeveloperGuide/cloudwatch_concepts.html#Dimension)).
As a possible convenience to ourselves we can just submit these metrics in duplicate, once for each aggregation scope of interest for any combination of name tokens and dimensions. This is best understood by examples.

----------------

> Scenario metric: Number of Requests to Service X, is a Counter with name "ServiceX Requests"

We have multiple machines in the *Service X* cluster. We want metrics over all machines as well as per. To enable
duplicate submission per each scope (global and per-machine), we could name the counter
**ServiceX Requests machine={insert machine id here}**. At runtime this might turn out to be

#### (CloudWatch Dimensions support) ####

  - ServiceX Requests machine=1.2.3.4
  - ServiceX Requests machine=7.5.3.1

The = signifies a CloudWatch Dimension. This segment would thus be translated into a dimension with dimension
name *machine* and dimension value *1.2.3.4* or *7.5.3.1* depending on the machine. Each machine submits one metric to
CloudWatch. In the CloudWatch UI there would be only two metrics. But wait, now we have to add these together to get
the global count. For 2 machines that's not so bad, but what if we had more? This is the limitation of the CloudWatch UI
**and** API -- there is no way to aggregate across custom metrics' dimensions (save doing it yourself). So for our
own convenience we can construct the metric names at runtime as such.

  - ServiceX Requests machine=1.2.3.4*
  - ServiceX Requests machine=7.5.3.1*

The * at the end of a dimension (or plain token) segment signifies that this component is *permutable*. The metric must be
submitted at least once with this component, and once without. The CloudWatchReporter on each machine will resolve this
to 2 metrics each.

#### (Permutable name components for multiple scopes duplicate metric submission) ####

  - ServiceX Requests machine=1.2.3.4*
    * ServiceX Requests
    * ServiceX Requests machine=1.2.3.4
  - ServiceX Requests machine=7.5.3.1*
    * ServiceX Requests
    * ServiceX Requests machine=7.5.3.1

In this way now in the CloudWatch UI we will find 3 unique metrics, one for each individual machine with dimension={ip
address}, and one *ServiceX Requests* which scopes all machines together.

Both simple name tokens and dimensions can be permuted (appended with *). Note that this can produce a multiplicative
number of submissions. e.g. Some machine constructs this individualized metric name

> "ServiceX Requests group-tag* machine=1.2.3.4* strategy=dolphin* environment=development"

This resolves to all of the following submissions, so be careful about multiplicity.

  - ServiceX Requests environment=development
  - ServiceX Requests environment=development machine=1.2.3.4
  - ServiceX Requests environment=development strategy=dolphin
  - ServiceX Requests environment=development machine=1.2.3.4 strategy=dolphin
  - ServiceX Requests group-tag environment=development
  - ServiceX Requests group-tag environment=development machine=1.2.3.4
  - ServiceX Requests group-tag environment=development strategy=dolphin
  - ServiceX Requests group-tag environment=development machine=1.2.3.4 strategy=dolphin



License
-------

Copyright 2013 BlackLocus under [the Apache 2.0 license](LICENSE)

