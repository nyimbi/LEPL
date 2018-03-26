## LEPL - A Parser Library for Python
Ported from Google Code by Nyimbi

This is the Development Site for Lepl, a Parser Library for Python. Thank-you for your interest in this project.

Experienced users will want to start with the manual which has a guided introduction and examples. It also includes download and installation details.

A longer introduction to Lepl is available in the tutorial.

# Features

The latest version (5+) includes: * Parsers are Python code, defined in Python itself. No separate grammar is necessary. * Friendly syntax using Python's operators. * Integrated, optional lexer simplifies handling whitespace. * Built-in AST support (a generic Node class). Improved support for the visitor pattern and tree re--writing. * Generic, pure-Python approach supports parsing a wide variety of data including bytes (Python 3+ only). * Well documented and easy to extend. * Unlimited recursion depth. The underlying algorithm is recursive descent, which can exhaust the stack for complex grammars and large data sets. LEPL avoids this problem by using Python generators as coroutines (aka "trampolining"). * Support for ambiguous grammars (complete backtracking). A parser can return more than one result (aka parse forests). * Packrat parsing. Parsers can be made much more efficient with automatic memoisation. * Parser rewriting. The parser can itself be manipulated by Python code. This gives limited opportunities for future expansion and optimisation. * Left recursive grammars. Memoisation can detect and control left--recursive grammars. Together with LEPL's support for ambiguity this means that "any" grammar can be supported. * Pluggable trace and resource management, including deepest match diagnostics and the ability to limit backtracking. * Various ways to improve parsing speed, including compilation to regular expressions. * Support for the offside rule (significant indentation levels). * Parsing of binary data (Python 3 only).

Maintained/Developed by Andrew Cooke andrew@acooke.com

