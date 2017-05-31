.. _variables:

================
TokuDB Variables
================

Like all storage engines, TokuDB has variables to tune performance and control behavior. Fractal Tree algorithms are designed for near optimal performance and TokuDB's default settings should work well in most situations, eliminating the need for complex and time consuming tuning in most cases.

.. contents::
  :local:

Client Session Variables
------------------------

``unique_checks``

  For tables with unique keys, every insertion into the table causes a lookup by key followed by an insertion, if the key is not in the table. This greatly limits insertion performance. If one knows by design that the rows being inserted into the table have unique keys, then one can disable the key lookup prior to insertion as follows:

  .. code-block:: sql

   SET unique_checks=OFF;

  If your primary key is an auto-increment key, and none of your secondary keys are declared to be unique, then setting ``unique_checks=OFF`` will provide limited performance gains. On the other hand, if your primary key has a lot of entropy (it looks random), or your secondary keys are declared unique and have a lot of entropy, then disabling unique checks can provide a significant performance boost.

  If ``unique_checks`` is disabled when the primary key is not unique, secondary indexes may become corrupted. In this case, the indexes should be dropped and rebuilt. This behavior differs from that of InnoDB, in which uniqueness is always checked on the primary key, and setting ``unique_checks`` to off turns off uniqueness checking on secondary indexes only. Turning off uniqueness checking on the primary key can provide large performance boosts, but it should only be done when the primary key is known to be unique.

``tokudb_commit_sync``

  Session variable ``tokudb_commit_sync`` controls whether or not the transaction log is flushed when a transaction commits. The default behavior is that the transaction log is flushed by the commit. Flushing the transaction log requires a disk write and may adversely affect the performance of your application.

  To disable synchronous flushing of the transaction log, disable the ``tokudb_commit_sync`` session variable as follows:

  .. code-block:: sql

   SET tokudb_commit_sync=OFF;

  Disabling this variable may make the system run faster. However, transactions committed since the last checkpoint are not guaranteed to survive a crash.

``tokudb_pk_insert_mode``

  This session variable controls the behavior of primary key insertions with the command ``REPLACE INTO`` and ``INSERT IGNORE`` on tables with no secondary indexes and on tables whose secondary keys whose every column is also a column of the primary key.

  For instance, the table ``(column_a INT, column_b INT, column_c INT, PRIMARY KEY (column_a,column_b), KEY (column_b))`` is affected, because the only column in the key of *column_b* is present in the primary key. TokuDB can make these insertions really fast on these tables. However, triggers may not work and row based replication definitely will not work in this mode. This variable takes the following values, to control this behavior. This only applies to tables described above, using the command ``REPLACE INTO`` or ``INSERT IGNORE``. All other scenarios are unaffected.

  * ``0``: Insertions are fast, regardless of whether triggers are defined on the table. ``REPLACE INTO`` and ``INSERT IGNORE`` statements fail if row based replication is enabled.

  * ``1`` (default): Insertions are fast, if there are no triggers defined on the table. Insertions may be slow if triggers are defined on the table. ``REPLACE INTO`` and ``INSERT IGNORE`` statements fail if row based replication is enabled.

  * ``2``: Insertions are slow, all triggers on the table work, and row based replication works on ``REPLACE INTO`` and ``INSERT IGNORE`` statements.

``tokudb_load_save_space``

  This session variable changes the behavior of the bulk loader. When it is disabled the bulk loader stores intermediate data using uncompressed files (which consumes additional CPU), whereas on compresses the intermediate files. It is enabled by default.

  .. note:: The location of the temporary disk space used by the bulk loader may be specified with the ``tokudb_tmp_dir`` server variable.

  If a ``load data infile`` statement fails with the error message ``ERROR 1030 (HY000): Got error 1`` from storage engine, then there may not be enough disk space for the optimized loader, so disable ``tokudb_prelock_empty`` and try again.

More information is available in :ref:`Known Issues <known-issues>`.

``tokudb_prelock_empty``

  By default, in 7.1.0, TokuDB preemptively grabs an entire table lock on empty tables. If one transaction is doing the loading, such as when the user is doing a table load into an empty table, this default provides a considerable speedup.

  However, if multiple transactions try to do concurrent operations on an empty table, all but one transaction will be locked out. Disabling ``tokudb_prelock_empty`` optimizes for this multi-transaction case by turning off preemptive prelocking.

  .. note:: If this variable is set to off, fast bulk loading is turned off as well.

.. _tokudb_create_index_online:

