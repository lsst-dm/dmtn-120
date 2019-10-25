.. Use Python-style namespaces everywhere, for consistency

.. default-domain:: py

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. _intro:

The Problem
===========

Since about 2013, a handful of poor/expedient design decisions in ``lsst.afw`` have been slowly compounding.
A full history of the situation can be found in a `Community post`_.
To summarize,

- We use the :class:`lsst.afw.image.Exposure` class throughout the code base to aggregate instances of all of the classes that characterize an image - PSFs, WCSs, aperture corrections, photometric calibrations, etc.

- Because :class:`~lsst.afw.image.Exposure` holds those components via direct C++ composition, they need to live either in ``lsst.afw.image`` or in a package it depends on.

- Because these components are written to and read from disk via the (C++) ``lsst.afw.table.io`` persistence framework, they are doubly-required to be implemented in C++, and they also need to live in either ``lsst.afw.table`` or a package that depends on it.

The net result is that a large fraction of our classes must:

- live in ``afw`` itself, leading to churn and bloat at the bottom of our dependency tree, and hence frequent, slow, rebuilds;

- be implemented in C++, a language much more complex and much less familiar to most of our developers than Python.

Furthermore, the circular dependency between the ``lsst.afw.image`` and ``lsst.afw.table`` subpackages makes the build and particular the Python import order for ``afw`` fragile, leading to hard-to-debug and even harder-to-fix bugs whenever a new :class:`~lsst.afw.image.Exposure`-component class is added.

