
.. index:: matchers, no_full_first_match()
.. _matchers:

Matchers
========

The `API Documentation <api/redirect.html#lepl.matchers>`_ contains an
exhaustive list of the matches available.  This chapter only describes the
most important.

The final section gives some `implementation details`_.

.. note::
   
   The examples here are fragments that illustrate some small detail.  They
   often include `.config.no_full_first_match() <api/redirect.html#lepl.core.config.ConfigBuilder.no_full_first_match>`_ so
   that a partial match can be displayed instead of an error message.

.. index:: Literal()

Literal 
-------

`[API] <api/redirect.html#lepl.matchers.core.Literal>`_ This matcher
identifies a given string.  For example, `Literal('hello')
<api/redirect.html#lepl.matchers.core.Literal>`_ will give the result "hello"
when that text is at the start of a stream::

  >>> matcher = Literal('hello')
  >>> matcher.config.no_full_first_match()
  >>> matcher.parse('hello world')
  ['hello']

In many cases it is not necessary to use `Literal() <api/redirect.html#lepl.matchers.core.Literal>`_ explicitly.  Most matchers,
when they receive a string as a constructor argument, will automatically
create a literal match from the given text.

.. index:: Any()

Any
---

`[API] <api/redirect.html#lepl.functions.Any>`_ This matcher identifies any
single character.  It can be restricted to match only characters that appear
in a given string.  For example::

  >>> matcher = Any()
  >>> matcher.config.no_full_first_match()
  >>> matcher.parse('hello world')
  ['h']

  >>> matcher = Any('abcdefghijklm')[0:]
  >>> matcher.config.no_full_first_match()
  >>> matcher.parse('hello world')
  ['h', 'e', 'l', 'l']

.. index:: And(), &

And (&)
-------

`[API] <api/redirect.html#lepl.matchers.combine.And>`_ This matcher combines
other matchers in order.  For example::

  >>> matcher = And(Any('h'), Any())
  >>> matcher.config.no_full_first_match()
  >>> matcher.parse('hello world')
  ['h', 'e']

All matchers must succeed for `And() <api/redirect.html#lepl.matchers.combine.And>`_ as a whole to succeed::

  >>> matcher = And(Any('a'), Any('b'))
  >>> matcher.parse('pq')
  [...]
  lepl.stream.maxdepth.FullFirstMatchException: The match failed in <string> at 'q' (line 1, character 2).
  >>> matcher.parse('ax')
  [...]
  lepl.stream.maxdepth.FullFirstMatchException: The match failed in <string> at '' (line 1, character 3).
  >>> matcher.parse('ab')
  ['a', 'b']

  .. note::

  It's worth noting that because the error message is based on the deepest
  match it can sometimes be "off by one" if a character was read but failed to
  match.

The ``&`` operator is equivalent unless a :ref:`separator <spaces>` is being
used::

  >>> matcher = Any('a') & Any('b')
  >>> matcher.parse('ab')
  ['a', 'b']

.. index:: Or(), |, parse_all()

Or (|)
------

`[API] <api/redirect.html#lepl.matchers.combine.Or>`_ This matcher searches
through a list of other matchers to find a successful match.  For example::

  >>> matcher = Or(Any('x'), Any('h'), Any('z'))
  >>> matcher.config.no_full_first_match()
  >>> matcher.parse('hello world')
  ['h']

The first match found is the one returned::

  >>> matcher = Or(Any('h'), Any()[3])
  >>> matcher.config.no_full_first_match()
  >>> matcher.parse('hello world')
  ['h']

But subsequent calls return other possibilities::

  >>> list(matcher.parse_all('hello world'))
  [['h'], ['h', 'e', 'l']]

This shows how Lepl supports "backtracking" --- a matcher may be called
several times before a result is found that "fits" with the rest of the
grammar.  All matchers upport this behaviour, but it is easiest to see with
`Or() <api/redirect.html#lepl.matchers.combine.Or>`_.

The `matcher.parse_all() <api/redirect.html#lepl.core.config.ParserMixin.parse_all>`_ method is similar
to `matcher.match() <api/redirect.html#lepl.core.config.ParserMixin.match>`_
introduced in the previous section, but returns only the results (it discards
the remaining streams).  Using ``list()`` converts the iterator returned by
the parser into a list that can be displayed.

