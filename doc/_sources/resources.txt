
.. index:: resources
.. _resources:

Resource Management
===================

.. index:: GeneratorWrapper, backtracking, generators

The Input Problem
-----------------

:ref:`backtracking` within Lepl is implemented using generators.  These are
blocks of code that can be paused and restarted.  Each matcher has, at its
core, one of these generators.  Backtracking is achieved by calling the
generator to continue searching the input for alternative matches.

This can be a problem, particularly when parsing input from files.  The
`parse_file() <api/redirect.html#lepl.core.config.ParserMixin.parse_file>`_
method calls `parse_iterable()
<api/redirect.html#lepl.core.config.ParserMixin.parse_iterable>`_ which
iterates over the lines in the file.  The streams created by `parse_iterable()
<api/redirect.html#lepl.core.config.ParserMixin.parse_iterable>`_ are lazy
linked lists that only reference later parts of the input.  That means that
Python can reclaim (garbage--collect) the memory used by earlier input while
the parser is still matching later input.  However, this cannot happen (memory
cannot be freed) if a generator keeps a reference to earlier input (which it
needs in case it is called to backtrack and provide a new match).

So the generators used in Lepl to provide backtracking frustrate Python's
garbage collection and lead to parsers consuming much more memory than would
otherwise be necessary.

The Output Problem
------------------

As a parser consumes input it also constructs the output.  In Lepl the output
is a list of data held in memory.  This output list is built by the parser
until all the input has been consumed, and then returned as a result.

A large input often implies a large output.  So even if we solve the "input
problem" above, a parser will still use a lot of memory to store the output.

The Input Solution
------------------

To solve the "input problem" we need to discard generators, but we need
generators while parsing because the parser needs to backtrack when it fails
to match something.  The solution to this apparent contradiction is to discard
only the generators that we will not need.

In general we cannot find an exact solution (I believe; finding which
generators we do not need is probably related to the halting problem), but we
can implement a simple heuristic that works well in practice: we discard
generators that have not been used "for a long time".

To do this, Lepl maintains a list of generators, sorted by last access time.
When the list exceeds some size the least-used generator is deleted.  This
reduces memory use while allowing a reasonable amount of backtracking.

The list of generators is maintained by a `GeneratorManager()
<api/redirect.html#lepl.core.manager.GeneratorManager>`_, but you do not need
to use this class directly.  Instead, call `.config.low_memory()
<api/redirect.html#lepl.core.config.ConfigBuilder.low_memory>`_.  That
configuration method takes a parameter ``queue_len`` which controls the size
of the list of "valid" generators.

The Output Solution
-------------------

We could reduce the amount of memory needed to hold the output if there were
some way of returning the output incrementally.  In normal use this is not
possible with Lepl, but there is a "neat hack" that works quite well.

Remember that a Lepl parser can produce a series of matches.  These are
alternative ways of parsing the input, given the parser.  But in most cases we
are only interested in the first match --- this is particularly true when
using `.config.low_memory()
<api/redirect.html#lepl.core.config.ConfigBuilder.low_memory>`_ as described
above, because the alternatives will be unavailable.

The hack is to return fragments of the output as alternative matches.  This
means:

 * Adding the matcher `Iterate() <api/redirect.html#lepl.matchers.complex.Iterate>`_ at the "top level" of the parser.  This
   will return each fragment in turn.

 * Calling `parse_all()
   <api/redirect.html#lepl.core.config.ParserMixin.parse_all>`_ and then
   treating the generator as a sequence of fragments, rather than a sequence
   of complete matches.

An example will make this clear.  Here is a matcher that simply matches every
line::

  >>> all_lines = Line(Token(Any()[:,...]))[:]

and here is the equivalent matcher than returns each line as a separate
fragment.  Note how the final ``[:]`` is replaced by `Iterate() <api/redirect.html#lepl.matchers.complex.Iterate>`_::

  >>> each_line = Iterate(Line(Token(Any()[:,...])))

We can use ``each_line`` as follows::

  >>> input = '''line one
  ... line two
  ... line three'''
  >>> each_line.config.no_full_first_match().lines()
  >>> parser = each_line.get_parse_all()
  >>> for line in parser(input):
  >>>     print(line)
  ['line one\n']
  ['line two\n']
  ['line three']
