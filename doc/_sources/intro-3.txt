
Part 3 - Recursion, Abstract Syntax Trees
=========================================

Recap
-----

In the previous section we defined a parser that could add two numbers, even
if the expression contained spaces.  Our final version used tokens::

  >>> from lepl import *
  >>> value = Token(UnsignedReal())
  >>> symbol = Token('[^0-9a-zA-Z \t\r\n]')
  >>> number = Optional(symbol('-')) + value >> float
  >>> add = number & ~symbol('+') & number > sum
  >>> add.parse('12 + -30')
  [-18.0]

(remember that I will not repeat the import statement in the examples below).

In this part of the tutorial we will extend this slightly, to handle a series
of additions, and then look at how to construct an internal representation of
the data --- an abstract syntax tree (AST).

Back to Basics
--------------

Since our aim is to construct an AST, I am going to stop calling ``sum`` to
process the results.  And to make clear exactly what rules we're matching,
I'll keep the addition symbol.

So here's what we're starting with::

  >>> value = Token(UnsignedReal())
  >>> symbol = Token('[^0-9a-zA-Z \t\r\n]')
  >>> number = Optional(symbol('-')) + value >> float
  >>> add = number & symbol('+') & number
  >>> add.parse('12 + -30')
  [12.0, '+', -30.0]

Adding Subtraction
------------------