.. index:: Repeat(), [], backtracking, breadth-first, depth-first
.. _repeat:

Repeat ([...])
--------------

`[API] <api/redirect.html#lepl.matchers.derived.Repeat>`_ Although `Repeat() <api/redirect.html#lepl.matchers.derived.Repeat>`_ can be used directly, it's
normal to use the ``[]`` array syntax instead (which, when used on a matcher,
is automatically translated into `Repeat() <api/redirect.html#lepl.matchers.derived.Repeat>`_).

At its simplest, ``[]`` indicates that a matcher should repeat a given number
of times::

  >>> matcher = Any()[3]
  >>> matcher.config.no_full_first_match()
  >>> matcher.parse('12345')
  ['1', '2', '3']
  >>> list(matcher.parse_all('12345'))
  [['1', '2', '3']]

  >>> matcher = Any()[3:3]
  >>> matcher.config.no_full_first_match()
  >>> matcher.parse('12345')
  ['1', '2', '3']

If only a lower bound to the number of repeats is given the match will be
repeated as often as possible::

  >>> matcher = Any()[3:]
  >>> matcher.config.no_full_first_match()
  >>> matcher.parse('12345')
  ['1', '2', '3', '4', '5']
  >>> list(matcher.parse_all('12345'))
  [['1', '2', '3', '4', '5'], ['1', '2', '3', '4'], ['1', '2', '3']]

If the match cannot be repeated the requested number of times no result is
returned::

  >>> matcher = Any()[3:]
  >>> matcher.config.no_full_first_match()
  >>> matcher.parse('12')
  None

As well as repetition, ``[]`` can also indicate that results should be joined
together.  This is done by adding ``...``::

  >>> matcher = Any()[3, ...]
  >>> matcher.config.no_full_first_match()
  >>> matcher.parse('12345')
  ['123']

And you can specify a separator that muct occur between repetitions (usually
this is used with `Drop() <api/redirect.html#lepl.matchers.derived.Drop>`_
which discards the value)::

  >>> matcher = Any()[3, ..., Drop('x')]
  >>> matcher.config.no_full_first_match()
  >>> matcher.parse('1x2x3x4x5')
  ['123']

.. index:: Lookahead(), ~
.. _lookahead:

Lookahead
---------

`[API] <api/redirect.html#lepl.matchers.core.Lookahead>`_ This matcher checks
whether another matcher --- its argument --- would succeed, but doesn't
actually match anything.  If the argument doesn't match then it fails, so any
following matchers joined with `And() <api/redirect.html#lepl.matchers.combine.And>`_ will not be called.

For example, to only parse numbers that begin with "2" (specifying a string as
matcher is equivalent to using `Literal() <api/redirect.html#lepl.matchers.core.Literal>`_)::

  >>> matcher = Lookahead('2') & Integer()
  >>> matcher.parse('234')
  ['234']
  >>> matcher.parse('123')
  [...]
  lepl.stream.maxdepth.FullFirstMatchException: The match failed in <string> at '23' (line 1, character 2).

When preceded by a ``~`` the logic is reversed::

  >>> matcher = ~Lookahead('2') & Integer()
  >>> matcher.parse('234')
  [...]
  lepl.stream.maxdepth.FullFirstMatchException: The match failed in <string> at '34' (line 1, character 2).
  >>> matcher.parse('123')
  ['123']

.. note::

  This change in behaviour is specific to `Lookahead() <api/redirect.html#lepl.matchers.core.Lookahead>`_ --- usually ``~`` applies
  `Drop() <api/redirect.html#lepl.matchers.derived.Drop>`_ as described below.

.. index:: Drop(), ~

Drop (~)
--------

`[API] <api/redirect.html#lepl.matchers.derived.Drop>`_ This matcher calls
another matcher, but discards the results::

  >>> (Drop('hello') / 'world').parse('hello world')
  [' ', 'world']
  >>> (~Literal('hello') / 'world').parse('hello world')
  [' ', 'world']

(The empty string in the result is from ``/`` which joins two matchers
together, with optional spaces between).

