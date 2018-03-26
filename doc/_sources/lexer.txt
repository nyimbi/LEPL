
.. index:: lexer
.. _lexer:

Lexer
=====

Lepl's lexer has not been used in earlier chapters of this manual.  However, 
for many applications it will both simplify the grammar and give a more 
efficient parser.  Some extensions, like :ref:`Line-Aware Parsing 
<offside>`, assume its use. 


Introduction
------------

The lexer pre-processes the stream that is being parsed, dividing it into
tokens that correspond to regular expressions.  The tokens, and their
contents, can then be matched in the grammar using the `Token() <api/redirect.html#lepl.lexer.matchers.Token>`_ matcher.

Tokens give a "rough" description of the text to be parsed.  A simple parser
of numerical expressions might have the following different token types:

  * Numeric values (eg. 1, 2.3, 4e5).

  * Function names (eg. sin, cos).

  * Symbols (eg. ``(``, ``+``).

Note that in the sketch above, a token is not defined for spaces.  The lexer
includes a spearate pattern that is used only if all others fail --- if this
matches, the matching text is discarded (otherwise, the lexer raise an error).
By default this "discard pattern" matches whitespace, so spaces are
automatically discarded and do not need to be specified in the grammar.


.. index:: Token()

Use
---

Within a grammar, a `Token() <api/redirect.html#lepl.lexer.matchers.Token>`_
appears much like any other matcher.  For a full example, see 
:ref:`tutorial`.

However, to use tokens correctly, it is necessary to understand how they work
in a little more detail.

Tokens are used in two ways.

First, they work like matchers (with the exceptions noted below) for the
regular expression given as their argument.  For example::

  >>> name = Token('[A-Z][a-z]*')
  >>> number = Token(Integer())

Here, ``name`` will match a string that starts with a capital letter and then
has zero or more lower case letters.  The second `Token() <api/redirect.html#lepl.lexer.matchers.Token>`_, ``number``, is similar, but
uses another matcher (`Integer() <api/redirect.html#lepl.matchers.derived.Integer>`_) to define the regular
expression that is matched.

.. note::

  Not all matchers can be used to define the pattern for a `Token() <api/redirect.html#lepl.lexer.matchers.Token>`_ --- only those that Lepl
  knows how to convert into regular expressions.

The second way in which tokens are used is by specialisation, which generates
a *new* matcher with an *additional* constraint.  This is easiest to see with
another example::

  >>> params = ...
  >>> function = Token('[a-z]*')
  >>> sin = function('sine')
  >>> cos = function('cosine')
  >>> call = (sin | cos) & params

Here the ``function`` `Token() <api/redirect.html#lepl.lexer.matchers.Token>`_
is restricted to match the strings "sine" and "cosine", creating two *new*
matchers, ``sin`` and ``cos``.

The example above used simple strings (which are converted to `Literal() <api/redirect.html#lepl.matchers.core.Literal>`_ matchers) as the additional
constraints, but any matcher expression can be used.

For a more realistic example of multiple specialisations, see the use of
``symbol`` in :ref:`this example <token_example>`.

Note that specialisation is optional.  It's OK to use `Token() <api/redirect.html#lepl.lexer.matchers.Token>`_ as a simple matcher without
adding any more constraints (providing that the use is consistent with the
limitations outlined below).


.. _limitations:

Limitations
-----------

The lexer has two limitations: it cannot backtrack and `Token() <api/redirect.html#lepl.lexer.matchers.Token>`_ matchers cannot be mixed with
non--Token matchers that consume input.

The first limitation means that the regular expressions associated with tokens
only match text once.  The lexer divides the input into fixed blocks that
match the largest possible fragment; no alternatives are considered.

This does not mean that a particular `Token() <api/redirect.html#lepl.lexer.matchers.Token>`_ is only called once, or that
it will return only a single result.  *Backtracking is still possible, but the
parser is restricted to exploring different interpretations of a single,
unchanging token stream.*

For example, consider "1-2", which might be parsed as two integers (1 and -2),
or as a subtraction expression (1 minus 2).  An appropriate matcher will give
both results, through backtracking::

  >>> matchers = (Integer() | Literal('-'))[:] & Eos()
  >>> list(matchers.parse_all('1-2'))
  [['1', '-2'], ['1', '-', '2']]

But when tokens are used, "-2" is preferred to "-", because it is a longer
match, so we get only the single result::

  >>> tokens = (Token(Integer()) | Token(r'\-'))[:] & Eos()
  >>> list(tokens.parse_all('1-2'))
  [['1', '-2']]

(In the examples above, ``list()`` is used to expand the generator and the
`Token() <api/redirect.html#lepl.lexer.matchers.Token>`_ is given ``r'\-'``
because its argument is a regular expression, not a literal value.)

The second limitation is more subtle.  The lexer is implemented via a
:ref:`rewriter <rewriting>` which adds a `Lexer() <api/redirect.html#lepl.lexer.lexer.Lexer>`_ instance to the head of the
matcher graph.  This divides the input into the "pieces" that the `Token() <api/redirect.html#lepl.lexer.matchers.Token>`_ matchers expect.

So matchers receive a stream of labelled fragments from `Lexer() <api/redirect.html#lepl.lexer.lexer.Lexer>`_.  It is only "inside" each
`Token() <api/redirect.html#lepl.lexer.matchers.Token>`_, when the fragment is
passed to the sub--matcher, that the stream is returned to its original
format.

As a consequence, matchers that read the stream --- those that consume data,
like `Any() <api/redirect.html#lepl.matchers.core.Any>`_ or `Literal() <api/redirect.html#lepl.matchers.core.Literal>`_ --- can only be used *inside*
`Token() <api/redirect.html#lepl.lexer.matchers.Token>`_.  If they are used
alongside the following error occurs::

  >>> matcher = Token(Any()) & Any()
  ...
  >>> matcher.parse(...)
  lepl.lexer.support.LexerError: The grammar contains a mix of Tokens and non-Token matchers at the top level. 
  If Tokens are used then non-token matchers that consume input must only appear "inside" Tokens.
  The non-Token matchers include: Any(None).

.. index:: lexer_rewriter(), Configuration()

Advanced Options
----------------

Configuration

  The lexer can be configured using `.config.lexer() <api/redirect.html#lepl.core.config.ConfigBuilder.lexer>`_ (see
  ref:`configuration`).  This can take an additional argument that specified
  the discard pattern.  Note that this is included in the default
  configuration (it does no harm if tokens are not used).

Completeness

  By default Tokens require that any sub--expression consumes the entire
  contents::

    >>> abc = Token('abc')
    >>> incomplete = abc(Literal('ab'))
    >>> incomplete.parse('abc')
    [...]
    lepl.stream.maxdepth.FullFirstMatchException: The match failed in <string> at 'c' (line 1, character 3).

  However, this constraint can be relaxed, in which case the matched portion is
  returned as a result::

    >>> abc = Token('abc')
    >>> incomplete = abc(Literal('ab'), complete=False)
    >>> incomplete.parse('abc')
    ['ab']


Example
-------

:ref:`tutorial` contains a complete, worked example using `Token() <api/redirect.html#lepl.lexer.matchers.Token>`_.


.. _lexer_process:

The Lexer Process
-----------------

In the explanations above I try to describe the `Token() <api/redirect.html#lepl.lexer.matchers.Token>`_ matcher in a fairly
declarative way.  However, I know that it is sometimes easier to understand
how to use a tool by first understanding how the tool itself works.  So here I
will sketch how the lexer is implemented by describing the steps involved when
a Python program uses the Lepl parser, with the lexer, to parse some text.

#. Python compilation

   The program containing Lepl code (and the Lepl library) are compiled.

#. Python execution

   The program is then run.

#. Creation of matcher graph

   A function, or set of statements, that generates the Lepl matchers is
   evaluated.  Matchers like `Token() <api/redirect.html#lepl.lexer.matchers.Token>`_, `And() <api/redirect.html#lepl.matchers.combine.And>`_, etc., are objects that
   link to each other.  The objects and their links form a graph (with a
   matcher object at each node).

   * Token numbering

     Each time a `Token() <api/redirect.html#lepl.lexer.matchers.Token>`_ is
     created it is assigned a unique number, which I will call the "tag".

   * Regular expression extraction

     Whenever a `Token() <api/redirect.html#lepl.lexer.matchers.Token>`_ is
     created with another matcher as an argument Lepl attempts to convert the
     matcher to a regular expression.  If it cannot do so, it raises an error.

   * Token specialisation

     A token is "specialised" when it is given a sub--matcher::

       >>> function = Token('[a-z]*')
       >>> sin = function('sine')

     In the example above, the first line creates a new `Token() <api/redirect.html#lepl.lexer.matchers.Token>`_, with a unique tag and a
     regular expression, as explained just above.  On the second line the
     token is specialised.  This creates another `Token() <api/redirect.html#lepl.lexer.matchers.Token>`_, which contains the given
     sub--matcher (a `Literal() <api/redirect.html#lepl.matchers.core.Literal>`_ in this case), but with
     the same tag and regular expression as the "parent".

     I call a token like this, which has the same tag and regular expression
     as the parent, but also contains a sub--matcher, a "specialised token" in
     the description below.

#. Parser compilation

   At some point Lepl internally "compiles" the matcher graph to generate a
   parser.  Exactly when this happens depends on how the matchers are used,
   but at the latest it happens just before the first match is calculated.

   "Compilation" is perhaps misleading --- the parser is not compiled to
   Python byte codes, for example.  What happens is that the matcher graph is
   processed in various ways.  The most important processing, in terms of the
   lexer, is...

#. Lexer rewriting

   The lexer rewriter uses the matcher graph to construct a `Lexer() <api/redirect.html#lepl.lexer.lexer.Lexer>`_
   instance:

   * `Token() <api/redirect.html#lepl.lexer.matchers.Token>`_ instances are
     collected.  

   * The graph is checked to make sure that tokens and non-token matchers are
     not used together (see :ref:`limitations` above).

   * The regular expressions and tags associated with the tokens are collected
     together.  Duplicate tags and expressions (from specialised tokens) are
     ignored --- at this part of the process, a specialised token is no
     different to the original unspecialised parent.

   * A regular expression matcher is generated, which can match the different
     expressions and return the text and tag(s) associated with the longest
     match.

   * A `Lexer() <api/redirect.html#lepl.lexer.lexer.Lexer>`_ is added to
     the "head" of the matcher graph.  It contains the regular expression
     matcher.

   The modified matcher graph is then complete and returned for evaluation.

#. Parser evaluation

   When the parser is finally called, by passing it some text to process, the
   matcher graph has already been prepared for lexing, as described above.
   The following processes then occur:

   * A stream may be constructed that wraps the input text.  Whether this
     happens depends on the method called.

   * The input (as stream or text) is passed to the head of the matcher graph,
     which is the `Lexer() <api/redirect.html#lepl.lexer.lexer.Lexer>`_
     instance constructed earlier.

   * The lexer generates a new stream, which encapsulates both the input text
     and the regular expression matcher.  This new stream is a stream of
     tagged fragments --- each fragment is a match from the regular expression
     matcher, and it is associated with the list of tags that identifies which
     tokens had regular expressions that matched the fragment (more than one
     of the token regular expressions may match a single piece of text).

   * The new stream of tagged fragments is passed to to the matcher graph in
     the same way as normal.

   * When a `Token() <api/redirect.html#lepl.lexer.matchers.Token>`_ receives
     the stream it checks whether the first item in the stream is tagged with
     its own tag.

     * If the tag does not match, the token matcher fails.

     * If the tag matches and the token contains a sub--matcher (ie, if it is
       a specialised token), then the fragment of text is passed to the
       sub--matcher for processing.  If the sub--matcher returns a result then
       that result is returned by the token.  Alternatively, if the
       sub--matcher fails then the token fails too.

     * If the tag matches and the token has no sub--matcher (ie, if it is not
       specialised), then the token returns the entire fragment as the result
       of a successful match.

   * Evaluation continues in the usual manner, returning a list of results.
