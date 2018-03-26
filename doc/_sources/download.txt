
.. _install:

Download And Installation
=========================

Lepl is available for Python 2.6+ and 3+.  See :ref:`versions` for more
information.


Installation
------------

There are several ways to install Lepl --- they are described below, simplest
first.  If you want a local copy of the manual you should also read the
:ref:`localdocs` section.

Install With Distribute / Setuptools (easy_install)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This installs the latest version.

`Distribute <http://pypi.python.org/pypi/distribute>`_ and
`setuptools <http://pypi.python.org/pypi/setuptools>`_ are very similar,
and either will install Lepl on Python 2.6.  However, I recommend using
`distribute <http://pypi.python.org/pypi/distribute>`_ since it also
works with Python 3 and appears to be better supported.

Once you have installed 
`distribute <http://pypi.python.org/pypi/distribute>`_ or
`setuptools <http://pypi.python.org/pypi/setuptools>`_ you can install
Lepl with the command::

  easy_install lepl

That's it.  There is no need to download anything beforehand;
``easy_install`` will do all the work.

.. _manual_install:

Install With Distutils / setup.py
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:ref:`download` and unpack a source package then run::

  python setup.py install

For example, on Gnu/Linux (in the instructions below, "lepl-xxx" would be
"lepl-\ |release|\ " for the current release)::

  wget http://lepl.googlecode.com/files/lepl-xxx.tar.gz
  tar xvfz lepl-xxx.tar.gz
  cd lepl-xxx
  python setup.py install

Manual Install (experts only)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:ref:`download` and unpack a source package then add to your ``PYTHONPATH``
or ``site-packages``.

.. _download:

Download
--------

You can download the source and documentation packages from the `Support Site
<http://code.google.com/p/lepl/downloads>`_.

Source packages are also available from the `Python Package Index
<http://pypi.python.org/pypi/LEPL/>`_.

.. note::

  The Google Code site doesn't allow free access to all people (in particular,
  Cubans are blocked).  However, the Python Package Index site (second link
  above) is hosted in the Netherlands and, I believe, does work.

  I do not live in the USA, and acooke.org is not hosted in the USA.

.. index:: Python version
.. _versions:

Testing
-------

Once installed you can test Lepl by running the self-test::

  >>> from lepl._test import all
  >>> all()

.. warning::

  Some test failures are expected with certain Python versions.  The test
  described above will check the failures against the version used and,
  if all is as expected, display "Looks OK to me!".

  Also, with easy_install and Python 2.6, a syntax error is printed during
  install (from a Python 3 print statement in lepl._example.separators).  You
  can safely ignore this.

Supported Versions
------------------

The code is targetted at Python 3, but various small modifications are added
to keep most packages (currently everything except binary parsing) working
with Python 2.6.

It is regularly tested on 2.6 and 3.1.

It does not work with Python 2.5.  Incompatibilities include:

  * with contexts
  * setter decorators
  * {} formatting
  * ABC metaclasses
  * changed heapq API
  * except syntax