This is different to `Lookahead() <api/redirect.html#lepl.matchers.core.Lookahead>`_ because the matcher after
`Drop() <api/redirect.html#lepl.matchers.derived.Drop>`_ receives a stream
that has "moved on" to the next part of the input.  With `Lookahead() <api/redirect.html#lepl.matchers.core.Lookahead>`_ the stream is not advanced
and so this example will fail::

  >>> (Lookahead('hello') / 'world').parse('hello world')
  [...]
  lepl.stream.maxdepth.FullFirstMatchException: The match failed in <string> at ' world' (line 1, character 6).

.. note::

   The error message is misleading here because it is based on the deepest
   match in the stream, which in this case is due to `Lookahead() <api/redirect.html#lepl.matchers.core.Lookahead>`_.

.. index:: Apply(), >, >=, args()

Apply (>, >=, args)
-------------------

.. note::

   See also :ref:`faq_apply`

`[API] <api/redirect.html#lepl.matchers.derived.Apply>`_ This matcher passes
the results of another matcher to a function, then returns the value from the
function as a new result::

  >>> def show(results):
  ...     print('results:', results)
  ...     return results
  >>> Apply(Any()[:,...], show).parse('hello world')
  results: ['hello world']
  [['hello world']]

The ``>`` operator is equivalent::

  >>> (Any()[:,...] > show).parse('hello world')
  results: ['hello world']
  [['hello world']]

The returned result is placed in a new list, which is not always what is
wanted (it is useful when you want :ref:`nestedlists`); setting ``raw=True``
uses the result directly::

  >>> Apply(Any()[:,...], show, raw=True).parse('hello world')
  results: ['hello world']
  ['hello world']
  >>> (Any()[:,...] >= show).parse('hello world')
  results: ['hello world']
  ['hello world']

Setting another optional argument, `args <api/redirect.html#lepl.matchers.derived.args>`_, to ``True`` changes the way the
function is called.  Instead of passing the results as a single list each is
treated as a separate argument.  This is familiar as the way ``*args`` works
in Python::

  >>> def format3(a, b, c):
  ...     return 'a: {0}; b: {1}; c: {2}'.format(a, b, c)
  >>> Apply(Any()[3], format3, args=True).parse('xyz')
  ['a: x; b: y; c: z']

There's no operator equivaluent for this, but a little helper function called
`args() <api/redirect.html#lepl.matchers.derived.args>`_ allows ``>`` to be
reused:

  >>> (Any()[3] > args(format3)).parse('xyz')
  ['a: x; b: y; c: z']

.. index:: **

KApply (**)
-----------

`[API] <api/redirect.html#lepl.matchers.derived.KApply>`_ This matcher passes
the results of another matcher to a function, along with additional
information about the match, then returns the value from the function as a new
result.  Unlike `Apply() <api/redirect.html#lepl.matchers.derived.Apply>`_,
this names the arguments as follows:

  stream_in
    The stream passed to the matcher before matching.

  stream_out
    The stream returned from the matcher after matching.

  results
    A list of the results returned.


.. index:: First(), Empty(), Regexp(), Delayed(), Trace(), AnyBut(), Optional(), Star(), ZeroOrMore(), Plus(), OneOrMore(), Map(), Add(), Substitute(), Name(), Eof(), Eos(), Identity(), Newline(), Space(), Whitespace(), Digit(), Letter(), Upper(), Lower(), Printable(), Punctuation(), UnsignedInteger(), SignedInteger(), Integer(), UnsignedFloat(), SignedFloat(), SignedEFloat(), Float(), Word(), String().

More
----

