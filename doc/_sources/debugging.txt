
.. index:: debug, errors
.. _debugging:

Debugging
=========

When a parser fails to match some text it can be difficult (slow, frustrating
work) to understand why.  Fortunately, Lepl includes some features that make
life easier.

.. note::

  This section does not describe *known errors* (for example, generating an
  error message for the user when they enter text that is wrong in an expected
  way).  That issue is addressed in :ref:`errors`.  What is discussed here are
  the *unknown errors* you face when a parser fails to work with good input.


.. index:: log, logging, No handlers could be found for logger

Logging
-------

Lepl uses the standard `Python logging library
<http://docs.python.org/3.1/library/logging.html>`_ and so will display the
message::

  No handlers could be found for logger ...

if no logging has been configured.

The simplest way to configure logging is to add the following to your
program::

  from logging import basicConfig, DEBUG
  basicConfig(level=DEBUG)

To reduce the amount of logging displayed, you can use levels like ``ERROR``
and ``WARN``.

Logging is sent to loggers named after modules and classes.  You can tailor
the logging configuration to only display messages from certain modules or
classes.  See the `Python logging documentation
<http://docs.python.org/3.1/library/logging.html>`_ for more details.

Errors (indicating that the program has failed) use the ``ERROR`` level,
warnings (indicating that you may be mis-using the library) and stack traces
for errors use the ``WARN`` level, and general debug messages use the
``DEBUG`` level.

.. warning::

  Whenever you have a problem with Lepl, the first thing to do is enable
  ``DEBUG`` logging (see above).  And then read the logs.  Common messages and
  errors are described below.


.. index:: stack traces, format_exc, trampolining

Stack Traces
------------

Lepl 4 has improved trampolining which should give exceptions whose traceback
information identifies the source of any problem (earlier versions printed a
separate stack trace to the log --- that is no longer necessary).


.. index:: variable traces, TraceVariable

Variable Traces
---------------

The traces described later in this section give a very detailed picture of the
processing that occurs  within Lepl, but they are  difficult to understand and
show an overwhelming amount of information.

