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

:class:`~lsst.afw.typehandling.GenericMap` attempts not only to serve as a suitable back-end for :class:`~lsst.afw.image.ExposureInfo`, but also to address the design problems that prevented the use of :class:`~lsst.daf.base.PropertySet` in this role:

* :class:`~lsst.afw.typehandling.GenericMap` relies on compile-time type safety as much as possible, with safeguards preventing invalid object retrieval in C++ and spurious type errors in Python.
* :class:`~lsst.afw.typehandling.GenericMap` provides separate interface and implementation classes, allowing mapping types with specific properties (e.g., ordering) to be created without forcing changes in client code.
  There are already two suggestions for alternatives to the reference implementation.
* :class:`~lsst.afw.typehandling.GenericMap` provides *only* a heterogeneous mapping with simple set-get behavior.
  The single responsibility makes :class:`~lsst.afw.typehandling.GenericMap` suitable as a back-end for a variety of applications, including more complex mapping types.

Initially, :class:`lsst.afw.typehandling.GenericMap` will be a component of ``afw`` because it depends, indirectly, on the ``lsst.table.io`` framework.
However, ``lsst.afw.typehandling`` has no other dependencies on ``afw``.
Once object persistence is decoupled from ``lsst.table.io``, the entire subpackage can be moved to a lower level in the Stack and :class:`~lsst.afw.typehandling.GenericMap` can be treated as a fundamental LSST type.

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
