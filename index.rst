:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. _live-efd:

Original Format EFD
===================

The design of the "live" Engineering and Facilities Database (EFD) provided by the Observatory Control System (OCS) is described in :lts:`210` :cite:`LTS-210`.
There are two replicas of this, one at the Summit and one at the Base.
The interface between the EFD and Data Management (DM) is controlled by :lse:`72` :cite:`LSE-72`, in particular requirements OCS-DM-COM-ICD-0023, 0024, 0025, 0026, 0027, 0028, 0029, and 0030.

.. note::

  OCS-DM-COM-ICD-0026 should refer to the "EFD Transformation Service" rather than the "EFD Replicator device".

  OCS-DM-COM-ICD-0029 should be synchronized with DMS-REQ-0102 mentioned below.

  OCS-DM-COM-ICD-0030 needs to be changed to indicate that we are not using MySQL replication anymore.

  We need to add a requirement to :lse:`72` that the OCS provide an interface to retrieve the metadata (schema, table structure) for the EFD that is described in :lts:`210` section 5.6.

  We also need to add a requirement to :lse:`72` that no data made visible to DM through the EFD query interface will ever be rolled back, altered in place, or removed.


.. _transformation:

Transformation
==============

The DM EFD Transformation Service will extract content from the Base replica of the EFD, transform it (lightly), and load it into the Transformed EFD within the Consolidated Database at NCSA.
The Concept of Operations for this service is described in :ldm:`230` :cite:`LDM-230` section 2.4.

This ETL process is under the control of the OCS via a commandable SAL component implemented by the Base Data Management Control System (DMCS).
The Transformed EFD will be replicated to the Base via Data Backbone mechanisms for use in the Chilean Data Access Center (DAC).
The latency requirement for delivery of transformed EFD records in both locations is very relaxed: 24 hours, as documented in DMS-REQ-0102 in :lse:`61` :cite:`LSE-61`.
However, we expect that the EFD Transformation Service will run continuously and should provide latencies of less than 5 minutes in normal operation.
After an outage, latencies will increase until catch-up has completed.
Outages could be intentional if the OCS disables the EFD Transformation Service, or they could be unintentional if a software, hardware, or network failure occurs.
Nevertheless, the EFD Transformation Service is not to be considered as observing-critical and will be maintained at an "offline" level of reliability and service.


.. _initial-schema:

Initial Schema
--------------

The initial schema will be a copy of the Original Format EFD with minor additions.
The list of tables, including one per telemetry, event, or command topic, will remain the same, with the possible exception of a "blacklisted" set configured into the EFD Transformation Service.
(This blacklist feature is not expected to be used.  In particular, any privacy-sensitive data is expected to be copied and transformed with limitations placed on access.)

For appropriate tables, a relationship to exposures will be added to the schema. In general relation between telemetry records and exposure records is many-to-many though there may be cases when relation type is many-to-one (one telemetry record has zero or one matching exposures). This relationship is implicit in the original EFD format and is based on timestamps, in the transformed schema it should be made explicit and efficient for queries that need to match telemetry with exposures.

There could be different implementation of the explicit relationship differing in complexity and efficiency:

* One option that does not require creating additional tables is to perform matching based on timestamps.
* For many-to-one relationship it can be implemented as an additional column in EFD tables which is a foreign key to the exposure table, indexing of that foreign key will be needed for efficient joins.
* For many-to-many case relationship can be implemented in general as a separate table matching telemetry primary keys and exposure primary keys.
* For many-to-many case it may also be possible to implement it by adding a foreign key column to EFD tables and duplicating telemetry records so that each telemetry record only refers to single exposure, this would lead to de-normalization but may be acceptable in some cases given that EFD data is non-modifiable.

Particular implementation for relationship chosen for particular table may depend on such factors as telemetry update rate and nature of relation with the exposure. It will require some experimentation to select best approach.


