
# Table of Contents
[Participate](#Participate)

[Introduction](#Introduction)

[Goals](#Goals)

[Non-Goals](#Non-Goals)

[Performance Interface](#Performance-Interface)

[PerformanceEntry Interface](#PerformanceEntry-Interface)

[PerformanceObserver Interface](#PerformanceObserver-Interface)

[Usage Example](#Usage-Example)

[Design Discussion](#Design-Discussion)

[Standards Status](#Standards-Status)

# Participate
[Issue tracker](https://github.com/w3c/performance-timeline/issues)

[Spec](https://w3c.github.io/performance-timeline/)

# Introduction
The Performance Timeline specification defines ways in which web developers can measure certain aspects of their web applications in order to make them faster.
It introduces two ways to obtain these measurements: a polling method via the Performance interface and a callback method via the PerformanceObserver.
The latter is the recommended way to measure in scenarios where there may be numerous measurements.

# Goals
This specification aims to:
* Provide a framework by which performance data can be exposed to web developers, to enable them to optimize their websites and provide better user experiences.
* Give an opinionated API to enable web developers to query performance data in an easy, performant, and predictable manner.

# Non-Goals
This specification does not concern itself with:
* Dictating which performance data is exposed to web developers.
* Providing any new measurements of any kind to web developers.

# Performance Interface
This specification extends the [Performance](#https://w3c.github.io/hr-time/#sec-performance) interface so that it can be used to query existing performance data about a web application.
Three polling methods are added to achieve this purpose:
* `getEntries()`: return all of the available entries.
  Not all performance data is available: the [registry](#https://w3c.github.io/timing-entrytypes-registry/#registry) notes which kind of performance data is exposed by this method via the `availableFromTimeline` flag.
* `getEntriesByType(type)`: return all of the available entries that are of the type passed on as a parameter.
* `getEntriesByName(name, optional type)`: return all of the available entries with the given name.
  If `type` is passed, this method only returns entries of the specified type.

In order to prevent the user agent from storing too much performance data in memory, there are limits to the number of entries of each type that are stored, and these are also in the registry as `maxBufferSize`.

# PerformanceEntry Interface
This specification defines the `PerformanceEntry` interface, which is used to host performance data of the web application.
A single `PerformanceEntry` object corresponds to one nugget of information about the performance of the website.
The entry has the following attributes:
* `name`: a string identifier for the object, also used to filter entries in the `getEntriesByName()` method.
* `entryType`: a string representing the type of performance data being exposed. It is also used to filter entries in the `getEntriesByType()` method and in the `PerformanceObserver`.
* `startTime`: a timestamp representing the starting point for the performance data being recorded. The semantics of this attribute depend on the `entryType`.
* `duration`: a time duration representing the duration of the performance data being recorded. The semantics of this one also depend on the `entryType`.

If these sound abstract, it’s because they are.
A specification whose goal is to expose new measurements to web developers will define a new interface which extends `PerformanceEntry`.
These are a few examples:
* [PerformanceResourceTiming](#https://w3c.github.io/resource-timing/#sec-performanceresourcetiming)
* [PerformanceNavigationTiming](#https://w3c.github.io/navigation-timing/#sec-PerformanceNavigationTiming)
* [PerformancePaintTiming](#https://w3c.github.io/paint-timing/#sec-PerformancePaintTiming)
* [PerformanceLongTaskTiming](#https://w3c.github.io/longtasks/#sec-PerformanceLongTaskTiming)
* [PerformanceMark](#https://w3c.github.io/user-timing/#performancemark) and [PerformanceMeasure](#https://w3c.github.io/user-timing/#performancemeasure)

# PerformanceObserver Interface
This specification defines the `PerformanceObserver` interface, which is used to register a callback that is run whenever some new performance data is logged about the web application.
A `PerformanceObserver` object can be notified of new `PerformanceEntry` objects that match the `entryType` values observed by the observer.

As there are many entryTypes and not all user agents support all of the entryTypes being proposed or incubated, the web developer can determine which of them are available via the `PerformanceObserver.supportedEntryTypes` static attribute.

When constructing a `PerformanceObserver`, the developer passes a callback as an argument.
This callback does not necessarily run once per `PerformanceEntry` nor immediately upon the creation of one.
Instead, entries are 'queued' at the `PerformanceObserver` level, and the user agent can asynchronously execute the callback later.
When the callback is executed, all queued entries are passed, and the queue for the `PerformanceObserver` is emptied.
The `PerformanceObserver` initially does not observe anything, so the `observe()` method is called to specify what kinds of entryTypes are to be observed.

To encourage developers to register performance-monitoring JavaScript code late on the page load, the `PerformanceObserver` is also equipped to receive entries that were logged before the observer was registered by setting the `buffered` flag in the `observe()` method.
In addition, the queued entries on the `PerformanceObserver` can be polled via the `takeRecords()` method, which is particularly useful when tearing down the performance monitoring code to ensure all data points are logged.
A `PerformanceObserver` will stop running callbacks when its disconnect() method is called.

# Usage Example

```js
// Helper to log a single entry.
function logEntry(entry => {
  const objDict = {
    "entry type":, entry.entryType,
    "name": entry.name,
    "start time":, entry.startTime,
    "duration": entry.duration
  };
  console.log(objDict);
});

const userTimingObserver = new PerformanceObserver(list => {
  list.getEntries().forEach(entry => {
    logEntry(entry);
  });
});

// Call to log all previous and future User Timing entries.
function logUserTiming() {
  if (!PerformanceObserver.supportedEntryTypes.includes("mark")) {
    console.log("Marks are not observable");
  } else {
    userTimingObserver.observe({type: "mark", buffered: true});
  }
  if (!PerformanceObserver.supportedEntryTypes.includes("measure")) {
    console.log("Measures are not observable");
  } else {
    userTimingObserver.observe({type: "measure", buffered: true});
  }
}

// Call to stop logging entries.
function stopLoggingUserTiming() {
  userTimingObserver.disconnect();
}

// Call to force logging queued entries immediately.
function flushLog() {
  userTimingObserver.takeRecords().forEach(entry => {
    logEntry(entry);
  });
}
```

# Design Discussion
This explainer was written long after the specification was standardized, so there is no “alternatives considered” section.
That said, the current API design has stood the test of time.
Over time, we’ve tried to encourage web developers to prefer the callback method of querying instead of polling, at least in cases where there’s more than one or two performance data points.
This is because `getEntries()` will return all of the entries, so calling it multiple times is going to be expensive.
In addition, using a callback approach allows the user agent to not have to store all of the entries during long-lived pages.
The registry has been a useful tool to document differences in behavior between different entryTypes.

The processing model is largely the same for all entryTypes.
Perhaps the only exception is Resource Timing, which has a secondary buffer and allows web developers to manually set the buffer size.
This was later deemed to introduce complexity and cause problems when there are multiple observers of the entries, so developer-set buffer sizes are not used in new specifications. Instead, the user agent is in charge of using a buffer size which is likely to meet the needs of most web applications, and is able to change such buffer size if feedback arises stating that there are missing entries.
We recently specified the `droppedEntriesCount` feature to enable this use-case.

Having both polling and callbacks has proved to be useful.
For entryTypes with a fixed amount of entries, like Paint Timing, using the polling method is more convenient, especially in the current state of the world where analytics providers often only send a single beacon per page load.
At the same time, the callback is more useful for cases where there is varying amount of entries of the given entryType, like in Resource Timing.
The web performance monitoring service can process all the performance information while the page is still running and only has to do very minimal work when the data needs to be reported.

# Standards Status
The Performance Timeline specification is widely approved.
There are differences in what kinds of performance data is exposed on different user agents, but this specification does not concern itself with that, and that is delegated to the new specifications that describe new data.
Most [tests](#https://wpt.fyi/results/performance-timeline?label=master&label=experimental&aligned) are green on all major user agents.
