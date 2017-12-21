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
tags = {
        'uuid': user_id,
        'request': request,
        'service_name': name,
        'service_version': "%s.%s" % (major, minor),
        'service_url': self._ftam_client.url()
}
with TraceScope('basnet.sendRequest', logger, details=tags) as ts:
    response = self._ftam_client.sendRequest(request)
```

* use @trace_me decorator to instrument a function of method

```python
@trace_me(logger, 'BcsStore')
    def get_stream(self, obj_hash, obj_type):
      ...
```

# Conditional logging
