
.. _lepl5:

Lepl 5 - Better Shape for the Future
====================================

I've simplified how Lepl handles strings (or lists, or iterables, or...).  As
a consequence, some of the API has changed.  So moving from Lepl 4 to 5 may
break your code.  I'm sorry!  But hopefully it should be easy to fix (see
below) and, once running again, should be a little faster.

In the longer term these changes help me maintain Lepl and continue to expand
it.

As part of the stream work I also revisited how Lepl handles memoisation,
particularly for left-recursive grammars.

The Big Picture
---------------

Understanding what happened "under the hood" may explain the API changes.  But
if you'd rather just have a list of issues please skip to "Changes" below.

Streams
~~~~~~~

Originally, Lepl pretended that all inputs were strings.  Indeed, it was
possible (using ``parse_null()``) to send a string directly to the matchers:
each matcher then passed a new string, containing whatever remained to be
parsed, to the next matcher.  One advantage of this was that it allowed Lepl's
regular expression code to be used with general sequences.

In practice, however, this approach wasn't so useful.  For example, there was
no way to tell how far you are through the data.  So in most cases (ie. using
`parse() <api/redirect.html#lepl.core.config.ParserMixin.parse>`_) Lepl
constructed a wrapper that carried the string and the extra location
information together, but which still looked sufficiently string-like for the
matchers to work correctly.

This solved the problem of knowing where you were, but made the code more
complicated.  To avoid duplicating that complex code I used the same wrapper
for many different input types.  And that only made things more complicated.
Worse, the "strings" slowly accumulated extra non-string methods to do other
useful things.

Eventually it became clear that always pretending that things were strings was
a bad idea.  The extra complexity of defining an interface for "a stream of
input data" might complicate matchers a little, but made everything else too
brittle.

Hence Lepl 5, in which I dropped the pretence that the input is "just" a
string.  Instead, matchers work with a "stream" using functions like `s_next()
<api/redirect.html#lepl.stream.core.s_next>`_ to get the next character, or
`s_line() <api/redirect.html#lepl.stream.core.s_line>`_ to read a line.

You may be surprised to hear that those are functions, and not methods, so I
will take a moment to sketch the implementation.  A stream is now implemented
as a tuple; a pair of values.  The first value is "state" and the second value
is a "helper".  Exactly what the state and helper are depend on the input, but
the helper must implement a standard interface and the ``s_xxx()`` functions
call that interface with the state.

That sounds more complex than it is.  Essentially::

    def s_next(stream):
        (state, helper) = stream
	return helper.next(state)

and the reason for *that* is efficiency.  For a string, for example, state is
the current index into the string.  So every time the stream advances (and
this happens a *lot*) we only have to construct a new tuple with a different
integer value.  The helper remains the same (in the case of a string it's a
wrapper round the the string itself).  This is much more efficient than
constructing a new, heavyweight object at every step (there is a downside,
which is an extra function invocation, but I think that will be insignificant
once PyPy becomes widely adopted).

Memoisation
~~~~~~~~~~~

Once I had simplified the streams code I started working through the unit
tests, fixing bugs.  At first, of course, I was fixing bugs in my new code and
because that was simpler than before it wasn't too hard to fix.  Then I
started hitting bugs that didn't seem to be in the new code, but in the old.
There were several mistakes / confusions / bugs in the old stream and
memoisation code that, together, "cancelled each other out" for at least some
inputs.

A less charitable way of describing the above is that I had "balanced" the
code well enough to pass the tests I had, but that the end result was still
broken.

In retrospect this isn't a huge surprise - worrying about this kind of problem
is what motivated the work to simplify stream handling.

So I have also rewritten much of the memoisation code, trying to simplify it,
adding new tests, and generally trying to understand in more detail what is
happening.  It now appears to be working, but I am still not convinced that I
understand everything, so I have changed the default configuration to more
actively discourage left-recursion.  These changes are described below.

Resource Management
~~~~~~~~~~~~~~~~~~~

While working on the above I also needed to check how the new streams
interacted with Python's garbage collection.  It turned out that, despite the
intention of the original design, Lepl had never been able to parse input
larger than the available memory.  So I added additional tests for this and
fixed things as necessary.

I also added support for avoiding consuming too much memory with the output.
See :ref:`resources` for more details.

Changes
-------

Streams
~~~~~~~

