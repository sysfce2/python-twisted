
:LastChangedDate: $LastChangedDate$
:LastChangedRevision: $LastChangedRevision$
:LastChangedBy: $LastChangedBy$


.. _core-howto-logger-main:

Logging with twisted.logger
===========================


The Basics
----------

Logging consists of two main endpoints: applications that emit events, and observers that receive and handle those events.
An event is simply a ``dict`` object containing the relevant data that describes something interesting that has occurred in the application.
For example: a web server might emit an event after handling each request that includes the URI of requested resource, the response's status code, a count of bytes transferred, and so on.
All of that information might be contained in a pair of objects representing the request and response, so logging this event could be as simple as:

.. code-block:: python

    log.info(request=request, response=response)

The above API would seem confusing to users of many logging systems, which are built around the idea of emitting strings to a file.
There is, after all, no string in the above call.
In such systems, one might expect the API to look like this instead:

.. code-block:: python

    log.info(
        "{uri}: status={status}, bytes={size}, etc..."
        .format(uri=request.uri, status=response.code, size=response.size)
    )

Here, a string is rendered by formatting data from our request and response objects.
The string can be easily appended to a log file, to be later read by an administrator, or perhaps teased out via some scripts or log gathering tools.

One disadvantage to this is that a strings meant to be human-readable are not necessarily easy (or even possible) for software to handle reliably.
While text files are a common medium for storing logs, one might want to write code that notices certain types of events and does something other than store the event.

In the web server example, if many requests from a given IP address are resulting in failed authentication, someone might be trying to break in to the server.
Perhaps blocking requests from that IP address would be useful.
An observer receiving the above event as a string would have to resort to parsing the string to extract the relevant information, which may perform poorly and, depending on how well events are formatted, it may also be difficult to avoid bugs.
Additionally, any information not encoded into the string is simply unavailable; if the IP address isn't in the string, it cannot be obtained from the event.

However, if the ``request`` and ``response`` objects are available to the observer, as in the first example, it would be possible to write an observer that accessed any attributes of those objects.
It would also be a lot easier to write an observer that emitted structured information by serializing these objects into a format such as JSON, rows in a database, etc.

Events-as-strings do have the advantage that it's obvious what an observer that writes strings, for example to a file, would emit.
We can solve this more flexibly by providing an optional format string in events that can be used for this purpose:

.. code-block:: python

    log.info(
        "{request.uri}: status={response.status}, bytes={response.size}, etc...",
        request=request, response=response
    )

Note that this looks very much like the events-as-strings version of the API, except that the string is not rendered by the caller; we leave that to the observer, *if and when it is needed* .
The observer also has direct access to the request and response objects, as well as their attributes.
Now a text-based observer can format the text in a prescribed way, and an observer that wants to handle these events in some other manner can do so as well.


Usage for emitting applications
-------------------------------

The first thing that an application that emits logging events needs to do is to instantiate a :py:class:`Logger <twisted.logger.Logger>` object, which provides the API to emit events.
A :py:class:`Logger <twisted.logger.Logger>` may be created globally for a module:

.. code-block:: python

    from twisted.logger import Logger
    log = Logger()

    def handleData(data):
        log.debug("Got data: {data!r}.", data=data)

A :py:class:`Logger <twisted.logger.Logger>` can also be associated with a class:

.. code-block:: python

    from twisted.logger import Logger

    class Foo(object):
        log = Logger()

        def oops(self, data):
            self.log.error(
                "Oops! Invalid data from server: {data!r}",
                data=data
            )

When associated with a class in this manner, the ``"log_source"`` key is set in the event.
For example:

:download:`logsource.py <listings/logger/logsource.py>`

.. literalinclude:: listings/logger/logsource.py

This example will show the string "object with value 7 doing a task" because the ``log_source`` key is automatically set to the ``MyObject`` instance that the ``Logger`` is retrieved from.


Handling Failures
~~~~~~~~~~~~~~~~~

:py:class:`Logger <twisted.logger.Logger>` provides a :py:meth:`failuresHandled <twisted.logger.Logger.failuresHandled>` method, which allows one to run some application code, then capture a :py:class:`Failure <twisted.python.failure.Failure>` in the log stream if that code raises an exception:

