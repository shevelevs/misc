# Overview

BQNT logging library is built as a layer on top of python 

There are a few basic considerations and requirements that influenced the design:
* normalized/standardized format that could be used by automatic processing and tools such as splunk
* multithreading support
* ability to trace the whole conversation (potentially over multiple threads or processes)
* ability to time operations in consistent way
* produce more output conditionally, i.e. in case of an error or timeout
* limited ability to easily link log records produced by standard python logging (for example, emitted from third party code) with records produced by our logging library

# Formatting 

Current version supports standardized formatting by 

* providing additional layer above the standard python that adds standardized pieces to every log record
* providing additional keyword *bqnt_extra* that can be used in the formatting string

%(asctime)s %(levelname)s %(name)s %(message)s *%(bqnt_extra)s*



# Trace Scope


# Conditional logging
