.. Use Python-style namespaces everywhere, for consistency

.. default-domain:: py

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   A design study for improving the persistence of Stack classes, particularly :class:`lsst.afw.image.Exposure` and its components.
   This note describes the design decisions behind the :class:`lsst.afw.typehandling.GenericMap` class template and explores options for an off-the-shelf persistence framework to replace ``lsst.afw.table.io``.

.. _intro:

The Problem
===========

Since about 2013, a handful of poor/lazy/expedient design decisions in ``afw`` have been slowly compounding.
A full history of the situation can be found in a `community post`_.
To summarize,

- We use the :class:`lsst.afw.image.Exposure` class throughout the codebase to aggregate instances of all of the classes that charactize an image - PSFs, WCSs, aperture corrections, photometric calibrations, etc.

- Because :class:`~lsst.afw.image.Exposure` holds those components via direct C++ composition, they need to live either in ``lsst.afw.image`` or in a package it depends on.

- Because these components are written to and read from disk via the (C++) ``lsst.afw.table.io`` persistence framework, they are doubly-required to be implemented in C++, and they also need to live in either ``lsst.afw.table`` or a package that depends on it.

The net result is that a large fraction of our classes must:

- live in ``afw`` itself, leading to churn and bloat at the bottom of our dependency tree, and hence frequent, slow, rebuilds;

- be implemented in C++, a language much more complex and much less familiar to most of our developers than Python.

Furthermore, the circular dependency between the ``lsst.afw.image`` and ``lsst.afw.table`` subpackages makes the build and particular the Python import order for ``afw`` fragile, leading to hard-to-debug and even harder-to-fix bugs whenever a new :class:`~lsst.afw.image.Exposure`-component class is added.