.. code-block:: python

    from twisted.logger import Logger
    log = Logger()

    with log.failuresHandled("While doing some math:") as math:
        1 / 0

    if math.succeeded:
        print("Surprisingly, division by zero is possible.")

The emitted event will have the ``"log_failure"`` key set, which is a :py:class:`Failure <twisted.python.failure.Failure>` that captures the exception.
This can be used by my observers to obtain a traceback.
For example, :py:class:`FileLogObserver <twisted.logger.FileLogObserver>` will append the traceback to its output::

    While doing some math:

    Traceback (most recent call last):
    --- <exception caught here> ---
      File "/tmp/test.py", line 8, in <module>
        1/0
    exceptions.ZeroDivisionError: integer division or modulo by zero

This API is meant to capture **unexpected and otherwise unhandled** errors (in other words: bugs, which is why tracebacks are preserved).
As such, it defaults to logging at the :py:attr:`critical <twisted.logger.LogLevel.critical>` level.
It is generally more appropriate to instead use ``log.error()`` when logging an expected type of error condition that was fully handled by your code.
To put it differently, use ``.failuresHandled(...)`` for handling bugs in your code, and ``.error()`` for handling errors in input data, user configuration, and the like.

The ``as`` clause binds an :py:class:`Operation <twisted.logger.Operation>` object that allows you to determine whether the code under the ``with`` block succeeded or failed, and inspect its failure if so.
For more performance-sensitive applications, a simpler context manager which does *not* provide an :py:class:`Operation <twisted.logger.Operation>` is also available, :py:meth:`failureHandler <twisted.logger.Logger.failureHandler>`.
Since :py:meth:`failureHandler <twisted.logger.Logger.failureHandler>` is designed to be created once and used repeatedly, it does not allow for passing any placeholder parameters or inspecting the result of the operation.  Using it would look more like this:

.. code-block:: python

     from twisted.logger import Logger
     log = Logger()

     dividingByZero = log.failureHandler("While doing some math:")

     def later() -> None:
         with dividingByZero:
             1 / 0

``.failuresHandled(...)`` and ``.failureHandler(...)`` are intended for frameworks (such as Twisted itself) which call out to “application code”, where misbehavior on the part of the application should not corrupt the state of the framework itself.
For example, a buggy protocol implementation will not cause the entire Twisted reactor to crash and exit, it will log a traceback, disconnect the protocol which caused the exception, and keep running.

Therefore, most of the time, you should be accepting the behavior of ``failuresHandled``, of catching everything via ``BaseException``.
However, if you require more customization of behavior beyond catching everything and continuing on, you can use :py:meth:`failure <twisted.logger.Logger.failure>` directly, like so:

.. code-block:: python

    try:
        1 / 0
    except ZeroDivisionError:
        log.failure("While dividing by zero, specifically:")

The ``.failure`` method requires a log message, but can discover an exception on the stack automatically, so if it is called from within an ``except:`` block it will do the right thing.

Namespaces
~~~~~~~~~~

All :py:class:`Logger <twisted.logger.Logger>` s have a namespace, which can be used to categorize events.
Namespaces may be specified by passing in a ``namespace`` argument to :py:class:`Logger <twisted.logger.Logger>` 's initializer, but if none is given, the logger will derive its namespace from the module name of the callable that instantiated it, or, in the case of a class, from the fully qualified name of the class.
A :py:class:`Logger <twisted.logger.Logger>` will add a ``log_namespace`` key to the events it emits.

In the first example above, the namespace would be ``some.module`` , and in the second example, it would be ``some.module.Foo`` .


Log levels
~~~~~~~~~~

:py:class:`Logger <twisted.logger.Logger>` s provide a number of methods for emitting events.
These methods all have the same signature, but each will attach a specific ``log_level`` key to events.
Log levels are defined by the :py:class:`LogLevel <twisted.logger.LogLevel>` constants container.
These are:

:py:attr:`debug <twisted.logger.LogLevel.debug>`

  Debugging events: Information of use to a developer of the software, not generally of interest to someone running the software unless they are attempting to diagnose a software issue.

:py:attr:`info <twisted.logger.LogLevel.info>`

  Informational events: Routine information about the status of an application, such as incoming connections, startup of a subsystem, etc.

