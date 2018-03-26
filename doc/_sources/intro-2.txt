

Part 2 - Handling Spaces, Repetition
====================================

Recap
-----

In the previous chapter we defined a parser that could identify the different
parts of an addition, separate out the numbers, and return their sum::

  >>> from lepl import *
  >>> number = Real() >> float
  >>> add = number & ~Literal('+') & number > sum
  >>> add.parse('12+30')
  [42.0]

(remember that I will not repeat the import statement in the examples below).

An obvious problem with this parser is that it does not handle spaces::

  >>> add.parse('12 + 30')
  [...]
  lepl.stream.maxdepth.FullFirstMatchException: The match failed in <string> at '+ 30' (line 1, character 4).

So in this section we'll look at the various ways we can handle spaces (and
learn more about other features of Lepl along the way).

.. index:: Space()

Explicit Spaces
---------------

The simplest way to handle spaces is to add them to the parser.  Lepl includes
the `Space() <api/redirect.html#lepl.matchers.derived.Space>`_ matcher which
recognises a single space::

  >>> number = Real() >> float
  >>> add = number & ~Space() & ~Literal('+') & ~Space() & number > sum
  >>> add.parse('12 + 30')
  [42.0]

But now our parser won't work without spaces!
::

  >>> add.parse('12+30')
  [...]
  lepl.stream.maxdepth.FullFirstMatchException: The match failed in <string> at '30' (line 1, character 4).

.. index:: Star()

The Star Matcher
----------------

To fix the problem described above, where we can match only a single space, we
would like to match any number of spaces.  There are various ways of doing
this.  If you are used to using regular expressions you may realise that this
is what the "*" symbol does.  And in Lepl we have something similar.

The `Star() <api/redirect.html#lepl.matchers.derived.Star>`_ matcher repeats its
argument as many times as necessary (including none at all).  This is what we
need for our spaces::

  >>> number = Real() >> float
  >>> spaces = ~Star(Space())
  >>> add = number & spaces & ~Literal('+') & spaces & number > sum
  >>> add.parse('12 + 30')
  [42.0]
  >>> add.parse('12+30')
  [42.0]
  >>> add.parse('12+     30')
  [42.0]

Note that I included a ``~`` in the definition of ``spaces`` so that they are
dropped from the results.

.. index:: [], repetition

Repetition
----------

As well as `Star() <api/redirect.html#lepl.matchers.derived.Star>`_, Lepl
supports a more general way of specifying repetitions.  This uses Python's
array syntax, which looks a bit odd at first, but turns out to be a really
neat, compact, powerful way of describing what we want.

The easiest way to show how this works is with some examples.

First, here's how we specify that exactly three things are matched:

  >>> a = Literal('a')
  >>> a[3].parse('aaa')
  ['a', 'a', 'a']

and here's how we specify that 2 to 4 should be matched:

  >>> a[2:4].parse('aa')
  ['a', 'a']
  >>> a[2:4].parse('aaaa')
  ['a', 'a', 'a', 'a']
  >>> list(a[2:4].parse_all('aaaa'))
  [['a', 'a', 'a', 'a'], ['a', 'a', 'a'], ['a', 'a']]

As we saw earlier `parse_all() <api/redirect.html#lepl.core.config.ParserMixin.parse_all>`_ returns a generator (which we convert to a
list) that contains all the different possible combinations: 2, 3 and 4
letters, while `parse() <api/redirect.html#lepl.core.config.ParserMixin.parse>`_ returns just the first result (repetition with
``[]`` returns the largest number of matches first).

If we give a range with a missing start value then the minimum number of
matches is zero:

  >>> list(a[:1].parse_all('a'))
  [['a'], []]