A related problem - a potentially more serious, but also less clear-cut one - is that ``lsst.afw.table.io`` was never intended to be a long-term or general persistence solution.\ [#general_persistence]_
It lacks any kind of built-in support for versioning, produces opaque and sometimes inefficient files, involves a lot of boilerplate to use, and is built on top of a bespoke C++ library (``lsst.afw.table`` itself) that presents a maintenance challenge and has I/O performance problems.
None of these limitations represents a particularly acute problem (we have managed to work around all of them for a while now), but as a persistence framework more or less specifies the on-disk format, significant (i.e., backwards incompatible) changes will become progressively harder to make as we approach operations.

.. _community post: https://community.lsst.org/t/how-the-exposure-class-and-afw-io-wrecked-the-codebase/3384

.. [#general_persistence] ``lsst.afw.table.io`` was written to enable writing :class:`~lsst.meas.algorithms.CoaddPsf` objects into FITS files on an HSC data release deadline, with the expectation that it would someday be replaced by the Data Access Team.
   That "Data Access" has since been reinterpreted to mean only *science user* data access (i.e., not pipeline data access or data access middleware), and the hence the responsibility for persistence frameworks has essentially fallen through the cracks.


.. _overview:

Proposal Overview
=================

Our proposal to address these problems (which is already partly implemented) starts with the following steps:

- Add a new heterogenous-type mapping interface and implementation in C++, :class:`~lsst.afw.typehandling.GenericMap`.
  This augments rather than replaces our existing :class:`~lsst.daf.base.PropertySet` and :class:`~lsst.daf.base.PropertyList` in large part because we want to avoid their behavioral idiosyncracies (especially with respect to vector-valued entries) in order to provide natural interfaces in both C++ and Python (e.g., the :class:`collections.abc.Mapping` API), but also recognize that attempting to change the behavior of the existing classes at this stage would be extremely disruptive.

- Define an interface, :class:`~lsst.afw.typehandling.Storable`, for objects that can be held by :class:`~lsst.afw.typehandling.GenericMap`.
  This includes basic functionality like stringification, comparisons, and cloning as well as persistence (currently still via ``lsst.afw.table.io``).

- Rewrite :class:`~lsst.afw.image.ExposureInfo` (which holds the non-image components of :class:`~lsst.afw.image.Exposure`) to hold its components via :class:`~lsst.afw.typehandling.GenericMap`, allowing any :class:`~lsst.afw.typehandling.Storable` to be attached to an :class:`~lsst.afw.image.Exposure`.
  Once we complete this stage (the one in progress as of this writing), :class:`~lsst.afw.image.Exposure` components could begin to be be defined in any package downstream of ``afw`` (but would still need to be implemented in C++ and use ``lsst.afw.table.io``).

To go further, and allow Python-implemented :class:`~lsst.afw.image.Exposure` components, we need to replace ``lsst.afw.table.io`` as the way :class:`~lsst.afw.typehandling.Storable`\ s are persisted, preferably by utilizing a third-party persistence library as much as possible.
There are of course many third-party persistence libraries we could consider, and these can differ quite substantially.

.. _genericmap:

The Design of GenericMap
========================

Rationale
---------

We wish to implement :class:`~lsst.afw.image.ExposureInfo` using a heterogeneous map (string keys to arbitrary objects) to decouple it from the details of what information is associated with an exposure.
Doing so will allow :class:`~lsst.afw.image.ExposureInfo` to store objects of classes unknown to ``afw``, and allow :class:`~lsst.afw.image.ExposureInfo` to be extended by pipeline or third-party packages without changing its code.
This concept requires some standardization of keys, but no more so than, for example, FITS header keywords.

At the time of writing, the LSST stack has at least three heterogeneous map types in C++: :class:`lsst.daf.base.PropertySet`, :class:`lsst.daf.base.PropertyList`, and :class:`lsst.pex.policy.Policy` (deprecated).
However, all these types are specialized for particular roles (e.g., :class:`~lsst.daf.base.PropertyList` is designed to represent FITS headers), and mix heterogeneous mapping with other functions.
As a result, these classes are difficult to adapt to new use cases.
In addition, the lack of a common codebase makes these classes difficult to maintain, and the limited type safety makes these classes easy to use incorrectly from both C++ and Python.
:class:`~lsst.afw.typehandling.GenericMap` attempts to not only serve as a suitable back-end for :class:`~lsst.afw.image.ExposureInfo`, but also address the design problems that prevented the use of :class:`~lsst.daf.base.PropertySet` in the first place.

Initially, :class:`lsst.afw.typehandling.GenericMap` will be a component of ``afw`` because it depends, indirectly, on the ``lsst.table.io`` framework.
However, ``lsst.afw.typehandling`` has no other dependencies on ``afw``.
Once object persistence is decoupled from ``lsst.table.io``, the entire subpackage can be moved to a lower level in the Stack and :class:`~lsst.afw.typehandling.GenericMap` can be treated as a fundamental LSST type.

Design Goals
------------

We designed and implemented :class:`~lsst.afw.typehandling.GenericMap` while striving to obey the following principles:

* provide both a C++ and a Python API for :class:`~lsst.afw.typehandling.GenericMap`, although it is unlikely that many Python users will use :class:`~lsst.afw.typehandling.GenericMap` directly, rather through classes such as :class:`~lsst.afw.image.ExposureInfo`.
* support storage of primitive types, as well as LSST classes written in either C++ or Python.
* provide C++ and Python APIs that are as idiomatic as possible in their respective languages, making full use of pybind11's ability to serve as an API adapter.
* do not provide any features beyond a heterogeneous mapping with simple set-get behavior.
  The single responsibility makes :class:`~lsst.afw.typehandling.GenericMap` suitable as a back-end for a variety of applications, including more ornate mapping types.
* provide separate interface (:class:`~lsst.afw.typehandling.GenericMap` and :class:`~lsst.afw.typehandling.MutableGenericMap`) and implementation classes.
  An abstract interface makes it easy to create mappings that require specific properties (as :class:`~lsst.daf.base.PropertySet` and :class:`~lsst.daf.base.PropertyList` do) without breaking code for other users, following the open-closed principle.
  Most code based on :class:`~lsst.afw.typehandling.GenericMap` can be agnostic to the implementation class.
* rely on compile-time type safety as much as possible, with safeguards preventing invalid element retrieval in C++ and spurious type errors in Python.
* allow element retrieval without knowing the exact type as which it was stored, so long as the conversion is valid (e.g., superclass vs. subclass, or different sizes of floating-point number).
  Supporting inexact types is not only user-friendly, it avoids unnecessary coupling between code changes at the points of storage and retrieval (which may be in different packages).


The GenericMap and MutableGenericMap APIs
-----------------------------------------

The design of :class:`~lsst.afw.typehandling.GenericMap` was inspired by a similar class that K. Findeisen wrote in Java, but had to make a number of compromises to accommodate an implementation in C++, Python, and pybind11.

While :class:`~lsst.afw.typehandling.GenericMap` was originally conceived as a class that could store values of *any* type, this proved incompatible with the need for a pybind11 wrapper and the desire for an idiomatic Python API.
In particular, unless ``__getitem__`` takes type information as part of its input, it needs a finite set of types it can test in C++ (the current implementation does so implicitly through a pybind11 wrapper for ``boost::variant``).
The final :class:`~lsst.afw.typehandling.GenericMap` supports values of built-in types as well as :class:`~lsst.afw.typehandling.Storable`, an interface that can be added as a mixin to any LSST class.

Inspired by the Python distinction between :class:`~collections.abc.Mapping` and :class:`~collections.abc.MutableMapping`, we provide separate interfaces for reading (:class:`~lsst.afw.typehandling.GenericMap`) and writing (:class:`~lsst.afw.typehandling.MutableGenericMap`).

C++
^^^

The C++ API is loosely based on the standard library mapping interface (taken as the intersection of the C++14 APIs for :cpp:class:`std::map` and :cpp:class:`std::unordered_map`).
:class:`~lsst.afw.typehandling.GenericMap` omits standard methods that would have paradoxical or surprising behavior when generalized to heterogeneous values.

The first level of type-safety is provided by a :cpp:class:`lsst::afw::typehandling::Key` class template, which combines a nominal key (e.g., a string) with the required type of the value.
:cpp:class:`~lsst::afw::typehandling::Key` objects are lightweight values, and can be passed around or created on the fly as easily as the underlying key type.

The original design for :class:`~lsst.afw.typehandling.GenericMap` called for the map to expose public method templates (e.g., ``void set(Key<K, T> const &, T const &)``) that would provide compile-time type safety.
Since method templates cannot be overridden in C++, these public templates would be implemented in terms of protected methods (e.g., ``void _set(Key<K, Storable> const &, Storable const &)``), which subclasses could use to define how the key-value pairs were stored and managed.

Prototyping revealed that this design had several major issues:

1. The process of delegating calls to a protected non-template method would strip away any information about *which* subclass of :class:`~lsst.afw.typehandling.Storable` was being stored.
   A careless implementation would make it legal to store a :class:`~lsst.afw.geom.SkyWcs` object and then ask for it as a :class:`~lsst.afw.image.Psf`, or vice versa.
2. An implementation of ``__getitem__`` would need to explicitly enumerate and test all possible value types, particularly all subclasses of :class:`~lsst.afw.typehandling.Storable`.
   This would, at the very least, introduce elaborate pybind11 code that would need to be kept in sync with the class definition, but would not have an obvious failure mode if the two files diverged.
3. The protected API would require multiple methods per supported type, including integers, floating point numbers, strings, and :class:`~lsst.afw.typehandling.Storable` (and ``const``/non-``const`` and value/smart pointer variants thereof).
   This would impose an enormous writing and maintenance burden on subclass authors.

We tried several solutions to these problems.
One of the simplest is to drop the interface-oriented architecture, and with it the need to delegate to specialized protected methods.
However, we did not pursue this approach because it did not solve the problem of how to implement ``__getitem__``, and the eventual solution to that problem made an interface-oriented design acceptable again.

One option to implement ``__getitem__`` without hardcoding a list of value types is to design :cpp:class:`~lsst::afw::typehandling::Key` objects to allow retrieval by superclasses of the desired type.
However, we could not find a satisfactory implementation of this approach in C++.
While it is possible to use templates to express questions like "Does a ``Key<Storable>`` match a ``Key<SkyWcs>``?", storing mixed :cpp:class:`~lsst::afw::typehandling::Key` types would require removing the compile-time information that enables such comparisons.
Adding run-time type information to :cpp:class:`~lsst::afw::typehandling::Key` would solve the information loss problem, but C++'s support for RTTI is very limited, and the standard API only allows tests for exact type equality.

The solution we adopted is to no longer store type information -- as a :cpp:class:`~lsst::afw::typehandling::Key` object or any other form -- in a :class:`~lsst.afw.typehandling.GenericMap`.
All queries internally pass the stored value through dynamic casting, which accesses RTTI in an implementation-dependent way that does account for subclasses.
This approach solves all the original design issues, at the cost of making :class:`~lsst.afw.typehandling.GenericMap` no longer strictly type-safe:

1. Queries for an object of the wrong type are blocked at the casting step.
2. ``__getitem__`` can retrieve any :class:`~lsst.afw.typehandling.Storable` by considering only ``Key<Storable>`` and ``Key<shared_ptr<Storable>>``.
3. The protected subclass API is greatly simplified because it no longer needs to accommodate a variety of :cpp:class:`~lsst::afw::typehandling::Key` classes.

In practice, the desire for type-unsafe storage was handled by making all protected methods work in terms of untyped keys and :cpp:class:`boost::variant` values.
Passing by :cpp:class:`~boost::variant` provides a convenient way to express, in code, which values are supported by :class:`~lsst.afw.typehandling.GenericMap` without committing subclasses to any particular storage mechanism.
Unfortunately, because :cpp:class:`boost::variant` cannot distinguish between ``const`` and non-``const`` versions of the same type, :class:`~lsst.afw.typehandling.GenericMap` does not currently support ``const`` values for any type except ``shared_ptr<Storable>``.
We expect to lift this restriction once we can migrate to :cpp:class:`std::variant` in C++17.

After the nature of :class:`~lsst.afw.typehandling.GenericMap`'s type handling, the largest remaining problem was how to handle shared pointers, which are used extensively by :class:`~lsst.afw.image.ExposureInfo`.
In keeping with C++ idioms, and in order to correctly handle polymorphism of :class:`~lsst.afw.typehandling.Storable`, :class:`~lsst.afw.typehandling.GenericMap` returns most values by reference.
However, because the implementation holds pointers as ``shared_ptr<Storable>`` yet must return them as pointers of the correct type, its accessors create a new pointer of the desired type, which cannot be returned by reference.

We chose to return smart pointers alone by value, though the inconsistency with other value types makes it much harder to write type-agnostic code against :class:`~lsst.afw.typehandling.GenericMap`.
The alternatives were to return everything by value, which would make it impossible to support non-smart-pointer storage of :class:`~lsst.afw.typehandling.Storable`, or to always return shared pointers as ``shared_ptr<Storable>``, which would force users to perform unsafe casts in their code.

It is not practical to design an idiomatic C++ API for iterating over a :class:`~lsst.afw.typehandling.GenericMap`.
Instead, we developed a system similar to the visitors used by :cpp:class:`boost::variant` and :cpp:class:`std::variant`, where the user represents the body of the loop by a callable object that accepts values of any supported type.
In practice the callables are usually private classes with templates or overloaded methods, but in rare cases a generic lambda can be used as well.
This approach involves considerable boilerplate, but is more natural to users than an API written in terms of :cpp:class:`~std::variant` or some kind of iterator-like proxy.


.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
