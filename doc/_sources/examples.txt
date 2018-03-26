
.. index:: examples
.. _examples:

Examples
========

Parsing a Python Argument List
------------------------------

This problem was discussed on `comp.lang.python
<http://groups.google.com/group/comp.lang.python/msg/3d0aedf525030865>`_,
where a pyparsing solution was posted.  The following is the equivalent Lepl
solution::

  >>> comma  = Drop(',') 
  >>> none   = Literal('None')                        >> (lambda x: None)
  >>> bool   = (Literal('True') | Literal('False'))   >> (lambda x: x == 'True')
  >>> ident  = Word(Letter() | '_', 
  >>>               Letter() | '_' | Digit())
  >>> float_ = Float()                                >> float 
  >>> int_   = Integer()                              >> int
  >>> str_   = String() | String("'")
  >>> item   = str_ | int_ | float_ | none | bool | ident

  >>> with Separator(~Regexp(r'\s*')):
  >>>     value  = Delayed()
  >>>     list_  = Drop('[') & value[:, comma] & Drop(']') > list
  >>>     tuple_ = Drop('(') & value[:, comma] & Drop(')') > tuple
  >>>     value += list_ | tuple_ | item  
  >>>     arg    = value                                   >> 'arg'
  >>>     karg   = (ident & Drop('=') & value              > tuple) >> 'karg'
  >>>     expr   = (karg | arg)[:, comma] & Drop(Eos())    > Node
            
  >>> parser = expr.get_parse_string()

  >>> ast = parser('True, type=rect, sizes=[3, 4], coords = ([1,2],[3,4])')
  >>> print(ast[0])
  Node
   +- arg True
   +- karg ('type', 'rect')
   +- karg ('sizes', [3, 4])
   `- karg ('coords', ([1, 2], [3, 4]))
  >>> ast[0].arg
  [True]
  >>> ast[0].karg
  [('type', 'rect'), ('sizes', [3, 4]), ('coords', ([1, 2], [3, 4]))]
  
  >>> ast = parser('None, str="a string"')
  >>> print(ast[0])
  Node
   +- arg None
   `- karg ('str', 'a string')
  >>> ast[0].arg
  [None]
  >>> ast[0].karg
  [('str', 'a string')]


.. index:: optimisation, efficiency, configuration
.. _config_example:

Configuration and Efficiency
----------------------------

This example shows how different choices of :ref:`configuration` can change
the compilation and parsing times.  Lepl 5 introduces `print_timing() <api/redirect.html#lepl.support.timer.print_timing>`_ which
makes this example much simpler than in previous versions::

  def matcher():
      '''A simple parser we'll use as an example.'''

      class Term(List): pass
      class Factor(List): pass
      class Expression(List): pass

      expr   = Delayed()
      number = Float()                                >> float

      with DroppedSpace():
	  term    = number | '(' & expr & ')'         > Term
	  muldiv  = Any('*/')
	  factor  = term & (muldiv & term)[:]         > Factor
	  addsub  = Any('+-')
	  expr   += factor & (addsub & factor)[:]     > Expression
	  line    = expr & Eos()

      return line

  if __name__ == '__main__':
      basicConfig(level=ERROR)
      m = matcher()
      print_timing(lambda: fmt('{0:4.2f} + {1:4.2f} * ({2:4.2f} + {3:4.2f} - {4:4.2f})',
			       random(), random(), random(), random(), random()),
		   {'default': m.clone(),
		    'clear': m.clone().config.clear().matcher,
		    'no memo': m.clone().config.no_memoize().matcher,
		    'low memory': m.clone().config.low_memory().matcher,
		    'nfa': m.clone().config.clear().compile_to_nfa().matcher,
		    'dfa': m.clone().config.clear().compile_to_dfa().matcher,
		    're': m.clone().config.clear().compile_to_re().matcher})

Running that program gives::

  Timing Results (ms)
  -------------------

  Compiling:  best of 3 averages over 10 repetition(s)
  Parse only: best of 3 averages over 10 repetition(s)

	       Matcher           Compiling | Parse only
  -----------------------------------------+------------------
	       default   167.30            |   5.86           
		 clear     7.84  (x   0.0) |   7.80  (x   1.3)
		   dfa    59.27  (x   0.4) |   2.39  (x   0.4)
	    low memory   151.84  (x   0.9) |  22.82  (x   3.9)
		   nfa    59.25  (x   0.4) |   3.55  (x   0.6)
	       no memo   143.76  (x   0.9) |   3.46  (x   0.6)
		    re    58.21  (x   0.3) |   2.32  (x   0.4)

The first column describes the configuration --- you can check the code to see
exactly what was used.  Note that we use a function to generate the input from
random data; that makes sure that memoisation does not artificially improve
the speeds when tests are repeated.

The second two columns are the time (and the ratio of that time relative to
the default) for using a parser that is re--compiled for each parse.  The time
includes the work needed to compile the parser and is appropriate when you're
only using a matcher once.