``tokudb_create_index_online``

  This variable controls whether indexes created with the ``CREATE INDEX`` command are hot (if enabled), or offline (if disabled). Hot index creation means that the table is available for inserts and queries while the index is being created. Offline index creation means that the table is not available for inserts and queries while the index is being created.

  .. note:: Hot index creation is slower than offline index creation.

  By default, ``tokudb_create_index_online`` is enabled.

``tokudb_disable_slow_alter``

  This variable controls whether slow alter tables are allowed. For example, the following command is slow because HCADER does not allow a mixture of column additions, deletions, or expansions:

  .. code-block:: sql

     ALTER TABLE table
     ADD COLUMN column_a INT, 
     DROP COLUMN column_b;

  By default, ``tokudb_disable_slow_alter`` is disabled, and the engine reports back to mysql that this is unsupported resulting in the following output:

  .. code-block:: none

     ERROR 1112 (42000): Table 'test_slow' uses an extension that doesn't exist in this MySQL version

``tokudb_block_size``

  Fractal tree internal and leaf nodes default to 4,194,304 bytes (4 MB). The session variable ``tokudb_block_size`` controls the target uncompressed size of these nodes.

  Changing the value of ``tokudb_block_size`` only affects subsequently created tables. The value of this variable cannot be changed for an existing table without a dump and reload.

``tokudb_read_block_size``

  Fractal tree leaves are subdivided into read blocks, in order to speed up point queries. The session variable ``tokudb_read_block_size`` controls the target uncompressed size of the read blocks. The units are bytes and the default is 65,536 (64 KB). A smaller value favors read performance for point and small range scans over large range scans and higher compression. The minimum value of this variable is 4096.

  Changing the value of ``tokudb_read_block_size`` only affects subsequently created tables. The value of this variable cannot be changed for an existing table without a dump and reload.

``tokudb_read_buf_size``

  This variable controls the size of the buffer used to store values that are bulk fetched as part of a large range query. Its unit is bytes and its default value is 131,072 (128 KB).

  A value of 0 turns off bulk fetching. Each client keeps a thread of this size, so it should be lowered if situations where there are a large number of clients simultaneously querying a table.

``tokudb_disable_prefetching``

  TokuDB attempts to aggressively prefetch additional blocks of rows, which is helpful for most range queries but may create unnecessary IO for range queries with ``LIMIT`` clauses. Prefetching is on by default, with a value of 0, and can be disabled by setting this variable to 1.

``tokudb_row_format``

  This session variable controls the default compression algorithm used to compress data when no row format is specified in the ``CREATE TABLE`` command. See :ref:`Compression Details <compress-details>`.

``tokudb_analyze_time``

  This session variable controls the number of seconds an analyze operation will spend on each index when calculating cardinality. Cardinality is shown by executing the following command:

  .. code-block:: sql

   SELECT INDEXES FROM table_name;

  If an analyze is never performed on a table then the cardinality is 1 for primary key indexes and unique secondary indexes, and NULL (unknown) for all other indexes. Proper cardinality can lead to improved performance of complex SQL statements. The default value is 5.

``tokudb_lock_timeout_debug``

  The following values are available:

  :0: No lock timeouts or lock deadlocks are reported.

  :1: A JSON document that describes the lock conflict is stored in the ``tokudb_last_lock_timeout`` session variable

  :2: A JSON document that describes the lock conflict is printed to the MySQL error log.

      *Supported since 7.5.5*: In addition to the JSON document describing the lock conflict, the following lines are printed to the MySQL error log:

      * A line containing the blocked thread id and blocked sql
      * A line containing the blocking thread id and the blocking sql.

  :3: A JSON document that describes the lock conflict is stored in the ``tokudb_last_lock_timeout`` session variable and is printed to the MySQL error log.

      *Supported since 7.5.5*: In addition to the JSON document describing the lock conflict, the following lines are printed to the MySQL error log:

      * A line containing the blocked thread id and blocked sql
      * A line containing the blocking thread id and the blocking sql.

``tokudb_last_lock_timeout``

  This session variable contains a JSON document that describes the last lock conflict seen by the current MySQL client. It gets set when a blocked lock request times out or a lock deadlock is detected.

  The ``tokudb_lock_timeout_debug`` session variable must have bit 0 set for this behavior, otherwise this session variable will be empty.

