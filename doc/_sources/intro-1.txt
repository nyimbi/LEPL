
.. _tutorial1:

Part 1 - Basic Matching
=======================

.. index:: parsing

What is Parsing?
----------------

When we talk about "parsing something" we generally have some data (often in
the form of text) and a description of the way that data is organised, and we
want to bring those two together so that we can break the data into known
pieces.

For example, we might want to write a program that lists the prices of various
items on Amazon's web site.  We can get the web page for each item (maybe
using Python's `HTTP client
<http://docs.python.org/3.0/library/http.client.html>`_), and we know that all
the pages have the same structure --- but how do we extract the price?  This is
where the parser comes in.  We give the parser a description of the page
structure, together with a web page, and it will return the price as a result.

That example is a little ambitious for a simple introduction.  Here we will
look at a simpler problem.  We will write a program that can take a simple
mathematical expression, like "1+2*3", understand the structure, and work out
the answer.  For example, when given "2+2" we want the result "4" (so we not
only break the input into pieces, but do something useful with it to get a
final result).

.. index:: Real(), import

Recognising a Number
--------------------

Lepl has built--in support for parsing a number.  We can see this by typing
the following at a Python prompt::

  >>> from lepl import *
  >>> Real().parse('123')
  ['123']

What is happening here?

The first line imports the contents of Lepl's main module.  Lepl is structured
as a collection of different packages, but the important functions and classes
are collected together in the `lepl <api/redirect.html#lepl>`_ module --- for most work this is all
you will need.

In the rest of the examples below I will assume that you have already imported
this module.

The second line creates a matcher --- `Real() <api/redirect.html#lepl.matchers.derived.Real>`_ (clicking on that link
will take you to the `API documentation <api>`_ that describes all Lepl's
modules, including the source code) --- and uses it to match the text "123".
The result is a list that contains the text "123".

In other words, `Real() <api/redirect.html#lepl.matchers.derived.Real>`_ looked at "123" and
recognised that it was a number.

What would happen if we gave `Real() <api/redirect.html#lepl.matchers.derived.Real>`_ something that wasn't
a number?  We can try it and see::

  >>> Real().parse('cabbage')
  [...]
  lepl.stream.maxdepth.FullFirstMatchException: The match failed in <string> at 'abbage' (line 1, character 2).

Which is reasonable enough --- "cabbage" is not a number.

But things are often not as simple as they first appear.  For example: why is
"123" a single number, and not three different numbers joined together?

.. index:: ambiguity

Ambiguity
---------

In fact, Lepl doesn't know that "123" is a single number.  Because of the way
`Real() <api/redirect.html#lepl.matchers.derived.Real>`_ is defined
internally, it gives the `longest` number it can find.  But that doesn't mean
it is the only result.  We can see all the different possibilities by calling
`parse_all() <api/redirect.html#lepl.core.config.ParserMixin.parse_all>`_ instead of `parse() <api/redirect.html#lepl.core.config.ParserMixin.parse>`_:

  >>> Real().parse_all('123')
  <map object at 0xdec950>

That will not seem very useful unless you already understand Python's
`generators <http://docs.python.org/3.0/glossary.html#term-generator>`_.  A
generator is something like a list that hasn't been built yet.  We can use it
in a ``for`` loop just like a list, for example::

  >>> for result in Real().parse_all('123'):
  ...     print(result)
  ...
  ['123']
  ['12']
  ['1']

Or we can create a list directly:

  >>> list(Real().parse_all('123'))
  [['123'], ['12'], ['1']]

Either way we can see that `Real() <api/redirect.html#lepl.matchers.derived.Real>`_ is giving us a choice
of different results.  It can match the number "123", or the number "12", or
the number "1".  Those are all the different numbers possible if you start
with the first character.

.. note::

   In the `parse_all() <api/redirect.html#lepl.core.config.ParserMixin.parse_all>`_ examples we don't get an error, even though the
   second and third matches don't match the whole stream.  That's because only
   the *first* match is checked to make sure that it consumes all the data
   (this is both for technical reasons and also because it's usually what you
   want).

.. index:: Float(), Real(), Integer()

More Ambiguity - Integers and Floats
------------------------------------

Sometimes we want a little less ambiguity when we are parsing numbers.  We may
want to match only Integers, or exclude integral values from reals.  We can do
both of these using `Integer()` and `Float()`.

  >>> Integer().parse('1')
  ['1']
  >>> Integer().parse('1.2')
  lepl.stream.maxdepth.FullFirstMatchException: The match failed in <string> at '2' (line 1, character 3).
  >>> Float().parse('1')
  lepl.stream.maxdepth.FullFirstMatchException: The match failed in <string> at '' (line 1, character 2).
  >>> Float().parse('1.2')
  ['1.2']
  >>> Real().parse('1')
  ['1']
  >>> Real().parse('1.2')
  ['1.2']

.. warning::

   The behaviour described above changed in version 4.4.  Before that,
   `Float() <api/redirect.html#lepl.support.warn.Float>`_ also matched integers.  To convert code from before version 4.4
   replace `Float() <api/redirect.html#lepl.support.warn.Float>`_ with `Real() <api/redirect.html#lepl.matchers.derived.Real>`_.


.. index:: &, And(), Literal()

Matching a Sum
--------------

So how do we extend matching a number to match a sum?

Here's the answer::

  >>> add = Real() & Literal('+') & Real()
  >>> add.parse('12+30')
  ['12', '+', '30']

In Lepl all that is necessary to join matchers together is ``&``.  This is
shorthand for::

  >>> add = And(Real(), Literal('+'), Real())
  >>> add.parse('12+30')
  ['12', '+', '30']