:py:attr:`warn <twisted.logger.LogLevel.warn>`

  Warning events: Events that may require greater attention than informational events but are not a systemic failure condition, such as authorization failures, bad data from a network client, etc.
  Such events are of potential interest to system administrators, and should ideally be phrased in such a way, or documented, so as to indicate an action that an administrator might take to mitigate the warning.

:py:attr:`error <twisted.logger.LogLevel.error>`

  Error conditions: Events indicating a systemic failure.
  For example, resource exhaustion, or the loss of connectivity to an external system, such as a database or API endpoint, without which no useful work can proceed.
  Similar to warnings, errors related to operational parameters may be actionable to system administrators and should provide references to resources which an administrator might use to resolve them.

:py:attr:`critical <twisted.logger.LogLevel.critical>`

  Critical failures: Errors indicating systemic failure (ie. service outage), data corruption, imminent data loss, etc. which must be handled immediately.
  This includes errors unanticipated by the software, such as unhandled exceptions, wherein the cause and consequences are unknown.

In the first example above, the call to ``log.debug`` will add a ``log_level`` key to the emitted event with a value of :py:attr:`LogLevel.debug <twisted.logger.LogLevel.debug>` .
In the second example, calling ``self.log.error`` would use a value of :py:attr:`LogLevel.error <twisted.logger.LogLevel.error>` .

The above descriptions are simply guidance, but it is worth noting that log levels have a reduced value if they are used inconsistently.
If one module in an application considers a message informational, and another module considers a similar message an error, then filtering based on log levels becomes harder.
This is increasingly likely if the modules in question are developed by different parties, as will often be the case with externally source libraries and frameworks.
(If a module tends to use higher levels than another, namespaces may be used to calibrate the relative use of log levels, but that is obviously suboptimal.)
Sticking to the above guidelines will hopefully help here.


Emitter method signatures
~~~~~~~~~~~~~~~~~~~~~~~~~

The emitter methods (:py:meth:`debug <twisted.logger.Logger.debug>` , :py:meth:`info <twisted.logger.Logger.info>` , :py:meth:`warn <twisted.logger.Logger.warn>` , etc.) all take an optional format string as a first argument, followed by keyword arguments that will be included in the emitted event.

Note that all three examples in the opening section of this HOWTO fit this signature.
The first omits the format, which doesn't lend itself well to text logging.
The second omits the keyword arguments, which hostile to anything other than text logging, and is therefore ill-advised.
Finally, the third provides both, which is the recommended usage.

These methods are all convenience wrappers around the :py:meth:`emit <twisted.logger.Logger.emit>` method, which takes a :py:class:`LogLevel <twisted.logger.LogLevel>` as its first argument.


Format strings
~~~~~~~~~~~~~~


Format strings provide observers with a standard way to format an event as text suitable for a human being to read.  Formatting is accomplished using the function :py:func:`eventAsText <twisted.logger.eventAsText>`.
When writing a format string, take care to present it in a manner which would make as much sense as possible to a human reader.
Particularly, format strings need not be written with an eye towards parseability or machine-readability.
If you want to save your log events along with their structure and then analyze them later, see the next section, on :ref:`"saving events for later" <core-howto-logger-saving-events-for-later>` .

Format strings should be

``unicode`` , and use `PEP 3101 <http://www.python.org/dev/peps/pep-3101/>`_ syntax to describe how the event should be rendered as human-readable text.
For legacy support and convenience in python 2, UTF-8-encoded ``bytes`` are also accepted for format strings, but unicode is preferred.
There are two variations from PEP 3101 in the format strings used by this module:

#.
   Positional (numerical) field names (eg. ``{0}`` ) are not permitted.
   Event keys are not ordered, which means positional field names do not make sense in this context.
   However, this is not an accidental limitation, but an intentional design decision.
   As software evolves, log messages often grow to include additional information, while still logging the same conceptual event.
   By using meaningful names rather than opaque indexes for event keys, these identifiers are more robust against future changes in the format of messages and the information provided.
