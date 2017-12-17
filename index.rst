:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

.. _data-products-definition:

Data Products Definition
========================

The data model given in the Data Products Definition Document (DPDD, :lse:`163`) is intentionally specified as a logical model, not a definitive physical model.
Among other things, the names of items may change, the exact data types used may be different, the organization into columns or arrays may vary, and certain quantities are left unspecified as they are algorithm-dependent.
The DPDD text describes the column contents, their expected units, and how columns are related to each other, but there is no specification for how that information is to be associated with a physical catalog.

.. _current-situation:

Current Situation
=================

Today's Science Pipelines algorithmic code generates catalog data as :py:mod:`lsst.afw.table` objects in memory, and these are currently persisted by the Data Butler as FITS binary table files.
The serialization/deserialization code in :py:mod:`lsst.afw.table` guarantees that the object in memory will exactly round-trip through the FITS binary table form if the Data Butler is subsequently used to retrieve it.
The names of columns used in the :py:mod:`lsst.afw.table` objects often contain more information than the DPDD specifies, but it should be possible to use the :py:mod:`lsst.afw.table` slots mechanism to define aliases that match or at least closely approximate the DPDD names.
These definitions must be generated in the Science Pipelines code, as they involve scientific judgements as to the appropriate mapping to the DPDD; they cannot originate from the database ingest code nor the Database Team.

The FITS binary tables can be loaded into a database using code in the :py:mod:`lsst.daf.ingest` package.
In the process, certain transformations can be applied, depending on column data type and name, under the control of a configuration.
In particular, data types are transformed into MySQL-compatible ones; ``NaN`` and infinite values are converted to SQL ``NULL``; :py:class:`lsst.afw.geom.Angle` columns are converted from radians to degrees; all non-alphanumeric characters in column names are converted to underscores unless overridden by a specific mapping; and additional empty columns (e.g. for indexing) can be added.
Slots defined in the :py:mod:`lsst.afw.table` are used to automatically create a view that allows access by the aliased name.
This database loading process does not currently preserve all of the information in the original :py:mod:`lsst.afw.table` object nor in the FITS binary table, so it is not possible for the Data Butler to exactly reproduce the in-memory object based on the results of a database query.

Some code has been written in :py:mod:`lsst.daf.io` to improve upon this situation.
The biggest change is that querying to create an in-memory :py:mod:`lsst.afw.table` is supported, as well as direct insertion from an in-memory :py:mod:`lsst.afw.table` without going through a persistent FITS binary table.
By using `SQLAlchemy`_ instead of a MySQL-specific interface, column names can be preserved exactly, and non-MySQL databases can be supported.

.. _SQLAlchemy: http://docs.sqlalchemy.org/en/latest/

Note that for the foreseeable future, all pipeline catalog outputs are expected to go through persisted files prior to loading into a database.

.. _column-metadata:

Column Metadata
===============

Additional information beyond column name, type, and value is required for science users to be able to understand the contents of the catalogs.
The DPDD provides units and a short description in its (:math:`\LaTeX`) tables, as well as typically having additional descriptive text nearby in the document.
Similar information must be captured for each published catalog column.
The metadata items that have been identified as necessary are:

- unit (already present)
- short description (already present but not always sufficiently descriptive)
- Unified Content Descriptor (`UCD`_, taken from the `UCD list`_)
- extended help text and/or link to same
- relationships between columns (e.g. that an error corresponds to a particular measurement)

.. _UCD: http://www.ivoa.net/documents/latest/UCD.html
.. _UCD list: http://www.ivoa.net/documents/latest/UCDlist.html

A database loading utility (whether based on :py:mod:`lsst.daf.ingest`, :py:mod:`lsst.daf.io`, a combination of the two, or something else), must accept this metadata and load it into "meta-tables" in the database that describe the catalog tables in the same database.
The catalog loading code will check that the :py:mod:`lsst.afw.table` objects that it is loading (from in-memory objects or persisted files) match with what is being or has been loaded into the meta-tables.

The units and short descriptions are currently provided by the Science Pipelines code at :py:mod:`lsst.afw.table` schema definition time.
UCDs will also be provided at the same time.
It is highly desirable for linkages be described at the same time, which will require an addition to the schema interface.
Much of the metadata can thus be loaded from the schema in an in-memory :py:mod:`lsst.afw.table` object, from a persisted schema in a persisted catalog, or from a separately-persisted schema as is typically written by pipeline tasks.

Extended help text and/or a link to such text, as well as inter-column relationships if not provided directly in the code, can be provided as a separate input to the database loading code on a per-column basis.
One possible form for providing this additional metadata is `VODML`_.
This specification is extremely complex, however; it may be appropriate to define a subset or adaptation of the standard and propose it to the IVOA as a modification.
See :ref:`below <yaml-metadata>` for a minimal step towards coming up with such a definition based on `YAML`_.

.. _VODML: http://www.ivoa.net/documents/VODML/index.html
.. _YAML: http://yaml.org

The Science Pipelines developers and scientists need to provide the content of this metadata and audit current column definitions to ensure they are complete and correct.
They also need to determine if any further transformations are required like those currently performed by the :py:mod:`lsst.daf.ingest` code.
Ideally such transformations would be performed by a meas_base BaseTransform subclass, but in certain cases where these are for the database alone, having this occur in the loading code (and undone in query code) may be appropriate.

They also need to determine if there are any columns generated by the pipeline algorithms that should be considered debugging or transitory and thus should not be loaded into the database.
Designating any columns as such will obviously prevent exact round-tripping of an :py:mod:`lsst.afw.table` object, but presumably this will be acceptable due to the nature of the missing columns.

.. _visit-catalogs:

Visit Catalogs
==============

Science Pipelines code does not currently generate visit and coadd catalogs, including metadata about each visit or coadd image.
Instead, this information is persisted per-image in Processed Visit Image or coadd FITS files as headers or as FITS binary tables in additional extensions.
In some cases, a subset of what should be in the visit catalog is present in the Butler registry, loaded from raw image header information during the repository ingest process (performed by different code in :py:mod:`lsst.daf.ingest` from the database loading code).

Database loading code should expect to be able to load visit catalog contents from both image files and separate persisted :py:mod:`lsst.afw.table` objects.
Some image metadata is expected to be stored as BLOBs or as pointers to separate files (e.g. containing persisted models) rather than explicit columns.

.. _integration-plans:

Integration Plans
=================

The catalog outputs from the periodic :abbr:`HSC (Subaru HyperSuprimeCam)` precursor data processing runs will be loaded into database tables on the `lsst-db`_ development environment machine.
As the code is improved to store meta-tables, these will also be loaded.

.. _lsst-db: https://developer.lsst.io/services/lsst-db.html

.. _yaml-metadata:

The MySQL-specific schema in the :py:mod:`lsst.cat` package, which was intended to be used to create catalog databases and is used as the basis for the Web-based schema browser, will be replaced by YAML files containing the tables, columns, and metadata described above.
Each of the :math:`\LaTeX` tables in the DPDD will be generated from this source-of-truth YAML file.
The YAML will also be used to generate validation code to test both the FITS binary and loaded database tables for consistency.

It is desirable for `Continuous Integration`_ runs to also load catalogs and their metadata into database tables, but having CI depend on a specific database server is undesirable.
Since the database loading code is being made portable across database implementations, one way of resolving this is to have CI load into a SQLite database.

.. _Continuous Integration: https://developer.lsst.io/build-ci/jenkins.html#jenkins-job-listing

One of the verification tasks will be to ensure that every product described in the DPDD has at least one completely defined and documented persisted format.

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :encoding: latex+latin
..    :style: lsst_aa
