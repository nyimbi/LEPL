
.. index:: recursive descent, generators, stack, parser combinators, implementation
.. _implementation:

Implementation
==============

Lepl is, in many ways, a very traditional recursive descent parser.  This
chapter does not describe the basic ideas behind recursive descent parsing
[*]_.  Instead I will focus on the details that are unique to this particular
implementation.

.. [*] There is a broad--bush description of how matchers work in
       :ref:`implementation_details`.
   

.. index:: trampolining
.. _trampolining:

Trampolining
------------

Overview
~~~~~~~~

A typical recursive descent parser uses at least one stack frame for each
recursive call to a matcher.  Unfortunately, the default Python stack is
rather small and there is no optimisation of tail--recursive calls.  So the
degree of recursion is limited.  This problem is exacerbated by a "clean",
orthogonal design that constructs matchers in a hierarchical manner
(eg. `Word() <api/redirect.html#lepl.matchers.derived.Word>`_ calls `Any()
<api/redirect.html#lepl.matchers.core.Any>`_ to handle character matching;
memoisation uses nested matchers to manage caches).

Trampolining removes this limitation by moving evaluation to a separate
function, which manages the evaluation sequence separately from the program
stack.

A trampolining implementation would typically use a continuation passing
style, but Python supports co--routines (via extended generators) which
automate the encapsulation of a local environment.  Trampolining then becomes
a simple loop that repeatedly evaluates co-routines stored in a stack
allocated on the heap.

The conversion from nested functions to trampolining with generators involves
little more than replacing calls to sub-matchers with ``yield`` (which
presents the target to the trampoline function for evaluation) and then again
using ``yield`` to return the match (rather than ``return``).

.. index:: direct_eval(), no_direct_eval()

Optimisations
~~~~~~~~~~~~~

The overhead of the trampolining is quite small (expensive operations in
Python appear to involve accessing attributes and calling functions; the time
spent in the logic of the trampoline loop is relatively unimportant).

However, in an attempt to improve performance wherever possible, I have
experimented with various approaches to reducing the number of "bounces" made.
These include calling function--based (those that do not need to evaluate
other matcher) directly; flattening nested `And()
<api/redirect.html#lepl.matchers.combine.And>`_ and `Or()
<api/redirect.html#lepl.matchers.combine.Or>`_ matchers; compiling simple leaf
matchers to regular expressions; combining nested transformations.

These are all relatively easy to implement using introspection of the matcher
DAG (see below).

.. index:: @function_matcher, @function_matcher_factory, function_matcher, function_matcher_factory
.. _new_matchers:

Simple Functional Matchers
~~~~~~~~~~~~~~~~~~~~~~~~~~

The simplest matchers can be defined trivially.  For example, to match a
single character::

  >>> @function_matcher
  >>> def char(support, stream):
  >>>     (char, next_stream) = s_next(stream)
  >>>     return ([char], next_stream)
  >>> char()[:].parse('ab')
  ['a', 'b']

The `@function_matcher
<api/redirect.html#lepl.matchers.support.function_matcher>`_ decorator does
the necessary work to place the given logic within an `OperatorMatcher()
<api/redirect.html#lepl.matchers.support.OperatorMatcher>`_ instance, so the
resulting matcher includes all the usual Lepl functionality (configuration,
operators, etc).

`s_next() <api/redirect.html#lepl.stream.core.s_next>`_ is part of the stream API (see below) --- it returns the next
character and the associated stream.  It automatically handles the
end--of--stream case by raising ``StopIteration()``.

This can extended to include configuration::

  >>> @function_matcher_factory()
  >>> def char_in(chars):
  >>>     def match(support, stream):
  >>>         if stream and stream[0] in chars:
  >>>             return ([stream[0]], stream[1:])
  >>>     return match

where the `@function_matcher_factory
<api/redirect.html#lepl.matchers.support.function_matcher_factory>`_ decorator
uses introspection to ensure that the final matcher takes the correct
arguments, and is associated with any given documentation.

Matchers of this kind may avoid trampolining when used with
`.config.direct_eval()
<api/redirect.html#lepl.core.config.ConfigBuilder.direct_eval>`_.

.. index:: @sequence_matcher, @sequence_matcher_factory, sequence_matcher, sequence_matcher_factory

Sequence Matchers
~~~~~~~~~~~~~~~~~