#.
   Field names ending in parentheses (eg. ``{foo()}`` ) will call the referenced object with no arguments, then call ``str`` on the result, rather than calling ``str`` on the referenced object directly.
   This extension to PEP 3101 format syntax is provided to make it as easy as possible to defer potentially expensive work until a log message must be emitted.
   For example, let's say that we wanted to log a message with some useful, but potentially expensive information from the 'request' object:

   .. code-block:: python

       log.info("{request.uri} useful, but expensive: {request.usefulButExpensive()}",
                request=request)

   In the case where this log message is filtered out as uninteresting and not saved, no formatting work is done *at all* ; and since we can use PEP3101 attribute-access syntax in conjunction with this parenthesis extension, the caller does not even need to build a function or bound method object to pass as a separate key.
   There is no support for specifying arguments in the format string; the goal is to make it idiomatic to express that work be done later, not to implement a full Python expression evaluator.


Event keys added by the system
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The logging system will add keys to emitted events.
All event keys that are inserted by the logging system with have a ``log_`` prefix, to avoid namespace collisions with application-provided event keys.
Applications should therefore not insert event keys using the ``log_`` prefix, as that prefix is reserved for the logging system.
System-provided event keys include:

``log_logger``

  :py:class:`Logger <twisted.logger.Logger>` object that the event was emitted to.

``log_source``

  The source object that emitted the event.
  When a :py:class:`Logger <twisted.logger.Logger>` is accessed as an attribute of a class, the class is the source.
  When accessed as an attribute of an instance, the instance is the source.
  In other cases, the source is ``None`` .

``log_level``

  The :py:class:`LogLevel <twisted.logger.LogLevel>` associated with the event.

``log_namespace``

  The namespace associated with the event.

``log_format``

  The format string provided for use by observers that wish to render the event as text.
  This may be ``None`` , if no format string was provided.

``log_time``

  The time that the event was emitted, as returned by :py:func:`time <time.time>` .

``log_failure``

  A :py:class:`Failure <twisted.python.failure.Failure>` object captured when the event was emitted.


Avoid mutable event keys
~~~~~~~~~~~~~~~~~~~~~~~~

Emitting applications should be cautious about inserting objects into events which may be mutated later.
While observers are called synchronously, it is possible that an observer will do something like queue up the event for later serialization, in which case the serialized object may be different than intended.


Capturing log events for testing
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you want to test that your code is logging the expected events, you can use the :py:func:`LogCapture <twisted.logger.capturedLogs>` context manager:

.. code-block:: python

    from twisted.logger import Logger, LogLevel, capturedLogs
    from twisted.trial.unittest import TestCase

    class SomeTests(TestCase):

        def test_capture(self):
            foo = object()

            with capturedLogs() as captured:
                self.log.debug("Capture this, please", foo=foo)

            self.assertTrue(len(captured) == 1)
            self.assertEqual(captured[0]["log_format"], "Capture this, please")
            self.assertEqual(captured[0]["log_level"], LogLevel.debug)
            self.assertEqual(captured[0]["foo"], foo)


.. _core-howto-logger-saving-events-for-later:

Saving events for later
-----------------------

For compatibility reasons, ``twistd`` will log to a text-based format by default.
However, it's much better to use a structured log file format which preserves information about the events being logged.
``twisted.logger`` provides two APIs: :py:func:`jsonFileLogObserver <twisted.logger.jsonFileLogObserver>` and :py:func:`eventsFromJSONLogFile <twisted.logger.eventsFromJSONLogFile>`, which allow you to save and retrieve structured log events with a basic level of fidelity.
Log events are serialized as JSON dictionaries, with serialization rules that are as lenient as possible; any unknown values are replaced with simple placeholder values.

``jsonFileLogObserver`` will create a log observer that will save events as structured data, like so:

:download:`saver.py <listings/logger/saver.py>`

.. literalinclude:: listings/logger/saver.py

And ``eventsFromJSONLogFile`` can load those events again; here, you can see that the event has preserved enough information to be formatted as human-readable again:

:download:`loader.py <listings/logger/loader.py>`

.. literalinclude:: listings/logger/loader.py

You can also, of course, feel free to access any of the keys in the ``event`` object as you load them, as well; basic structures such as lists, numbers, dictionaries and strings (anything serializable with JSON) will be preserved:

:download:`loader-math.py <listings/logger/loader-math.py>`

.. literalinclude:: listings/logger/loader-math.py