Initial Schema Examples
```````````````````````

Suppose that schema for some Original Format EFD table looks like::

    CREATE TABLE original_EFD_Table (
        date_time DATETIME(6),
        float value1,
        float value2,
        PRIMARY KEY (date_time)
    );

#. If data in this table is fast-changing and each record in the table can only be related to a single exposure or no exposure then association to exposures can be implemented for this table as a foreign key column (assuming ``Exposure`` table exists with primary key column named ``exposureId``)::

     CREATE TABLE transformed_EFD_Table (
         date_time DATETIME(6),
         exposureId BIGINT NULL DEFAULT NULL,
         float value1,
         float value2,
         PRIMARY KEY (date_time),
         FOREIGN KEY (exposureId) REFERENCES Exposure (exposureId)
     );

   In case association to more than one type of data is needed this will result in multiple columns and foreign keys added to transformed table schema. Note that MySQL automatically indexes foreign key only in InnoDB tables, if MyISAM is used then index on ``exposureId`` will have to be added.

#. If record in original table can be associated with more than one exposure then one option is to duplicate records and assign different records to different exposures. This will also require change in primary key as individual ``date_time`` become non-unique. Additional complication in this case is that foreign key becomes a part of primary key and cannot be null (meaning that each record must be associated to some exposure)::

     CREATE TABLE transformed_EFD_Table (
         date_time DATETIME(6),
         exposureId BIGINT NOT NULL,
         float value1,
         float value2,
         PRIMARY KEY (date_time, exposureId),
         FOREIGN KEY (exposureId) REFERENCES Exposure (exposureId)
     );

#. Alternative to that which still keeps foreign key nullable is to use synthetic primary key::

     CREATE TABLE transformed_EFD_Table (
         date_time DATETIME(6),
         serial BIGINT NOT NULL AUTO_INCREMENT,
         exposureId BIGINT NULL DEFAULT = NULL,
         float value1,
         float value2,
         PRIMARY KEY (date_time, serial),
         FOREIGN KEY (exposureId) REFERENCES Exposure (exposureId)
     );

#. And most generic implementation allowing many-to-many association between exposure and EFD tables using separate table::

     CREATE TABLE transformed_EFD_Table_Exposure_Match (
         exposureId BIGINT NOT NULL,
         efd_date_time DATETIME(6) NOT NULL,
         PRIMARY KEY (exposureId, efd_date_time),
         INDEX (efd_date_time),
         FOREIGN KEY (exposureId) REFERENCES Exposure (exposureId),
         FOREIGN KEY (efd_date_time) REFERENCES transformed_EFD_Table (date_time)
     );

   One attractive feature of this schema is that it does not require any changes to the original EFD schema for transformed tables (foreign key above can actually point to an ``original_EFD_Table``), this can be a benefit for schema management operations such as schema migration.

#. It is also possible to completely avoid adding additional schema elements by simply matching exposure time (or any other time) with the set of records from original EFD tables. If original tables are indexed on timestamp this matching can be done efficiently by using two classes of queries:

   a. Finding a single telemetry record that matches exactly or precedes given timestamp (e.g. ``exposureBegin``). This query should return a single record, or no records if there was no telemetry recorded before given time::

        SELECT *
            FROM original_EFD_Table
            WHERE date_time <= exposureBegin
            ORDER BY date_time DESC
            LIMIT 1;

   b. Finding a range of records contained withing exposure window (this intentionally excludes ``exposureBegin`` as one should use previous query for finding that)::

        SELECT *
            FROM original_EFD_Table
            WHERE date_time > exposureBegin AND date_time < exposureEnd;

.. _future-schemas:

Future Schemas
--------------

Future schemas may add columns to the EFD tables and may add additional tables.
Such additions might include join tables linking visits, epochs of stability, or other identifiers with time ranges.
They might also include convenience or summary tables of extracted values, potentially averaged, filtered, interpolated, or otherwise processed from a window of values to a single number.

Because the schema will evolve and the methods for processing windows of values will evolve, particularly for each Data Release, we may need to save different versions of the transformed/processed data (a form of bitemporality).
If the schema evolution is strictly additive, this occurs naturally through accumulation of tables and columns. Even in that case some additional bookkeeping information may be needed to describe evolution of those additional columns and tables (which may be a part of the meta-information used by ETL for performing transforms).
In more complex cases schema versioning and schema migration process will have to be implemented. Exact nature of that process is difficult to predict without knowing more about structure of the summary tables and how they are produced or used. In any case the process will have to be controlled by meta-information which is likely to be a central piece of the Transformed EFD database.


Future Schema Examples
``````````````````````

One possible option to support schema changes in original EFD without significant changes in summary tables is to make summary table schemas flexible enough to support addition or removal of columns at arbitrary time. This implies some departure from regular relational model which also means that NoSQL options for summary data storage may be a reasonable approach.

In a context of relational database such flexibility can be implemented via storing all related attributes of the same type in a single table. One of the columns in this table would be an original column name or, for efficiency, corresponding integer index. Summary values will likely be associated with a particular type of the time window such as exposure or visit so it is also natural to have a foreign key of the corresponding item in the table schema too. In the minimal approach schema for table which stores summary data for the above ``original_EFD_Table`` may look like::

  -- Table which maps column names to unique indices
  CREATE TABLE summary_column_names (
      columnId INT NOT NULL,
      columnName CHAR(64) NOT NULL,
      PRIMARY KEY (columnId),
      INDEX (columnName)
  );

  -- Per-exposure summary table for floating point columns in original_EFD_Table
  CREATE TABLE summary_EFD_Table_per_exposure (
      exposureId BIGINT NOT NULL,
      columnId BIGINT NOT NULL,
      double summaryValue,
      PRIMARY KEY (exposureId, columnId),
      FOREIGN KEY (exposureId) REFERENCES Exposure (exposureId),
      FOREIGN KEY (columnId) REFERENCES summary_column_names (columnId)
   );

Each individual data type of the resulting summary table will likely need a separate summary table, unless they can be stored without loss of precision as a double precision type. If summary data for some column is represented as a set of values (set of integer status values) then above schema needs to be extended for those columns to support multiple values.

Supporting different versions of summary algorithms can mean different things:

* Having only one active algorithm at a time but recording algorithm being in use at the moment. That could be achieved via some sort of provenance mechanism by storing algorithm specification in a separate provenance table and adding provenance identifier column to the summary table.
* Having multiple active algorithms for the same time window (either during the same processing or re-processing at a later time). To store multiple summary values they need to be distinguished by algorithm identifier.

In latter case schema may look like::

  -- Table which maps algorithm names to unique indices
  CREATE TABLE summary_algo_names (
      algoId INT NOT NULL,
      algoName CHAR(64) NOT NULL,
      PRIMARY KEY (algoId),
      UNIQUE INDEX (algoName)
  );

  -- Per-exposure summary table for floating point columns in original_EFD_Table
  CREATE TABLE summary_EFD_Table_per_exposure (
      exposureId BIGINT NOT NULL,
      columnId BIGINT NOT NULL,
      algoId BIGINT NOT NULL,
      double summaryValue,
      provenanceId BIGINT NOT NULL,
      PRIMARY KEY (exposureId, columnId, algoId),
      -- additional indices may be needed to support efficient search
      FOREIGN KEY (exposureId) REFERENCES Exposure (exposureId),
      FOREIGN KEY (columnId) REFERENCES summary_column_names (columnId),
      FOREIGN KEY (algoId) REFERENCES summary_algo_names (algoId),
      FOREIGN KEY (provenanceId) REFERENCES summary_provenance (provenanceId)
   );

This is of course only one of many possibles ways to implement limited versioning support and it certainly does not cover all possible schema evolution options.


.. _large-file-annex:

Large File Annex
================

The EFD Large File Annex is a set of files pointed to by entries in the other EFD tables.
These files will be ingested into the Data Backbone under control of the EFD Transformation Service.
The "pointer" entries must not be published in a Transformed EFD instance until the files are available locally.


.. _other-considerations:

Other Considerations
====================

The physical implementation of the schema may differ between the Original Format EFD and the Transformed EFD.
In particular, partitioning schemes appropriate for the Original Format EFD may be different in the Transformed EFD.

It may be desirable to provide different policies for handling extraction of
data when catch-up is required; other similar commandable SAL components such as the Image Archiver will have this capability.
However, because of the time-ordering of the EFD data and the need for having windows of data to compute transformed results, it may be tricky to implement any policy other than "oldest-first".


.. _schedule:

Schedule
========

The Original Format EFD will begin accepting data when the Summit Facility achieves beneficial occupancy and environmental and OCS systems are installed, currently expected to occur by the end of calendar 2017.
The EFD transformation service was originally scheduled to meet an early integration exercise date of April 2018.
With potential delays in the date of Auxiliary Telescope Spectrograph delivery from August 2018 to later in the year, that integration exercise could occur later as well.
The DAX team has resources assigned to design the (logical and physical) schema for the Transformed EFD in the Fall 2018 cycle.
The DAX T/CAM has agreed that a few story points from this will be advanced into calendar 2017 to finish the :ref:`initial schema <initial-schema>`.
Any other EFD schema work necessary to support initial production will be advanced to Spring 2018.


.. rubric:: References

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :encoding: latex+latin
   :style: lsst_aa
