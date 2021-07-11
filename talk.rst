:title: Building SciPy Kernels with Pythran
:data-transition-duration: 150
:skip-help: true
:slide-numbers: true
:css: font.css


Intro
=====

Building SciPy kernels with Pythran

- Ralf Gommers (SciPy maintainer, Quansight Labs)
- Serge Guelton (Pythran author)

.. RG: explain here in 10 seconds what Pythran is; no need to explain SciPy

----

The goal for SciPy
==================

SciPy contains algorithmic code. It needs to be fast. Approaches:

- Python: for the glue, and non critical parts
- Cython: for critical parts
- Fortran 77: for very old critical parts
- C, C++: for ultra critical parts :-)

*Our goal: make it easier to write fast SciPy kernels!*

----

The Pythran Approach
====================

Keep input code portable and high-level:

- takes pure Python code as input
- understands NumPy high-level constructs
- delivers performance *by transpiling to C++*

But still

- efficient explicit looping in Python
- and without any runtime dependencies!

----

A typical Pythran Kernel for SciPy
==================================

.. code:: python

    #pythran export _max_len_seq_inner(intp[], int8[], int, int, int8[])
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

.. RG: we can merge this with the slide before (in Google Slides)
.. SG: ok, easier to follow for the audience

----

Works in a Jupyter notebook
===========================

.. code:: python

    %%pythran
    #pythran export _max_len_seq_inner(intp[], int8[], int, int, int8[])
    def _max_len_seq_inner(taps, state, nbits, length, seq):
        n_taps = taps.shape[0]
        # ...
        return np.roll(state, -idx, axis=0)

----

Easy build system integration
=============================

.. code:: python

    from distutils.core import setup
    from pythran.dist import PythranExtension, PythranBuildExt

    setup(...,
          ext_modules=[PythranExtension("mymodule", ["mymodule.py"])],
          cmdclass={"build_ext": PythranBuildExt})

Or precompile to C++ to use with any build system:

.. code:: bash

   $ pythran -E mykernel.py -o mykernel.cpp

----

Isn't Cython Enough?
====================

Cython is a **great** tool

- incremental conversion / mixed mode
- great for gluing existing native code / library with Python
- good portability, no runtime requirements

- but still has a non-negligible learning curve
- tends to be closer to C than Python when performance matters

----

What About Numba Then?
======================

Numba is a **great** tool

- Just-in-Time compilation
- GPU support
- Pure Python syntax

- but it has more runtime dependencies
- tends to require lower-level programming


..
  @SG we should mention Numba. How about reusing the table from
  https://fluiddyn.netlify.app/transonic-vision.html#Overall-comparison-between-Cython,-Numba-and-Pythran
  ?
  @RG: I added a section on numba, and I'm fine to reuse that table as a
  concluding slide on these aspects

----

Comparing Cython, Numba and Pythran
===================================

TODO RG: add table

----


SciPy build-time and runtime dependencies
=========================================

.. image:: SciPy_build_dependency_graph_with_Pythran.png
    :height: 600px

..
  RG: I want to talk here about build-time vs. runtime dependencies. It depends
  on where you are in the stack. The lower you go, the more you want to avoid
  runtime dependencies. On the other hand, if you go up in the stack to
  packages that do not yet have build-time dependencies, adding Pythran (or
  Cython) is very costly - that is where Numba makes sense (e.g. ship a single
  pure Python wheel vs. needing to ship ~20).

----

When do I use which tool?
=========================

Our advice:

- for higher-level, pure Python packages: use Numba

Once you have compiled code in your package:

- use Pythran for standalone kernels
- use Cython for binding C/C++ code, or if you need to interact with the
  Python or NumPy C API

----

Current Usage in SciPy
======================

- Largest extension: `RBFInterpolator`
- Several small extensions:

  .. code-block:: shell

    $ git grep -l  '#pythran'
    scipy/optimize/_group_columns.py
    scipy/signal/_max_len_seq_inner.py
    scipy/signal/_spectral.py
    scipy/stats/_hypotests_pythran.py


- More PRs in progress.

----

GSoC student: Xingyu Liu
------------------------

Xingyu is going through SciPy's code base, looking for kernels to benchmark and
accelerate:

- ``stats.binned_statistic_dd``: 2-30x
- ``stats.somersd``: 4-20x
- ``spatial.SphericalVoronoi.sort_vertices_of_regions``: 3x

And more to come - read the log of her journey:

https://blogs.python-gsoc.org/en/xingyu-lius-blog/

----


Benefits for SciPy
==================

Key benefit: **easiest way to write fast kernels**

- Developer experience about as good as with Numba, accessible to almost every
  contributor
- It's fast (typically >= Cython, even without SIMD)
- Produced binaries are much smaller than those from Cython
- Pythran itself is easy to contribute to, and has a responsive maintainer
- Build system integration is easy(-ish)

----


Limitation wrt. SciPy
=====================

Still gaps in functionality - not all of NumPy covered:

- `numpy.random`
- APIs with too much "dynamic" behavior
- There is no "escape hatch" - if something is not supported, it must be
  implemented in Pythran itself first
- No threading - OpenMP is forbidden in SciPy (see https://github.com/scipy/scipy/pull/13576, went with Cython there)
- Extra constraint on Windows: must build with ``clang-cl``


----

A recent hiccup: circular dependencies
======================================

1. SciPy depends on Pythran
2. Pythran uses introspection to optimize some functions
3. Pythran knows about some ``scipy.special`` functions
4. ``(2.) and (3.) => Pythran depends on SciPy``
5. ``(1.) and (4.) => SciPy depends on SciPy``

And more recently

1. SciPy depends on Pythran
2. Pythran depends on Networkx
3. Networkx depends on SciPy
4. ``(1.) and (2.) and (3.) => SciPy depends on SciPy``


----

Integration Status
==================

Currently Pythran is:

- **enabled** by default in the SciPy build
- still an **optional** dependency (to disable: ``export SCIPY_USE_PYTHRAN=0``)

Lessons from the recent SciPy ``1.7.0`` release:

- Portability issues on AIX
- Status with PyPy unclear (PyPy has other issues that need resolving first)
- Other than that, mostly smooth sailing

Note:

- Several Pythran releases were needed to fix distutils integration
  - native code + multiple platform = <3

----

Conclusion
==========

- SciPy contributors like Pythran
- Pythran is indeed an easier way to write fast kernels
- Pythran will likely become a hard build dependency for or after SciPy 1.8.0


Bonus question: can we combine Pythran with CuPy's Python-to-CUDA JIT? It emits
C++ code too, so we could get fast CPU + GPU code like that.

.. SG: that's a bold move ;-)


----

Backup slides
=============

Stuff that doesn't fit in

----

A packager's choice
===================

::

    >>> performant library
    <<< easy to deploy

    --- native code
    +++ compatible with PyPI

----

Pythran Conversion
==================


.. code:: shell

    $ sed -i -e '1 i #pythran export _max_len_seq_inner(intp[], int8[], int, int, int8[])' kernel.py
    $ pythran kernel.py

----

Discussion
==========

- compiling with ``-DUSE_XSIMD -march=native`` for auto-vectorization at the
  expense of portability

- compiling with ``-fopenmp`` and adding openmp annotation at the expense of
  portability (again)

- Linux, Windows and macOS portability