.. note::

   Later, when we meet :ref:`separators <separators>`, we'll see that `And() <api/redirect.html#lepl.matchers.combine.And>`_ and ``&`` aren't always
   exactly the same.  That's because ``&`` is an operator and operators can be
   redefined in Lepl (in the case of separators, for example, we redefine
   ``&`` to add extra spaces).

The parser above also used `Literal() <api/redirect.html#lepl.matchers.core.Literal>`_.  Like its name suggests,
this matches whatever value it is given::

  >>> matcher = Literal('hello')
  >>> matcher.config.no_full_first_match()
  >>> matcher.parse('hello world')
  ['hello']

Implicit Literals
-----------------

Often we can just use an ordinary string, instead of `Literal() <api/redirect.html#lepl.matchers.core.Literal>`_, and Lepl will still understand
what we mean::

  >>> add = Real() & '+' & Real()
  >>> add.parse('12+30')
  ['12', '+', '30']

Unfortunately this doesn't always work, and predicting exactly when it's going
to fail can be difficult (technically, the string must be an argument to a
matcher's overloaded operator or constructor).  So if you get a strange error
on a line with strings, try adding a `Literal() <api/redirect.html#lepl.matchers.core.Literal>`_ around the text --- after a
while you'll get a feeling for when it is needed, and when not.

Anyway, we still haven't added those numbers.  To do that we need to do
something with the results.

.. index:: ~, Drop()

Ignoring Values
---------------

To simplify adding the two values, we need to get rid of the "+" (please just
trust me on this; it will be clear why in a few more sections).

It is quite common when parsing data that we do not need to see all the values
we have matched.  That doesn't mean that it isn't important to do the match
--- in this case we need to check that there is a "+" between the two numbers
to be sure that we are doing the right thing by adding them --- but once we
have done that check, we don't actually want the "+" to be returned as a
result.

We can indicate that a match should be ignored by preceding the matcher with
``~``::

  >>> add = Real() & ~Literal('+') & Real()
  >>> add.parse('12+30')
  ['12', '30']

Just like ``&``, this is shorthand for another matcher, in this case
`Drop() <api/redirect.html#lepl.matchers.derived.Drop>`_::

  >>> add = Real() & Drop(Literal('+')) & Real()
  >>> add.parse('12+30')
  ['12', '30']

.. index:: >>

Creating Numbers
----------------

Our result above, ``['12', '30']``, is a list of numbers.  But the numbers are
still strings.  We need to convert them to floats before we can add them.  To
see what I mean, consider the two examples below::

  >>> 12 + 30
  42
  >>> '12' + '30'
  '1230'

We want the first case, not the second.

To do this we can define a new matcher, which takes the output from
`Real() <api/redirect.html#lepl.matchers.derived.Real>`_ (a list of strings) and passes each value in the list to the
Python built--in function, ``float()``::

  >>> number = Real() >> float

We can test this by calling `parse() <api/redirect.html#lepl.core.config.ParserMixin.parse>`_::

  >>> number = Real() >> float
  >>> number.parse('12')
  [12.0]

So now we can re-define `add <api/redirect.html#lepl.matchers.derived.add>`_ to use this matcher instead::

  >>> number = Real() >> float
  >>> add = number & ~Literal('+') & number
  >>> add.parse('12+30')
  [12.0, 30.0]

(I have repeated the definition of number here and in the previous example so
that each is complete by itself).

Note that, because ``>>`` works on each result in turn, we could have written
this in a different, but equivalent way::

  >>> add = (Real() & Drop(Literal('+')) & Real()) >> float
  >>> add.parse('12+30')
  [12.0, 30.0]

But as a general rule it is better to process results as soon as possible.
This usually keeps the parser simpler.

For more on ``>>`` you may find it useful to read :ref:`faq_apply`

Adding Values
-------------

Now that we have just the two numbers, we can add them.  How?  Well, we have a
list of numbers that we need to add, and Python has a function that does
exactly this, called ``sum()``::

  >>> sum([1,2,3])
  6

So we can send our results to that function::

  >>> number = Real() >> float
  >>> add = number & ~Literal('+') & number > sum
  >>> add.parse('12+30')
  [42.0]

which gives the answer we wanted!

.. note::

   The difference between ``>`` and ``>>`` is quite subtle, but important:
   ``>`` sends the entire list of results to a function as a single argument
   (so the function must take a list of values), while ``>>`` sends each
   result separately (so the function must take a single value).

We have come a long way --- from nothing to a parser that can add two numbers.
In the next section we will make this more robust, allowing us to have spaces
in the expression.

Summary
-------

What have we learnt so far?

* Parsing is all about recognising structure (eg. mathematical expressions).

* Once we have recognised structure we can process it (eg. adding numbers
  together).

* To use Lepl we must first use import the lepl module: ``from lepl import
  *``.

* Lepl builds up a parser using functions (which I call "matchers").

* Matchers can return one value (with `parse() <api/redirect.html#lepl.core.config.ParserMixin.parse>`_) or all possible values
  (with `parse_all() <api/redirect.html#lepl.core.config.ParserMixin.parse_all>`_).

* We can join matchers together with ``&`` or `And() <api/redirect.html#lepl.matchers.combine.And>`_.

* We can ignore the results of a matcher with ``~`` or `Drop() <api/redirect.html#lepl.matchers.derived.Drop>`_.

* We can process each value in a list of results with ``>>``.

* We can process the list of results (as a complete list) with ``>``.