..  TODO: command-line option for twistd to do this


Implementing an observer
------------------------

An observer must provide the :py:class:`ILogObserver <twisted.logger.ILogObserver>` interface.
That interface simply describes a 1-argument callable that takes a ``dict`` , so a simple implementation may simply use the handy :py:class:`provider <zope.interface.provider>` decorator on a function that takes one argument:

.. code-block:: python

    from zope.interface import provider
    from twisted.logger import ILogObserver, eventAsText

    @provider(ILogObserver)
    def simpleObserver(event):
        print(eventAsText(event))

The :py:func:`eventAsText <twisted.logger.eventAsText>` function returns a textual (``unicode`` ) representation of the event.

While it is recommended, in most cases it is not required that observers declare their compliance with :py:class:`ILogObserver <twisted.logger.ILogObserver>` .
This flexibility exists to allow for pre-existing callables and lambda expressions to be used as observers.
As an example, if one would like to accumulate events in a ``list`` , then ``list.append`` may be used as an observer.

When implementing your own log observer, however, you should always keep in mind that unlike most objects within Twisted, a log observer *must be thread safe* .

Specifically, a log observer:

- must be prepared to be called from threads other than the main thread (or I/O thread, or reactor thread)
- must be prepared to be called from multiple threads concurrently
- must not interact with other Twisted APIs that are not explicitly thread-safe without first taking precautions like using :py:meth:`callFromThread <twisted.internet.interfaces.IReactorFromThreads.callFromThread>`

Keep in mind that this is true even if you elect not to explicitly interact with any threads from your program.
Twisted itself may log messages from threads, and Twisted may internally use APIs like :py:meth:`callInThread <twisted.internet.interfaces.IReactorInThreads.callInThread>` ; for example, Twisted uses threads to look up hostnames when making an outgoing connection.

Given this extra wrinkle, it's usually best to see if you can find an existing log observer implementation that does what you need before implementing your own; thread safety can be tricky to implement.
Luckily, :py:mod:`twisted.logger` comes with several useful observers, which are documented below.


Writing an observer for event analysis
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Twisted includes log observers which take care of most of the "normal" uses of logging, like writing messages to a file, saving them as text, filtering them, and so on.
There are two common reasons you might want to write a log observer of your own.
The first is to ship log messages over to a different kind of external system which Twisted itself does not support.
If you've read the above, you now know enough to do that; simply implement a function that converts the dictionary to something amenable to your external system or file format, and send it there or save it.
The second task is to do some kind of analysis, either real-time within your process or after the fact offline.

You're probably going to have to aggregate information from your own code and from libraries you're using, including Twisted itself, and you may not have much in the way of control over how those messages are organized.
You're also probably trying to aggregate information from ad-hoc messages into some kind of structure.

One way to extract such semi-structured information is to feed your logs into an external system like `logstash <http://logstash.net>`_ , and process them there.
Such systems are quite powerful and ``twisted.logger`` does not try to replace them.
However, such systems are necessarily external, and therefore if you want to react to the log analysis *within* your application - for example, denying access in response to an abusive client - you would have to write some glue code to push messages from your log-analysis system back into your application.
For such cases, it's useful to be able to write log analysis code as a log observer.

Assuming that the libraries whose log events you're interested in analyzing are making use of ``twisted.logger`` , you can analyze log events either live as they're being logged, or loaded from a saved log file.

If you're writing code to log events directly for potential analysis, then you should simply structure your messages to include all necessary information as serialization-friendly values in events, and then simply pull them out, like the ``loader-math.py`` example above.

However, let's say you're trying to interact with a system that logs messages like so:

:download:`ad_hoc.py <listings/logger/ad_hoc.py>`

.. literalinclude:: listings/logger/ad_hoc.py

In this example, the ``AdHoc`` object is not itself serializable, but it has two relevant attributes, which the log message's format string calls out as attributes of the ``log_source`` field, as described above: ``a`` and ``b`` .

To analyze this event, we can't pursue the same strategy shown above in ``loader-math.py`` .
We don't have any key/value pairs of the log event to examine directly; ``a`` and ``b`` are not present as keys themselves.
We could look for the ``log_source`` key within the event and access its ``a`` and ``b`` attributes, but that wouldn't work once the event had been serialized and loaded again, since an ``AdHoc`` instance isn't a basic type that can be saved to JSON.

