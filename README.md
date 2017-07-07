Metrics
=======

[![Build Status](https://travis-ci.org/mikejihbe/metrics.svg?branch=master)](https://travis-ci.org/mikejihbe/metrics)

A node.js port of codahale's metrics library: https://github.com/codahale/metrics

Metrics provides an instrumentation toolkit to measure the behavior of your critical systems while they're running in production.

License
---------
The MIT License (MIT)
Copyright (c) 2012 Mike Ihbe

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.


How to Use
----------

**Import metrics library**

```javascript
metrics = require('metrics');
```

**Create your metrics**

```javascript
var myCounter = new metrics.Counter;
var myMeter = new metrics.Meter;
var myHistogram = new metrics.Histogram;
var myTimer = new metrics.Timer;
```

**Start a metrics server**

```javascript
var metricsServer = new metrics.Server(config.metricsPort || 9091);
```

Servers are only one way to report your metrics.  It's actually a thin layer on top of metrics.Report, which you could use to build other reporting mechanisms.  Some other reporting methods include Grapite, console reporting, and CSV reporting.

**Add the metrics to the server**

```javascript
metricsServer.addMetric('myApp.groupA.thingA', myCounter);
metricsServer.addMetric('myApp.groupA.thingB', myHistogram);
metricsServer.addMetric('myApp.groupA.thingC', mySecondHisto);
metricsServer.addMetric('myApp.groupB.thingD', myMeter);
metricsServer.addMetric('myApp.groupB.thingE', myTimer);
```

**Setting up a Reporter**

A reporting interface exists for reporting metrics on a recurring interval.  Reporters can be found in [reporting/](reporting).

```javascript
// Report to console every 1000ms.
var report = new metrics.Report();
report.addMetric('com.co.thingA', counter);
var reporter = new metrics.ConsoleReporter(report);

reporter.start(1000);
```

Types of Metrics
----------------
This library includes four different types of metrics, listed here in increasing order of complexity. All metrics are created by calling:

```javascript
// Require metrics once
var metrics = require('metrics');
// Create your metric
var myMetric = metrics.Counter;
```

**Counter**

A Counter is a simple metric which tracks a single numerical value. It can be modified by calling `myMetric.inc(x)` or `myMetric.dec(x)`, where `x` is an integer. Counters are the simplest metric, and their output looks like:

```json
{
  "myMetricName": {
    "type": "counter",
    "count": 1
  }
}
```

**Meter**

A Meter is another relatively simple metric which tracks events over time.  It can be activated by calling `myMetric.mark()`, which marks an event.  Meters track the total number of events along with their frequency.  Output looks like this:

```json
{
  "myMetricName": {
      "type": "meter",
      "count": 2,
      "m1": 0.16929634497812285,
      "m5": 0.1934432200964012,
      "m15": 0.19779007785878447,
      "mean": 0.08485722771437057,
      "unit": "seconds"
    }
}
```

**Histogram**

A Histogram measures distribution of values and provides a lot more information than a Meter.  Histograms are modified by calling `myMetric.update(x)` where x is a numerical value.  Output:

```json
{
  "myMetricName": {
    "type": "histogram",
    "min": 12,
    "max": 52,
    "sum": 152,
    "variance": 250.3,
    "mean": 30.4,
    "std_dev": 15.820872289478858,
    "count": 5,
    "median": 35,
    "p75": 43.5,
    "p95": 52,
    "p99": 52,
    "p999": 52
  }
}
```

**Timer**

The Timer measures time between events, and stores the results in both a Meter and a Histogram.  That is, the frequency of events (technically `.stop()` calls) is recorded in a Meter, and the duration of events (the time between `.time()` and `.stop()`) is stored in a Histogram.  As mentioned, Timers are activated with `myMetric.time()` and `myMetric.stop()`.  Here's the output:

```json
"myMetricName": {
    "type": "timer",
    "duration": {
      "type": "histogram",
      "min": 0.1,
      "max": 0.25,
      "sum": 0.35,
      "variance": 0.15,
      "mean": 0.175,
      "std_dev": 0.106,
      "count": 2
    },
    "rate": {
      "type": "meter",
      "count": 2,
      "m1": 2,
      "m5": 2,
      "m15": 2,
      "mean": 2,
      "unit": "seconds"
    }
  }
```

Advanced Usage
--------------
Typical production deployments have multiple Node processes per server.  Rather than each process exposing metrics on different ports, it makes more sense to expose the metrics from the "master" process.  

Writing a thin wrapper around this api to perform the process communication is trivial, with a message passing setup, the client processes could look something like this:

```javascript
var Metric = exports = module.exports = function Metrics(messagePasser, eventType) {
  this.messagePasser = messagePasser;
  this.eventType = eventType;
}

Metric.prototype.newMetric = function(type, eventType) {
  this.messagePasser.sendMessage({
    method: 'createMetric'
    , type: type
    , eventType: eventType
  });
}
Metric.prototype.forwardMessage = function(method, args) {
  this.messagePasser.sendMessage({
    method: 'updateMetric'
    , metricMethod: method
    , metricArgs: args
    , eventType: this.eventType
  }); 
}

Metric.prototype.update = function(val) { return this.forwardMessage('update', [val]); }
Metric.prototype.mark = function(n) { return this.forwardMessage('mark', [n]); }
Metric.prototype.inc = function(n) { return this.forwardMessage('inc', [n]); }
Metric.prototype.dec = function(n) { return this.forwardMessage('dec', [n]); }
Metric.prototype.clear = function() { return this.forwardMessage('clear'); }
```

And the server side that receives the createMetric and updateMetric rpcs could look something like this:

```javascript
{
  createMetric: function(msg) {
      if (metricsServer) {
        msg.type = msg.type[0].toUpperCase() + msg.type.substring(1)
        metricsServer.addMetric(msg.eventType, new metrics[msg.type]);
      }
   }
  updateMetric: function(msg) {
    if (metricsServer) {
    var namespaces = msg.eventType.split('.')
      , event = namespaces.pop()
      , namespace = namespaces.join('.');
    var metric = metricsServer.trackedMetrics[namespace][event];
    metric[msg.metricMethod].apply(metric, msg.metricArgs);
  }
}
```

For multiple server deployments, you have more options, but the best approach will be highly application dependent. Best of luck, and always be tracking!
Using the metrics server you can hit the server on your configured port and you'll get a json representation of your metrics.  You should collect these periodically to generate timeseries to monitor the longterm health of your application.  The metrics.Reporting object would let you write to a log periodically or however else you'd like to expose your metrics.
