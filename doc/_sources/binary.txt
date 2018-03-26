
.. _binary:

Binary Data
===========

.. warning::

   This package does not work with Python 2.6 or 2.7 (and never will do so -
   the ``bytes()`` type is not usable until version 3.0).

   This documentation is rather short and incomplete; it will be extended as
   the package is developed further.

The ``bin`` package supports both parsing and (re-)constructing binary data.
It uses the core Lepl engine which, because of Python's relaxed typing and
calling conventions, is not restricted to parsing text.  But it also adds
additional support to help address problems that are unique to handling binary
data.

The first few sections below discuss the representation of data.  This (rather
tedious) detail is necessary to precisely define how binary values are handled
by Lepl.

Abstract Model
--------------

Lepl models binary data as an array of bits, visualised from left to right.
The zeroth index is the least significant bit and is visualised as being
"leftmost".  For a value with `n` bits, the `n-1` index is the most
significant bit, visualised as being "rightmost".

In the API and documentation, `byte` refers to 8 bits (an octet).

Literal Representation
----------------------

Binary values can be expressed in a variety of ways.  Conversion from a
literal form to Lepl's internal model may require passing through an
intermediate byte-based form.  If so, Lepl preserves bit order within a byte
(ie the least significant bit of a Python ``byte`` corresponds to the least
significant bit in Lepl's abstract model).  However, the `order of bytes` will
vary, depending on whether the encoding is little- or big-endian.

The following table summarises the representations support by Lepl:

+-----------+--------+----------------------------------------+-----------+
|Base       |Notation|Description                             | n(bits)   |
+===========+========+========================================+===========+
|2 binary   |0b010   |Native Python int with prefix -         |32         |
|           |        |converted to an integer according to    |           |
|           |        |Python's own rules.                     |           |
|           |        |Resulting bytes stored little-endian.   |           |
|           +--------+----------------------------------------+-----------+
|           |'0b010' |String, converted by Lepl using Python's|n(digits)  |
|           |        |int() function, but with length from    |           |
|           |        |number of characters following '0b'.    |           |
|           |        |Resulting bytes stored little-endian.   |           |
|           +--------+----------------------------------------+-----------+
|           |'010b0' |String, taken as **literal              |n(digits)  |
|           |        |representation of a series of bits**    |           |
|           |        |(lsb to left).                          |           | 
+-----------+--------+----------------------------------------+-----------+
|8 octal    |0o073   |Native Python int with prefix -         |32         |
|           |        |converted to an integer according to    |           |
|           |        |Python's own rules.                     |           |
|           |        |Resulting bytes stored little-endian.   |           |
|           +--------+----------------------------------------+-----------+
|           |'0o073' |String, converted by Lepl using Python's|3*n(digits)|
|           |        |int() function, but with length from    |           |
|           |        |number of characters following '0o'.    |           |
|           |        |Resulting bytes stored little-endian.   |           |
|           +--------+----------------------------------------+-----------+
|           |'073o0' |String, converted by Lepl using Python's|3*n(digits)|
|           |        |int() function, but with length from    |           |
|           |        |number of characters preceding 'o0'.    |           |
|           |        |Resulting bytes stored **big-endian**.  |           |
|           |        |Note: Number of bits must be a multiple |           |
|           |        |of 8 (ie whole number of bytes).        |           |
+-----------+--------+----------------------------------------+-----------+
|10 decimal |1980    |Native Python int -                     |32         |
|           |        |converted to an integer according to    |           |
|           |        |Python's own rules.                     |           |
|           |        |Resulting bytes stored little-endian.   |           |
|           +--------+----------------------------------------+-----------+
|           |'0d1980'|String, converted by Lepl using Python's|32         |
|           |'1980'  |int() function.                         |           |
|           |        |Resulting bytes stored little-endian.   |           |
|           +--------+----------------------------------------+-----------+
|           |'1980d0'|String, converted by Lepl using Python's|32         |
|           |        |int() function.                         |           |
|           |        |Resulting bytes stored **big-endian**.  |           |
+-----------+--------+----------------------------------------+-----------+
|16         |0xfe01  |Native Python int with prefix -         |32         |
|hexadecimal|        |converted to an integer according to    |           |
|           |        |Python's own rules.                     |           |
|           |        |Resulting bytes stored little-endian.   |           |
|           +--------+----------------------------------------+-----------+
|           |'0xfe01'|String, converted by Lepl using Python's|4*n(digits)|
|           |        |int() function, but with length from    |           |
|           |        |number of characters following '0o'.    |           |
|           |        |Resulting bytes stored little-endian.   |           |
|           +--------+----------------------------------------+-----------+
|           |'fe01x0'|String, converted by Lepl using Python's|4*n(digits)|
|           |        |int() function, but with length from    |           |
|           |        |number of characters preceding 'o0'.    |           |
|           |        |Resulting bytes stored **big-endian**.  |           |
|           |        |Note: Number of bits must be a multiple |           |
|           |        |of 8 (ie whole number of bytes).        |           |
+-----------+--------+----------------------------------------+-----------+

.. note::

   Big-endian encoded values must be a whole number of bytes.  This is because
   the byte order must be reversed during translation to Lepl's internal
   model, and this process is undefined if the data do not completely define a
   series of bytes.

   In contrast, little-endian encoded data match Lepl's abstract model and
   so the value can be truncated directly at the given number of bits.

Examples
--------

The table below shows, for various representations, the corresponding binary
data in Lepl's internal model.

============= =======================
Literal Value Internal Model (length)
============= =======================
0b101100      00110100 00000000 00000000 00000000 (32)
------------- -----------------------
'0b101100'    001101 (6)
------------- -----------------------
'001101b0'    001101 (6)
------------- -----------------------
0o073         11011100 00000000 00000000 00000000 (32)
------------- -----------------------
'0o073'       11011100 0 (9)
------------- -----------------------
'073o0'       Error
------------- -----------------------
'0o01234567'  11101110 10011100 10100000 (24)
------------- -----------------------
'01234567o0'  10100000 10011100 11101110 (24)
------------- -----------------------
1980          00111101 11100000 00000000 00000000 (32)
------------- -----------------------
'0d1980'      00111101 11100000 00000000 00000000 (32)
------------- -----------------------
'1980'        00111101 11100000 00000000 00000000 (32)
------------- -----------------------
'1980d0'      00000000 00000000 11100000 00111101 (32)
------------- -----------------------
0xfe01        10000000 01111111 00000000 00000000 (32)
------------- -----------------------
'0xfe01'      10000000 01111111 (16)
------------- -----------------------
'fe01x0'      01111111 10000000 (16)
============= =======================

Lengths
-------

In various parts of the API it is necessary to specify the length of binary
data (eg. when matching).  If an integer value is given, it is taken as the
number of bits.  A float value, however, is taken as bytes, with the `decimal
portion as the number of bits`.  So, for example, ``3.4`` is equivalent to 28
bits (3x8+4).

BitString
---------

The `BitString() <api/redirect.html#lepl.bin.bits.BitString>`_ class is an
implementation of Lepl's abstract model described above.  It has similar
semantics to Python's strings, in that a single entry (a bit - the equivalent
of a character in a string) is still a `BitString() <api/redirect.html#lepl.bin.bits.BitString>`_::

  >>> from lepl.bin.bits import BitString
  >>> b = BitString.from_int('00110101b0')
  >>> str(b)
  '00110101b0/8'
  >>> type(b)
  <class 'lepl.bin.bits.BitString'>
  >>> str(b[0])
  '0b0/1'
  >>> type(b[0])
  <class 'lepl.bin.bits.BitString'>
  >>> str(b[1:4])
  '011b0/3'

  >>> s = 'abc'
  >>> type(s)
  <class 'str'>
  >>> s[0]
  'a'
  >>> type(s[0])
  <class 'str'>