The next most complex kind of matchers can return multiple values (ie they
support backtracking)::

  >>> @sequence_matcher
  >>> def any_char(support, stream):
  >>>     while stream:
  >>>         yield ([stream[0]], stream[1:])
  >>>         stream = stream[1:]

  >>> @sequence_matcher_factory()
  >>> def any_char_in(chars):
  >>>     def match(support, stream):
  >>>         while stream:
  >>>             if stream[0] in chars:
  >>>                 yield ([stream[0]], stream[1:])
  >>>             stream = stream[1:]
  >>>     return match

(these will discard any characters that do match, and return those that do as
successive possibilities).

Again, matchers of this kind may avoid trampolining when used with
`.config.direct_eval()
<api/redirect.html#lepl.core.config.ConfigBuilder.direct_eval>`_.

.. index:: @trampoline_matcher, @trampoline_matcher_factory, trampoline_matcher, trampoline_matcher_factory

Trampoline Matchers
~~~~~~~~~~~~~~~~~~~

The most general matchers evaluate other matchers.  It is difficult to think
of a simple example to add here, but the curious can check the implementation
of `And() <api/redirect.html#lepl.matchers.combine.And>`_ and `Or()
<api/redirect.html#lepl.matchers.combine.Or>`_ (the API documentation includes
source).

These matchers are defined using `@trampoline_matcher
<api/redirect.html#lepl.matchers.support.trampoline_matcher>`_ and
`@trampoline_matcher_factory
<api/redirect.html#lepl.matchers.support.trampoline_matcher_factory>`_ and
cannot avoid trampolining.

.. index:: memoisation, Norvig, Frost, Hafiz, left-recursion
.. _memoisation_impl:

Memoisation
-----------

The simple memoizer, `RMemo() <api/redirect.html#lepl.matchers.memo.RMemo>`_,
is equivalent to the approach described by `Norvig 1991
<http://acl.ldc.upenn.edu/J/J91/J91-1004.pdf>`_.

During the application of left--recursive grammars a matcher may be called with
the same stream, but within different contexts (eg. consider ``a = Optional(a)
& b``, where each repeated call to ``a`` is from an additional "step down").

.. warning::

   Without memoisation left recursion will cause an infinite loop and crash the
   program.

`Frost and Hafiz 2006 <http://www.cs.uwindsor.ca/~hafiz/p46-frost.pdf>`_
observed that there is a natural limit to the number of times left recursion
can be meaningful, which is the length of the remaining input (since you have
to consumer *something* each time round).  They therefore recommended
extending the simple cache with a counter that blocks recursion past that
depth.

This approach is implemented in `LMemo()
<api/redirect.html#lepl.matchers.memo.LMemo>`_ which makes Lepl robust to
left--recursive grammars.

However, the implementation is non-trivial.  In particular, each occurrence of
a matcher in the DAG needs to be curtailed separately.  Since matchers are
pure they are often references in multiple places; this conflicts with the
need to treat each location as a distinct entity and so some care is needed to
duplicate appropriate nodes when rewriting (the end--user does not need to
worry about this, but it makes implementation and testing more difficult).


.. index:: rewriting, graph, flattening

Parser Rewriting
----------------

A parser is constructed from a set matchers that form a directed (possibly
cyclic) graph.  By storing the constructor arguments for the matcher objects
(and knowing their types, which are constructors in Python) we can reconstruct
(and, more generally, rewrite) the graph.

The base classes for the graph are in the ``graph`` package (the ``node`` package, used for ASTs, builds on these
classes so many of the tools used internally within Lepl may also be useful to
process ASTs).  Matcher graph rewriting occurs during parser construction
(see the ``parser`` package).

Parser rewriting allows memoisation to be transparently added to all nodes,
for example.

Tree traversal (without rewriting) is also useful; it is used to generate
various textual representations of the matchers (and the pretty ASCII trees
for `Node() <api/redirect.html#lepl.support.node.Node>`_--based ASTs).


.. index:: streams, SimpleStream(), LocationStream(), StreamFactory()
.. _streams:

Streams
-------

Since Lepl 5 all input is wrapped within a "stream" abstraction.  This is a
tuple containing state and a helper: the state changes at each position, but
the helper remains the same.  So for strings the state is an index, while the
helper wraps the original input.

Helpers provide additional functionality, like recording the deepest match,
formatting, and calculating hash keys.  Access to these functions is provided
through the generic ``s_xxx`` functions defined in `lepl.stream.core
<api/redirect.html#lepl.stream.core>`_.