Luckily, :py:mod:`twisted.logger` provides an API for doing just this: :py:func:`extractField <twisted.logger.extractField>` .
You use it like so:

:download:`analyze.py <listings/logger/analyze.py>`

.. literalinclude:: listings/logger/analyze.py

This ``analyze`` function can then be used in two ways.
First, you can analyze your application "live", as it's running.

:download:`online_analyze.py <listings/logger/online_analyze.py>`

.. literalinclude:: listings/logger/online_analyze.py

If you run this script, you will see that it extracts the values from the ``AdHoc`` object's log message.

However, you can also analyze output from a saved log file, using the exact same code.
First, you'll need to produce a log file with a message from an AdHoc object in it:

:download:`ad_hoc_save.py <listings/logger/ad_hoc_save.py>`

.. literalinclude:: listings/logger/ad_hoc_save.py

If you run that script, it will produce a ``log.json`` file which can be analyzed in the same manner as the ``loader.py`` example above:

:download:`offline_analyze.py <listings/logger/offline_analyze.py>`

.. literalinclude:: listings/logger/offline_analyze.py

When doing analysis, it always seems like you should have saved more structured data for later.
However, log messages are often written early on in development, in an ad-hoc way, before a solid understanding has developed about what information will be useful and what will be redundant.
Log messages are therefore a stream-of-consciousness commentary on what is going on as a system is being developed.

``extractField`` lets you acknowledge the messy reality of how log messages are written, but still take advantage of structured analysis later on.
Just always be sure to use event format fields, not string concatenation, to reference information about a particular event, and you'll be able to easily pull apart hastily-written ad-hoc messages from multiple versions of a system, either as it's running or once you've saved a log file.

..  TODO: explain filtering


Registering an observer
-----------------------

One way to register an observer is to construct a :py:class:`Logger <twisted.logger.Logger>` object with it:

.. code-block:: python

    from twisted.logger import Logger
    from myobservers import PrintingObserver

    log = Logger(observer=PrintingObserver())

    log.info("Hello")

This will cause all of a logger's events to be sent to the given observer.
In the above example, all events emitted by the logger (eg. ``"Hello"`` ) will be printed.
While that is useful in some cases, it is common to register multiple observers, and to do so globally for all (or most) loggers in an application.


The global log publisher
------------------------

The global log publisher is a log observer whose purpose is to capture log events not directed to a specific observer.
In a typical application, the majority of log events will be emitted to the global log publisher.
Observers can register themselves with the global log publisher in order to be forwarded these events.

When a :py:class:`Logger <twisted.logger.Logger>` is created without specifying an observer to send events to, the logger will send its events to the global log publisher, which is accessible via the name :py:attr:`globalLogPublisher <twisted.logger.globalLogPublisher>` .

The global log publisher is a singleton instance of a private subclass of :py:class:`LogPublisher <twisted.logger.LogPublisher>` , which is itself an :py:class:`ILogObserver <twisted.logger.ILogObserver>` .
What this means is that the global log publisher accepts events like any other observer, and that it forwards those events to other observers.
Observers can be registered to be forwarded events by calling the :py:class:`LogPublisher <twisted.logger.LogPublisher>` method :py:meth:`addObserver <twisted.logger.LogPublisher.addObserver>` , and unregister by calling :py:meth:`removeObserver <twisted.logger.LogPublisher.removeObserver>` :

.. code-block:: python

    from twisted.logger import globalLogPublisher
    from myobservers import PrintingObserver

    log = Logger()

    globalLogPublisher.addObserver(PrintingObserver())

    log.info("Hello")

The result here is the same as the previous example, except that additional observers can be (and may already have been) registered.
We know that ``"Hello"`` will be printed.
We don't know, but it's very possible, that the same event will also be handled by other observers.

There is no supported API to discover what other observers are registered with a :py:class:`LogPublisher <twisted.logger.LogPublisher>` ; in general, one doesn't need to know.
If an application is running in ``twistd`` , for example, it's likely that an observer is streaming events to a file by the time the application code is in play.
If it is running in a ``twistd`` web container, there will probably be another observer writing to the access log.