* Some `parse() <api/redirect.html#lepl.core.config.ParserMixin.parse>`_
  methods have changed:

  * ``parse_null()`` no longer exists.  `parse_sequence()
    <api/redirect.html#lepl.core.config.ParserMixin.parse_sequence>`_ might be
    the best replacement (see above).

  * ``parse_items()`` no longer exists.  You will probably want either
    `parse_list()
    <api/redirect.html#lepl.core.config.ParserMixin.parse_list>`_ (for lists)
    or `parse_iterable()
    <api/redirect.html#lepl.core.config.ParserMixin.parse_iterable>`_ (for
    iterables).

* The idea of a *line* is more clearly defined.  This has two effects:

  * You may need to care more about which `parse()
    <api/redirect.html#lepl.core.config.ParserMixin.parse>`_ method to use.  A
    list or sequence (the only difference between those is that the formatting
    for displaying lists is a little prettier) treats all the data as one
    line, but a string uses the newline (``'\n'``) character to define lines.
    And an iterable assumes that each value is a new line (which matches the
    way iterating over files works).  In most cases the generic `parse()
    <api/redirect.html#lepl.core.config.ParserMixin.parse>`_ will do the right
    thing for you by dispatching on type.

  * Regular expressions don't match across lines.  There is no way to pass a
    general "stream" to regular expressions, so I pass a "line" instead.  If
    you really want to match multiple lines in a string then use
    `parse_sequence()
    <api/redirect.html#lepl.core.config.ParserMixin.parse_sequence>`_ which,
    as I just explained, treats the entire input as a single line.

* ``parse_items()`` had an option to add an extra list around each value.
  This is not present in `parse_list()
  <api/redirect.html#lepl.core.config.ParserMixin.parse_list>`_ because the
  way that a character is accessed has changed.  In Lepl 4 the first character
  was ``stream[0]``, but in Lepl 5 it is ``stream[0:1]``.  This has no impact
  on strings, but for lists it fixes a subtle bug (the problem was a confusion
  between "first character" and "a sub-section of the input stream containing
  the first character" - the latter approach makes lists and strings
  consistent).

