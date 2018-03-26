
.. index:: patterns
.. _style:

Patterns
========

Some guidelines for writing good parsers.


.. index:: operators

Operators are Matchers
----------------------

Remember that operators are just shorthand for matchers (``&`` instead of
`And() <api/redirect.html#lepl.matchers.combine.And>`_ etc).  You don't have
to use operators --- see the discussion on :ref:`caveatsandlimitations`.

But remember that ``&`` and `And() <api/redirect.html#lepl.matchers.combine.And>`_ *do differ* when using
:ref:`separators`.


.. index:: Or()
.. _complexor:

Use Or() With Complex Alternatives
----------------------------------

Use `Or(..., ..., ...) <api/redirect.html#lepl.matchers.combine.Or>`_ for
alternatives with productions.  The ``|`` syntax can lead to errors because it
binds more tightly than ``>``.


.. index:: Apply()
.. _applycase:

``>`` Capitalised; ``>>`` lowercase
-----------------------------------

Nodes are typically made with constructors and invoked with ``>``, while the
processing of results is usually done with mapped functions.  So ``>`` is
followed by a Capitalised name, while ``>>`` is followed by a lowercase name.
Noticing and following this convention can help avoid issues with the
behaviour of `Apply() <api/redirect.html#lepl.matchers.derived.Apply>`_ with
``raw=False`` (the default implementation of ``>``), which adds an extra level
of nesting and is usually inappropriate for use with functions.

``throw`` can be used with ``>>`` or ``>``.


.. index:: Separator()
.. _separator:

Define Words Before Using DroppedSpace (or Separator)
-----------------------------------------------------

`DroppedSpace() <api/redirect.html#lepl.matchers.operators.DroppedSpace>`_ and `Separator() <api/redirect.html#lepl.matchers.operators.Separator>`_
simplify the handling of spaces in the grammar.  To avoid confusion, split
your grammar into two.  The first part, defining words, should come before
``with``; the second part should come after.

  >>> # words defined here
  >>> word = Letter()[:,...]
  >>> with DroppedSpace():
  >>>     # sentences defined here
  >>>     sentence = word[1:]


Explicitly Exclude Spaces
-------------------------

By default, repetition in Lepl is greedy.  This means that, no matter what
`Separator() <api/redirect.html#lepl.matchers.operators.Separator>`_ is used, `Any()[:]
<api/redirect.html#lepl.match.Any>`_ will swallow the *entire* input.

So handling spaces in a grammar takes two steps:

1. Exclude the spaces from matchers that produce results.

2. Use `Separator() <api/redirect.html#lepl.matchers.operators.Separator>`_ to "mop up" the
   input that remains.


.. index:: Delayed()

Use Delayed() For Recursive Definitions
---------------------------------------

Sometimes a grammar needs to refer to matchers before they are defined.  The
`Delayed() <api/redirect.html#lepl.matchers.core.Delayed>`_ matcher acts as a
placeholder which can be passed to other functions.  It can be defined later
using ``+=``::

  >>> expr   = Delayed()
  >>> number = Digit()[1:,...]
  >>> with Separator(r'\s*'):
  >>>     term    = number | '(' & expr & ')'
  >>>     muldiv  = Any('*/')
  >>>     factor  = term & (muldiv & term)[:]
  >>>     addsub  = Any('+-')
  >>>     expr   += factor & (addsub & factor)[:]


Imports
-------

The most commonly used classes are exposed via the `lepl <api/redirect.html#lepl>`_ module, so simple
scripts can use::

  from lepl import *

