:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. _live-efd:

Live EFD
========

The design of the "live" Engineering and Facilities Database (EFD) provided by the Observatory Control System (OCS) is described in :cite:`LTS-210`.
There are two replicas of this, one at the Summit and one at the Base.
The interface between the EFD and Data Management (DM) is controlled by :cite:`LSE-72`, in particular requirements OCS-DM-COM-ICD-0023, 0024, 0025, 0026, 0027, 0028, 0029, and 0030.

.. note::

  OCS-DM-COM-ICD-0026 should refer to the "EFD Transformation Service" rather than the "EFD Replicator device".
  OCS-DM-COM-ICD-0029 should be synchronized with DMS-REQ-0102 mentioned below.
  OCS-DM-COM-ICD-0030 needs to be changed to indicate that we are not using MySQL replication anymore.
  We need to add a requirement to LSE-72 that the OCS provide an interface to retrieve the metadata (schema, table structure) for the EFD that is described in LTS-210 section 5.6.
  We also need to add a requirement to LSE-72 that no data made visible to DM through the EFD query interface will ever be rolled back, altered in place, or removed.


.. _transformation:

Transformation
==============

The DM EFD Transformation Service will extract content from the Base replica of the EFD, transform it (lightly), and load it into the Transformed EFD within the Consolidated Database at NCSA.
This ETL process is under the control of the OCS via a commandable SAL component implemented by the Base Data Management Control System (DMCS).
The Transformed EFD will be replicated to the Base for use in the Chilean Data Access Center (DAC).
The latency requirement for delivery of transformed EFD records in both locations is very relaxed: 24 hours, as documented in DMS-REQ-0102 in :cite:`LSE-61`.
However, we expect that the EFD Transformation Service will run continuously and should provide latencies of less than 5 minutes in normal operation.
After an outage, latencies will increase until catch-up has completed.
Outages could be intentional if the OCS disables the EFD Transformation Service, or they could be unintentional if a software, hardware, or network failure occurs.
Nevertheless, the EFD Transformation Service is not to be considered as observing-critical and will be maintained at an "offline" level of reliability and service.

.. _initial-schema:

Initial schema
--------------

The initial schema will be a copy of the Live EFD with minor additions.
The list of tables, including one per telemetry, event, or command topic, will remain the same, with the possible exception of a "blacklisted" set configured into the EFD Transformation Service.
(This blacklist feature is not expected to be used.  In particular, any privacy-sensitive data is expected to be copied and transformed with limitations placed on access.)
For appropriate tables, a column will be added with an exposure identifier.
Note that there may be multiple rows associated with a single exposure, and many rows may not be associated with any exposure.

.. _future-schemas:

Future schemas
--------------

Future schemas may add columns to the EFD tables and may add additional tables.
Such additions might include join tables linking visits, epochs of stability, or other identifiers with time ranges.
They might also include convenience tables of extracted values, potentially averaged, filtered, interpolated, or otherwise processed from a window of values to a single number.


.. _large-file-annex:

Large File Annex
================

The EFD Large File Annex is a set of files pointed to by entries in the other EFD tables.
These files will be copied to a suitable filesystem at NCSA.


.. _other-considerations:

Other Considerations
====================

The physical implementation of the schema may differ between the Live EFD and the Transformed EFD.
In particular, partitioning schemes appropriate for the Live EFD may be different in the Transformed EFD.

It may be desirable to provide different policies for handling extraction of
data when catch-up is required; other similar commandable SAL components such as the Image Archiver will have this capability.
However, because of the time-ordering of the EFD data and the need for having windows of data to compute transformed results, it may be tricky to implement any policy other than "oldest-first".


.. _schedule:

Schedule
========

The Live EFD will begin accepting data when the Summit Facility achieves beneficial occupancy and environmental and OCS systems are installed, currently expected to occur by the end of calendar 2017.
The EFD transformation service was originally scheduled to meet an early integration exercise date of April 2018.
With potential delays in the date of Auxiliary Telescope Spectrograph delivery from August 2018 to later in the year, that integration exercise could occur later as well.
The DAX team has resources assigned to design the (logical and physical) schema for the Transformed EFD in the Fall 2018 cycle.
Fritz says that a few story points from this could be advanced into calendar 2017 to finish the :ref:`initial schema <initial-schema>`.
Any other EFD schema work necessary to support initial production will be advanced to Spring 2018.


.. .. rubric:: References

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :encoding: latex+latin
..    :style: lsst_aa