so here we have 0 or 1 matches (zero matches means we get an empty list of
results --- that's `not` the same as failing to match).

And if the end value is missing as many as possible will be matched:

  >>> list(a[4:].parse_all('aaaaa'))
  [['a', 'a', 'a', 'a', 'a'], ['a', 'a', 'a', 'a']]

Finally, we can get the shortest number of matches first by specifying an
array index "step" of ``'b'`` (short for "breadth--first search"; the default
is ``'d'`` for "depth--first")::

  >>> a24 = Literal('a')[2:4:'b']
  >>> a24.config.no_full_first_match()
  >>> list(a24.parse_all('aaaa'))
  [['a', 'a'], ['a', 'a', 'a'], ['a', 'a', 'a', 'a']]

Putting all that together, `Star() <api/redirect.html#lepl.matchers.derived.Star>`_ is the same as ``[:]`` (which
starts at zero, takes as many as possible, and returns the longest match
first).

So we can write our parser like this::

  >>> number = Real() >> float
  >>> spaces = ~Space()[:]
  >>> add = number & spaces & ~Literal('+') & spaces & number > sum
  >>> add.parse('12 + 30')
  [42.0]
  >>> add.parse('12+30')
  [42.0]
  >>> add.parse('12+     30')
  [42.0]

That's perhaps not as clear as using `Star() <api/redirect.html#lepl.matchers.derived.Star>`_, but personally I prefer this
approach so I'll continue to use it below.

.. index:: ...

More Repetition
---------------

While we are looking at ``[]`` I should quickly explain two extra features
which are often useful.

First, including ``...`` will join together the results::

  >>> a[3].parse('aaa')
  ['a', 'a', 'a']
  >>> a[3,...].parse('aaa')
  ['aaa']

Second, we can specify a "separator" that is useful when matching lists.  This
is used to match "in-between" whatever we are repeating.  For example, we
might have a sequence of "a"s separated by "x"s, which we want to ignore::

  >>> a[3,Drop('x')].parse('axaxa')
  ['a', 'a', 'a']


.. index:: Separator()
.. _separators:

Separators
----------

Enough about repetition; let's return to our main example.

The solution above works fine, but it gets a bit tedious adding ``spaces``
everywhere.  It would be much easier if we could just say that they should be
added wherever there is a ``&``.  Luckily, we can do that in Lepl::

  >>> number = Real() >> float
  >>> spaces = ~Space()[:]
  >>> with Separator(spaces):
  ...   add = number & ~Literal('+') & number > sum
  ...
  >>> add.parse('12 + 30')
  [42.0]
  >>> add.parse('12+30')
  [42.0]

Which works as before, but can save some typing in longer programs.

`Separator() <api/redirect.html#lepl.matchers.operators.Separator>`_
redefines the ``&`` and ``[]`` operators to include spaces.  The matcher
associated with any operator can be redefined in Lepl, but doing so is pretty
advanced and outside the scope of this tutorial.

Because `Separator() <api/redirect.html#lepl.matchers.operators.Separator>`_
changes everything "inside" the "with" it's usually best to define matchers
that *don't* need spaces beforehand.

.. warning::

   `Separator() <api/redirect.html#lepl.matchers.operators.Separator>`_ only
   modifies ``&`` and ``[]``, which can lead to (at least) two surprising
   results.

   First, there's nothing added before or after any pattern that's defined.
   For that, you still need to explicitly add spaces as described earlier.
   `Separator() <api/redirect.html#lepl.matchers.operators.Separator>`_ only
   adds spaces *between* items joined with ``&``.

   Second, if you specify *at least one* space (rather than *zero or more*)
   then *every* ``&`` in the separator's context *must* have a space.  This
   can be surprising if you have, for example, ``& Eos()`` because it means
   that there *must* be a space before the end of the stream.

   You can avoid spaces in two ways.  Either define matchers that don't need
   spaces *before* you use `Separator() <api/redirect.html#lepl.matchers.operators.Separator>`_, or use `And() <api/redirect.html#lepl.matchers.combine.And>`_ instead.

Finally, because this is so common, `DroppedSpace() <api/redirect.html#lepl.matchers.operators.DroppedSpace>`_, is pre--defined::

  >>> number = Real() >> float
  >>> with DroppedSpace():
  ...   add = number & ~Literal('+') & number > sum
  ...
  >>> add.parse('12 + 30')
  [42.0]
  >>> add.parse('12+30')
  [42.0]

.. index:: regular expressions

Regular Expressions
-------------------

I'm going to take a small diversion now to discuss regular expressions.  Once
I've finished I'll return to the issue of spaces with a different approach.

Regular expressions are like "mini-parsers".  They are used in a variety of
languages, and Python has a `module
<http://docs.python.org/3.0/library/re.html>`_ that supports them.  I don't
have space here (or the time and energy) to explain them in detail, but the
basic idea is that you can write description (an "expression") for a sequence
of letters to be matched.  This expression can contain things like "." which
matches any letter, or "[a-m]" which matches any letter between "a" and "m",
for example.

So regular expressions are very like a parser.  But a parser can usually
(exact details depend on the language and parser) describe more complicated
structures and tends to be easier to use for "big" problems.

That doesn't mean that regular expressions don't play a part in Lepl.  In
fact, Lepl supports three kinds of regular expressions, and I will describe
these below.  But please note that all the options below have limitations ---
Lepl is a parser in its own right and does not need powerful regular
expressions.


.. index:: Regexp()

Regexp()
--------

The `Regexp() <api/redirect.html#lepl.matchers.core.Regexp>`_ matcher calls
the Python regular expression library.  So if you are experienced at using
that you may find it useful.

However, there are some limitations.  First, the interface exposed by Lepl
doesn't include all Python's options (it would make things too complicated and
Lepl has other ways of doing things --- sorry!).

Second, the expression is only matched against the "current line".  Exactly
what the "current line" is depends on some internal details (sorry again), but
you should work on the assumption that the regular expression will only
receive data up to the next newline character.

The reason for this second limitation is that Lepl is quite careful about how
it manages memory.  In theory it should be possible to process huge amounts of
text, because only a section of the document is held in memory at any one
time.  Unfortunately that doesn't play well with Python's regular expressions,
which expect all the data to be in a single string.

Here are some examples showing what is possible::

  >>> matcher = Regexp('a+')
  >>> matcher.config.no_full_first_match()
  >>> matcher.parse('aaabb')
  ['aaa']
  >>> matcher = Regexp(r'\w+')
  >>> matcher.config.no_full_first_match()
  >>> matcher.parse('abc def')
  ['abc']
  >>> matcher = Regexp('a*(b*)c*(d*)e*')
  >>> matcher.config.no_full_first_match()
  >>> matcher.parse('abbcccddddeeeeee')
  ['bb', 'dddd']

The last example above shows how groups can be used to define results.

.. index:: DfaRegexp()

DfaRegexp()
-----------

The `DfaRegexp() <api/redirect.html#lepl.regexp.matchers.DfaRegexp>`_ matcher
calls Lepl's own regular expression library.  It understands simple regular
expressions, but it does not support grouping, references, etc.

  >>> matcher = DfaRegexp('a*b')
  >>> matcher.config.no_full_first_match()
  >>> matcher.parse('aabbcc')
  ['aab']

.. index:: NfaRegexp()

NfaRegexp()
-----------

This is implemented by Lepl's own regular expression library and, like
`DfaRegexp() <api/redirect.html#lepl.regexp.matchers.DfaRegexp>`_, is limited
in what it supports.

`NfaRegexp() <api/redirect.html#lepl.regexp.matchers.NfaRegexp>`_ differs from
"normal" regular expressions in that it can return multiple matches (usually a
regular expression returns only the "longest match")::

  >>> list(NfaRegexp('a*').parse_all('aaa'))
  [['aaa'], ['aa'], ['a'], ['']]
  >>> list(DfaRegexp('a*').parse_all('aaa'))
  [['aaa']]
  >>> list(Regexp('a*').parse_all('aaa'))
  [['aaa']]

.. index:: tokens, Token()

Tokens (First Attempt)
----------------------

Now that we have discussed regular expressions I can explain the final
alternative for handling spaces.

This approach uses regular expressions to classify the input into different
"tokens".  It then lets us match both the token type and, optionally, the
token contents.

By itself, this doesn't make handling spaces any simpler, but we can also tell
Lepl to ignore certain values.  So if we define tokens for the different
"words" we will need, we can then tell Lepl to discard any spaces that occur
between (in fact, by default, spaces are discarded, so we don't need to
actually say that below).

For more detailed information on tokens, see :ref:`lexer` in the manual.


First, let's define the tokens we will match.  We don't have to be very
precise here because we can add more conditions later --- it's enough to
identify the basic types of input.  For our parser these will be values and
symbols::

  >>> value = Token(Real())
  >>> symbol = Token('[^0-9a-zA-Z \t\r\n]')

I said that we defined tokens with regular expressions, but the definition of
``value`` above seems to use the matcher `Real() <api/redirect.html#lepl.matchers.derived.Real>`_.  This is because Lepl
can automatically convert some matchers into regular expressions, saving us
the work (it really does convert them, piece by piece, so it is not limited to
the built--in matchers, but it is limited by how the matcher is constructed --
it cannot see "inside" arbitrary function calls, for example, so any matcher
that includes ``>`` or ``>>`` won't work).

The second token, defined with the regular expression "[^0-9a-zA-Z \\t\\r\\n]"
means "any single character that is not a digit, letter, or space".  Obviously
we will need to add extra conditions for matching "+" and, later, "*", "-",
etc.

With those tokens we can now try to rewrite our parser::

  >>> number = value >> float
  >>> add = number & ~symbol('+') & number > sum
  >>> add.parse('12+30')
  [...]
  lepl.stream.maxdepth.FullFirstMatchException: The match failed in <string> at '+30' (line 1, character 3).

Ooops.  That is not what we wanted!

Before we fix the problem, though, I need to explain a detail above.

The matcher, ``symbol('+')`` is the same as ``symbol(Literal('+'))`` and means
that we require a symbol token *and* that the text in that token matches "+"
(this is what I was referring to when I said that we match both the *type* of
token and it's *contents*).  A token used like this can contain any Lepl
matcher as a constraint (well, anything except `Token() <api/redirect.html#lepl.lexer.matchers.Token>`_ itself).

.. index:: debugging

Debugging
---------

What went wrong in the example above?

There is a clue in the error message --- when we use tokens the "match failed
at" message shows the token::

  lepl.stream.maxdepth.FullFirstMatchException: The match failed in <string> at '+30' (line 1, character 3).

That means that we have a token whose value is "+30", which is not what we
were expecting.  We expected that the tokens would be "12", "+", and "30".
Instead, it seems that the tokens generated are "12" and "+30".

So we can see that the lexer (the part of Lepl that generates the tokens) is
identifying two `Real() <api/redirect.html#lepl.matchers.derived.Real>`_ matches.  Matching "+"
as a ``symbol`` is ignored because `the lexer chooses the token with the
longest match` and "+30" is longer than "+".

In a little more detail: the lexer takes the input and breaks it down into
tokens, from left to right.  So in this case it starts with "12+30", tries
matching the various tokens, and finds that "12" is the longest (and only)
match.  It then starts again with what remains, "+30" and finds a match of "+"
for ``symbol`` and a match of "+30" for ``value``.  It chooses the latter
because it is longest, and is done.

This illustrates an important restriction on the use of tokens: you have to be
careful to avoid ambiguity.  This might make them seem pointless, but in
practice their advantages --- in particular, simplifying handling spaces ---
often make them worthwhile.

.. index:: tokens
.. _token_example:

Tokens (Second Attempt)
-----------------------

We can avoid the problem above by using unsigned numbers.  But that means that
we need to worry about signs that are "part of the number" in the parser
itself.  Since people don't really care about a leading "+" I've only included
the "-" case (negative numbers) below::

  >>> value = Token(UnsignedReal())
  >>> symbol = Token('[^0-9a-zA-Z \t\r\n]')
  >>> number = Optional(symbol('-')) + value >> float
  >>> add = number & ~symbol('+') & number > sum
  >>> add.parse('12+30')
  [42.0]
  >>> add.parse('12 + -30')
  [-18.0]

The important changes here are:

* ``value`` is changed to an `UnsignedReal() <api/redirect.html#lepl.matchers.derived.UnsignedReal>`_

* number has an `Optional() <api/redirect.html#lepl.matchers.derived.Optional>`_ minus (we could also
  have written this ``symbol('-')[0:1]``)

* the ``+`` joins the optional ``-`` and ``value`` together into a single
  string, so that when passed to ``float()`` a negative number will be
  created

Alternative Spaces
------------------

Finally, it is worth noting that you can specify an alternative regular
expression that will be used to match spaces between tokens.  The way that
Lepl works is as follows:

1. An attempt is made to match a token.

2. If no token matches, an attempt is made to match spaces.

3. If no spaces could be matched, an error is raised.

The spaces matched in step 2 are defined via a regular expression, which can
be passed to the :ref:`configuration` (the ``discard`` parameter to
`.config.lexer() <api/redirect.html#lepl.core.config.ConfigBuilder.lexer>`_).
If no value value is given, "[\\r\\n\\t ]+" is used.

Summary
-------

What more have we learnt?

* To handle spaces, we can specify them explicitly.

* The ``[]`` syntax for repetition is compact and powerful.

* `Separator() <api/redirect.html#lepl.matchers.operators.Separator>`_ can
  automate the addition of spaces wherever we use ``&`` or ``[]``.

* Regular expressions are supported, in various different ways.

* Lepl has an optional lexer, which generates tokens using regular
  expressions.

* Because regular expressions are "greedy", always matching the longest amount
  of text possible, we need to be careful exactly how we define our tokens.

* In particular, we should worry when two different tokens overlap (in our
  case, a possible ``symbol``, "+", was also the start of a valid ``value``,
  "+3.0").
