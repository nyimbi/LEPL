
.. _lepl4:

Lepl 4 - Simpler, Faster, Easier
================================

I've made Lepl simpler to use.  For example, if a parser fails then an
exception shows which part of the input could not be matched --- but if that's
not what you want, it can be disabled by calling
`.config.no_full_first_match() <api/redirect.html#lepl.core.config.ConfigBuilder.no_full_first_match>`_ on
the matcher (configuration got simpler too!).

Another example: it's easier to add new matchers.  Before, you had to subclass
a complex class.  Now, you can add a decorator to a simple function.

Even debugging is simpler.  If you want to understand what the parser is
doing, add ``with TraceVariables()`` and the progress of the match will be
printed to the screen.  That includes the variable names that you used in the
code, so it's easy to understand.  And if there's a bug in a matcher,
tracebacks are now clearer (you no longer get something that has been mangled
by the trampolining).

Generating ASTs is simpler too.  There is extra support for using nested
lists, with the new `List() <api/redirect.html#lepl.support.list.List>`_
class, which means that the more complex `Node() <api/redirect.html#lepl.support.node.Node>`_ classes are often not needed (the
examples in the documentation have been updated to reflect this).

Sometimes, when software is made simpler, it becomes slower.  The reverse is
true for Lepl - the new, simpler, approach supports new optimisations and
makes fixing bugs easier.  In my tests, parsers using the default
configuration are up to 10 times faster.

Finally, licensing: some people were worried about the details of the `LGPL
<http://www.opensource.org/licenses/lgpl-3.0.html>`_; you can now choose to
use Lepl under the `MPL <http://www.opensource.org/licenses/mozilla1.1.php>`_.

Below I'll explain these new features in much more detail, but if you want to
get started with Lepl now, installation instructions are on the (new, simpler)
`front page <index.html>`_.


A Simpler API
-------------

Configuration
~~~~~~~~~~~~~

Matches are now configured via methods on the ``matcher.config`` attribute.
There's no need to hunt round the documentation looking for rewriters --
everything is right at your fingertips::

   >>> matcher = Any()
   >>> dir(matcher.config)
   [...]
   >>> help(matcher.config.left_memoize)
   
   Help on method left_memoize in module lepl.core.config:
   
   left_memoize(self) method of lepl.core.config.ConfigBuilder instance
       Add memoization that can detect and stabilise left-recursion.  This
       makes the parser more robust (so it can handle more grammars) but
       also significantly slower.

Each configuration option has two methods --- one to turn it on, and one to
turn it off.  These changes are relative to the default :ref:`configuration
<configuration>` unless you first call `.config.clear() <api/redirect.html#lepl.core.config.ConfigBuilder.clear>`_ (which removes all
options).

So, for example::

  >>> matcher.config.no_lexer()

removes lexer support from the default configuration, while

::

  >>> matcher.config.clear().lexer()

gives a configuration that *only* has lexer support.


Full Match
~~~~~~~~~~

Often, particularly with a simple parser, you expect all the input to be
matched.  If it isn't, something went wrong, and you'd like to know where.

In Lepl 4 you get all that by default::

  >>> matcher = Any()[5]
  
  >>> try:
  >>>     matcher.parse('1234567')
  >>> except FullMatchException as e:
  >>>     print(str(e))
  The match failed at '67',
  Line 1, character 5 of str: '1234567'.

Of course, you can disable this with `.config.no_full_first_match() <api/redirect.html#lepl.core.config.ConfigBuilder.no_full_first_match>`_.

For more details, see :ref:`configuration`.


Multiple Matches, Parsers
~~~~~~~~~~~~~~~~~~~~~~~~~

The new `matcher.parse_all() <api/redirect.html#lepl.core.config.ParserMixin.parse_all>`_ method (and
related `matcher.parse_string_all() <api/redirect.html#lepl.core.config.ParserMixin.parse_string_all>`_, etc)
returns a generator of all possible matches.  This is similar to the old
`matcher.match() <api/redirect.html#lepl.core.config.ParserMixin.match>`_
method (which still exists), but without the remaining streams (which were
usually not interesting).  If you need multiple matches you'll probably find
that `matcher.parse_all() <api/redirect.html#lepl.core.config.ParserMixin.parse_all>`_ simplifies your
code.

Also, parsers are now cached (this isn't strictly new - it was also present in
later Lepl 3 versions).  This means that you can call `matcher.parse() <api/redirect.html#lepl.core.config.ParserMixin.parse>`_
repeatedly without worrying about wasting time re-compiling the parser.

Cached parsers and configuration interact like you would expect --- changing
the configuration clears the cache so that a new parser is compiled with the
new settings.  If you want to keep a copy of the parser with the old settings
(useful in tests) then try `matcher.get_parse() <api/redirect.html#lepl.core.config.ParserMixin.get_parse>`_.