The static method ``BitString.from_int()`` understands all the representations
described earlier.

Matching
--------

A `BitString() <api/redirect.html#lepl.bin.bits.BitString>`_ can be passed to a Lepl matcher in the same way as a Python
string.  The matchers will "automatically" match and construct the binary data.

The `lepl.bin.matchers <api/redirect.html#lepl.bin.matchers>`_ package defines some additional matchers to help
match literal binary values.  These include `Const() <api/redirect.html#lepl.bin.matchers.Const>`_ for matching a
constant value, and ``BEnd(length)`` for matching a big-endian value of a
certain length (``LEnd(length)`` is similar for little-endian values).

The example below is rather detailed, but it shows `Const() <api/redirect.html#lepl.bin.matchers.Const>`_ and `BEnd() <api/redirect.html#lepl.bin.matchers.BEnd>`_
in use::

  from lepl.bin.bits import BitString
  from lepl.bin.encode import dispatch_table, simple_serialiser
  from lepl.bin.literal import parse
  from lepl.bin.matchers import BEnd, Const
  from lepl.node import Node

  # first, define some test data - we'll use a simple definition
  # language, but you could also construct this directly in Python
  # (Frame, Header etc are auto-generated subclasses of Node). 
  mac = parse('''
  Frame(
    Header(
      preamble  = 0b10101010*7,
      start     = 0b10101011,
      destn     = 010203040506x0,
      source    = 0708090a0b0cx0,
      ethertype = 0800x0
    ),
    Data(1/8,2/8,3/8,4/8),
    CRC(234d0/4.)
  )
  ''')

  # next, define a parser for the header structure
  # this is mainly literal values, but we make the two addresses
  # big-endian integers, which will be read from the data

  # this looks very like "normal" lepl because it is - there's 
  # nothing in lepl that forces the data being parsed to be text. 

  preamble  = ~Const('0b10101010')[7]
  start     = ~Const('0b10101011')
  destn     = BEnd(6.0)                > 'destn'
  source    = BEnd(6.0)                > 'source'
  ethertype = ~Const('0800x0') 
  header    = preamble & start & destn & source & ethertype > Node

  # so, what do the test data look like?
  print(mac)
  # Frame
  #  +- Header
  #  |   +- preamble BitString(b'\xaa\xaa\xaa\xaa\xaa\xaa\xaa', 56, 0)
  #  |   +- start BitString(b'\xab', 8, 0)
  #  |   +- destn BitString(b'\x01\x02\x03\x04\x05\x06', 48, 0)
  #  |   +- source BitString(b'\x07\x08\t\n\x0b\x0c', 48, 0)
  #  |   `- ethertype BitString(b'\x08\x00', 16, 0)
  #  +- Data
  #  |   +- BitString(b'\x01', 8, 0)
  #  |   +- BitString(b'\x02', 8, 0)
  #  |   +- BitString(b'\x03', 8, 0)
  #  |   `- BitString(b'\x04', 8, 0)
  #  `- CRC
  #      `- BitString(b'\x00\x00\x00\xea', 32, 0)    

  # we can serialise that to a BitString        
  b = simple_serialiser(mac, dispatch_table())
  assert str(b) == 'aaaaaaaaaaaaaaab123456789abc801234000eax0/240'

  # and then we can parse it
  p = header.parse(b)[0]
  print(p)
  # Node
  #  +- destn Int(1108152157446,48)
  #  `- source Int(7731092785932,48)

  # the destination address
  assert hex(p.destn[0]) == '0x10203040506'

  # the source address
  assert hex(p.source[0]) == '0x708090a0b0c'

Binary Literals
---------------

The first part of the example above shows how a binary data structure
(``mac``) can be generated from a string representation.

Serialisation
-------------

The package also contains support for serialising data (``simple_serialiser``
in the example above).

Sized Integers
--------------

The results of the parsing are sized integers (`Int() <api/redirect.html#lepl.bin.bits.Int>`_).  These include both
an integer value and a bit count.  They are subclasses of Python's ``int`