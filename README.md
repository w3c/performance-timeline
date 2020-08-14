Performance Timeline
====================

### Overview

The PerformanceTimeline specification defines ways in which web developers can measure specific aspects of their web applications in order to make them faster.
It introduces two main ways to obtain these measurements: via getter methods from the [Performance](https://w3c.github.io/hr-time/#sec-performance) interface and via the PerformanceObserver interface.
The latter is the recommended way to reduce the performance impact of querying these measurements.

### PerformanceEntry

A PerformanceEntry object can host performance data of a certain metric.
A PerformanceEntry has 4 attributes: `name`, `entryType`, `startTime`, and `duration`.
This specification does not define concrete PerformanceEntry objects.
Examples of specifications that define new concrete types of PerformanceEntry objects are [Paint Timing](https://github.com/w3c/paint-timing), [User Timing](https://github.com/w3c/user-timing), [Resource Timing](https://github.com/w3c/resource-timing), and [Navigation Timing](https://github.com/w3c/navigation-timing/).

### Performance getters

The [Performance](https://w3c.github.io/hr-time/#sec-performance) interface is augmented with three new methods that can return a list of PerformanceEntry objects:

* `getEntries()`: returns all of the entries available to the [Performance](https://w3c.github.io/hr-time/#sec-performance) object.
* `getEntriesByType(type)`: returns all of the entries available to the [Performance](https://w3c.github.io/hr-time/#sec-performance) object whose `entryType` matches *type*.
* `getEntriesByName(name, type)`: returns all of the entries available to the [Performance](https://w3c.github.io/hr-time/#sec-performance) object whose `name` matches *name*.
If the optional parameter *type* is specified, it only returns entries whose `entryType` matches *type*.

#### Using the getters in JavaScript

The following example shows how `getEntriesByName()` could be used to obtain the first paint information:

```javascript
// Returns the FirstContentfulPaint entry, or null if it does not exist.
function getFirstContentfulPaint() {
  // We want the entry whose name is "first-contentful-paint" and whose entryType is "paint".
  // The getter methods all return arrays of entries.
  const list = performance.getEntriesByName("first-contentful-paint", "paint");
  // If we found the entry, then our list should actually be of length 1,
  // so return the first entry in the list.
  if (list.length > 0)
    return list[0];
  // Otherwise, the entry is not there, so return null.
  else
    return null;
}
```

### PerformanceObserver

A PerformanceObserver object can notified of new PerformanceEntry objects, according to their `entryType` value.
The constructor of the object must receive a callback, which will be ran whenever the user agent is dispatching new entries whose `entryType` value match one of the ones being observed by the observer.
This callback is not run once per PerformanceEntry nor immediately upon creation of a PerformanceEntry.
Instead, entries are 'queued' at the PerformanceObserver, and the user agent can execute the callback later.
When the callback is executed, all queued entries are passed onto the function, and the queue for the PerformanceObserver is reset.
The PerformanceObserver initially does not observer anything: the `observe()` method must be called to specify what kind of PerformanceEntry objects are to be observed.
The `observe()` method can be called with either an 'entryTypes' array or with a single 'type' string, as detailed below.
Those modes cannot be mixed, or an exception will be thrown.

#### PerformanceObserverCallback

The callback passed onto `PerformanceObserver` upon construction is a `PerformanceObserverCallback`. It is a void callback with the following parameters:

* `entries`: a `PerformanceObserverEntryList` object containing the list of entries being dispatched in the callback.
* `observer`: the `PerformanceObserver` object that is receiving the above entries.
* `hasDroppedEntries`: a `boolean` indicating whether `observer` is currently observing an `entryType` for which at least one entry has been lost due to the corresponding buffer being full. See the buffered flag [section](#buffered-flag).

#### `supportedEntryTypes`

The static `PerformanceObserver.supportedEntryTypes` returns an array of the `entryType` values which the user agent supports, sorted in alphabetical order.
It can be used to detect support for specific types.

#### `observe(entryTypes)`

In this case, the PerformanceObserver can specify various `entryTypes` values with a single call to `observe()`.
However, no additional parameters are allowed in this case.
Multiple `observe()` calls will override the kinds of objects being observed.
Example of a call: `observer.observe({entryTypes: ['resource', 'navigation']})`.

#### `observe(type)`

In this case, the PerformanceObserver can only specify a single type per call to the `observe()` method.
Additional parameters are allowed in this case.
Multiple `observe()` calls will stack, unless a call to observer the same `type` has been made in the past, in which case it will override.
Example of a call: `observer.observe({type: "mark"})`.

#### `buffered` flag

One parameter that can be used with `observe(type)` is defined in this specification: the `buffered` flag, which is unset by default.
When this flag is set, the user agent dispatches records that it has buffered prior to the PerformanceObserver's creation, and thus they are received in the first callback after this `observe()` call occurs.
This enables web developers to register PerformanceObservers when it is convenient to do so without missing out on entries dispatched early on during the page load.
Example of a call using this flag: `observer.observe({type: "measure", buffered: true})`.

Each `entryType` has special characteristics around buffering, described in the [registry](https://w3c.github.io/timing-entrytypes-registry/#registry). In particular, note that there are limits to the numbers of entries of each type that are buffered. When the buffer of an `entryType` becomes full, no new entries are buffered. A PerformanceObserver may query whether an entry was dropped (not buffered) due to the buffer being full via the `hasDroppedEntry` parameter of its callback.

#### `disconnect()`

This method can be called when the PerformanceObserver should no longer be notified of entries any more.

#### `takeRecords()`

This method returns a list of entries that have been queued for the PerformanceObserver but for which the callback has not yet run.
The queue of entries is also emptied for the PerformanceObserver.
It can be used in tandem with `disconnect()` to ensure that all entries up to a specific point in time are processed.

#### Using the PerformanceObserver

The following example logs all [User Timing](https://github.com/w3c/user-timing), [Resource Timing](https://github.com/w3c/resource-timing) entries by using a PerformanceObserver which observers marks and measures.

```javascript
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
