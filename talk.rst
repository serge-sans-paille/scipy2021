:title: Building SciPy Kernels with Pythran
:data-transition-duration: 150
:skip-help: true
:slide-numbers: true
:css: font.css


Intro
=====

Scipy & Pythran

Ralf Gommers & Serge Guelton

----

A packager's choice
===================

>>> performant library
<<< easiy to deploy

--- native code
+++ compatible with PyPI

----

How SciPy is Built
==================

SG: @ralf instert cool image here

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

- take pure Python code as input
- understands Numpy high-level constructs
- deliver performance
- without much runtime dependencies

----

Isn't Cython Enough?
====================

Cython is a **great** tool

- incremental conversion / mixed mode
- great for gluing existing native code/library with Python
- good portability, no runtime requirements

- but still has a non-neglectible learning curve
- tends to be closer to C than Python when performance matters

----

The Pythran Approach
====================

Keep input code portable and High-Level

- Pure Python
- Using Numpy idioms

But still

- Efficient explicit looping

----

A Typical Scipy Kernel
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

Anatomy of a Scipy Kernel
=========================

- Uses Numpy: ``import numpy as np``
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

Discussion
==========

- compiling with ``-DUSE_XSIMD -march=native`` for auto-vectorization at the
  expense of portability

- compiling with ``-fopenmp`` and adding openmp annotation at the expense of
  portability (again)

- Linux, windows and OSX portability

----

Conclusion
==========

Let's pretend we're smart