A related problem is that ``lsst.afw.table.io`` was never intended to be a long-term or general persistence solution.\ [#general_persistence]_
It lacks any kind of built-in support for versioning, produces opaque and sometimes inefficient files, involves a lot of boilerplate to use, and is built on top of a bespoke C++ library (``lsst.afw.table`` itself) that presents a maintenance challenge and has I/O performance problems.
None of these limitations represents a particularly acute problem (we have managed to work around all of them for a while now), but as a persistence framework more or less specifies the on-disk format, significant (i.e., backwards incompatible) changes will become progressively harder to make as we approach operations.

.. _Community post: https://community.lsst.org/t/how-the-exposure-class-and-afw-io-wrecked-the-codebase/3384

.. [#general_persistence] ``lsst.afw.table.io`` was written to enable writing :class:`~lsst.meas.algorithms.CoaddPsf` objects into FITS files on an HSC data release deadline, with the expectation that it would later be replaced by the Data Access Team.
   That "Data Access" has since been reinterpreted to mean only *science user* data access (i.e., not pipeline data access or data access middleware), and the hence the responsibility for persistence frameworks has fallen through the cracks.


.. _overview:

Overview
========

We've taken the following actions to mitigate the design problems imposed by :class:`~lsst.afw.image.Exposure` and to prepare for future work:

- Add a new heterogeneous-type mapping interface and implementation in C++, :class:`~lsst.afw.typehandling.GenericMap`.
  This is an alternative to rather than a replacement of our existing :class:`~lsst.daf.base.PropertySet` and :class:`~lsst.daf.base.PropertyList` to avoid their behavioral idiosyncrasies (especially with respect to vector-valued entries) and to provide natural interfaces in both C++ and Python (e.g., the :class:`collections.abc.Mapping` API).
  We also recognize that attempting to change the behavior of the existing classes at this stage would be extremely disruptive.

- Define an interface, :class:`~lsst.afw.typehandling.Storable`, for objects that can be held by :class:`~lsst.afw.typehandling.GenericMap`.
  This includes basic functionality like stringification, comparisons, and cloning as well as persistence (currently still via ``lsst.afw.table.io``).

- Rewrite :class:`lsst.afw.image.ExposureInfo` (which holds the non-image components of :class:`~lsst.afw.image.Exposure`) to hold its components as a map (originally a :class:`~lsst.afw.typehandling.GenericMap`, later replaced with a more specialized type), allowing any :class:`~lsst.afw.typehandling.Storable` to be attached to an :class:`~lsst.afw.image.Exposure`.
  This allows :class:`~lsst.afw.image.Exposure` components to be defined in any package downstream of ``afw`` (but still need to be implemented in C++ and use ``lsst.afw.table.io``).

To go further, and allow Python-implemented :class:`~lsst.afw.image.Exposure` components, we need to replace ``lsst.afw.table.io`` as the way :class:`~lsst.afw.typehandling.Storable`\ s are persisted, preferably using a third-party persistence library as much as possible.
We finish with a brief review of some candidate libraries.

.. _genericmap:

The Design of GenericMap
========================

While the current implementation of :class:`~lsst.afw.image.ExposureInfo` no longer uses :class:`~lsst.afw.typehandling.GenericMap`, developing this class was a significant part of our :class:`~lsst.afw.image.ExposureInfo`-related work.
In addition, we believe that LSST developers can learn from both the solutions we found and the mistakes we made in developing :class:`~lsst.afw.typehandling.GenericMap`.

Rationale
---------

We wished to implement :class:`~lsst.afw.image.ExposureInfo` using a map (string keys to object values) to decouple it from the details of what information is associated with an exposure.
Doing so allows :class:`~lsst.afw.image.ExposureInfo` to store objects of classes unknown to ``afw``, and allows :class:`~lsst.afw.image.ExposureInfo` to be extended by pipeline or third-party packages without changing its code.
This concept requires some standardization of keys, but no more so than, for example, FITS header keywords.

Our original plan involved a heterogeneous map (i.e., one that stores values of multiple formal types), in order to support appending completely arbitrary metadata to an :class:`~lsst.afw.image.Exposure`, possibly including primitive types.
The LSST stack previously had at least three heterogeneous map types in C++: :class:`lsst.daf.base.PropertySet`, :class:`lsst.daf.base.PropertyList`, and :class:`lsst.pex.policy.Policy` (deprecated).
However, all these types are specialized for particular roles (e.g., :class:`~lsst.daf.base.PropertyList` is designed to represent FITS headers), and mix heterogeneous mapping with other functions.
As a result, these classes are difficult to adapt to new use cases.
In addition, the lack of a common code base makes these classes difficult to maintain, and the limited type safety makes these classes easy to use incorrectly from both C++ and Python.
:class:`~lsst.afw.typehandling.GenericMap` attempted to not only serve as a suitable back-end for :class:`~lsst.afw.image.ExposureInfo`, but also address the design problems that prevented the use of :class:`~lsst.daf.base.PropertySet` in the first place.

Currently, :class:`lsst.afw.typehandling.GenericMap` is a component of ``afw`` because it depends, indirectly, on the ``lsst.table.io`` framework.
However, ``lsst.afw.typehandling`` has no other dependencies on ``afw``.
Once object persistence is decoupled from ``lsst.table.io``, the entire subpackage can be moved to a lower level in the Stack and :class:`~lsst.afw.typehandling.GenericMap` can be treated as a fundamental LSST type.

Design Goals
------------

We designed and implemented :class:`~lsst.afw.typehandling.GenericMap` while striving to obey the following principles:

* provide both a C++ and a Python API for :class:`~lsst.afw.typehandling.GenericMap`, although it is unlikely that many Python users will use :class:`~lsst.afw.typehandling.GenericMap` directly, rather through classes such as :class:`~lsst.afw.image.ExposureInfo`.
* support storage of primitive types, as well as LSST classes written in either C++ or Python.
* provide C++ and Python APIs that are as idiomatic as possible in their respective languages, making full use of pybind11's ability to serve as an API adapter.
* do not provide any features beyond a heterogeneous mapping with simple set-get behavior.
  The single responsibility makes :class:`~lsst.afw.typehandling.GenericMap` suitable as a back-end for a variety of applications, including more ornate mapping types like :class:`~lsst.daf.base.PropertySet`.
* provide separate interface (:class:`~lsst.afw.typehandling.GenericMap` and :class:`~lsst.afw.typehandling.MutableGenericMap`) and implementation classes.
  An abstract interface makes it easy to create mappings that require specific properties (as :class:`~lsst.daf.base.PropertySet` and :class:`~lsst.daf.base.PropertyList` do) without breaking code for other users, following the open-closed principle.
  Most code based on :class:`~lsst.afw.typehandling.GenericMap` can be agnostic to the implementation class.
* rely on compile-time type safety as much as possible, with safeguards preventing invalid element retrieval in C++ and spurious type errors in Python.
* allow element retrieval without knowing the exact type as which it was stored, so long as the conversion is valid (e.g., superclass vs. subclass, or different sizes of floating-point number).
  Supporting inexact types is not only user-friendly, it avoids unnecessary coupling between code changes at the points of storage and retrieval (which may be in different packages).


The GenericMap and MutableGenericMap APIs
-----------------------------------------

The design of :class:`~lsst.afw.typehandling.GenericMap` is based on some preliminary design work by J. Bosch and on a similar class that K. Findeisen wrote in Java.
The design ended up a hybrid of the two concepts, with a number of compromises to work in a mixture of C++, Python, and pybind11.

While :class:`~lsst.afw.typehandling.GenericMap` was originally conceived as a class that could store values of *any* type, this proved incompatible with the need for a pybind11 wrapper and the desire for an idiomatic Python API.
In particular, unless ``__getitem__`` takes type information as part of its input, it needs a finite set of types it can test in C++ (the current implementation does so implicitly through a pybind11 wrapper for ``boost::variant``).
The final :class:`~lsst.afw.typehandling.GenericMap` supports values of built-in types as well as :class:`~lsst.afw.typehandling.Storable`, an interface that can be added as a mixin to any LSST class.

Inspired by the Python distinction between :class:`~collections.abc.Mapping` and :class:`~collections.abc.MutableMapping`, we provide separate interfaces for reading (:class:`~lsst.afw.typehandling.GenericMap`) and writing (:class:`~lsst.afw.typehandling.MutableGenericMap`).

C++
^^^

The C++ API is loosely based on the standard library mapping interface (taken as the intersection of the C++14 APIs for :cpp:class:`std::map` and :cpp:class:`std::unordered_map`).
:class:`~lsst.afw.typehandling.GenericMap` omits standard methods that would have paradoxical or surprising behavior when generalized to heterogeneous values, such as the new element created by :cpp:member:`~std::map::operator[]` when a key does not match.

The first level of type-safety is provided by a :cpp:class:`lsst::afw::typehandling::Key` class template, which combines a nominal key (e.g., a string) with the required type of the value.
:cpp:class:`~lsst::afw::typehandling::Key` objects are lightweight values, and can be passed around or created on the fly as easily as the underlying key type.

The original design for :class:`~lsst.afw.typehandling.GenericMap` called for the map to expose public method templates (e.g., ``void set(Key<K, T> const &, T const &)``) that would provide compile-time type safety.
Since method templates cannot be overridden in C++, these public templates would be implemented in terms of protected methods (e.g., ``void _set(Key<K, Storable> const &, Storable const &)``), which subclasses could use to define how the key-value pairs are stored and managed.

Prototyping revealed that this design had several major issues:

1. The process of delegating calls to a protected non-template method would strip away any information about *which* subclass of :class:`~lsst.afw.typehandling.Storable` was being stored.
   A careless implementation would make it legal to store a :class:`~lsst.afw.geom.SkyWcs` object and then ask for it as a :class:`~lsst.afw.image.Psf`, or vice versa.
2. An implementation of ``__getitem__`` would need to explicitly enumerate and test all possible value types, particularly all subclasses of :class:`~lsst.afw.typehandling.Storable`.
   This would, at the very least, introduce elaborate pybind11 code that would need to be kept in sync with the class definition, but would not have an obvious failure mode if the two files diverged.
3. The protected API would require multiple methods per supported type, including integers, floating point numbers, strings, and :class:`~lsst.afw.typehandling.Storable` (and ``const``/non-``const`` and value/smart pointer variants thereof).
   This would impose an enormous writing and maintenance burden on subclass authors.

We tried several solutions that would still let us use a heterogeneous map.
One of the simplest was to drop the use of separate interface and implementation classes, and with it the need to delegate to specialized protected methods.
However, we did not pursue this approach because it did not solve the problem of how to implement ``__getitem__``, and the eventual solution to that problem made an interface-oriented design acceptable again.

One option to implement ``__getitem__`` without hard-coding a list of value types was to design :cpp:class:`~lsst::afw::typehandling::Key` objects to allow retrieval by superclasses of the desired type.
However, we could not find a satisfactory implementation of this approach in C++.
While it is possible to use templates to express questions like "Does a ``Key<Storable>`` match a ``Key<SkyWcs>``?", storing mixed :cpp:class:`~lsst::afw::typehandling::Key` types would require removing the compile-time information that enables such comparisons.
Adding run-time type information to :cpp:class:`~lsst::afw::typehandling::Key` would solve the information loss problem, but C++'s support for RTTI is very limited, and the standard API only allows tests for exact type equality.

The solution we adopted is to no longer store type information -- as a :cpp:class:`~lsst::afw::typehandling::Key` object or in any other form -- in a :class:`~lsst.afw.typehandling.GenericMap`.
All queries internally pass the stored value through dynamic casting, which accesses RTTI in an implementation-dependent way that does account for subclasses.
This approach solves all the original design issues, at the cost of making :class:`~lsst.afw.typehandling.GenericMap` no longer strictly type-safe (and much slower):

1. Queries for an object of the wrong type are blocked at the casting step.
2. ``__getitem__`` can retrieve any :class:`~lsst.afw.typehandling.Storable` by considering only ``Key<Storable>`` and ``Key<shared_ptr<Storable>>``.
3. The protected API is greatly simplified because it no longer needs to accommodate a variety of :cpp:class:`~lsst::afw::typehandling::Key` classes.

In practice, the desire for type-unsafe storage was handled by making all protected methods work in terms of untyped keys and :cpp:class:`boost::variant` values.
Passing by :cpp:class:`~boost::variant` provides a convenient way to express, in code, which values are supported by :class:`~lsst.afw.typehandling.GenericMap` without committing subclasses to any particular storage mechanism.
Unfortunately, :cpp:class:`boost::variant` handles implicitly convertible types very poorly (in particular, it gets confused when expected to convert numeric types, and cannot distinguish between ``const`` and non-``const`` versions of the same type).
This internal detail has consequences for the public API: :class:`~lsst.afw.typehandling.GenericMap` does not currently support ``const`` values for any type except ``shared_ptr<Storable>``, and operations involving primitive types must match them exactly.
We hope to lift these restrictions once we can migrate to :cpp:class:`std::variant` in C++17.

After the nature of :class:`~lsst.afw.typehandling.GenericMap`'s type handling, the largest remaining problem was how to handle shared pointers, which are used extensively by :class:`~lsst.afw.image.ExposureInfo`.
In keeping with C++ idioms, and in order to correctly handle polymorphism of :class:`~lsst.afw.typehandling.Storable`, :class:`~lsst.afw.typehandling.GenericMap` returns most values by reference.
However, because the implementation holds pointers as ``shared_ptr<Storable>`` yet must return them as pointers of the correct type, its accessors create a new pointer of the desired type, which cannot be returned by reference.

We chose to return smart pointers alone by value, though the inconsistency with other value types makes it much harder to write type-agnostic code against :class:`~lsst.afw.typehandling.GenericMap`.
The alternatives were to return everything by value, which would make it impossible to support non-smart-pointer storage of :class:`~lsst.afw.typehandling.Storable`, or to always return shared pointers as ``shared_ptr<Storable>``, which would force users to perform unsafe casts in their code.

It is not practical to design an idiomatic C++ API for iterating over a :class:`~lsst.afw.typehandling.GenericMap`.
Instead, we developed a system similar to the visitors used by :cpp:class:`boost::variant` and :cpp:class:`std::variant`, where the user represents the body of the loop by a callable object that accepts values of any supported type.
In practice the callables are usually private classes with templates or overloaded methods, but in rare cases a generic lambda can be used as well.
This approach involves considerable boilerplate, but is more natural to users than an API written in terms of :cpp:class:`~std::variant` or some kind of iterator-like proxy.

Python
^^^^^^

In Python, :class:`~lsst.afw.typehandling.GenericMap` follows the :class:`~collections.abc.Mapping` API almost exactly, aside from the need for homogeneous keys and the specific set of value types imposed by the C++ implementation.
Operations that would violate these constraints raise :class:`TypeError`.
As in C++, LSST-specific types can only be stored in a :class:`~lsst.afw.typehandling.GenericMap` if they inherit from :class:`~lsst.afw.typehandling.Storable`.
However, these types need not be implemented in C++; :class:`~lsst.afw.typehandling.Storable` is designed to be subclassed by Python types as well (see :ref:`storable`).

In C++, :class:`~lsst.afw.typehandling.GenericMap` is a class template parametrized by the key type.
Each specialization has its own pybind11 wrapper, but these wrappers are hidden by an :class:`lsst.utils.TemplateMeta` facade.
Attempts to construct a :class:`~lsst.afw.typehandling.GenericMap` in Python will infer the key type from any input data, so most users need not specify a key type explicitly.

Since the compile-time type safety provided by :cpp:class:`lsst::afw::typehandling::Key` is irrelevant in Python, :cpp:class:`~lsst::afw::typehandling::Key` does not have a pybind11 wrapper.
Instead, all :class:`~lsst.afw.typehandling.GenericMap` methods take the underlying key type (e.g., a string instead of ``Key<string>``), and the pybind11 code expresses the operations in terms of :cpp:class:`~lsst::afw::typehandling::Key`-based equivalents.

.. _storable:

The Design of Storable
======================

Rationale
---------

As noted in :ref:`genericmap`, we were unable to develop a design for :class:`lsst.afw.typehandling.GenericMap` that could accept objects of any type.
We introduced the :class:`lsst.afw.typehandling.Storable` interface to let :class:`~lsst.afw.typehandling.GenericMap` interact with LSST-specific types.
Any user-defined class must inherit from :class:`~lsst.afw.typehandling.Storable` to be stored in a :class:`~lsst.afw.typehandling.GenericMap` or :class:`~lsst.afw.image.ExposureInfo`.

To make it easier to work with :class:`~lsst.afw.typehandling.Storable` objects in C++, the interface declares several optional methods that provide common operations like equality comparison.
These methods add some clutter to implementation classes that don't define them, but make it possible to persist :class:`~lsst.afw.typehandling.Storable` objects and perform generic object manipulation without the need for unsafe casting in user code.

Design Goals
------------

We designed :class:`~lsst.afw.typehandling.Storable` to meet the following goals:

* support subclasses written in either C++ or Python
* support ``lsst.afw.table.io`` persistence
* support the smallest reasonable subset of generic operations, chosen to be equality comparison, hashing, copying, and string representation
* do not conflict with existing APIs of classes that may be retrofitted with :class:`~lsst.afw.typehandling.Storable`

The Storable API
----------------

:class:`lsst.afw.typehandling.Storable` is a subclass of :cpp:class:`lsst::afw::table::io::Persistable`, though it does not require that persistence be implemented.
This relationship ensures that :class:`lsst.afw.image.ExposureInfo` can persist :class:`~lsst.afw.typehandling.Storable` objects using the same mechanism as most of its original members.

:class:`lsst.afw.typehandling.Storable` provides methods ``equals``, ``hash_value``, ``cloneStorable``, and ``toString`` to allow comparisons, hashing, copying, and printing in C++.
``equals`` defaults to object identity comparisons, while the others throw an exception by default.
The method names, including the underscore in ``hash_value``, were chosen to avoid collisions with existing APIs (e.g., a ``clone`` method that returns a smart pointer to a more specific type than :class:`~lsst.afw.typehandling.Storable`).
We preferred this approach over a more elaborate delegation system, such as that used in the AST library and many ``table::io`` classes, because the latter approach requires that authors remember to write a new method for each subclass, no matter how remotely descended from the base.

:class:`~lsst.afw.typehandling.Storable` can be inherited from by Python classes, which should override the appropriate Python methods (e.g., ``__repr__`` instead of ``toString``).
The inheritance is handled using the `pybind11 API for Python inheritance <https://pybind11.readthedocs.io/en/stable/advanced/classes.html#overriding-virtual-functions-in-python>`_, including a "trampoline" helper class.
While the helper class has hooks for all of :class:`~lsst.afw.typehandling.Storable`'s C++ methods, :class:`~lsst.afw.typehandling.Storable`'s pybind11 wrapper does not include them to keep the Python API from being cluttered by default implementations.
In practice, C++ classes that implement these operations declare them in their own Python wrappers anyway.

.. _exposureinfo:

The Design of ExposureInfo
==========================

Rationale
---------

:class:`lsst.afw.image.ExposureInfo` is one of the most fundamental classes in the LSST Science Pipelines, and any breaking changes to it can have far-reaching effects.
We will almost certainly need to break :class:`~lsst.afw.image.ExposureInfo` when adopting a new persistence framework, as the ``lsst.afw.table.io`` framework is built into both the API and the persisted form.
Therefore, we avoided introducing breaking changes in the conversion to a map-based implementation, to keep users from having to change their code or data twice.

Design Goals
------------

We implemented our changes to :class:`~lsst.afw.image.ExposureInfo` based on the following goals:

* do not change the behavior of any existing method on :class:`~lsst.afw.image.ExposureInfo`, particularly component retrieval methods like :meth:`~lsst.afw.image.ExposureInfo.getSkyWcs`.
* keep the new code compatible with the previous :class:`~lsst.afw.image.Exposure` file format
* keep the :class:`~lsst.afw.image.Exposure` file format readable by old science pipelines code.
  In practice, this means that components stored in "archives" can be rearranged and new header keywords can be added, but no other changes are possible.
* minimize the number of new API elements added to allow operations on unknown components

.. _exposureinfo_code:

ExposureInfo Code Changes
-------------------------

:class:`lsst.afw.image.ExposureInfo` now contains a map that stores the :class:`~lsst.afw.image.ExposureInfo` components.
This map is not exposed to client code.
However, the C++ API for inserting and removing components does take a :cpp:class:`lsst::afw::typehandling::Key`, as there is no better way to make these methods type-safe.
In Python, as for :class:`lsst.afw.typehandling.GenericMap`, the key arguments are simple strings.

In the original conversion, :class:`~lsst.afw.image.ExposureInfo` stored its components in a :class:`MutableGenericMap\<string> <lsst.afw.typehandling.MutableGenericMap>`.
The need for backwards-compatibility in :class:`~lsst.afw.image.ExposureInfo`, and the intersection of type requirements imposed by ``lsst.table.io`` and :class:`~lsst.afw.typehandling.GenericMap`, meant that all generic components had to be stored as shared pointer to ``const`` :class:`~lsst.afw.typehandling.Storable` (see :ref:`exposureinfo_code_constraints`).
When it became clear that :class:`~lsst.afw.typehandling.GenericMap` will be a brittle class until at least C++17, we replaced the :class:`~lsst.afw.typehandling.MutableGenericMap` with a private class, ``StorableMap``, that only holds shared pointers to :class:`~lsst.afw.typehandling.Storable` but still checks for the appropriate subclass.
This change makes :class:`~lsst.afw.image.ExposureInfo` much easier to maintain, at the cost of being unable to incorporate a broader range of types in the future.

All but three of :class:`~lsst.afw.image.ExposureInfo`'s traditional components have been migrated to map-based storage.
The three exceptions are:

* the image metadata are stored in a :class:`lsst.daf.base.PropertySet`, which cannot inherit from :class:`~lsst.afw.typehandling.Storable` because it's in a dependency of ``lsst.afw``.
* the visit info is stored inside the metadata rather than as a separate component, so it must continue to be written there for backward compatibility.
  We chose not to duplicate it (storing both as metadata and as a :class:`~lsst.afw.typehandling.Storable`) for simplicity.
* the filter is stored inside the metadata, like the visit info.
  In addition, :class:`~lsst.afw.image.ExposureInfo`'s filter-related methods pass and return a filter object, not a shared pointer, so we cannot migrate it without either changing the existing API to use shared pointers or introducing inconsistencies between, for example, the return types of :meth:`~lsst.afw.image.ExposureInfo.getFilter` and :meth:`~lsst.afw.image.ExposureInfo.getComponent`.

.. _exposureinfo_code_constraints:

Constraints Imposed by the Use of GenericMap
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

While :class:`~lsst.afw.typehandling.GenericMap` can store both :class:`~lsst.afw.typehandling.Storable`\ s and shared pointers to :class:`~lsst.afw.typehandling.Storable`, it cannot preserve this distinction after persistence -- the ``lsst.table.io`` framework always depersists objects as shared pointers to :class:`~lsst.afw.table.io.Persistable`, so the information on whether an element was originally retrievable by reference or by shared pointer is lost.
To avoid inconsistencies in saving and restoring :class:`~lsst.afw.image.Exposure`\ s, we required that generic components in :class:`~lsst.afw.image.ExposureInfo` be pointers to :class:`~lsst.afw.typehandling.Storable` until we could change to a more flexible persistence framework.

The original access methods for the migrated components were very inconsistent in their use of ``const``, with the majority using ``shared_ptr<T const>``, some using ``shared_ptr<T>``, and some using ``const`` inconsistently between input and output.
Since:

* :class:`~lsst.afw.typehandling.GenericMap` cannot support both shared pointers to ``const`` and shared pointers to non-``const`` before C++17,
* keeping the existing mixture would require lots of (potentially unsafe) casts both in :class:`~lsst.afw.image.ExposureInfo` and in client code,
* changing the inputs to non-``const`` would likely break client code, and
* changing the outputs to ``const`` would be relatively safe,

we chose to standardize both inputs and outputs to ``const``, and to modify :class:`~lsst.afw.typehandling.GenericMap` to hold shared pointers to ``const`` *instead of* pointers to non-``const``.
Standardizing output to ``const`` was a breaking change to the C++ interfaces for :meth:`~lsst.afw.image.ExposureInfo.getPsf` and :meth:`~lsst.afw.image.ExposureInfo.getCoaddInputs`.
However, as there was no C++ code in Science Pipelines that stored the results as pointer to non-``const``, the change caused no problems to our knowledge.

ExposureInfo Persistence Changes
--------------------------------

The :class:`lsst.afw.image.Exposure` persistence format stores :class:`~lsst.afw.image.ExposureInfo` components in binary "archives" in FITS extensions, with the extension number stored in the header using keys like ``SKYWCS_ID``.
Generic components generalize this format by creating a header key from the component's key string (e.g., ``MYCOMPONENT_ID``).
The extensions containing generic components are not guaranteed to be arranged in any particular order, but neither the original nor the generalized formats depend on ordering.

The above conventions, combined with the need for backwards compatibility, imply that components that have been migrated to generic storage must have a component key that matches the original header key (e.g., ``SKYWCS`` for WCS information).
The awkward names add some inconvenience to :class:`~lsst.afw.image.ExposureInfo` clients, but an alternative would need to both generate an independent FITS header key in a backward-compatible way and store the component key inside the archive in a backward-compatible way.
Basing the FITS header key on the component key was deemed a less error-prone solution.

While the FITS header keys listing the extensions could previously be hard-coded into :class:`~lsst.afw.image.ExposureInfo`'s depersistence code, the depersistence code for generic components must search the header for keys of the expected format.
Queries for ``[arbitrary string]_ID`` are vulnerable to false positives from header keys like ``VISIT_ID`` or ``CCD_ID``; if associated values are small integers, then there is no way to  distinguish such keys from real archive IDs.

We therefore added a second convention for archive IDs, of the form ``ARCHIVE_ID_[component]``.
Old components are depersisted using the ``*_ID`` syntax, to retain compatibility with old files, while new ones are depersisted using ``ARCHIVE_ID_*``.
The new code strips both sets of header keywords when they are detected; new files read using old code may have leftover ``ARCHIVE_ID_`` keywords.

.. _newpersistence:

Alternative Persistence Frameworks
==================================

Rationale
---------

Despite recent improvements to :class:`lsst.afw.image.ExposureInfo`, our ability to attach data to an :class:`~lsst.afw.image.Exposure` is limited by the ``lsst.afw.table.io`` persistence framework.
We still have circular dependencies within ``afw``, we still need persistable objects to be written in C++, and we still need to work around the non-optimal nature of ``lsst.afw.table.io`` itself.
To go further, we need to replace ``lsst.afw.table.io`` with a well-maintained package that, unlike ``afw.table``, was designed for persistence.

Adoption Constraints
--------------------

We are looking for a persistence framework that meets many of the following criteria:

* it should have a GPL3-compatible license.
* it should allow persistence of both C++ and Python objects.
* it should allow persistence to multiple formats, including FITS, JSON, and YAML.
  Compatibility with JSON and YAML requires a framework that can represent objects as key-value pairs.
* it should allow versioning of persistence formats.
* it should allow depersistence of old files written with the ``lsst.afw.table.io`` framework.
* it should allow efficient storage of arrays, particularly images, but not necessarily in formats other than FITS.
* it should correctly depersist polymorphic types that are stored in C++ by their base class (e.g., :class:`lsst.afw.detection.Psf`), reproducing their exact type (e.g., :class:`lsst.meas.algorithms.ImagePsf`).
* it should be able to read in part of a persisted object, such as only the WCS from a persisted :class:`~lsst.afw.image.Exposure`.
* it should be able to store relationships between objects that refer to each other.
  This would allow us to separately store composite objects and their components, such as the individual PSFs used to create a :class:`lsst.meas.algorithms.CoaddPsf`, simplifying provenance tracking.
* it should persist files in a human-readable form, where practical, as a debugging aid.

We do not have any expectation that we should be able to easily change persistence frameworks in the future.

Option: Avro
------------

.. _Avro: https://avro.apache.org/

`Avro`_ is a table-like persistence library provided by Apache.
It defines persistence formats in terms of schemas, which may be composed (e.g., the user can declare that a :class:`lsst.geom.Box2I` is stored as a pair of :class:`lsst.geom.Point2I`, as long as there is a persisted form for :class:`~lsst.geom.Point2I`).
In C++, the schemas are typically compiled into proxy classes, whose data are then accessed using object member syntax.
In Python, no code generation is required, and data are represented using :class:`dict`-like objects.
Avro uses a schema definition language based on JSON, but as the notation is similar in spirit to :class:`lsst.pipe.base.PipelineTaskConnections`, it should not be a major conceptual burden on LSST developers.

Because of its record-based approach, Avro is best-suited for bulk data: one could store a :class:`~lsst.afw.table.Catalog` very naturally using Avro, but a free compound object like an :class:`~lsst.afw.image.Exposure` would have some overhead (essentially, its persisted form would be a collection of one-row tables).

Avro meets many, but not all, of our criteria:

* it uses the Apache 2.0 license.
* it has built-in support for both C++ and Python, although the Python API is much better.
* it has built-in support for JSON, as well as a proprietary format. We can, in principle, support additional formats using stream I/O, but the interfaces do not appear to have been designed for third-party implementations.
* it supports schema versioning.
* it may be possible to depersist ``lsst.afw.table.io`` files by treating it as a custom format.
* it natively supports arrays of arbitrary type, including numeric primitives.
* it does not natively handle polymorphism, though we could emulate it by storing an explicit class name (much like ``lsst.afw.table.io`` does).
  Avro makes heavy use of dynamic typing on depersistence, so differences in the persisted form among related classes are not a problem.
* its tables are row-oriented, so it does not have any special ability to partially read objects.
* it does not natively store object references, though we could emulate them by introducing a unique object ID data type.
* in Python, the persisted form is a :class:`dict` from field names to field values, which is reasonably human-readable.

Option: cereal
--------------

.. _cereal: https://uscilab.github.io/cereal/

.. _Boost Serialization: https://www.boost.org/doc/libs/release/libs/serialization/

`cereal`_ is a C++ persistence library provided by the University of Southern California.
It's similar in style to `Boost Serialization`_, but more lightweight.
References to objects are passed to a :cpp:class:`cereal::InputArchive` or :cpp:class:`cereal::OutputArchive` object, which calls user-defined methods or functions to handle non-built-in types (these methods usually call the archive recursively).

cereal was not designed for long-term storage, and makes no guarantee that a later version of the library will correctly load a file written by an earlier version.
We can work around this limitation by locking the Stack's version of cereal.

cereal meets many, but not all, of our criteria:

* it uses the 3-clause BSD license.
* it does not support languages other than C++.
  It probably cannot handle pure-Python classes (it needs C++ functions or methods to define the persisted form), but it might be able to handle Python subclasses of a C++ interface like :class:`lsst.afw.typehandling.Storable`.
* it explicitly supports subclassing the archive classes, including specializing them for new output formats.
* it supports class versioning, using a system similar to that currently used for :class:`~lsst.afw.image.ExposureInfo`.
* it is possible to depersist ``lsst.afw.table.io`` files by treating it as a custom format.
* it supports raw binary output for individual fields, which can be used to efficiently store arrays.
* it supports polymorphism, but requires explicit registration of all subclasses.
  The registration code for a particular class must be aware of all possible archive implementations, making it difficult to add new file formats.
* it supports "out of order loading" of sub-objects in specific archive implementations, including the built-in JSON archive.
* it can self-consistently track object relationships, but requires that all objects (such as a :class:`lsst.meas.algorithms.CoaddPsf` and its constituents) be in the same archive.
  It cannot be used to store such objects in separate files, though we could emulate such functionality by introducing a unique object ID to the persisted form.
* it supports human-readable key-value pairs, as well as a "simplified" output format for human-readability.


Option: FlatBuffer
------------------

.. _FlatBuffer: http://google.github.io/flatbuffers/

`FlatBuffer`_ is a struct-like persistence library provided by Google.
It defines persistence formats in terms of schemas, which may be composed (e.g., the user can declare that a :class:`lsst.geom.Box2I` is stored as a pair of :class:`lsst.geom.Point2I`, as long as there is a persisted form for :class:`~lsst.geom.Point2I`).
The schemas are compiled into proxy classes (in both C++ and Python), whose data are then accessed using object member syntax.
FlatBuffer uses a custom schema definition language, but as the notation is similar in spirit to :class:`lsst.pipe.base.PipelineTaskConnections`, it should not be a major conceptual burden on LSST developers.

Because the persisted form of each persistable type is represented by a different class, it may be difficult to write generic code against persistables.
However, FlatBuffer does provide a reflection API for working with generic persisted forms.

FlatBuffer meets some of our criteria:

* it uses the Apache 2.0 license.
* it has built-in support for both C++ and Python, although Python features are very limited.
  The C++ API is also cleaner, particularly with vector types.
* it only natively supports a proprietary file format (though it allows import from JSON).
  In C++, it may be possible to use reflection to support other formats.
* it supports schema versioning.
* it probably cannot depersist old files, as FlatBuffer makes strict demands on persistence formats in the interest of efficiency.
* it natively supports arrays of primitive type.
* it does not natively handle polymorphism, though we could emulate it by storing an explicit class name (much like ``lsst.afw.table.io`` does) and using the FlexBuffers feature to avoid knowing the schema a priori.
* it supports streaming input of large files, at least in C++, but does not support reading only a predetermined part of an object.
  However, it does depersist at close to disk-limited speed, so partial reads may be less performance-critical.
* it has native support for object relationships, even among different persisted files.
* its persisted form is highly efficient, and therefore not human-readable.

Recommendations
---------------

We do not yet recommend any of these frameworks as a replacement for ``lsst.afw.table.io``, and are continuing our research into alternatives.

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