The final two columns are the time (and the ratio of that time relative to the
default) for re--using a cached parser.  This doesn't include the time needed
to compile the parser and is appropriate for when you're using the same
matcher many times (in which case the compilation time is relatively
unimportant).

Note that you don't need to worry about caching parsers yourself --- a matcher
will automatically cache the parser when it is used.  The test code is complex
because it is trying to *disable* caching in various places.

What can we learn from these results?

#. Compilation isn't cheap.  The "re-compiled" times are, except for "clear",
   much larger than the "cached" times.  So if you are dynamically generating
   matchers and using each one just once, you might want to use
   `.config.clear() <api/redirect.html#lepl.core.config.ConfigBuilder.clear>`_.

#. But compilation isn't hugely expensive either.  If you're using a matcher
   more than about 20 times, it's worth using the default configuration
   (rather than `.config.clear() <api/redirect.html#lepl.core.config.ConfigBuilder.clear>`_) to get better peformance.

#. Disabling memoisation made the cached parser faster, but you should only do
   this once (1) you are sure you don't have a left-recursive grammar (if you
   do, the default configuration, with caching, will warn you) and (2) you've
   tested it for your particular case.

#. Low memory use is slow, but this is a specialised configuration that you
   won't need in normal use.

For anyone interested in absolute speed, the values above are milliseconds
required per iteration on a Dual Core laptop (a Lenovo X60, a couple of years
old), with sufficient memory to avoid paging.

It would be interesting to compare this with different versions.
Unfortunately the table wasn't updated regularly in previous manuals, but when
I re-ran the Lepl 4 code I found that Lepl 5 was typically around 20% faster.

.. index:: tables, columns, tabular data, Columns()
.. _table_example:

Tabular Data
------------

This is a simple example that shows how to parse data in a fixed, tabular
format using the `Columns() <api/redirect.html#lepl.matchers.complex.Columns>`_ matcher::

    def columns_example():
        # http://www.swivel.com/data_sets/spreadsheet/1002196
        table = '''
        US Foreign Aid, top recipients, constant dollars
        Year            Iraq          Israel           Egypt
        2005   6,981,200,000   2,684,100,000   1,541,900,000
        2004   8,333,400,000   2,782,400,000   2,010,600,000
        2003   4,150,000,000   3,878,300,000   1,849,600,000
        2002      41,600,000   2,991,200,000   2,362,800,000
        '''
        spaces = ~Space()[:]
        integer = (spaces & Digit()[1:, ~Optional(','), ...] & spaces) >> int
        cols = Columns((4,  integer),
                   # if we give widths, they follow on from each other
                   (16, integer),
                   # we can also specify column indices
                   ((23, 36), integer),
                   # and then start with widths again
                   (16, integer))
        # by default, Columns consumes a whole line (see skip argument), so
        # for the whole table we only need to (1) drop the text and (2) put
        # each row in a separate list.
        parser = ~SkipTo(Digit(), include=False) & (cols > list)[:]
        parser.parse(table)

    columns_example()

Which prints::

    [[2005, 6981200000, 2684100000, 1541900000],
     [2004, 8333400000, 2782400000, 2010600000],
     [2003, 4150000000, 3878300000, 1849600000],
     [2002, 41600000, 2991200000, 2362800000]]
 

.. index::  Block(), BLine(), offside rule, whitespace sensitive parsing

Simpler Offside Example
-----------------------

Here's a simpler example of how to use offside parsing, as described in
:ref:`offside`.  The idea is that we have a configuration file format with
named sections and subsections; in the subsections are name/value pairs::

  from string import ascii_letters
  from lepl import *

  def config_parser():
      word        = Token(Any(ascii_letters)[1:, ...])
      key_value   = (word & ~Token(':') & word) > tuple
      subsection  = Line(word) & (Block(Line(key_value)[1:] > dict)) > list
      section     = Line(word) & Block(subsection[1:]) > list
      config_file = (section | ~Line(Empty(), indent=False))[:] > list
      config_file.config.lines(block_policy=explicit)
      return config_file.get_parse()

  parser = config_parser()
  parser('''
  one
     a
	foo: bar
	baz: poop
     b
	snozzle: berry

  two
     c
	apple: orange
  ''')[0]

Which prints::

  [['one', ['a', {'foo': 'bar', 'baz': 'poop'}], ['b', {'snozzle': 'berry'}]], ['two', ['c', {'apple': 'orange'}]]]

Note that the name/value pairs are in dictionaries; this is because we passed
a list of tuples to ``dict()``.


.. index:: Line(), Word(), SOL, EOL

Parsing Lines of Words
----------------------

Here are a set of progressively more complex parsers that split each line into
a list of words.

We start with a simple parser that explicitly manages spaces::

  >>> with DroppedSpace():
  >>>     line = (Word()[:] & Drop('\n')) > list
  >>>     lines = line[:]
  >>> lines.parse('abc de f\n pqr\n')
  [['abc', 'de', 'f'], ['pqr']]