``tokudb_bulk_fetch``

  This session variable determines if our bulk fetch algorithm is used for ``SELECT`` and ``DELETE`` statements. ``SELECT`` statements include pure ``SELECT ...`` statements, as well as ``INSERT INTO table-name ... SELECT ...``, ``CREATE TABLE table-name ... SELECT ...``, ``REPLACE INTO table-name ... SELECT ...``, ``INSERT IGNORE INTO table-name ... SELECT ...``, and ``INSERT INTO table-name ... SELECT ... ON DUPLICATE KEY UPDATE``.

  By default, ``tokudb_bulk_fetch`` is enabled.

``tokudb_support_xa``

  This session variable defines whether or not the prepare phase of an XA transaction performs an ``fsync()``.

  By default, ``tokudb_support_xa`` is enabled.

``tokudb_optimize_throttling``

  *Supported since 7.5.5*

  By default, table optimization will run with all available resources. To limit the amount of resources, it is possible to limit the speed of table optimization. The ``tokudb_optimize_throttling`` session variable determines an upper bound on how many fractal tree leaf nodes per second are optimized. The default is 0 (no upper bound) with a valid range of [0,1000000].

``tokudb_optimize_index_name``

  *Supported since 7.5.5*

  To optimize a single index in a table, the ``tokudb_optimize_index_name`` session variable can be enabled to select the index by name.

``tokudb_optimize_index_fraction``

  *Supported since 7.5.5*

  For patterns where the left side of the tree has many deletions (a common pattern with increasing id or date values), it may be useful to delete a percentage of the tree. In this case, it’s possible to optimize a subset of a fractal tree starting at the left side. The ``tokudb_optimize_index_fraction`` session variable controls the size of the sub tree. Valid values are in the range [0.0,1.0] with default 1.0 (optimize the whole tree).

``tokudb_backup_throttle``

  This session level variable throttles the write rate in bytes per second of the backup to prevent Hot Backup from crowding out other jobs in the system. The default and max values are 18446744073709551615

``tokudb_backup_dir``

  *Supported since 7.5.5*

  When enabled, this session level variable serves two purposes, to point to the destination directory where the backups will be dumped and to kick off the backup as soon as it is set.


``tokudb_backup_exclude``
 
 *Supported since 7.5.5*

 Use this variable to set a regular expression that defines source files excluded from backup. For example, to exclude all `lost+found` directories, use the following command:

 .. code-block:: console

   mysql> set tokudb_backup_exclude='/lost\\+found($|/)';

``tokudb_backup_last_error``

  *Supported since 7.5.5*

  This session variable will contain the error number from the last backup. 0 indicates success.

``tokudb_backup_last_error_string``

  *Supported since 7.5.5*

  This session variable will contain the error string from the last backup.

MySQL Server Variables
----------------------

``tokudb_loader_memory_size``

  Limits the amount of memory that the TokuDB bulk loader will use for each loader instance, defaults to 100 MB. Increasing this value may provide a performance benefit when loading extremely large tables with several secondary indexes.

  .. note:: Memory allocated to a loader is taken from the TokuDB cache, defined as ``tokudb_cache_size``, and may impact the running workload's performance as existing cached data must be ejected for the loader to begin.

``tokudb_fsync_log_period``

  Controls the frequency, in milliseconds, for ``fsync()`` operations. If set to 0 then the ``fsync()`` behavior is only controlled by the tokudb commit sync, which is on or off. The default values is 0.

``tokudb_cache_size``

  This variable configures the size in bytes of the TokuDB cache table. The default cache table size is 1/2 of physical memory. Percona highly recommends using the default setting if using buffered IO, if using direct IO then consider setting this parameter to 80% of available memory.

  Consider decreasing ``tokudb_cache_size`` if excessive swapping is causing performance problems. Swapping may occur when running multiple mysql server instances or if other running applications use large amounts of physical memory.

``tokudb_directio``

  When enabled, TokuDB employs Direct IO rather than Buffered IO for writes. When using Direct IO, consider increasing ``tokudb_cache_size`` from its default of 1/2 physical memory.

  By default, ``tokudb_directio`` is disabled.

``tokudb_lock_timeout``

  This variable controls the amount of time that a transaction will wait for a lock held by another transaction to be released. If the conflicting transaction does not release the lock within the lock timeout, the transaction that was waiting for the lock will get a lock timeout error. The units are milliseconds. A value of 0 disables lock waits. The default value is 4000 (four seconds).

  If your application gets a ``lock wait timeout`` error (-30994), then you may find that increasing the ``tokudb_lock_timeout`` may help. If your application gets a ``deadlock found`` error (-30995), then you need to abort the current transaction and retry it.

``tokudb_data_dir``

  This variable configures the directory name where the TokuDB tables are stored. The default location is the MySQL data directory.

