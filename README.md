# Overview

BQNT logging library is built as a layer on top of python standard logging library

There are a few basic considerations and requirements that influenced the design:
* normalized/standardized format that could be used by automatic processing and tools such as splunk
* multithreading support
* ability to trace the whole conversation (potentially over multiple threads or processes)
* ability to time operations in consistent way
* ability to easily trace external calls
* produce more output conditionally, i.e. in case of an error or timeout
* limited ability to easily link log records produced by standard python logging (for example, emitted from third party code) with records produced by our logging library

# Formatting 

Current version supports standardized formatting by 

* providing additional layer above the standard python that adds standardized pieces to every log record
* providing additional keyword **bqnt_extra** that can be used in the formatting string

Example format string:

%(asctime)s %(levelname)s %(name)s %(message)s **%(bqnt_extra)s**

# Classes

Figure 1 shows a brief overview of main classes and the way they are related to standard python logging classes:

![Class Diagram](logging_classes.svg "Figure 1")

# Trace Scope

Trace scope is a fundamental concept that many other elements of the design are based on. It's the abstraction that models hierarchical structure of instrumented code. Each trace scope has an Id and a reference to its parent (via parent id). Current way of Id generation is hierarchical as well, which allows exploring parent/child relations by inspecting scope ids. Figiure 2 illustrates this on a simple example of interactions happening when handling the data upload message.

![Scope Tree](scope_tree.svg "Figure 2")

Implementation uses thread local storage since only one trace scope can be active at any given time within a given thread (assuming there's no *async io*).

Common use patterns for the TraceScope objects include:

* wrapping an operation in with(...) statement

```python
tags = { 'uuid': user_id, 'service_name': name, 'service_version': "%s.%s" % (major, minor) }
with TraceScope('basnet.sendRequest', logger, details=tags) as ts:
    response = self._ftam_client.sendRequest(request)
```

* use of the @trace_me decorator to instrument a function of method

```python
@trace_me(logger, 'BcsStore')
def get_stream(self, obj_hash, obj_type):
   ...
```

# Conditional logging

## Overview 
Conditional logging refers to the ability to dynamically lower the level of logging (i.e. output more details) in case of an error or a slow operation. For brevity, condition that triggers output of the detailed log information from the buffer will be referred to as *flush condition*. The basic idea behind this is buffering the set log records for each scope and flushing all of them or only a filtered portion depending on whether the flush condition occured. This is accomplished via *ScopedConditionalMemoryHandler* class which wraps around any log Handler and implements the buffering logic outlined above. It interacts with current TraceScope by retrieving scope details from thread local storage. 

## Flush level and default level
ScopedConditionalMemoryHandler can be customized by providing two different logging levels: default level and flush level. Default level is the one used when the end of a scope is reached and no flush condition is encountered. For example, if the default level is set to INFO, the code below will only emit log records of INFO level and above:
```python
with TraceScope("some-op", self.logger) as trace:
    trace.info(info_key="info_val")
    trace.debug(debug_key="debug_val")
    # no flush condition encountered, so DEBUG level logs are discarded at the end of the scope
```

Flush level is the one that defines a flush condition. Basically, flush condition is triggered by any logging at the customizable *flush_level* or above. For example, if the flush level set to *WARNING*, any log call with level of WARNING or ERROR will trigger the flush condition. Example:
```python
with TraceScope("some-op", self.logger, log_level=logging.INFO) as trace:
    trace.info(info_key="info_val")
    trace.debug(debug_key="debug_val")
    trace.error(error_key="error_val")
    # the .error call above will trigger flush condition and therefore the output of the DEBUG level info
```

Slow operation detection is implemented in TraceScope by providing a threshold latency above which a WARNING is emitted thus triggering a flush condition if the flush level is set to WARNING.
```python
with TraceScope("some-op", self.logger, warn_duration=1) as trace:
    trace.info(info_key="info_val")
    trace.debug(debug_key="debug_val")
    time.sleep(0.002) # sleep for 2 milliseconds, which is slower than warn_duration

# the "slowness" in the scope above will trigger a warning and thus a flush condition, if flush level is set to WARNING
```

There are two alternative approaches to the way scopes could be handled in case of a flush condition that make sense.

1. Bubble up. In this approach, flush conditions for each scope are handled independently of flush conditions in its sibling or child scopes. Once the flush condition is encountered, records for the current scope are flushed and flush condition is propagated to its parent scope. This ensures that we output all the details that led to the error, but if the error is occured in the parent scope, we don't need to output details of the child scope. Examples:


2. Trickle down. 
