:title: Building SciPy Kernels with Pythran
:data-transition-duration: 150
:skip-help: true
:slide-numbers: true
:css: font.css


Intro
=====

SciPy & Pythran

Ralf Gommers & Serge Guelton

----

A packager's choice
===================

>>> performant library
<<< easy to deploy

--- native code
+++ compatible with PyPI

----

How SciPy is Built
==================

.. image:: SciPy_build_dependency_graph_with_Pythran.png
    :height: 600px

----

How SciPy is Written
====================

- ``.py``: for the glue, and non critical parts
- ``.pyx``: for critical parts
- ``.cpp``: for ultra critical parts :-)

----

Lower the maintenance cost
==========================

Looking for a tool which:

- takes pure Python code as input
- understands NumPy high-level constructs
- delivers performance
- without many runtime dependencies

----

Isn't Cython Enough?
====================

Cython is a **great** tool

- incremental conversion / mixed mode
- great for gluing existing native code/library with Python
- good portability, no runtime requirements

- but still has a non-negligible learning curve
- tends to be closer to C than Python when performance matters

----

The Pythran Approach
====================

Keep input code portable and high-level

- Pure Python
- Using NumPy idioms

But still

- Efficient explicit looping

----

A Typical SciPy Kernel
======================

    #pythran export _max_len_seq_inner(intp[], int8[], int, int, int8[])

.. code:: python

    import numpy as np
    def _max_len_seq_inner(taps, state, nbits, length, seq):
        n_taps = taps.shape[0]
        idx = 0
        for i in range(length):
            feedback = state[idx]
            seq[i] = feedback
            for ti in range(n_taps):
                feedback ^= state[(taps[ti] + idx) % nbits]
            state[idx] = feedback
            idx = (idx + 1) % nbits
        return np.roll(state, -idx, axis=0)

----

Anatomy of a SciPy Kernel
=========================

- Uses NumPy: ``import numpy as np``
- Explicit looping: ``for i in range(length):``
- Explicit indexing: ``state[(taps[ti] + idx) % nbits]``
- High-Level idiom: ``np.roll(state, -idx, axis=0)``

⇒ Interleaving low-level and high-level abstractions

----

Pythran Conversion
==================


.. code:: shell

    $ sed -i -e '1 i #pythran export _max_len_seq_inner(intp[], int8[], int, int, int8[])' kernel.py
    $ pythran kernel.py

----

Notebook Playground
===================

.. code:: python

    %%pythran
    #pythran export _max_len_seq_inner(intp[], int8[], int, int, int8[])
    def _max_len_seq_inner(taps, state, nbits, length, seq):
        n_taps = taps.shape[0]
        # ...
        return np.roll(state, -idx, axis=0)

----

Distutils Playground
====================

.. code:: python

    from distutils.core import setup

    # These two lines are required to be able to use pythran in the setup.py
    import setuptools
    setuptools.dist.Distribution(dict(setup_requires='pythran'))

    from pythran.dist import PythranExtension, PythranBuildExt
    setup(...,
          ext_modules=[PythranExtension("mymodule", ["mymodule.py"])],
          cmdclass={"build_ext": PythranBuildExt})

----

Benefits for SciPy
===================

SG: @Ralf?


----

Limitation wrt. SciPy
=====================

SG: @Ralf?

----

Integration Status
==================

SG: @Ralf?

----

Migration Feedback
==================

- Several Pythran releases have been requested to fix distutils integration
  - native code + multiple platform = <3
- Portability issues on AIX
- Windows's extra requirements: clang-cl

----

GSoC Student: Xingyu-Liu
------------------------

Crawling in SciPy's code base, looking for kernel to benchmark and convert

Read the log of her journey:

https://blogs.python-gsoc.org/en/xingyu-lius-blog/



----

Discussion
==========

- compiling with ``-DUSE_XSIMD -march=native`` for auto-vectorization at the
  expense of portability

- compiling with ``-fopenmp`` and adding openmp annotation at the expense of
  portability (again)

- Linux, Windows and macOS portability

----

Conclusion
==========

Let's pretend we're smart