* Line-aware and offside parsing have changed.  These should make things
  simpler and more consistent:

  * Both offside parsing (where the indentation is significant) and simpler
    line--aware parsing (which has lines, but doesn't care about indentation)
    use the same matchers.  So ``BLine()`` no longer exists, for example ---
    use `Line() <api/redirect.html#lepl.lexer.lines.matchers.Line>`_ instead.

  * For configuration, use `.config.lines()
    <api/redirect.html#lepl.core.config.ConfigBuilder.lines>`_.

  * If you have specified ``block_policy`` or ``block_indent`` in
    `.config.lines()
    <api/redirect.html#lepl.core.config.ConfigBuilder.lines>`_ then the
    `LineStart() <api/redirect.html#lepl.lexer.lines.matchers.LineStart>`_
    token will include leading white spaces and Lepl will automatically check
    that the indentation is consistent with the `Block()
    <api/redirect.html#lepl.lexer.lines.matchers.Block>`_.  To disable this,
    use ``indent=False`` as a parameter in `Line()
    <api/redirect.html#lepl.lexer.lines.matchers.Line>`_ or `LineStart()
    <api/redirect.html#lepl.lexer.lines.matchers.LineStart>`_.  So, for
    example, to match a blank line with any indentation, use ``Line(Empty(),
    indent=False)``.

  * Lines are now handled by adding extra tokens (before, the lexer added
    extra "characters" which were then matched by special tokens).  That means
    that you can no longer match ``(\*SOL\*)`` and ``(\*EOL\*)`` in regular
    expressions (more generally, you must use tokens with line aware and
    offside parsing - before it was technically possible to not do so).

* The values available when generating an error message inside the parser have
  changed.  The value names are `documented here
  <api/lepl.stream.core-pysrc.html#StreamHelper.kargs>`_, and typically are
  prefixed by ``in_`` and ``out_`` for the input and output streams.

* The configuration for "managed generators" has changed from
  `.config.manage()
  <api/redirect.html#lepl.core.config.ConfigBuilder.manage>`_ to
  `.config.low_memory()
  <api/redirect.html#lepl.core.config.ConfigBuilder.low_memory>`_.  This also
  adds some additional settings that are needed to reduce memory use and
  restricts the size of "secondary" stacks used in search / repetition.  The
  result is that Lepl really can handle inputs larger than available memory -
  see :ref:`resources`.

* If you define your own matchers you will need to use `s_next()
  <api/redirect.html#lepl.stream.core.s_next>`_ and friends instead of
  accessing the "string".  So replace:: char = stream[0] next_stream =
  stream[1:] with ``(char, next_stream) = s_next(stream)``.  The full set of
  functions is documented in `lepl.stream.core <api/redirect.html#lepl.stream.core>`_ and the source is full of
  examples.

* `TraceResults() <api/redirect.html#lepl.core.trace.TraceResults>`_,
  configured by ``config.trace()``, is now `TraceStack()
  <api/redirect.html#lepl.core.trace.TraceStack>`_, configured by
  `.config.trace_stack()
  <api/redirect.html#lepl.core.config.ConfigBuilder.trace_stack>`_.  The
  output will not show the stream when `.config.low_memory()
  <api/redirect.html#lepl.core.config.ConfigBuilder.low_memory>`_ has been
  called because retaining the streams for debug display introduces a memory
  "leak".

* Repetition joins values using a "repeat" operator.  By default this joins
  lists, as before, but you can redefine it to define a fold over results.
  See the example :ref:`low_memory`.

* (Implementation detail) The "wrapper" around trampolining matchers is no
  longer "transformable".  This should have no effect on your code unless you
  are looking at the detailed structure of the matcher tree (it should make
  your code faster as it removes the need to call a generator that does
  nothing but call another generator - something anyone who has watched Lepl
  in a debugger cannot fail to have wondered about...)

Memoisation
~~~~~~~~~~~

* The default configuration now *includes* memoisation for right-recursive
  grammars.  For many problems this will make the parser slower.  It can be
  removed with `.config.no_memoize()
  <api/redirect.html#lepl.core.config.ConfigBuilder.no_memoize>`_, but is
  included by default because it detects left-recursive grammars (which would
  otherwise loop indefinitely) and raises an error with helpful text.

  * **To repeat, the first thing to try when optimising your code is**
    `.config.no_memoize()
    <api/redirect.html#lepl.core.config.ConfigBuilder.no_memoize>`_.

* To enable handling of (some) left-recursive grammars, the simplest option is
  to use `.config.auto_memoize()
  <api/redirect.html#lepl.core.config.ConfigBuilder.auto_memoize>`_ which will
  add `LMemo() <api/redirect.html#lepl.matchers.memo.LMemo>`_ caches where
  required.

* For more detailed control, you can also use:

  * `.config.left_memoize()
    <api/redirect.html#lepl.core.config.ConfigBuilder.left_memoize>`_ - add
    `LMemo() <api/redirect.html#lepl.matchers.memo.LMemo>`_ everywhere

  * ``config.auto_memoize(full=True)`` - add `RMemo()
    <api/redirect.html#lepl.matchers.memo.RMemo>`_ in addition to `LMemo()
    <api/redirect.html#lepl.matchers.memo.LMemo>`_.

  * `.config.optimize_or()
    <api/redirect.html#lepl.core.config.ConfigBuilder.optimize_or>`_ -
    re-arrange `Or() <api/redirect.html#lepl.matchers.combine.Or>`_ contents.
    This has the potential to make left-recursive parsers much faster (it will
    change the order of multiple results - generally for the better).

* Both `.config.left_memoize()
  <api/redirect.html#lepl.core.config.ConfigBuilder.left_memoize>`_ and
  `.config.auto_memoize()
  <api/redirect.html#lepl.core.config.ConfigBuilder.auto_memoize>`_ take a
  ``d`` parameter that can be used to specify a maximum recursion depth.  If
  this is not given the length of the remaining input is used - generally this
  is much too large, so ``d`` can significantly reduce time spent exploring
  incorrect matches.  However, if too small, it has the potential to exclude a
  correct match.

The left-memoisation code does the following:

* Wrappers (`LMemo() <api/redirect.html#lepl.matchers.memo.LMemo>`_ instances)
  are added to the matcher DAG.  These are do two things:

  * Most simply cache values.  Cached values are stored by input and call
    depth.

  * Wrappers around `Delayed()
    <api/redirect.html#lepl.matchers.core.Delayed>`_ instances, which
    represent "entry points" into loops, curtail the number of calls
    (according to either the length of the remaining input or the ``d``
    parameter described earlier).

* In addition, `Delayed() <api/redirect.html#lepl.matchers.core.Delayed>`_
  instances (and the `LMemo() <api/redirect.html#lepl.matchers.memo.LMemo>`_
  wrappers) are duplicated when multiple references exist.  This is necessary
  so that the "curtailment" is not duplicated at multiple points in the
  matcher graph (in general it doe snot matter of the same node is used at
  various point in the parser, because the parser is "pure", but memoisation
  adds state).

This is my interpretation of the approach described in Frost and Hafiz 2006.
However, the extra complexity implied by the generated / objects based
approach used here means that I am not completely sure that it is correct.


Further Reading
---------------

* `Front Page <index.html>`_
* :ref:`manual`
* :ref:`tutorial`
* :ref:`contents`
