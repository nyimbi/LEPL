

Results
=======

This chapter describes various ways in which results can be structured while
parsing data with Lepl.


Flat List
---------

This is the default behaviour.  Simple declarations produce a single list of
tokens (ignoring :ref:`backtracking`).  For example::

  >>> from lepl import *
  
  >>> expr   = Delayed()
  >>> number = Digit()[1:,...]
  
  >>> with Separator(r'\s*'):
  >>>     term    = number | '(' & expr & ')'
  >>>     muldiv  = Any('*/')
  >>>     factor  = term & (muldiv & term)[:]
  >>>     addsub  = Any('+-')
  >>>     expr   += factor & (addsub & factor)[:]
  >>>     line    = expr & Eos()
  >>> line.parse('1 + 2 * (3 + 4 - 5)')
  ['1', ' ', '', '+', ' ', '2', ' ', '*', ' ', '(', '', '3', ' ', '', '+', ' ', '4', ' ', '', '-', ' ', '5', '', '', ')', '']

.. index:: Drop()
.. note::

  The empty strings are a result of the separator and could be removed by
  using ``with Separator(Drop(Regexp(r'\s*'))):``


.. _nestedlists:

Nested Lists
------------

.. index:: s-expressions, list, nested lists

Simple Lists
~~~~~~~~~~~~

Nested lists (S-Expressions) are a traditional way of structuring parse
results.  With Lepl they are easy to construct with ``> list``::

  >>> expr   = Delayed()
  >>> number = Digit()[1:,...]

  >>> with DroppedSpace():
  >>>     term    = number | (Drop('(') & expr & Drop(')') > list)
  >>>     muldiv  = Any('*/')
  >>>     factor  = (term & (muldiv & term)[:])
  >>>     addsub  = Any('+-')
  >>>     expr   += factor & (addsub & factor)[:]
  >>>     line    = expr & Eos()
  >>> line.parse('1 + 2 * (3 + 4 - 5)')
  ['1', '+', '2', '*', ['3', '+', '4', '-', '5']]

.. note::

  ``list`` is just the usual Python constructor.

.. index:: List(), AST, abstract syntax tree, syntax tree, tree

Extended List Support
~~~~~~~~~~~~~~~~~~~~~

Lepl includes additional classes to smplify working with nested lists.

First, the `List() <api/redirect.html#lepl.support.list.List>`_ class (which
subclasses Python's ``list``) and sub-classes can be used to identify nodes
and display a tree::

  >>> class Factor(List): pass
  >>> class Expression(List): pass
            
  >>> expr   = Delayed()
  >>> number = Digit()[1:,...]
        
  >>> with DroppedSpace():
  >>>     term    = number | '(' & expr & ')'
  >>>     muldiv  = Any('*/')
  >>>     factor  = term & (muldiv & term)[:]         > Factor
  >>>     addsub  = Any('+-')
  >>>     expr   += factor & (addsub & factor)[:]     > Expression
  >>>     line    = expr & Eos()

  >>> ast = line.parse_string('1 + 2 * (3 + 4 - 5)')[0]
  >>> print(ast)
  Expression
   +- Factor
   |   `- '1'
   +- '+'
   `- Factor
       +- '2'
       +- '*'
       +- '('
       +- Expression
       |   +- Factor
       |   |   `- '3'
       |   +- '+'
       |   +- Factor
       |   |   `- '4'
       |   +- '-'
       |   `- Factor
       |       `- '5'
       `- ')'

  >>> print(ast[2][0][0])
  2

Second, we can use `sexpr_fold()
<api/redirect.html#lepl.support.list.sexpr_fold>`_ to manipulate this
structure in various ways::

  >>> def per_list(type_, list_):
  >>>     return str(eval(''.join(list_)))
  >>> def calculate(list_):
  >>>     return sexpr_fold(per_list=per_list)(list_)[0]
  >>> calculate(ast)
  5

  >>> sexpr_fold(per_list=lambda t_, l: list(l))(ast)
  [['1'], '+', ['2', '*', '(', [['3'], '+', ['4'], '-', ['5']], ')']]


.. index:: Node()
.. _trees:

Nodes
------

Lepl includes another class, `Node()
<api/redirect.html#lepl.support.node.Node>`_, that can also be used to
construct trees::

  >>> class Term(Node): pass
  >>> class Factor(Node): pass
  >>> class Expression(Node): pass

  >>> expr   = Delayed()
  >>> number = Digit()[1:,...]                        > 'number'

  >>> with Separator(r'\s*'):
  >>>     term    = number | '(' & expr & ')'         > Term
  >>>     muldiv  = Any('*/')                         > 'operator'
  >>>     factor  = term & (muldiv & term)[:]         > Factor
  >>>     addsub  = Any('+-')                         > 'operator'
  >>>     expr   += factor & (addsub & factor)[:]     > Expression
  >>>     line    = expr & Eos()

  >>> ast = line.parse('1 + 2 * (3 + 4 - 5)')[0]
  >>> print(ast)
  Expression
   +- Factor
   |   +- Term
   |   |   `- number '1'
   |   `- ' '
   +- ''
   +- operator '+'
   +- ' '
   `- Factor
       +- Term
       |   `- number '2'
       +- ' '
       +- operator '*'
       +- ' '
       `- Term
           +- '('
           +- ''
           +- Expression
           |   +- Factor
           |   |   +- Term
           |   |   |   `- number '3'
           |   |   `- ' '
           |   +- ''
           |   +- operator '+'
           |   +- ' '
           |   +- Factor
           |   |   +- Term
           |   |   |   `- number '4'
           |   |   `- ' '
           |   +- ''
           |   +- operator '-'
           |   +- ' '
           |   `- Factor
           |       +- Term
           |       |   `- number '5'
           |       `- ''
           +- ''
           `- ')

The `Node() <api/redirect.html#lepl.support.node.Node>`_ class functions like
an array of the original results (including spaces)::

  >>> [child for child in ast]
  [Factor(...), '', '+', ' ', Factor(...)]

  >>> [ast[i] for i in range(len(ast))]
  [Factor(...), '', '+', ' ', Factor(...)]

.. warning::

   This has changed slightly; before Lepl 4 iterating over values set by named
   pairs would return the pair (``('operator', '+')`` instead of ``+``).

Nodes also provide attribute access to child nodes and named pairs.  These are
returned as lists, since sub--node types and names need not be unique::

  >>> [(name, getattr(ast, name)) for name in dir(ast)]
  [('Factor', [Factor(...), Factor(...)]), ('operator', ['+'])]

  >>> ast.Factor[1].Term[0].number[0]
  '2'

As you can see, `Node() <api/redirect.html#lepl.support.node.Node>`_ combines
aspects of ``list`` and ``dict``.  This makes it very powerful, but also
complicates the API considerably.  For example, no single method describes the
contents completely, so iteration over Nodes is via the constructor arguments
exposed by `ConstructorGraphNode() <api/redirect.html#lepl.support.graph.ConstructorGraphNode>`_.