Often, to understand a problem, it is sufficient to see how the matchers
associated with variables are being matched.  This is displayed when the
variables are defined inside a ``with TraceVariables()`` scope::

  >>> with TraceVariables():
  ...     word = ~Lookahead('OR') & Word()
  ...     phrase = String()
  ...     with DroppedSpace():
  ...         text = (phrase | word)[1:] > list
  ...         query = text[:, Drop('OR')]
  ...
  >>> query.parse('spicy meatballs OR "el bulli restaurant"')
	phrase failed                             stream = 'spicy meatballs OR...
	  word = ['spicy']                        stream = ' meatballs OR "el ...
	phrase failed                             stream = 'meatballs OR "el b...
	  word = ['meatballs']                    stream = ' OR "el bulli rest...
	phrase failed                             stream = 'OR "el bulli resta...
	  word failed                             stream = 'OR "el bulli resta...
	phrase failed                             stream = ' OR "el bulli rest...
	  word failed                             stream = ' OR "el bulli rest...
	  text = [['spicy', 'meatballs']]         stream = ' OR "el bulli rest...
	phrase = ['el bulli restaurant']          stream = ''
	phrase failed                             stream = ''
	  word failed                             stream = ''
	  text = [['el bulli restaurant']]        stream = ''
	 query = [['spicy', 'meatballs'], ['el... stream = ''
  [['spicy', 'meatballs'], ['el bulli restaurant']]

The display above shows, on the left, the current match.  On the right is the
head of the stream (what is left after being matched).


.. index:: longest match, print_longest()
.. _deepest_match:

Deepest Matches
---------------

The `.config.full_first_match() <api/redirect.html#lepl.core.config.ConfigBuilder.full_first_match>`_ option,
enabled by default, gives a simple error indicating the deepest match within
the stream.  A more detailed report is also possible via
`.config.record_deepest() <api/redirect.html#lepl.core.config.ConfigBuilder.record_deepest>`_.

The following code is similar to that used in :ref:`getting-started`, but
fails to match the given input.  It has been modified to print information
about the longest match::

  >>> from logging import basicConfig, INFO
  >>> basicConfig(level=INFO)

  >>> name    = Word()              > 'name'
  >>> phone   = Integer()           > 'phone'
  >>> line    = name / ',' / phone  > make_dict
  >>> matcher = line[0:,~Newline()]
  >>> matcher.config.clear().record_deepest()
  >>> matcher.parse('andrew, 3333253\n bob, 12345')
  INFO:lepl.core.trace._RecordDeepest:
  Up to 6 matches before and including longest match:
  00105 '3333253\n' 8/1.9 (0008) 006 (['3333253'], (15, <helper>))  ->  Transform(And, TransformationWrapper(<add>))(8:'3')  ->  (['3333253'], (15, <helper>))
  00106 '3333253\n' 8/1.9 (0008) 005 (['3333253'], (15, <helper>))  ->  Transform(Transform, TransformationWrapper(<apply>))(8:'3')  ->  ([('phone', '3333253')], (15, <helper>))
  00107 'andrew...' 0/1.1 (0000) 004 ([('phone', '3333253')], (15, <helper>))  ->  And(And, Transform, Transform)(0:'a')  ->  ([('name', 'andrew'), ',', ' ', ('phone', '3333253')], (15, <helper>))
  00108 'andrew...' 0/1.1 (0000) 003 ([('name', 'andrew'), ',', ' ', ('phone', '3333253')], (15, <helper>))  ->  Transform(And, TransformationWrapper(<apply>))(0:'a')  ->  ([{'phone': '3333253', 'name': 'andrew'}], (15, <helper>))
  00113 '\n'        15/1.16 (0015) 004 next(Literal('\n')(15:'\n'))  ->  (['\n'], (16, <helper>))
  00114 '\n'        15/1.16 (0015) 005 (['\n'], (16, <helper>))  ->  Or(Literal, Literal)(15:'\n')  ->  (['\n'], (16, <helper>))
  Up to 2 failures following longest match:
  00123 ' bob, ...' 16/2.1 (0016) 008 next(NfaRegexp('[^ \t\n\r\x0b\x0c]', <Unicode>)(16:' '))  ->  stop
  00124 ' bob, ...' 16/2.1 (0016) 009 stop  ->  And(NfaRegexp, DepthFirst)(16:' ')  ->  stop
  Up to 2 successful matches following longest match:
  00139 'andrew...' 0/1.1 (0000) 002 stop  ->  DepthFirst(None, None, ([], <built-in function __add__>), rest=And, 0, first=Transform)(0:'a')  ->  ([{'phone': '3333253', 'name': 'andrew'}], (15, <helper>))

This looks a little daunting, but is extremely useful once you understand it.

The left column is a counter that increases with time.  The next column is the
stream, with offset information (line.character and total characters in
parentheses).  After that is the current stack depth.  Finally, there is a
description of the current action.

Lines are generated *after* matching, so the innermost of a set of nested
matchers is shown first.

The number of entries displayed is controlled by optional parameters supplied
to `.config.record_deepest() <api/redirect.html#lepl.core.config.ConfigBuilder.record_deepest>`_.

Looking at the output we can see that the first failure after the deepest
match was a `Lookahead() <api/redirect.html#lepl.matchers.core.Lookahead>`_ on the
input ``' bob, ...'``, after matching a newline, `Literal('\n')
<api/redirect.html#lepl.matchers.core.Literal>`_.  So we are failing to match a
space after the newline that separates lines --- this is why the original (see
:ref:`repetition`) had::

  >>> newline = spaces & Newline() & spaces
  >>> matcher = line[0:,~newline]

.. note::

   If you have a matcher that is failing you will need to use
   `.config.no_full_first_match() <api/redirect.html#lepl.core.config.ConfigBuilder.no_full_first_match>`_ to disable the error message, or you will
   not see the expected output.


.. index:: execution trace, Trace(), logging

Trace Output
------------

The same data can also be displayed to the logs with the `Trace() <api/redirect.html#lepl.matchers.monitor.Trace>`_ matcher.  This takes a
matcher as an argument --- tracing is enabled when the selected matcher is
called::

  >>> from logging import basicConfig, INFO
  >>> basicConfig(level=INFO)

  >>> name    = Word()                   > 'name'
  >>> phone   = Trace(Integer())         > 'phone'
  >>> line    = name / ',' / phone       > make_dict
  >>> matcher = line[0:,~Newline()]
  >>> matcher.config.clear().trace_stack()
  >>> matcher.parse('andrew, 3333253\n bob, 12345')
  INFO:lepl.core.trace._TraceResults:00176 '3333253\n'   1.8   (0008) 013 stop  ->  DepthFirst(0, 1, rest=Any, first=Any)('andrew, 3333253\n'[8:])  ->  ([], 'andrew, 3333253\n'[8:])
  INFO:lepl.core.trace._TraceResults:00177 '3333253\n'   1.8   (0008) 012 ([], 'andrew, 3333253\n'[8:])  ->  DepthFirst(0, 1, rest=Any, first=Any)('andrew, 3333253\n'[8:])  ->  ([], 'andrew, 3333253\n'[8:])
  INFO:lepl.core.trace._TraceResults:00182 '3333253\n'   1.8   (0008) 013 next(Any('0123456789')('andrew, 3333253\n'[8:]))  ->  (['3'], 'andrew, 3333253\n'[9:])
  INFO:lepl.core.trace._TraceResults:00184 '333253\n'    1.9   (0009) 013 next(Any('0123456789')('andrew, 3333253\n'[9:]))  ->  (['3'], 'andrew, 3333253\n'[10:])
  INFO:lepl.core.trace._TraceResults:00186 '33253\n'     1.10  (0010) 013 next(Any('0123456789')('andrew, 3333253\n'[10:]))  ->  (['3'], 'andrew, 3333253\n'[11:])
  INFO:lepl.core.trace._TraceResults:00188 '3253\n'      1.11  (0011) 013 next(Any('0123456789')('andrew, 3333253\n'[11:]))  ->  (['3'], 'andrew, 3333253\n'[12:])
  INFO:lepl.core.trace._TraceResults:00190 '253\n'       1.12  (0012) 013 next(Any('0123456789')('andrew, 3333253\n'[12:]))  ->  (['2'], 'andrew, 3333253\n'[13:])
  INFO:lepl.core.trace._TraceResults:00192 '53\n'        1.13  (0013) 013 next(Any('0123456789')('andrew, 3333253\n'[13:]))  ->  (['5'], 'andrew, 3333253\n'[14:])
  INFO:lepl.core.trace._TraceResults:00194 '3\n'         1.14  (0014) 013 next(Any('0123456789')('andrew, 3333253\n'[14:]))  ->  (['3'], 'andrew, 3333253\n'[15:])
  INFO:lepl.core.trace._TraceResults:00197 '3333253\n'   1.8   (0008) 014 stop  ->  DepthFirst(1, None, rest=Any, first=Any)('andrew, 3333253\n'[8:])  ->  (['3', '3', '3', '3', '2', '5', '3'], 'andrew, 3333253\n'[15:])
  INFO:lepl.core.trace._TraceResults:00198 '3333253\n'   1.8   (0008) 013 (['3', '3', '3', '3', '2', '5', '3'], 'andrew, 3333253\n'[15:])  ->  DepthFirst(1, None, rest=Any, first=Any)('andrew, 3333253\n'[8:])  ->  (['3', '3', '3', '3', '2', '5', '3'], 'andrew, 3333253\n'[15:])
  INFO:lepl.core.trace._TraceResults:00199 '3333253\n'   1.8   (0008) 012 (['3', '3', '3', '3', '2', '5', '3'], 'andrew, 3333253\n'[15:])  ->  Transform(DepthFirst, TransformationWrapper(<add>))('andrew, 3333253\n'[8:])  ->  (['3333253'], 'andrew, 3333253\n'[15:])
  INFO:lepl.core.trace._TraceResults:00200 '3333253\n'   1.8   (0008) 011 (['3333253'], 'andrew, 3333253\n'[15:])  ->  And(DepthFirst, Transform)('andrew, 3333253\n'[8:])  ->  (['3333253'], 'andrew, 3333253\n'[15:])
  INFO:lepl.core.trace._TraceResults:00201 '3333253\n'   1.8   (0008) 010 (['3333253'], 'andrew, 3333253\n'[15:])  ->  And(DepthFirst, Transform)('andrew, 3333253\n'[8:])  ->  (['3333253'], 'andrew, 3333253\n'[15:])
  INFO:lepl.core.trace._TraceResults:00202 '3333253\n'   1.8   (0008) 009 (['3333253'], 'andrew, 3333253\n'[15:])  ->  Transform(And, TransformationWrapper(<add>))('andrew, 3333253\n'[8:])  ->  (['3333253'], 'andrew, 3333253\n'[15:])
  INFO:lepl.core.trace._TraceResults:00203 '3333253\n'   1.8   (0008) 008 (['3333253'], 'andrew, 3333253\n'[15:])  ->  Trace(Transform, True)('andrew, 3333253\n'[8:])  ->  (['3333253'], 'andrew, 3333253\n'[15:])

Unlike the deepest match output, I don't find this very useful in most cases.

.. note::

  `Trace() <api/redirect.html#lepl.matchers.monitor.Trace>`_ expects the parser to be configured with the `TraceStack() <api/redirect.html#lepl.core.trace.TraceStack>`_
  monitor.  This is done with `.config.trace_stack() <api/redirect.html#lepl.core.config.ConfigBuilder.trace_stack>`_.


.. index:: common errors

Common Errors
-------------

This section is smaller than it was, because I have changed Lepl to
automatically avoid some common problems that occurred in earlier versions.
But please suggest new entries via the `mailing list / discussion group
<http://groups.google.com/group/lepl>`_.

.. index:: Lexer rewriter used but no tokens found

Missing Tokens
~~~~~~~~~~~~~~

The default `Configuration() <api/redirect.html#lepl.core.config.Configuration>`_ includes processing for
lexers.  If no lexers are present, this message is logged::

  Lexer rewriter used, but no tokens found.

This is not a problem (assuming you didn't intend to use lexing, of course).