Many more matchers are described in the `API Documentation
<api/redirect.html#lepl.matchers>`_, including 
`Add() <api/redirect.html#lepl.matchers.derived.Add>`_,
`AnyBut() <api/redirect.html#lepl.matchers.derived.AnyBut>`_,
`Columns() <api/redirect.html#lepl.matchers.complex.Columns>`_,
`Delayed() <api/redirect.html#lepl.matchers.core.Delayed>`_,
`Digit() <api/redirect.html#lepl.matchers.derived.Digit>`_,
`Empty() <api/redirect.html#lepl.matchers.core.Empty>`_,
`Eof() <api/redirect.html#lepl.matchers.core.Eof>`_,
`Eos() <api/redirect.html#lepl.matchers.core.Eos>`_,
`First() <api/redirect.html#lepl.matchers.combine.First>`_,
`Float() <api/redirect.html#lepl.support.warn.Float>`_, 
`Identity() <api/redirect.html#lepl.matchers.derived.Identity>`_,
`Integer() <api/redirect.html#lepl.matchers.derived.Integer>`_,
`Letter() <api/redirect.html#lepl.matchers.derived.Letter>`_,
`Lower() <api/redirect.html#lepl.matchers.derived.Lower>`_,
`Map() <api/redirect.html#lepl.matchers.derived.Map>`_,
`Name() <api/redirect.html#lepl.matchers.derived.Name>`_,
`Newline() <api/redirect.html#lepl.matchers.derived.Newline>`_,
`OneOrMore() <api/redirect.html#lepl.matchers.derived.OneOrMore>`_,
`Optional() <api/redirect.html#lepl.matchers.derived.Optional>`_,
`Plus() <api/redirect.html#lepl.matchers.derived.Plus>`_,
`Printable() <api/redirect.html#lepl.matchers.derived.Printable>`_,
`Punctuation() <api/redirect.html#lepl.matchers.derived.Punctuation>`_,
`Regexp() <api/redirect.html#lepl.matchers.core.Regexp>`_,
`SignedEFloat() <api/redirect.html#lepl.support.warn.SignedEFloat>`_,
`SignedFloat() <api/redirect.html#lepl.support.warn.SignedFloat>`_,
`SignedInteger() <api/redirect.html#lepl.matchers.derived.SignedInteger>`_,
`SkipTo() <api/redirect.html#lepl.matchers.derived.SkipTo>`_,
`Space() <api/redirect.html#lepl.matchers.derived.Space>`_,
`Star() <api/redirect.html#lepl.matchers.derived.Star>`_,
`String() <api/redirect.html#lepl.matchers.derived.String>`_,
`Substitute() <api/redirect.html#lepl.matchers.derived.Substitute>`_,
`Trace() <api/redirect.html#lepl.matchers.monitor.Trace>`_,
`UnsignedFloat() <api/redirect.html#lepl.support.warn.UnsignedFloat>`_,
`UnsignedInteger() <api/redirect.html#lepl.matchers.derived.UnsignedInteger>`_,
`Upper() <api/redirect.html#lepl.matchers.derived.Upper>`_,
`Whitespace() <api/redirect.html#lepl.matchers.derived.Whitespace>`_,
`Word() <api/redirect.html#lepl.matchers.derived.Word>`_ and
`ZeroOrMore() <api/redirect.html#lepl.matchers.derived.ZeroOrMore>`_.

.. index:: generator, results, failure, implementation, Matcher, BaseMatcher, ABC
.. _implementation_details:

Implementation Details
----------------------

All matchers accept a stream of data and return an iterator over possible
``([results], stream)`` pairs, where the new stream continues from after the
matched text (and which may then be passed to another matcher to continue the
process of parsing).  These iterators are typically implemented as Python
generators [*]_.

A matcher may succeed, but provide no results --- the iterator will include a
tuple containing an empty list and the new stream.  When there are no more
possible matches, the iterator will terminate.

Simple matchers will return an iterator containing a single entry.  Matchers
that return multiple values support backtracking.  For example, the `Or() <api/redirect.html#lepl.matchers.combine.Or>`_ generator may yield once for
each sub--match in turn (in practice some sub-matchers may return generators
that themselves return many values, while others may fail immediately, so it
is not a direct 1--to--1 correspondence).

(It is probably obvious if you have used combinator libraries before, but all
matchers implement this same interface, whether they are "fundamental" --- do
the real work of matching against the stream --- or delegate work to other
sub--matchers, or modify results.  This consistency is the source of their
expressive power.)

Lepl includes several function decorators that help simplify the creation of
new matchers.  See :ref:`new_matchers` and following sections.

.. [*] I am intentionally omitting details about trampolining here to focus on
       the process of matching.  A more complete description of the entire
       implementation can be found in :ref:`trampolining`.
