Building SciPy Kernels with Pythran
===================================

The SciPy library faces an interesting challenge: SciPy developers want an easy
to maintain codebase, while SciPy users want their computations to run fast. The
problem is currently solved by using NumPy for most operations and specializing
some kernels with Cython when NumPy cannot provide a satisfactory answer.

Since late 2020, SciPy developers have been exploring a different trade-off
between maintainability and performance, through the use of the Pythran
compiler. Pythran is an ahead-of-time compiler for scientific kernels written in
Python. It has the unique property of generating high-performance C++ code from
pure Python code as input, while supporting many high-level NumPy constructs.
This makes the developer's task easier, compared to writing Cython (or C++),
without sacrificing performance.

This talk first introduces the main characteristics of the Pythran compiler,
then focuses on early results of using it in SciPy. We will attempt to answer
questions that may be valuable for other projects:

- What are the pros and cons of using an ahead-of-time compiler?
- How does Pythran impact portability of the codebase?
- How does Pythran impact performance of the generated code in terms of
  execution time, build time, and size of generated binaries compared to
  pure Python code and compared to Cython code?
- What would the long term impact of accelerating more code with Pythran
  be?
- How does Pythran integrate with SciPy's build system?