Next, we use tokens (and spaces are handled automatically)::

  >>> word = Token(Word())
  >>> newline = ~Token('\n')
  >>> line = (word[:] & newline) > list
  >>> lines = line[:]
  >>> lines.parse('abc de f\n pqr\n')
  [['abc', 'de', 'f'], ['pqr']]

We can also use line-aware parsing with tokens to handle the newline::

  >>> word = Token(Word())
  >>> line = Line(word[:]) > list
  >>> lines = line[:]
  >>> lines.config.lines()
  >>> lines.parse('abc de f\n pqr\n')
  [['abc', 'de', 'f'], ['pqr']]

.. _low_memory:
.. index:: low_memory(), Override()

Low Memory Use
--------------

This next example shows how data larger than the available memory can be
parsed by Lepl.  Since Lepl is written in Python this is an unusual
requirement (if a task is that large Lepl will probably be too slow), but it
may be useful in some cases::

  from sys import getsizeof
  from logging import basicConfig, DEBUG, ERROR
  from itertools import count, takewhile
  try:
      from itertools import imap
  except ImportError:
      imap = map

  from lepl import *
  from lepl.support.lib import fmt
  from lepl.stream.iter import Cons


  if __name__ == '__main__':

      def source(n_max):
	  '''
	  A source of integers from 1 to n_max inclusive.
	  '''
	  return imap(str, takewhile(lambda n: n <= n_max, count(1)))


      @sequence_matcher
      def Digits(support, stream):
	  '''
	  A matcher that returns each digit (as an int) in turn.
	  '''
	  (number, next_stream) = s_line(stream, False)
	  for digit in number:
	      yield ([int(digit)], next_stream)


      def parser():

	  # a reduce function and the associated zero - this will sum the values
	  # returned by Digit() instead of appending them to a list.  this is
	  # to avoid generating a large result that may confuse measurements of
	  # how much memory the parser is using.
	  sum = ([0], lambda a, b: [a[0] + b[0]])

	  with Override(reduce=sum):
	      total = Digits()[:] & Eos()

	  # configure for reduced memory use
	  total.config.low_memory()

	  return total

      # some basic tests to make sure everything works
      l = list(source(9))
      assert l == ['1', '2', '3', '4', '5', '6', '7', '8', '9'], l
      p = parser()
      print(p.tree())

      r = list(p.parse_iterable_all(source(9)))
      # the sum of digits 1-9 is 45
      assert r == [[45]], r

      r = list(p.parse_iterable_all(source(10)))
      # the digits in 1-10 can sum to 45 or 46 depending on whether we use the
      # '1' or the '0' from 10.
      assert r == [[46],[45]], r

      # if we have 10^n numbers then we have about 10^n * n characters which
      # is 2 * 10^n * n bytes for UTF16
      def size(n):
	  gb = 10**n * (n * 2 + getsizeof(Cons(None))) / 1024**3.0
	  return fmt('{0:4.2f}', gb)
      s = size(8)
      assert s == '8.94', s
      s = size(7)
      assert s == '0.88', s

      # we'll test with 10**7 - just under a GB of data, according to the above
      # (on python2.6)

      # guppy only works for python 2 afaict
      # and it's broken for 2.7
      from guppy import hpy
      from gc import get_count, get_threshold, set_threshold, collect
      #basicConfig(level=DEBUG)
      basicConfig(level=ERROR)

      r = p.parse_iterable_all(source(10**7))
      next(r) # force the parser to run once, but keep the parser in memory
      h = hpy()
      print(h.heap())

This generates the following output::

  Partition of a set of 50077 objects. Total size = 6924832 bytes.
   Index  Count   %     Size   % Cumulative  % Kind (class / dict of class)
       0  19758  39  1790456  26   1790456  26 str
       1  11926  24   958744  14   2749200  40 tuple
       2    149   0   444152   6   3193352  46 dict of module
       3   3011   6   361320   5   3554672  51 types.CodeType
       4    604   1   359584   5   3914256  57 dict (no owner)
       5   2918   6   350160   5   4264416  62 function
       6    334   1   303568   4   4567984  66 dict of type
       7    334   1   299256   4   4867240  70 type
       8    150   0   157200   2   5024440  73 dict of lepl.core.config.ConfigBuilder
       9    140   0   149792   2   5174232  75 dict of class
  <178 more rows. Type e.g. '_.more' to view.>

The output is generated by the Guppy library and shows memory use.  The
simplest thing to note is that there are no objects with a count of 10,000,000
even though that many values were parsed.  That means that parsed data are
garbage-collected as they are processed, which is critical for parsing large
data sets.

For a longer discussion of this work see the `notes I made during development
<http://www.acooke.org/cute/Processing1.html>`_ (the syntax improved since
that was written, but the motivation and general details for the test are
still very relevant).