Upgrading from Lepl 3
~~~~~~~~~~~~~~~~~~~~~

Lepl 4.0 is a major release, which means that it contains changes to
often-used methods.  If you have code from a previous version the changes
described here are important, because they will probably cause your program to
fail.  The good news is that the parts of the API with most changes are those
that are called only once (configuration, creating the parser, etc).  So
updating your code should be relatively easy.  In particular, the way that the
grammar is specified is unchanged.


Easier to Extend
----------------

Roll Your Own Matcher
~~~~~~~~~~~~~~~~~~~~~

Adding a new matcher to Lepl is now as easy as writing a function::

  >>> @function_matcher
  >>> def Capital(support, stream):
  ...    '''A matcher for capital letters.'''
  ...    if stream[0] in ascii_uppercase:
  ...        return ([stream[0]], stream[1:])
  ...
  >>> Capital.config.no_full_first_match()
  >>> Capital.parse('ABC')
  ['A']

If the matcher supports multiple results then it should ``yield`` them::

  >>> @sequence_matcher
  ... def Digit(support, stream):
  ...     '''Provide all possible telephone keypresses.'''
  ...     digits = {'1': '',     '2': 'abc',  '3': 'def',
  ...               '4': 'ghi',  '5': 'jkl',  '6': 'mno',
  ...               '7': 'pqrs', '8': 'tuv',  '9': 'wxyz',
  ...               '0': ''}
  ...     if stream:
  ...         digit, tail = stream[0], stream[1:]
  ...         yield ([digit], tail)
  ...         if digit in digits:
  ...             for letter in digits[digit]:
  ...                 yield ([letter], tail)
  ...
  >>> list(Digit()[3, ...].parse_all('123'))
  [['123'], ['12d'], ['12e'], ['12f'], ['1a3'], ['1ad'], ['1ae'], ['1af'], 
  ['1b3'], ['1bd'], ['1be'], ['1bf'], ['1c3'], ['1cd'], ['1ce'], ['1cf']]

Note how these matchers inherit the full functionality of Lepl!

For more information, including support for matchers that process other
matchers, or that can be configured in the grammar, see :ref:`new_matchers`
and following.


General Transformations
~~~~~~~~~~~~~~~~~~~~~~~

Lepl has always supported functions that transform results, but the underlying
implementation is now significantly more powerful.  For example, a function may
add alternative matches, or abort the matching early.

This functionality is unlikely to be used in grammars, but will make adding
cool new features easier.


Easier Debugging
----------------

The `Trace() <api/redirect.html#lepl.matchers.monitor.Trace>`_ functionality in Lepl has never been easy to understand, for
two reasons.  First, it tracks *every* matcher.  Second, it's unclear which
matcher corresponds to which part of the grammar.

Normally, when we debug a program, things are simpler because we can see the
*variables*.  So I have added that to Lepl.  The implementation has some rough
corners, because it uses parts of Python that were not intended to be used in
this way, but I think you'll agree that the result is worth the effort.

Here's an example.  The variables that will be displayed must be defined
inside ``with TraceVariables()``::

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
  [['spicy', 'meatballs'], ['el bulli restaurant']]



Faster Parsers
--------------

Faster Defaults
~~~~~~~~~~~~~~~

I spent time profiling, experimenting with different configurations, and have
tweaked the default settings so that, on average, parsers are faster.  In
particular, memoisation is used only to detect left--recursive loops (if you
do want full memoisation you can still configure it, of course, with
`.config.auto_memoize(full=True)
<api/redirect.html#lepl.core.config.ConfigBuilder.auto_memoize>`_).


No Trampolining
~~~~~~~~~~~~~~~

Lepl is unique (I believe) in using trampoling and co-routines to implement
the recursive descent.  This has several advantages, but introduces some
overhead.

I have measured the overhead, and it's surprisingly small, but even so it
seems silly to have it when it's not needed.  But the problem has always been:
when is it not needed?  The ability to define matchers via functions,
described above, finally gave an answer to that question.

Matchers that are defined as functions are simpler than a completely general
matcher.  So Lepl exploits this to remove trampolining when they are used.
And, of course, matchers provided by Lepl are implemented this way when
possible.

The end result is that trampoling is removed when the grammar is unlikely to
need it.  If you disagree you add it back through the configuration
(`.config.no_direct_eval() <api/redirect.html#lepl.core.config.ConfigBuilder.no_direct_eval>`_).


Better Memoisation
~~~~~~~~~~~~~~~~~~

Sometimes memoisation is a *big* win.  It's not enabled by default, so you
still need to experiment to find out when to use it.  But until now it had a
stupid bug that made it less likely to work.  That bug is now fixed, so when
you need memoisation, it will be there for you.


Further Reading
---------------

* `Front Page <index.html>`_
* :ref:`manual`
* :ref:`tutorial`
* :ref:`contents`
