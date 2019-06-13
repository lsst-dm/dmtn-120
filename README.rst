.. image:: https://img.shields.io/badge/dmtn--120-lsst.io-brightgreen.svg
   :target: https://dmtn-120.lsst.io
.. image:: https://travis-ci.org/lsst-dm/dmtn-120.svg
   :target: https://travis-ci.org/lsst-dm/dmtn-120
..
  Uncomment this section and modify the DOI strings to include a Zenodo DOI badge in the README
  .. image:: https://zenodo.org/badge/doi/10.5281/zenodo.#####.svg
     :target: http://dx.doi.org/10.5281/zenodo.#####

########################################################################
Improving Extensibility in afw.image.Exposure and Replacing afw.table.io
########################################################################

DMTN-120
========

A design study for improving the persistence of Stack classes, particularly `lsst.afw.image.Exposure` and its components. This note describes the design decisions behind the `lsst.afw.typehandling.GenericMap` class template and explores options for an off-the-shelf persistence framework to replace `lsst.afw.table.io`.

**Links:**

- Publication URL: https://dmtn-120.lsst.io
- Alternative editions: https://dmtn-120.lsst.io/v
- GitHub repository: https://github.com/lsst-dm/dmtn-120
- Build system: https://travis-ci.org/lsst-dm/dmtn-120


Build this technical note
=========================

You can clone this repository and build the technote locally with `Sphinx`_:

.. code-block:: bash

   git clone https://github.com/lsst-dm/dmtn-120
   cd dmtn-120
   pip install -r requirements.txt
   make html

.. note::

   In a Conda_ environment, ``pip install -r requirements.txt`` doesn't work as expected.
   Instead, ``pip`` install the packages listed in ``requirements.txt`` individually.

The built technote is located at ``_build/html/index.html``.

Editing this technical note
===========================

You can edit the ``index.rst`` file, which is a reStructuredText document.
The `DM reStructuredText Style Guide`_ is a good resource for how we write reStructuredText.

Remember that images and other types of assets should be stored in the ``_static/`` directory of this repository.
See ``_static/README.rst`` for more information.

The published technote at https://dmtn-120.lsst.io will be automatically rebuilt whenever you push your changes to the ``master`` branch on `GitHub <https://github.com/lsst-dm/dmtn-120>`_.

Updating metadata
=================

This technote's metadata is maintained in ``metadata.yaml``.
In this metadata you can edit the technote's title, authors, publication date, etc..
``metadata.yaml`` is self-documenting with inline comments.

Using the bibliographies
========================

The bibliography files in ``lsstbib/`` are copies from `lsst-texmf`_.
You can update them to the current `lsst-texmf`_ versions with::

   make refresh-bib

Add new bibliography items to the ``local.bib`` file in the root directory (and later add them to `lsst-texmf`_).

.. _Sphinx: http://sphinx-doc.org
.. _DM reStructuredText Style Guide: https://developer.lsst.io/restructuredtext/style.html
.. _this repo: ./index.rst
.. _Conda: http://conda.pydata.org/docs/
.. _lsst-texmf: https://lsst-texmf.lsst.io