``tokudb_log_dir``

  This variable specifies the directory where the TokuDB log files are stored. The default location is the MySQL data directory. Configuring a separate log directory is somewhat involved. Please contact Percona support for more details.

.. _tokudb-tmp-dir:

``tokudb_tmp_dir``

  This variable specifies the directory where the TokuDB bulk loader stores temporary files. The bulk loader can create large temporary files while it is loading a table, so putting these temporary files on a disk separate from the data directory can be useful.

  For example, it can make sense to use a high-performance disk for the data directory and a very inexpensive disk for the temporary directory. The default location for temporary files is the MySQL data directory.

``tokudb_checkpointing_period``

  This variable specifies the time in seconds between the beginning of one checkpoint and the beginning of the next. The default time between TokuDB checkpoints is 60 seconds. We recommend leaving this variable unchanged.

``tokudb_write_status_frequency``, ``tokudb_read_status_frequency``

  TokuDB shows statement progress of queries, inserts, deletes, and updates in ``SHOW PROCESSLIST``. Queries are defined as reads, and inserts, deletes, and updates are defined as writes.

  Progress for updated is controlled by ``tokudb_write_status_frequency``, which is set to 1000, that is, progress is measured every 1000 writes.

  Progress for reads is controlled by ``tokudb_read_status_frequency`` which is set to 10,000.

  For slow queries, it can be helpful to set these variables to 1, and then run ``show processlist`` several times to understand what progress is being made.

``tokudb_fs_reserve_percent``

  This variable controls the percentage of the file system that must be available for inserts to be allowed. By default, this is set to 5. We recommend that this reserve be at least half the size of your physical memory. See :ref:`Full Disks <full-disks>` for more information.

``tokudb_cleaner_period``

  This variable specifies how often in seconds the cleaner thread runs. The default value is 1. Setting this variable to 0 turns off cleaner threads.

``tokudb_cleaner_iterations``

  This variable specifies how many internal nodes get processed in each ``tokudb_cleaner_period`` period. The default value is 5. Setting this variable to 0 turns off cleaner threads.

``tokudb_backup_throttle``

  This variable specifies the maximum number of bytes per second the copier of a hot backup process will consume. Lowering its value will cause the hot backup operation to take more time but consume less IO on the server. The default value is 18446744073709551615.

``tokudb_rpl_lookup_rows``

  When disabled, TokuDB replication slaves skip row lookups for *delete row* log events and *update row* log events, which eliminates all associated read IO for these operations.

  .. note:: Optimization is only enabled when ``read_only`` is 1 ``binlog_format`` is ROW.

  By default, ``tokudb_rpl_lookup_rows`` is enabled.

``tokudb_rpl_lookup_rows_delay``

  This server variable allows for simulation of long disk reads by sleeping for the given number of microseconds prior to the row lookup query, it should only be set to a non-zero value for testing.

  By default, ``tokudb_rpl_lookup_rows_delay`` is disabled.

``tokudb_rpl_unique_checks``

  When disabled, TokuDB replication slaves skip uniqueness checks on inserts and updates, which eliminates all associated read IO for these operations.

  .. note:: Optimization is only enabled when ``read_only`` is 1, `binlog_format` is ROW.

  By default, ``tokudb_rpl_unique_checks`` is enabled.

``tokudb_rpl_unique_checks_delay``

  This server variable allows for simulation of long disk reads by sleeping for the given number of microseconds prior to the row lookup query, it should only be set to a non-zero value for testing.

  By default, ``tokudb_rpl_unique_checks_delay`` is disabled.

``tokudb-backup-plugin-version``

  *Supported since 7.5.5:*

  This server variable documents the version of the hot backup plugin

``tokudb_backup_version``

  *Supported since 7.5.5:*

  This server variable documents the version of the hot backup library.

``tokudb_backup_allowed_prefix``

  *Supported since 7.5.5:*

  This system-level variable restricts the location of the destination directory where the backups can be located. Attempts to backup to a location outside of the directory this variable points to or its children will result in an error.

  The default is null, backups have no restricted locations. This read only variable can be set in the :file:`my.cnf` file and displayed with the ``show variables`` command.

``tokudb_rpl_check_readonly``

  *Supported since 7.5.5:*

  The TokuDB replication code will run row events from the binlog with RFR when the slave is in read only mode. The ``tokudb_rpl_check_readonly`` variable is used to disable the slave read only check in the TokuDB replication code.

  This allows RFR to run when the slave is NOT read only. By default, ``tokudb_rpl_check_readonly`` is enabled (check slave read only). Do NOT change this value unless you completely understand the implications!