This doesn't involve anything new; we just need to add the alternative to our
parser.  We could do this in various ways.  One way would be to change
``symbol('+')`` to ``symbol(Any('+-'))``.  But that will make constructing the
AST harder (it's better to have one line per AST node), so instead I'm going
to go with::

  >>> value = Token(UnsignedReal())
  >>> symbol = Token('[^0-9a-zA-Z \t\r\n]')
  >>> number = Optional(symbol('-')) + value >> float
  >>> add = number & symbol('+') & number
  >>> sub = number & symbol('-') & number
  >>> expr = add | sub
  >>> expr.parse('12 + -30')
  [12.0, '+', -30.0]
  >>> expr.parse('12 -30')
  [12.0, '-', 30.0]

That should be clear enough, I hope.  Remember that ``|`` is another way of
writing `Or() <api/redirect.html#lepl.matchers.combine.Or>`_.

.. index:: recursion

Recursion
---------

One limitation of our parser is that it doesn't handle repeated addition and
subtraction.  There's a neat trick that can fix that: in the definition of
`add <api/redirect.html#lepl.matchers.derived.add>`_ and ``sub`` we can replace the second ``number`` with ``expr``, which
will allow us to match more and more additions / subtractions.  The only
problem with that is that we have no way to stop --- each time we try to match
`add <api/redirect.html#lepl.matchers.derived.add>`_, say, we need to match another ``expr``, which means matching another
`add <api/redirect.html#lepl.matchers.derived.add>`_ or ``sub``...  To allow the cycle to end we must allow ``expr`` to be
a simple ``number`` (which we needed anyway, if we want our expression parser
to be general enough to match a single value).

What I've just described above is easy to implement, but unfortunately won't
work::

  >>> value = Token(UnsignedReal())
  >>> symbol = Token('[^0-9a-zA-Z \t\r\n]')
  >>> number = Optional(symbol('-')) + value >> float
  >>> add = number & symbol('+') & expr
  >>> sub = number & symbol('-') & expr
  >>> expr = add | sub | number

.. warning::

  The parser above is broken.  Do not use it!  Depending on what you have
  already typed in to Python, it may not even compile.

What is wrong with the code above?

The problem is that ``expr`` is used before it is defined.  So either it won't
compile or (worse!) it will use a definition of ``expr`` from an earlier
example that you had typed into Python.  There's no simple fix, because if you
put ``expr`` before the `add <api/redirect.html#lepl.matchers.derived.add>`_, for example, then the `add <api/redirect.html#lepl.matchers.derived.add>`_ in the
definition of ``expr`` won't have been defined...

More generally, the problem is that we have a circular set of references,
because we have a recursive grammar.

But it's unfair to call this a "problem".  Recursive grammars are very useful.
The real problem is that I haven't shown how to handle recursive definitions
in Lepl.

.. index:: Delayed(), recursion

Delayed Matchers
----------------

The solution to our problem is to use the `Delayed() <api/redirect.html#lepl.matchers.core.Delayed>`_ matcher.  This lets us
introduce something, so that we can use it, and then add a definition later.
That might sound odd, but it's really simple to use::

  >>> value = Token(UnsignedReal())
  >>> symbol = Token('[^0-9a-zA-Z \t\r\n]')
  >>> number = Optional(symbol('-')) + value >> float
  >>> expr = Delayed()
  >>> add = number & symbol('+') & expr
  >>> sub = number & symbol('-') & expr
  >>> expr += add | sub | number

Note the use of ``+=`` when we give the final definition.  This works
perfectly::

  >>> expr.parse('1+2-3 +4-5')
  [1.0, '+', 2.0, '-', 3.0, '+', 4.0, '-', 5.0]

.. index:: AST, abstract syntax tree, List()

Building an AST with List
-------------------------

OK, finally we are at the point where it makes sense to build an AST.  The
motivation for the sections above (apart from the sheer joy of learning, of
course) is that we needed something complicated enough for this to be
worthwhile.

The simplest way of building a tree is almost trivial.  We just send the
results for the addition and subtraction to `List() <api/redirect.html#lepl.support.list.List>`_::

  >>> value = Token(UnsignedReal())
  >>> symbol = Token('[^0-9a-zA-Z \t\r\n]')
  >>> number = Optional(symbol('-')) + value >> float
  >>> expr = Delayed()
  >>> add = number & symbol('+') & expr > List
  >>> sub = number & symbol('-') & expr > List
  >>> expr += add | sub | number
  >>> expr.parse('1+2-3 +4-5')
  [List(...)]

OK, not so exciting, but let's look at that first result::

  >>> ast = expr.parse('1+2-3 +4-5')[0]
  >>> print(ast)
  List
   +- 1.0
   +- '+'
   `- List
       +- 2.0
       +- '-'
       `- List
           +- 3.0
           +- '+'
           `- List
               +- 4.0
               +- '-'
               `- 5.0

That's our first AST.  It's a bit of a lop--sided tree, I admit --- we will
make some more attractive trees later --- but if you have worked through this
tutorial from zero, this is a major achievement.  Congratulations!

(I hope it's clear that the result above is a "picture" of a tree built with
nested lists.  The root list has three children: the value ``1.0``; the symbol
``'+'``; a child `List <api/redirect.html#lepl.support.list.List>`_ with a first grandchild of ``2.0`` etc.)

.. index:: lists, nodes, List(), Node()

Lists, S-Expressions, and Nodes
-------------------------------

There's a long tradition of using nested lists to represent trees of data ---
it is fundamental to the Lisp programming language, for example.  Lists used
in this way are often called "S-Expressions".

The `List() <api/redirect.html#lepl.support.list.List>`_ class is a simple
subclass of Python's ``list``.  That makes it easy to understand and use.

Lepl includes tools that simplify working with nested lists, including
`sexpr_fold() <api/redirect.html#lepl.support.list.sexpr_fold>`_,
`sexpr_flatten() <api/redirect.html#lepl.support.list.sexpr_flatten>`_ and
`sexpr_to_tree() <api/redirect.html#lepl.support.list.sexpr_to_tree>`_.  These
all work with any kind of nested iterable (except strings, which are treated
as single values rather than sequences of characters).  That means that you
can also use tuples, plain old Python lists, and even sub--classes of `List() <api/redirect.html#lepl.support.list.List>`_ to structure your AST (the next
section will use sub--classes to identify different kinds of values).

For more complex cases, Lepl also includes a `Node() <api/redirect.html#lepl.support.node.Node>`_ class (this used to appear in the
examples; `List() <api/redirect.html#lepl.support.list.List>`_ is new in Lepl
4).  `Node() <api/redirect.html#lepl.support.node.Node>`_ tries to combine
Python's ``list`` and ``dict`` classes into one type, which sounds incredibly
useful, but ends up being a little too complex.

If you use `Node() <api/redirect.html#lepl.support.node.Node>`_, your code
will continue to work, but I would encourage you to consider switching to
`List() <api/redirect.html#lepl.support.list.List>`_.

Summary
-------

What more have we learnt?

* Recursive grammars are supported with `Delayed() <api/redirect.html#lepl.matchers.core.Delayed>`_.

* The results of parsing are stored in trees of data, called Abstract Syntax
  Trees (ASTs).

* The simples way to build an AST is with nested lists; the `List() <api/redirect.html#lepl.support.list.List>`_ class
  subclasses Python's list to add the ability to display the tree in a text
  diagram.

* A `Node() <api/redirect.html#lepl.support.node.Node>`_ combines list and
  dict behaviour.