A caveat here is that events are ``dict`` objects, which are mutable, so it is possible for an observer to modify an event that it sees.
Because doing so will modify what other observers will see, modifying a received event can be problematic and should be strongly discouraged.
It can be particularly harmful to edit or remove the content of an existing event key.
Furthermore, no guarantees are made as to the order in which observers are called, so the effect of such modifications on other observers may be non-deterministic.


Starting the global log publisher
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When the global log publisher is created, it uses a :py:class:`LimitedHistoryLogObserver <twisted.logger.LimitedHistoryLogObserver>` (see below) to store events that are logged by the application in memory until logging is started.
Logging is started by registering the first set of observers with the global log publisher by calling :py:meth:`beginLoggingTo <twisted.logger.LogBeginner.beginLoggingTo>` :

.. code-block:: python

    from twisted.logger import globalLogBeginner
    from myobservers import PrintingObserver

    log = Logger()

    log.info("Hello")

    observers = [PrintingObserver()]

    globalLogBeginner.beginLoggingTo(observers)

    log.info("Hello, again")

This:

* Adds the given observers (in this example, the ``PrintingObserver`` ) to the global log observer
* Forwards all of the events that were stored in memory prior to calling :py:meth:`beginLoggingTo <twisted.logger.LogBeginner.beginLoggingTo>` to these observers
* Gets rid of the :py:class:`LimitedHistoryLogObserver <twisted.logger.LimitedHistoryLogObserver>` , as it is no longer needed.

It is an error to call :py:meth:`beginLoggingTo <twisted.logger.LogBeginner.beginLoggingTo>` more than once.

.. note:: If the global log publisher is never started, the in-memory event buffer holds (a bounded number of) log events indefinitely.
	  This may unexpectedly increase application memory or CPU usage.
	  It is highly recommended that the global log publisher be started as early as feasible.


Provided log observers
----------------------

This module provides a number of pre-built observers for applications to use:

:py:class:`LogPublisher <twisted.logger.LogPublisher>`

  Forwards events to other publishers.
  This allows one to create a graph of observers.

:py:class:`LimitedHistoryLogObserver <twisted.logger.LimitedHistoryLogObserver>`

  Stores a limited number of received events, and can re-play those stored events to another observer later.
  This is useful for keeping recent logging history in memory for inspection when other log outputs are not available.

:py:class:`FileLogObserver <twisted.logger.FileLogObserver>`

  Formats events as text, prefixed with a time stamp and a "system identifier", and writes them to a file.
  The system identifier defaults to a combination of the event's namespace and level.

:py:class:`FilteringLogObserver <twisted.logger.FilteringLogObserver>`

  Forwards events to another observer after applying a set of filter predicates (providers of :py:class:`ILogFilterPredicate <twisted.logger.ILogFilterPredicate>` ).
  :py:class:`LogLevelFilterPredicate <twisted.logger.LogLevelFilterPredicate>` is a predicate that be configured to keep track of which log levels to filter for different namespaces, and will filter out events that are not at the appropriate level or higher.


Compatibility with standard library logging
-------------------------------------------

:py:class:`STDLibLogObserver <twisted.logger.STDLibLogObserver>` is provided for compatibility with the standard library's :py:mod:`logging <logging>` module.
Log levels are mapped between the two systems, and the various attributes of standard library log records are filled in properly.

Note that standard library logging is a blocking API, and logging can be configured to block for long periods (eg. it may write to the network).
No protection is provided to prevent blocking, so such configurations may cause Twisted applications to perform poorly.


Compatibility with twisted.python.log
-------------------------------------

This module provides some facilities to enable the existing :py:mod:`twisted.python.log` module to compatibly forward it's messages to this module.
As such, existing clients of :py:mod:`twisted.python.log` will begin using this module indirectly, with no changes to the older module's API.


Incrementally porting observers
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Observers have an incremental path for porting to the new module.
:py:class:`LegacyLogObserverWrapper <twisted.logger.LegacyLogObserverWrapper>` is an :py:class:`ILogObserver <twisted.logger.ILogObserver>` that wraps a log observer written for the older module.
This allows an old-style observer to be registered with a new-style logger or log publisher compatibly.
