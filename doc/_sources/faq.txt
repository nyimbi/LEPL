
.. _faq:

Frequently Asked Questions
==========================

 * :ref:`Why do I get "Cannot parse regexp..."? <faq_regexp>`
 * :ref:`Why isn't my parser matching the full expression? <faq_lefttoright>`
 * :ref:`Why does using Or() stop a full match from happening? <faq_or_bug>`
 * :ref:`How do I parse an entire file? <faq_file>`
 * :ref:`When I change from > to >> my function isn't called <faq_precedence>`
 * :ref:`How do I choose between > and >> ? <faq_apply>`
 * :ref:`Why am I seeing "No handlers could be found..." messages? <faq_logging>`
 * :ref:`Why does my matcher take so long to compile? <faq_slowregexp>`


.. _faq_regexp:

Why do I get "Cannot parse regexp..."?
--------------------------------------

*Why do I get "Cannot parse regexp '(' using ..." for Token('(')?*

String arguments to `Token() <api/redirect.html#lepl.lexer.matchers.Token>`_
are treated as regular expressions.  Because ``(`` has a special meaning in a
regular expression you must escape it, like this: ``Token('\\(')``, or like
this: ``Token(r'\(')``


.. _faq_lefttoright:

Why isn't my parser matching the full expression?
-------------------------------------------------

*In the code below*::

    word = Token('[a-z]+')
    lpar = Token('\\(')
    rpar = Token('\\)')
    expression = word | (word & lpar & word & rpar)
    
*why does expression.parse('hello(world)') match just 'hello'*?

In general Lepl is greedy (it tries to matches the longest possible string), 
but for `Or() <api/redirect.html#lepl.matchers.combine.Or>`_ it will try alternatives left-to-right.  So in this case you 
should rewrite the parser as::

    expression = (word & lpar & word & rpar) | word
    
Alternatively, you can force the parser to match the entire input by ending
with `Eos() <api/redirect.html#lepl.matchers.core.Eos>`_::

    expression = word | (word & lpar & word & rpar)
    complete = expression & Eos()   

See also the next answer.


.. _faq_or_bug:

Why does using Or() stop a full match from happening?
-----------------------------------------------------

*Why does this code*::

    >>> matcher = Letter() | "ab"
    >>> matcher.parse("a")
    ['a']
    >>> matcher.parse("ab")
    lepl.stream.maxdepth.FullFirstMatchException: The match failed at 'b'

*fail?*

OK, so this behaviour does seem odd, I agree.  But it's a logical consequence
of some other design decisions, all of which seem individually reasonable.  So
I'll explain those and, hopefully, that will shed some light on this.

#. `Or() <api/redirect.html#lepl.matchers.combine.Or>`_ is not greedy

   Repetition in Lepl is greedy by default, but Or() isn't.  If it can match
   the first option, it will do so.  But it will try other possibilities if
   that fails, or if all possible parses are requested.

   This is because there is no way to predict which option will return "most".
   So if `Or() <api/redirect.html#lepl.matchers.combine.Or>`_ were greedy it
   would need to evaluate every possible option, measure them, and return the
   "largest".  This could require a lot of memory and time.  Instead, it
   returns the first match it finds, but then supports backtracking.

   (Note that this is similar to regular expressions, except that Perl regexps
   are even worse - they don't backtrack).

   If that's not what you want there is, fortunately, a solution.  Please read
   on...

#. Lepl doesn't force you to match the entire input

   The "fundamental" parsing operation in Lepl is `matcher.match() <api/redirect.html#lepl.core.config.ParserMixin.match>`_.  This returns a
   list of pairs.  Each pair combines a result list with `the remaining
   input`.  There's nothing in that that says you need to match the entire
   input, because that's not the most general behaviour.

   For example::

    >>> matcher = Letter() | "ab"
    >>> matcher.config.no_full_first_match()
    >>> matcher.match("ab")
    <generator object trampoline at 0x916640>
    >>> list(matcher.match("ab"))
    [([u'a'], (1, <helper>)), (['ab'], (2, <helper>))]

   Here you can see, in detail, what Lepl is doing.  The ``(n, <helper>)``
   values are the remaining input, from index 1 and 2 respectively.

   If you *want* to match the whole input you can add `Eos() <api/redirect.html#lepl.matchers.core.Eos>`_ to the matcher::

    >>> matcher = (Letter() | "ab") & Eos()
    >>> list(matcher.match("ab"))
    [(['ab'], ''[0:])]

#. The "full first match" implementation is very simple.  It checks the
   remaining stream (see above) for the first match.  If it is not empty, then
   the error is raised.

   Why didn't I make this also add `Eos() <api/redirect.html#lepl.matchers.core.Eos>`_?  I could have done so, and
   then I wouldn't have had to write this explanation, but it would have meant
   adding more "magic" to the configuration system.  I did start to do this,
   but then I realised that *disabling the check could change the parse
   results*.  And I think that's a worse problem than the current (imperfect)
   compromise.

In summary then, this is a consequence of the way `Or() <api/redirect.html#lepl.matchers.combine.Or>`_ works (for efficiency), and
the way that Lepl does backtracking (for generality) and a desire to keep the
"full first match" code separate from "what the parser matches".  I know it's
a little confusing at first, but I don't see a better solution.  Sorry!

See also the previous answer.


.. _faq_file:

How do I parse an entire file?
------------------------------

*I understand how to parse a string, but how do I parse an entire file?*

Instead of `matcher.parse() <api/redirect.html#lepl.core.config.ParserMixin.parse>`_ or
`matcher.parse_string() <api/redirect.html#lepl.core.config.ParserMixin.parse_string>`_ use
`matcher.parse_file() <api/redirect.html#lepl.core.config.ParserMixin.parse_file>`_.  For example:

  >>> with open('myfile') as input:
  ...     return matcher.parse_file(input)

Matchers extend `ParserMixin() <api/redirect.html#lepl.core.config.ParserMixin>`_, which provides these
methods.


.. _faq_precedence:

When I change from > to >> my function isn't called
---------------------------------------------------

*Why, when I change my code from*::

    inverted = Drop('[^') & interval[1:] & Drop(']') > invert
    
*to*::
          
    inverted = Drop('[^') & interval[1:] & Drop(']') >> invert      

*is the `invert` function no longer called?*

This is because of operator precedence.  ``>>`` binds more tightly than ``>``,
so ``>>`` is applied only to the result from `Drop(']')
<api/redirect.html#lepl.matchers.derived.Drop>`_, which is an empty list
(because `Drop() <api/redirect.html#lepl.matchers.derived.Drop>`_ discards the
results).  Since the list is empty, the function ``invert`` is not called.

To fix this place the entire expression in parentheses::

    inverted = (Drop('[^') & interval[1:] & Drop(']')) >> invert      


.. _faq_apply:

How do I choose between > and >> ?
----------------------------------

To understand > and >> it's important that you first see that Lepl is designed
to work with lists of results.  For example, `Any() <api/redirect.html#lepl.matchers.core.Any>`_, the most basic
matcher, places the matched character in a list::

  >>> Any().parse('a')
  ['a']

Similarly, repetition returns a list of results::

  >>> Any()[:].parse('ab')
  ['a', 'b']

as does `And() <api/redirect.html#lepl.matchers.combine.And>`_::

  >>> (Any() & Any()).parse('ab')
  ['a', 'b']

Even when the strings are joined, they are still in a list::

  >>> Any()[:, ...].parse('ab')
  ['ab']
  >>> (Any() + Any()).parse('ab')
  ['ab']

You may not want this -- you may want a parser that returns a single object
rather than a list.  The best way to return a single value is to wrap the
*final* parser in an extra function that returns the first value from the
list::

  >>> def my_letter_parser(text):
  ...   return Any().parse(text)[0]
  ...
  >>> my_letter_parser('a')
  'a'

What does all this have to do with > and >>?  It's important because *you want
the result of applying a function to return a list*.

Given that, there are two obvious ways to apply functions to results.

The first way is to take a a list of results (which might contain just one
value -- that's completely normal and OK) and **apply the function to each
result in the list**.  This is what ``>>`` does::

  >>> def add_x(text):
  ...   return text + 'x'
  ...
  >>> ( Any() >> add_x ).parse('a')
  ['ax']
  >>> ( (Any() & Any()) >> add_x ).parse('ab')
  ['ax', 'bx']

This (``>>``) is useful when:

* You want to modify each result, one at a time, all in the same way.

* You know that your matcher gives a *single* result, and you want to change
  it.  For example,

  * Translating escaped characters.

  * Converting a number in a string to a float value.

Usually, if you are calling a *function* (``float()``, ``lambda`` etc) you
want to use ``>>``.

The second way that you can process a list of results is by **passing the
entire list to a function**.  Because we still want a list afterwards, Lepl
*adds an extra list around the result*.  This is what ``>`` does::

  >>> def first(my_list):
  ...   return my_list[0]
  ...
  >>> ( Any() > first ).parse('a')
  ['a']
  >>> ( (Any() & Any()) > first ).parse('ab')
  ['a']

This is also useful for structuring results::

  >>> ( (Any() & Any()) > tuple ).parse('ab')
  [('a', 'b')]
  >>> ( (Any() & Any()) > list ).parse('ab')
  [['a', 'b']]
  >>> (( (Any() & Any()) > list ) & Any()).parse('abc')
  [['a', 'b'], 'c']

So ``>`` is useful when:

* You want to select some results.

* You want to build data structures around the results.

Usually, if you are calling a *constructor* (`Node() <api/redirect.html#lepl.support.node.Node>`_, ``tuple()`` etc.) you want to
use ``>``.

.. _faq_logging:

Why am I seeing "No handlers could be found..." messages?
---------------------------------------------------------

*Why do I see this warning printed to stderr?*

::

  No handlers could be found for logger "lepl.parser.trampoline"

This is because Lepl is sending messages to the Python logging system (usually
debug information), but you don't have logging configured.

You can suppress the warning by adding the following somewhere in your code::

  from logging import basicConfig, ERROR
  basicConfig(level=ERROR)

but only do this if you are not using the logging package!

.. _faq_slowregexp:

Why does my matcher take so long to compile?
--------------------------------------------

*Why is the matcher taking several seconds just to compile?*

You are probably using `Float() <api/redirect.html#lepl.support.warn.Float>`_ or `Real() <api/redirect.html#lepl.matchers.derived.Real>`_ which are being compiled
internally to regular expressions.  The current regexp implementation is very
ineffecient when compiling such values.

In the future Lepl will move to a new regular expression engine.  For now, if
you don't need backtracking within the number and you are using a simple
parser without tokens (ie. no lexer), you can use these replacements (which
delegate to the system ``re`` library)::

  Real = lambda: Regexp(r'[\+\-]?(?:[0-9]*\.[0-9]+|[0-9]+\.|[0-9]+)(?:[eE][\+\-]?[0-9]+)?')
  Float = lambda: Regexp(r'[\+\-]?(?:[0-9]*\.[0-9]+(?:[eE][\+\-]?[0-9]+)?|[0-9]+\.(?:[eE][\+\-]?[0-9]+)?|[0-9]+[eE][\+\-]?[0-9]+)')

However, those will not improve the speed of the lexer (which will convert
them back to the the internal DFA implementation).

Another alternative is to use `.config.no_compile_regexp() <api/redirect.html#lepl.core.config.ConfigBuilder.no_compile_regexp>`_ which will avoid
the compilation in some circumstances.  Again, this won't help when the lexer
is used.

Finally, remember that you can avoid recompiling your parser by making your
matcher just once and then re-using it.  It may be worth, for example,
creating a matcher in a global variable (or during set-up for the entire
suite) to re-use in a series of unit tests.

