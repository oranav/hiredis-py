# hiredis-py

[![Build Status](https://travis-ci.org/pietern/hiredis-py.png?branch=master)](https://travis-ci.org/pietern/hiredis-py)

Python extension that wraps protocol parsing code in
[hiredis](http://github.com/antirez/hiredis).
It primarily speeds up parsing of multi bulk replies.

## Install

hiredis-py is available on [PyPi](http://pypi.python.org/pypi/hiredis), and
can be installed with:

    easy_install hiredis

## Usage

The `hiredis` module contains the `Reader` class. This class is responsible for
parsing replies from the stream of data that is read from a Redis connection.
It does not contain functionality to handle I/O.

### Reply parser

The `Reader` class has two methods that are used when parsing replies from a
stream of data. `Reader.feed` takes a string argument that is appended to the
internal buffer. `Reader.gets` reads this buffer and returns a reply when the
buffer contains a full reply. If a single call to `feed` contains multiple
replies, `gets` should be called multiple times to extract all replies.

Example:

    >>> reader = hiredis.Reader()
    >>> reader.feed("$5\r\nhello\r\n")
    >>> reader.gets()
    'hello'

When the buffer does not contain a full reply, `gets` returns `False`. This
means extra data is needed and `feed` should be called again before calling
`gets` again:

    >>> reader.feed("*2\r\n$5\r\nhello\r\n")
    >>> reader.gets()
    False
    >>> reader.feed("$5\r\nworld\r\n")
    >>> reader.gets()
    ['hello', 'world']

#### Unicode

`hiredis.Reader` is able to decode bulk data to any encoding Python supports.
To do so, specify the encoding you want to use for decoding replies when
initializing it:

    >>> reader = hiredis.Reader(encoding="utf-8")
    >>> reader.feed("$3\r\n\xe2\x98\x83\r\n")
    >>> reader.gets()
    u'☃'

When bulk data in a reply could not be properly decoded using the specified
encoding, it will be returned as a plain string. When the encoding cannot be
found, a `LookupError` will be raised after calling `gets` for the first reply
with bulk data (identical to what Python's `unicode` method would do).

#### Error handling

When a protocol error occurs (because of multiple threads using the same
socket, or some other condition that causes a corrupt stream), the error
`hiredis.ProtocolError` is raised. Because the buffer is read in a lazy
fashion, it will only be raised when `gets` is called and the first reply in
the buffer contains an error. There is no way to recover from a faulty protocol
state, so when this happens, the I/O code feeding data to `Reader` should
probably reconnect.

Redis can reply with error replies (`-ERR ...`). For these replies, the custom
error class `hiredis.ReplyError` is returned, **but not raised**.

When other error types should be used (so existing code doesn't have to change
its `except` clauses), `Reader` can be initialized with the `protocolError` and
`replyError` keywords. These keywords should contain a *class* that is a
subclass of `Exception`. When not provided, `Reader` will use the default
error types.

## Benchmarks

The repository contains a benchmarking script in the `benchmark` directory,
which uses [gevent](http://gevent.org/) to have non-blocking I/O and redis-py
to handle connections. These benchmarks are done with a patched version of
redis-py that uses hiredis-py when it is available.

All benchmarks are done with 10 concurrent connections.

* SET key value + GET key
  * redis-py: 11.76 Kops
  * redis-py *with* hiredis-py: 13.40 Kops
  * improvement: **1.1x**

List entries in the following tests are 5 bytes.

* LRANGE list 0 **9**:
  * redis-py: 4.78 Kops
  * redis-py *with* hiredis-py: 12.94 Kops
  * improvement: **2.7x**
* LRANGE list 0 **99**:
  * redis-py: 0.73 Kops
  * redis-py *with* hiredis-py: 11.90 Kops
  * improvement: **16.3x**
* LRANGE list 0 **999**:
  * redis-py: 0.07 Kops
  * redis-py *with* hiredis-py: 5.83 Kops
  * improvement: **83.2x**

Throughput improvement for simple SET/GET is minimal, but the larger multi bulk replies
get, the larger the performance improvement is.

## License

This code is released under the BSD license, after the license of hiredis.
