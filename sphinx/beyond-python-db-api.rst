#######################################################################
Native Database Engine Features and Extensions Beyond the Python DB API
#######################################################################

.. currentmodule:: firebirdsql

Programmatic Database Creation and Deletion
===========================================

The Firebird engine stores a database in a fairly straightforward
manner: as a single file or, if desired, as a segmented group of
files.

The engine supports dynamic database creation via the SQL statement
`CREATE DATABASE`.

The engine also supports dropping (deleting) databases dynamically,
but dropping is a more complicated operation than creating, for
several reasons: an existing database may be in use by users other
than the one who requests the deletion, it may have supporting objects
such as temporary sort files, and it may even have dependent shadow
databases. Although the database engine recognizes a `DROP DATABASE`
SQL statement, support for that statement is limited to the `isql`
command-line administration utility. However, the engine supports the
deletion of databases via an API call, which pyfirebirdsql exposes to
Python (see below).

pyfirebirdsql supports dynamic database creation and deletion via the
module-level function :func:`firebirdsql.create_database` and the method
:meth:`~firebirdsql.Connection.drop_database`. These are documented below,
then demonstrated by a brief example.

.. function:: create_database()

   Creates a database according to the supplied `CREATE DATABASE` SQL
   statement. Returns an open connection to the newly created database.

   Arguments:

   :sql:     string containing the `CREATE DATABASE` statement. Note that
             this statement may need to include a username and password.

.. method:: Connection.drop_database()

   Deletes the database to which the connection is attached.

   This method performs the database deletion in a responsible fashion.
   Specifically, it:

   + raises an `OperationalError` instead of deleting the database if
     there are other active connections to the database
   + deletes supporting files and logs in addition to the primary
     database file(s)

   This method has no arguments.

   Example program:

   .. sourcecode:: python

      import firebirdsql

      con = firebirdsql.create_database(
            "create database '/temp/db.db' user 'sysdba' password 'pass'"
            )
      con.drop_database()



Advanced Transaction Control
============================

For the sake of simplicity, pyfirebirdsql lets the Python programmer
ignore transaction management to the greatest extent allowed by the
Python Database API Specification 2.0. The specification says, "if the
database supports an auto-commit feature, this must be initially off".
At a minimum, therefore, it is necessary to call the `commit` method
of the connection in order to persist any changes made to the
database. Transactions left unresolved by the programmer will be
`rollback`ed when the connection is garbage collected.

Remember that because of `ACID 
<http://philip.greenspun.com/panda/databases-choosing#acid>`__,
every data manipulation operation in the Firebird database engine
takes place in the context of a transaction, including operations
that are conceptually "read-only", such as a typical `SELECT`.
The client programmer of pyfirebirdsql establishes a transaction
implicitly by using any SQL execution method, such as 
:meth:`~Connection.execute_immediate()`,
:meth:`Cursor.execute()`, or :meth:`Cursor.callproc()`.

Although pyfirebirdsql allows the programmer to pay little attention to
transactions, it also exposes the full complement of the database
engine's advanced transaction control features: transaction
parameters, retaining transactions, savepoints, and distributed
transactions.

Explicit transaction start
--------------------------

In addition to the implicit transaction initiation required by
Python Database API, pyfirebirdsql allows the programmer to
start transactions explicitly via the `Connection.begin` method.

.. method:: Connection.begin(tpb)

   Starts a transaction explicitly. This is never *required*; a
   transaction will be started implicitly if necessary.

   :tpb: Optional transaction parameter buffer (TPB) populated with
         `firebirdsql.isc_tpb_*` constants. See the Firebird API guide
         for these constants' meanings.


Transaction Parameters
----------------------

The database engine offers the client programmer an optional facility
called *transaction parameter buffers* (TPBs) for tweaking the
operating characteristics of the transactions he initiates. These
include characteristics such as whether the transaction has read and
write access to tables, or read-only access, and whether or not other
simultaneously active transactions can share table access with the
transaction.

Connections have a :attr:`default_tpb` attribute that can be changed to set
the default TPB for all transactions subsequently started on the
connection. Alternatively, if the programmer only wants to set the TPB
for a single transaction, he can start a transaction explicitly via
the :meth:`~Connection.begin()` method and pass a TPB for
that single transaction.

For details about TPB construction, see the Firebird API documentation.
In particular, the :file:`ibase.h` supplied with Firebird contains all
possible TPB elements -- single bytes that the C API defines as
constants whose names begin with `isc_tpb_`. pyfirebirdsql makes all of
those TPB constants available (under the same names) as module-level
constants in the form of single-character strings. A transaction
parameter *buffer* is handled in C as a character array; pyfirebirdsql
requires that TPBs be constructed as Python strings. Since the
constants in the `firebirdsql.isc_tpb_*` family are single-character
Python strings, they can simply be concatenated to create a TPB.

.. warning::

   This method requires good knowledge of `tpc_block` structure and
   proper order of various parameters, as Firebird engine will raise an error
   when badly structured block would be used. Also definition of `table
   reservation` parameters is uncomfortable as you'll need to mix binary codes
   with table names passed as Pascal strings (characters preceded by string length).

The following program uses explicit transaction initiation and TPB
construction to establish an unobtrusive transaction for read-only
access to the database:

.. sourcecode:: python

    import firebirdsql

    con = firebirdsql.connect(dsn='localhost:/temp/test.db', user='sysdba', password='pass')

    # Construct a TPB by concatenating single-character strings (bytes)
    # from the firebirdsql.isc_tpb_* family.
    customTPB = (
          firebirdsql.isc_tpb_read
        + firebirdsql.isc_tpb_read_committed
        + firebirdsql.isc_tpb_rec_version
      )

    # Explicitly start a transaction with the custom TPB:
    con.begin(tpb=customTPB)

    # Now read some data using cursors:
    ...

    # Commit the transaction with the custom TPB.  Future transactions
    # opened on con will not use a custom TPB unless it is explicitly
    # passed to con.begin every time, as it was above, or
    # con.default_tpb is changed to the custom TPB, as in:
    #   con.default_tpb = customTPB
    con.commit()


For convenient and safe construction of custom `tpb_block`, pyfirebirdsql provides
special utility class `TPB`.

.. class:: TPB

   .. attribute:: access_mode

      Required access mode. Default `isc_tpb_write`.

   .. attribute:: isolation_level

      Required Transaction Isolation Level. Default `isc_tpb_concurrency`.

   .. attribute:: lock_resolution

      Required lock resolution method. Default `isc_tpb_wait`.

   .. attribute:: lock_timeout

      Required lock timeout. Default `None`.

   .. attribute:: table_reservation

      Table reservation specification. Default `None`.
      Instead of changing the value of the table_reservation object itself,
      you must change its *elements* by manipulating it as though it were
      a dictionary that mapped "TABLE_NAME": (sharingMode, accessMode)
      For example:

      .. sourcecode:: python

         tpbBuilder.table_reservation["MY_TABLE"] =
           (firebirdsql.isc_tpb_protected, firebirdsql.isc_tpb_lock_write)

   .. method:: render()

      Returns valid `transaction parameter block` according to current
      values of member attributes.

.. sourcecode:: python

    import firebirdsql

    con = firebirdsql.connect(dsn='localhost:/temp/test.db', user='sysdba', password='pass')

    # Use TPB to construct valid transaction parameter block
    # from the firebirdsql.isc_tpb_* family.
    customTPB = TPB()
    customTPB.access_mode = firebirdsql.isc_tpb_read
    customTPB.isolation_level = firebirdsql.isc_tpb_read_committed
        + firebirdsql.isc_tpb_rec_version

    # Explicitly start a transaction with the custom TPB:
    con.begin(tpb=customTPB.render())

    # Now read some data using cursors:
    ...

    # Commit the transaction with the custom TPB.  Future transactions
    # opened on con will not use a custom TPB unless it is explicitly
    # passed to con.begin every time, as it was above, or
    # con.default_tpb is changed to the custom TPB, as in:
    #   con.default_tpb = customTPB.render()
    con.commit()

If you want to build only `table reservation` part of `tpb` (for example
to add to various custom built parameter blocks), you can use class
`TableReservation` instead `TPB`.

.. class:: TableReservation

   This is a `dictionary-like` class, where keys are table names and values
   must be tuples of access parameters, i.e. "TABLE_NAME": (sharingMode, accessMode)

   .. method:: render()

      Returns properly formatted table reservation part of `transaction parameter
      block` according to current values.

Connection object also exposes two methods that return information about
current transaction:

.. class:: Connection

   .. method:: trans_info(request)

      Pythonic wrapper around :meth:`~Connection.transaction_info()` call.

      :request:      One or more information request codes (see transaction_info
                     for details). Multiple codes must be passed as tuple.

      Returns decoded response(s) for specified request code(s). When multiple requests
      are passed, returns a dictionary where key is the request code and value is the
      response from server.

   .. method:: transaction_info(request, result_type)

      Thin wrapper around Firebird API `isc_transaction_info` call. This function
      returns information about active transaction. Raises `ProgrammingError`
      exception when transaction is not active.

      :request:      One from the next constants:

                       + isc_info_tra_id
                       + isc_info_tra_oldest_interesting
                       + isc_info_tra_oldest_snapshot
                       + isc_info_tra_oldest_active
                       + isc_info_tra_isolation
                       + isc_info_tra_access
                       + isc_info_tra_lock_timeout

                     See Firebird API Guide for details.

      :result_type:  String code for result type: 

                       + 'i' for Integer
                       + 's' fro String

Retaining Operations
--------------------

The `commit` and `rollback` methods of `firebirdsql.Connection` accept
an optional boolean parameter `retaining` (default `False`) to
indicate whether to recycle the transactional context of the
transaction being resolved by the method call.

If `retaining` is `True`, the infrastructural support for the
transaction active at the time of the method call will be "retained"
(efficiently and transparently recycled) after the database server has
committed or rolled back the conceptual transaction.

In code that commits or rolls back frequently, "retaining" the
transaction yields considerably better performance. However, retaining
transactions must be used cautiously because they can interfere with
the server's ability to garbage collect old record versions. For
details about this issue, read the "Garbage" section of `this document
<http://www.ibphoenix.com/main.nfs?a=ibphoenix&s=1123236035:18161&page
=ibp_expert4>`__ by Ann Harrison.

For more information about retaining transactions, see Firebird documentation.


Savepoints
----------

Firebird 1.5 introduced support for transaction savepoints. Savepoints
are named, intermediate control points within an open transaction that
can later be rolled back to, without affecting the preceding work.
Multiple savepoints can exist within a single unresolved transaction,
providing "multi-level undo" functionality.

Although Firebird savepoints are fully supported from SQL alone via
the `SAVEPOINT 'name'` and `ROLLBACK TO 'name'` statements,
pyfirebirdsql also exposes savepoints at the Python API level for the
sake of convenience. 

.. method::  Connection.savepoint(name)

   Establishes a savepoint with the specified `name`. To roll back to a
   specific savepoint, call the :meth:`~firebirdsql.Connection.rollback()`
   method and provide a value (the name of the savepoint) for the optional
   `savepoint` parameter. If the `savepoint` parameter of 
   :meth:`~firebirdsql.Connection.rollback()` is not specified, the active
   transaction is cancelled in its entirety, as required by the Python
   Database API Specification.

The following program demonstrates savepoint manipulation via the
pyfirebirdsql API, rather than raw SQL.

.. sourcecode:: python

    import firebirdsql

    con = firebirdsql.connect(dsn='localhost:/temp/test.db', user='sysdba', password='pass')
    cur = con.cursor()

    cur.execute("recreate table test_savepoints (a integer)")
    con.commit()

    print 'Before the first savepoint, the contents of the table are:'
    cur.execute("select * from test_savepoints")
    print ' ', cur.fetchall()

    cur.execute("insert into test_savepoints values (?)", [1])
    con.savepoint('A')
    print 'After savepoint A, the contents of the table are:'
    cur.execute("select * from test_savepoints")
    print ' ', cur.fetchall()

    cur.execute("insert into test_savepoints values (?)", [2])
    con.savepoint('B')
    print 'After savepoint B, the contents of the table are:'
    cur.execute("select * from test_savepoints")
    print ' ', cur.fetchall()

    cur.execute("insert into test_savepoints values (?)", [3])
    con.savepoint('C')
    print 'After savepoint C, the contents of the table are:'
    cur.execute("select * from test_savepoints")
    print ' ', cur.fetchall()

    con.rollback(savepoint='A')
    print 'After rolling back to savepoint A, the contents of the table are:'
    cur.execute("select * from test_savepoints")
    print ' ', cur.fetchall()

    con.rollback()
    print 'After rolling back entirely, the contents of the table are:'
    cur.execute("select * from test_savepoints")
    print ' ', cur.fetchall()


The output of the example program is shown below.

.. sourcecode:: python

    Before the first savepoint, the contents of the table are:
      []
    After savepoint A, the contents of the table are:
      [(1,)]
    After savepoint B, the contents of the table are:
      [(1,), (2,)]
    After savepoint C, the contents of the table are:
      [(1,), (2,), (3,)]
    After rolling back to savepoint A, the contents of the table are:
      [(1,)]
    After rolling back entirely, the contents of the table are:
      []


Using multiple transactions with the same connection
----------------------------------------------------


Python Database API 2.0 was created with assumption that connection
can support only one transactions per single connection. However,
Firebird can support multiple independent transactions that can run
simultaneously within single connection / attachment to the database.
This feature is very important, as applications may require multiple
transaction opened simultaneously to perform various tasks, which
would require to open multiple connections and thus consume more
resources than necessary.

pyfirebirdsql surfaces this Firebird feature through new class
:class:`Transaction` and extensions to 
:class:`~firebirdsql.Connection` and :class:`~firebirdsql.Cursor`
classes.

.. class:: Connection

   .. method:: trans(tpb=None)

      Creates a new Transaction that operates within the context of this
      connection.  Cursors can be created within that Transaction via its
      .cursor() method.

   .. attribute:: transactions

      `read-only property` 

      List of non-close()d `Transaction` objects associated
      with this `Connection`. An element of this list may represent a resolved or
      unresolved physical transaction. Once a `Transaction` object has been
      created, it is only removed from the Connection's tracker if the
      Transaction's `close()` method is called (`Transaction.__del__` triggers
      an implicit close() call if necessary), or (obviously) if the Connection
      itself is close()d. The initial implementation will not make any guarantees
      about the order of the Transactions in this list. 

   .. attribute:: main_transaction

      `read-only property`

      Transaction object that represents the DB-API implicit transaction.
      The implementation guarantees that the same Transaction object will be
      reused across all DB-API transactions during the lifetime of the Connection.

   .. method:: prepare()

      Manually triggers the first phase of a two-phase commit (2PC). Use of this
      method is optional; if preparation is not triggered manually, it will be
      performed implicitly by commit() in a 2PC. See also the 
      `Distributed Transactions`_ section for details.

.. class:: Cursor

   .. attribute:: transaction

      `read-only property`

      Transaction with which this Cursor is associated. `None` if the Transaction has
      been close()d, or if the Cursor has been close()d.

.. class:: Transaction

   .. method:: __init__(connection,tpb=None)

      Constructor requires open :class:`~firebirdsql.Connection` object and optional
      `tpb` specification.

   .. attribute:: connection

      `read-only property`

      Connection object on which this Transaction is based.
      When the Connection's close() method is called, all Transactions that depend
      on the connection will also be implicitly close()d. If a Transaction has been
      close()d, its connection property will be None.

   .. attribute:: closed

      `read-only property` 

      `True` if Transaction has been closed (explicitly or implicitly).

   .. attribute:: n_physical

      `read-only property (int)`

      Number of physical transactions that have been executed via this Transaction
      object during its lifetime.

   .. attribute:: resolution

      `read-only property (int)` 

      `Zero` if this Transaction object is currently managing an open physical transaction.
      `One` if the physical transaction has been resolved normally. Note that this is an int
      property rather than a bool, and is named `resolution` rather than `resolved`, so that
      the non-zero values other than one can be assigned to convey specific information about
      the state of the transaction, in a future implementation (consider distributed
      transaction prepared state, limbo state, etc.).

   .. attribute:: cursors 

      List of non-close()d Cursor objects associated with this Transaction. When
      Transaction's close() method is called, whether explicitly or implicitly, it will
      implicitly close() each of its Cursors. Current implementation do not make any
      guarantees about the order of the Cursors in this list.

   .. method:: begin(tpb)

      See :meth:`Connection.begin()` for details.

   .. method:: commit(retaining=False)

      See :meth:`firebirdsql.Connection.commit()` for details.

   .. method:: close()

      Permanently closes the Transaction object and severs its associations with other
      objects. If the physical transaction is unresolved when this method is called,
      a rollback() will be performed first.

   .. method:: prepare()

      See :meth:`Connection.prepare()` for details.

   .. method:: rollback(retaining=False)

      See :meth:`firebirdsql.Connection.rollback()` for details.

   .. method:: savepoint()

      See :meth:`Connection.savepoint()` for details.

   .. method:: trans_info()

      See :meth:`Connection.trans_info()` for details.

   .. method:: transaction_info()

      See :meth:`Connection.transaction_info()` for details.

   .. method:: cursor()

      Creates a new Cursor that will operate in the context of this Transaction.
      The association between a Cursor and its Transaction is set when the Cursor
      is created, and cannot be changed during the lifetime of that Cursor.
      See :meth:`Connection.cursor()` for more details.

If you don't want multiple transactions, you can use implicit transaction object
associated with `Connection` and control it via transaction-management and cursor
methods of the :class:`Connection`.

Alternatively, you can directly access the implicit transaction exposed as
:attr:`~firebirdsql.Connection.main_transaction` and control it via its
transaction-management methods.

To use additional transactions, create new :class:`~firebirdsql.Transaction` object
calling :meth:`Connection.trans()` method.


Prepared Statements
===================

When you define a Python function, the interpreter initially parses
the textual representation of the function and generates a binary
equivalent called bytecode. The bytecode representation can then be
executed directly by the Python interpreter any number of times and
with a variety of parameters, but the human-oriented textual
definition of the function never need be parsed again.

Database engines perform a similar series of steps when executing a
SQL statement. Consider the following series of statements:

.. sourcecode:: python

    cur.execute("insert into the_table (a,b,c) values ('aardvark', 1, 0.1)")
    ...
    cur.execute("insert into the_table (a,b,c) values ('zymurgy', 2147483647, 99999.999)")


If there are many statements in that series, wouldn't it make sense to
"define a function" to insert the provided "parameters" into the
predetermined fields of the predetermined table, instead of forcing
the database engine to parse each statement anew and figure out what
database entities the elements of the statement refer to? In other
words, why not take advantage of the fact that the form of the
statement ("the function") stays the same throughout, and only the
values ("the parameters") vary? Prepared statements deliver that
performance benefit and other advantages as well.

The following code is semantically equivalent to the series of insert
operations discussed previously, except that it uses a single SQL
statement that contains Firebird's parameter marker ( `?`) in the
slots where values are expected, then supplies those values as Python
tuples instead of constructing a textual representation of each value
and passing it to the database engine for parsing:

.. sourcecode:: python

    insertStatement = "insert into the_table (a,b,c) values (?,?,?)"
    cur.execute(insertStatement, ('aardvark', 1, 0.1))
    ...
    cur.execute(insertStatement, ('zymurgy', 2147483647, 99999.999))


Only the values change as each row is inserted; the statement remains
the same. For many years, pyfirebirdsql has recognized situations
similar to this one and automatically reused the same prepared
statement in each :meth:`Cursor.execute` call. In pyfirebirdsql 3.2, the
scheme for automatically reusing prepared statements has become more
sophisticated, and the API has been extended to offer the client
programmer manual control over prepared statement creation and use.

The entry point for manual statement preparation is the `Cursor.prep`
method.

.. method:: Cursor.prep(sql)

   :sql:  string parameter that contains the SQL statement to be prepared.
          Returns a :class:`PreparedStatement` instance.

.. class:: PreparedStatement

   `PreparedStatement` has no public methods, but does have the following
   public read-only properties:

   .. attribute:: sql

      A reference to the string that was passed to 
      :meth:`~Cursor.prep()` to create this `PreparedStatement`.

   .. attribute:: statement_type

      An integer code that can be matched against the statement type
      constants in the `firebirdsql.isc_info_sql_stmt_*` series.
      The following statement type codes are currently available: 

        + `isc_info_sql_stmt_commit`
        + `isc_info_sql_stmt_ddl`
        + `isc_info_sql_stmt_delete`
        + `isc_info_sql_stmt_exec_procedure`
        + `isc_info_sql_stmt_get_segment`
        + `isc_info_sql_stmt_insert`
        + `isc_info_sql_stmt_put_segment`
        + `isc_info_sql_stmt_rollback`
        + `isc_info_sql_stmt_savepoint`
        + `isc_info_sql_stmt_select`
        + `isc_info_sql_stmt_select_for_upd`
        + `isc_info_sql_stmt_set_generator`
        + `isc_info_sql_stmt_start_trans`
        + `isc_info_sql_stmt_update`

   .. attribute:: n_input_params

      The number of input parameters the statement requires.

   .. attribute:: n_output_params

      The number of output fields the statement produces.

   .. attribute:: plan

      A string representation of the execution plan generated for this
      statement by the database engine's optimizer. This property can
      be used, for example, to verify that a statement is using the
      expected index.

   .. attribute:: description

      A Python DB API 2.0 description sequence (of the same format as
      :attr:`Cursor.description`) that describes the statement's output
      parameters. Statements without output parameters have a `description`
      of `None`.

In addition to programmatically examining the characteristics of a SQL
statement via the properties of `PreparedStatement`, the client
programmer can submit a `PreparedStatement` to :meth:`Cursor.execute` or
:meth:`Cursor.executemany` for execution. The code snippet below is
semantically equivalent to both of the previous snippets in this
section, but it explicitly prepares the `INSERT` statement in advance,
then submits it to :meth:`Cursor.executemany` for execution:

.. sourcecode:: python

    insertStatement = cur.prep("insert into the_table (a,b,c) values (?,?,?)")
    inputRows = [
        ('aardvark', 1, 0.1),
        ...
        ('zymurgy', 2147483647, 99999.999)
      ]
    cur.executemany(insertStatement, inputRows)


**Example Program**

The following program demonstrates the explicit use of
PreparedStatements. It also benchmarks explicit `PreparedStatement`
reuse against pyfirebirdsql's automatic `PreparedStatement` reuse, and
against an input strategy that prevents `PreparedStatement` reuse.

.. sourcecode:: python

    import time
    import firebirdsql

    con = firebirdsql.connect(dsn=r'localhost:D:\temp\test-20.firebird',
        user='sysdba', password='masterkey'
      )

    cur = con.cursor()

    # Create supporting database entities:
    cur.execute("recreate table t (a int, b varchar(50))")
    con.commit()
    cur.execute("create unique index unique_t_a on t(a)")
    con.commit()

    # Explicitly prepare the insert statement:
    psIns = cur.prep("insert into t (a,b) values (?,?)")
    print 'psIns.sql: "%s"' % psIns.sql
    print 'psIns.statement_type == firebirdsql.isc_info_sql_stmt_insert:', (
        psIns.statement_type == firebirdsql.isc_info_sql_stmt_insert
      )
    print 'psIns.n_input_params: %d' % psIns.n_input_params
    print 'psIns.n_output_params: %d' % psIns.n_output_params
    print 'psIns.plan: %s' % psIns.plan

    print

    N = 10000
    iStart = 0

    # The client programmer uses a PreparedStatement explicitly:
    startTime = time.time()
    for i in xrange(iStart, iStart + N):
        cur.execute(psIns, (i, str(i)))
    print (
        'With explicit prepared statement, performed'
        '\n  %0.2f insertions per second.' % (N / (time.time() - startTime))
      )
    con.commit()

    iStart += N

    # pyfirebirdsql automatically uses a PreparedStatement "under the hood":
    startTime = time.time()
    for i in xrange(iStart, iStart + N):
        cur.execute("insert into t (a,b) values (?,?)", (i, str(i)))
    print (
        'With implicit prepared statement, performed'
        '\n  %0.2f insertions per second.' % (N / (time.time() - startTime))
      )
    con.commit()

    iStart += N

    # A new SQL string containing the inputs is submitted every time, so
    # pyfirebirdsql is not able to implicitly reuse a PreparedStatement.  Also, in a
    # more complicated scenario where the end user supplied the string input
    # values, the program would risk SQL injection attacks:
    startTime = time.time()
    for i in xrange(iStart, iStart + N):
        cur.execute("insert into t (a,b) values (%d,'%s')" % (i, str(i)))
    print (
        'When unable to reuse prepared statement, performed'
        '\n  %0.2f insertions per second.' % (N / (time.time() - startTime))
      )
    con.commit()

    # Prepare a SELECT statement and examine its properties.  The optimizer's plan
    # should use the unique index that we created at the beginning of this program.
    print
    psSel = cur.prep("select * from t where a = ?")
    print 'psSel.sql: "%s"' % psSel.sql
    print 'psSel.statement_type == firebirdsql.isc_info_sql_stmt_select:', (
        psSel.statement_type == firebirdsql.isc_info_sql_stmt_select
      )
    print 'psSel.n_input_params: %d' % psSel.n_input_params
    print 'psSel.n_output_params: %d' % psSel.n_output_params
    print 'psSel.plan: %s' % psSel.plan

    # The current implementation does not allow PreparedStatements to be prepared
    # on one Cursor and executed on another:
    print
    print 'Note that PreparedStatements are not transferrable from one cursor to another:'
    cur2 = con.cursor()
    cur2.execute(psSel)

Output:

.. sourcecode:: python

    psIns.sql: "insert into t (a,b) values (?,?)"
    psIns.statement_type == firebirdsql.isc_info_sql_stmt_insert: True
    psIns.n_input_params: 2
    psIns.n_output_params: 0
    psIns.plan: None

    With explicit prepared statement, performed
      9551.10 insertions per second.
    With implicit prepared statement, performed
      9407.34 insertions per second.
    When unable to reuse prepared statement, performed
      1882.53 insertions per second.

    psSel.sql: "select * from t where a = ?"
    psSel.statement_type == firebirdsql.isc_info_sql_stmt_select: True
    psSel.n_input_params: 1
    psSel.n_output_params: 2
    psSel.plan: PLAN (T INDEX (UNIQUE_T_A))

    Note that PreparedStatements are not transferrable from one cursor to another:
    Traceback (most recent call last):
      File "adv_prepared_statements__overall_example.py", line 86, in ?
        cur2.execute(psSel)
    firebirdsql.ProgrammingError: (0, 'A PreparedStatement can only be used with the
     Cursor that originally prepared it.')


As you can see, the version that prevents the reuse of prepared
statements is about five times slower -- *for a trivial statement*. In
a real application, SQL statements are likely to be far more
complicated, so the speed advantage of using prepared statements would
only increase.

As the timings indicate, pyfirebirdsql does a good job of reusing
prepared statements even if the client program is written in a style
strictly compatible with the Python DB API 2.0 (which accepts only
strings -- not :class:`PreparedStatement` objects -- to the :meth:`Cursor.execute()`
method). The performance loss in this case is less than one percent.



Named Cursors
=============

To allow the Python programmer to perform scrolling `UPDATE` or
`DELETE` via the "`SELECT ... FOR UPDATE`" syntax, pyfirebirdsql
provides the read/write property `Cursor.name`.

.. attribute:: Cursor.name

   Name for the SQL cursor. This property can be ignored entirely
   if you don't need to use it.

**Example Program**

.. sourcecode:: python

    import firebirdsql

    con = firebirdsql.connect(dsn='localhost:/temp/test.db', user='sysdba', password='pass')
    curScroll = con.cursor()
    curUpdate = con.cursor()

    curScroll.execute("select city from addresses for update")
    curScroll.name = 'city_scroller'
    update = "update addresses set city=? where current of " + curScroll.name

    for (city,) in curScroll:
        city = ... # make some changes to city
        curUpdate.execute( update, (city,) )

    con.commit()



Parameter Conversion
====================

pyfirebirdsql converts bound parameters marked with a `?` in SQL code in
a standard way. However, the module also offers several extensions to
standard parameter binding, intended to make client code more readable
and more convenient to write.


Implicit Conversion of Input Parameters from Strings
----------------------------------------------------

The database engine treats most SQL data types in a weakly typed
fashion: the engine may attempt to convert the raw value to a
different type, as appropriate for the current context. For instance,
the SQL expressions `123` (integer) and `'123'` (string) are treated
equivalently when the value is to be inserted into an `integer` field;
the same applies when `'123'` and `123` are to be inserted into a
`varchar` field.

This weak typing model is quite unlike Python's dynamic yet strong
typing. Although weak typing is regarded with suspicion by most
experienced Python programmers, the database engine is in certain
situations so aggressive about its typing model that pyfirebirdsql
must `compromise <http://sourceforge.net/tracker/index.php?func=detail&
aid=531828&group_id=9913&atid=309913>`__ in order to remain an elegant
means of programming the database engine.

An example is the handling of "magic values" for date and time fields.
The database engine interprets certain string values such as
`'yesterday'` and `'now'` as having special meaning in a date/time
context. If pyfirebirdsql did not accept strings as the values of
parameters destined for storage in date/time fields, the resulting
code would be awkward. Consider the difference between the two Python
snippets below, which insert a row containing an integer and a
timestamp into a table defined with the following DDL statement:

.. sourcecode:: python

   create table test_table (i int, t timestamp)

.. sourcecode:: python

   i = 1
   t = 'now'
   sqlWithMagicValues = "insert into test_table (i, t) values (?, '%s')" % t
   cur.execute( sqlWithMagicValues, (i,) )

.. sourcecode:: python

   i = 1
   t = 'now'
   cur.execute( "insert into test_table (i, t) values (?, ?)", (i, t) )

If pyfirebirdsql did not support weak parameter typing, string
parameters that the database engine is to interpret as "magic values"
would have to be rolled into the SQL statement in a separate operation
from the binding of the rest of the parameters, as in the first Python
snippet above. Implicit conversion of parameter values from strings
allows the consistency evident in the second snippet, which is both
more readable and more general.

It should be noted that pyfirebirdsql does not perform the conversion
from string itself. Instead, it passes that responsibility to the
database engine by changing the parameter metadata structure
dynamically at the last moment, then restoring the original state of
the metadata structure after the database engine has performed the
conversion.

A secondary benefit is that when one uses pyfirebirdsql to import large
amounts of data from flat files into the database, the incoming values
need not necessarily be converted to their proper Python types before
being passed to the database engine. Eliminating this intermediate
step may accelerate the import process considerably, although other
factors such as the chosen connection protocol and the deactivation of
indexes during the import are more consequential. For bulk import
tasks, the database engine's external tables also deserve
consideration. External tables can be used to suck semi-structured
data from flat files directly into the relational database without the
intervention of an ad hoc conversion program.


Dynamic Type Translation
------------------------

Dynamic type translators are conversion functions registered by the
Python programmer to transparently convert database field values to
and from their internal representation.

The client programmer can choose to ignore translators altogether, in
which case pyfirebirdsql will manage them behind the scenes. Otherwise,
the client programmer can use any of several :ref:`standard type translators
<included-translators>` included with pyfirebirdsql, register custom
translators, or set the translators to `None` to deal directly with
the pyfirebirdsql-internal representation of the data type. When
translators have been registered for a specific SQL data type, Python
objects on their way into a database field of that type will be passed
through the input translator before they are presented to the database
engine; values on their way out of the database into Python will be
passed through the corresponding output translator. Output and input
translation for a given type is usually implemented by two different
functions.


Specifics of the Dynamic Type Translation API
---------------------------------------------

Translators are managed with next methods of :class:`~firebirdsql.Connection`
and :class:`~firebirdsql.Cursor`. 

.. method:: Connection.get_type_trans_in()

   Retrieves the inbound type translation map.

.. method:: Connection.set_type_trans_in(trans_dict)

   Changes the inbound type translation map.

.. method:: Cursor.get_type_trans_in()

   Retrieves the inbound type translation map.

.. method:: Cursor.set_type_trans_in(trans_dict)

   Changes the inbound type translation map.

The `set_type_trans_[in|out]` methods accept a single argument: a mapping
of type name to translator. The `get_type_trans[in|out]` methods return
a copy of the translation table. 

`Cursor`s inherit their `Connection`'s translation settings, but can
override them without affecting the connection or other cursors
(much as subclasses can override the methods of their base classes).

The following code snippet installs an input translator for fixed
point types ( `NUMERIC`/ `DECIMAL` SQL types) into a connection:

.. sourcecode:: python

   con.set_type_trans_in( {'FIXED': fixed_input_translator_function} )


The following method call retrieves the type translation table for
`con`:

.. sourcecode:: python

    con.get_type_trans_in()


The method call above would return a translation table (dictionary)
such as this:

.. sourcecode:: python

   {
     'DATE': <function date_conv_in at 0x00920648>,
     'TIMESTAMP': <function timestamp_conv_in at 0x0093E090>,
     'FIXED': <function <lambda> at 0x00962DB0>,
     'TIME': <function time_conv_in at 0x009201B0>
   }


Notice that although the sample code registered only one type
translator, there are four listed in the mapping returned by the
`get_type_trans_in` method. By default, pyfirebirdsql uses dynamic type
translation to implement the conversion of `DATE`, `TIME`,
`TIMESTAMP`, `NUMERIC`, and `DECIMAL` values. For the source code
locations of pyfirebirdsql's reference translators, see the :ref:`table 
<included-translators>` in the next section.

In the sample above, a translator is registered under the key
`'FIXED'`, but Firebird has no SQL data type named `FIXED`. The
following table lists the names of the database engine's SQL data
types in the left column, and the corresponding pyfirebirdsql-specific
key under which client programmers can register translators in the
right column.

.. _table-mapping-to-keys:

**Mapping of SQL Data Type Names to Translator Keys**

========================= ===========================================
SQL Type(s)               Translator Key 
========================= ===========================================
CHAR / VARCHAR            'TEXT' for fields with charsets
                          `NONE`, `OCTETS`, or `ASCII`

                          'TEXT_UNICODE' for all other charsets 
BLOB                      'BLOB'
SMALLINT/INTEGER/BIGINT   'INTEGER'
FLOAT/ DOUBLE PRECISION   'FLOATING'
NUMERIC / DECIMAL         'FIXED'
DATE                      'DATE'
TIME                      'TIME'
TIMESTAMP                 'TIMESTAMP'
========================= ===========================================

Database Arrays
---------------

pyfirebirdsql converts database arrays *from* Python sequences (except
strings) on input; *to* Python lists on output. On input, the Python
sequence must be nested appropriately if the array field is multi-
dimensional, and the incoming sequence must not fall short of its
maximum possible length (it will not be "padded" implicitly--see
below). On output, the lists will be nested if the database array has
multiple dimensions.

Database arrays have no place in a purely relational data model, which
requires that data values be *atomized* (that is, every value stored
in the database must be reduced to elementary, non-decomposable
parts). The Firebird implementation of database arrays, like that
of most relational database engines that support this data type,
is fraught with limitations.

Database arrays are of fixed size, with a predeclared number of
dimensions (max. 16) and number of elements per dimension. Individual array
elements cannot be set to `NULL` / `None`, so the mapping between
Python lists (which have dynamic length and are therefore *not*
normally "padded" with dummy values) and non-trivial database arrays
is clumsy.

Stored procedures cannot have array parameters.

Finally, many interface libraries, GUIs, and even the isql command
line utility do not support database arrays.

In general, it is preferable to avoid using database arrays unless you
have a compelling reason.


**Example Program**

The following program inserts an array (nested Python list) into a
single database field, then retrieves it.

.. sourcecode:: python

    import firebirdsql

    con = firebirdsql.connect(dsn='localhost:/temp/test.db', user='sysdba', password='pass')
    con.execute_immediate("recreate table array_table (a int[3,4])")
    con.commit()

    cur = con.cursor()

    arrayIn = [
        [1, 2, 3, 4],
        [5, 6, 7, 8],
        [9,10,11,12]
      ]

    print 'arrayIn:  %s' % arrayIn
    cur.execute("insert into array_table values (?)", (arrayIn,))

    cur.execute("select a from array_table")
    arrayOut = cur.fetchone()[0]
    print 'arrayOut: %s' % arrayOut

    con.commit()


Output:

.. sourcecode:: python

    arrayIn:  [[1, 2, 3, 4], [5, 6, 7, 8], [9, 10, 11, 12]]
    arrayOut: [[1, 2, 3, 4], [5, 6, 7, 8], [9, 10, 11, 12]]


.. _blob-conversion:

Blobs
-----

pyfirebirdsql supports the insertion and retrieval of blobs either
wholly in memory ("materialized mode") or in chunks ("streaming mode")
to reduce memory usage when handling large blobs. The default handling
mode is "materialized"; the "streaming" method is selectable via a
special case of Dynamic Type Translation.

In **materialized** mode, input and output blobs are represented as
Python `str` objects, with the result that the entirety of each blob's
contents is loaded into memory. Unfortunately, flaws in the database
engine's C API `prevent <http://sourceforge.net/forum/forum.php?thread_
id=1299756&forum_id=30917>`__ automatic Unicode conversion from
applying to textual blobs in the way it applies to Unicode `CHAR` and
`VARCHAR` fields in any Firebird version prior to version 2.1.

.. note:: pyfirebirdsql 3.3 introduces new :ref:`type_conv mode 300
  <typeconv-values>` that enables automatic type conversion for textual
  blobs when you're working with Firebird 2.1 and newer.

In **streaming** mode, any Python "file-like" object is acceptable as
input for a blob parameter. Obvious examples of such objects are
instances of :class:`file` or :class:`StringIO`. Each output blob is represented by
a :class:`firebirdsql.BlobReader` object. 

.. class:: BlobReader 

  BlobReader is a "file-like" class, so it acts much like a `file`
  instance opened in `rb` mode.

  `BlobReader` adds one method not found in the "file-like" interface:

  .. method:: chunks()

    Takes a single integer parameter that specifies the number of bytes
    to retrieve in each chunk (the final chunk may be smaller).

    For example, if the size of the blob is `50000000` bytes, 
    `BlobReader.chunks(2**20)` will return `47` one-megabyte chunks, and
    a smaller final chunk of `716928` bytes.

Due to the combination of CPython's deterministic finalization with
careful programming in pyfirebirdsql's internals, it is not strictly
necessary to close `BlobReader` instances explicitly. A `BlobReader`
object will be automatically closed by its `__del__` method when it
goes out of scope, or when its `Connection` closes, whichever comes
first. However, it is always a better idea to close resources
explicitly (via `try...finally`) than to rely on artifacts of the
CPython implementation. (For the sake of clarity, the example program
does not follow this practice.)


**Example Program**

The following program demonstrates blob storage and retrieval in both
*materialized* and *streaming* modes.

.. sourcecode:: python

    import os.path
    from cStringIO import StringIO

    import firebirdsql

    con = firebirdsql.connect(dsn=r'localhost:D:\temp\test-20.firebird',
        user='sysdba', password='masterkey'
      )

    cur = con.cursor()

    cur.execute("recreate table blob_test (a blob)")
    con.commit()

    # --- Materialized mode (str objects for both input and output) ---
    # Insertion:
    cur.execute("insert into blob_test values (?)", ('abcdef',))
    cur.execute("insert into blob_test values (?)", ('ghijklmnop',))
    # Retrieval:
    cur.execute("select * from blob_test")
    print 'Materialized retrieval (as str):'
    print cur.fetchall()

    cur.execute("delete from blob_test")

    # --- Streaming mode (file-like objects for input; firebirdsql.BlobReader
    #     objects for output) ---
    cur.set_type_trans_in ({'BLOB': {'mode': 'stream'}})
    cur.set_type_trans_out({'BLOB': {'mode': 'stream'}})

    # Insertion:
    cur.execute("insert into blob_test values (?)", (StringIO('abcdef'),))
    cur.execute("insert into blob_test values (?)", (StringIO('ghijklmnop'),))

    f = file(os.path.abspath(__file__), 'rb')
    cur.execute("insert into blob_test values (?)", (f,))
    f.close()

    # Retrieval using the "file-like" methods of BlobReader:
    cur.execute("select * from blob_test")

    readerA = cur.fetchone()[0]

    print '\nStreaming retrieval (via firebirdsql.BlobReader):'

    # Python "file-like" interface:
    print 'readerA.mode:    "%s"' % readerA.mode
    print 'readerA.closed:   %s'  % readerA.closed
    print 'readerA.tell():   %d'  % readerA.tell()
    print 'readerA.read(2): "%s"' % readerA.read(2)
    print 'readerA.tell():   %d'  % readerA.tell()
    print 'readerA.read():  "%s"' % readerA.read()
    print 'readerA.tell():   %d'  % readerA.tell()
    print 'readerA.read():  "%s"' % readerA.read()
    readerA.close()
    print 'readerA.closed:   %s'  % readerA.closed

    # The chunks method (not part of the Python "file-like" interface, but handy):
    print '\nFor a blob with contents "ghijklmnop", iterating over'
    print 'BlobReader.chunks(3) produces:'
    readerB = cur.fetchone()[0]
    for chunkNo, chunk in enumerate(readerB.chunks(3)):
        print 'Chunk %d is: "%s"' % (chunkNo, chunk)


Output:

.. sourcecode:: python

    Materialized retrieval (as str):
    [('abcdef',), ('ghijklmnop',)]

    Streaming retrieval (via firebirdsql.BlobReader):
    readerA.mode:    "rb"
    readerA.closed:   False
    readerA.tell():   0
    readerA.read(2): "ab"
    readerA.tell():   2
    readerA.read():  "cdef"
    readerA.tell():   6
    readerA.read():  ""
    readerA.closed:   True

    For a blob with contents "ghijklmnop", iterating over
    BlobReader.chunks(3) produces:
    Chunk 0 is: "ghi"
    Chunk 1 is: "jkl"
    Chunk 2 is: "mno"
    Chunk 3 is: "p"

.. _connection-timeout:

Connection Timeouts
===================

Connection timeouts allow the programmer to request that a connection
be automatically closed after a specified period of inactivity. The
simplest uses of connection timeouts are trivial, as demonstrated by
the following snippet:

.. sourcecode:: python

   import firebirdsql

   con = firebirdsql.connect(dsn=r'localhost:D:\temp\test.db',
       user='sysdba', password='masterkey',
       timeout={'period': 120.0} # time out after 120.0 seconds of inactivity
     )

   ...

The connection created in the example above is *eligible* to be
automatically closed by pyfirebirdsql if it remains idle for at least
120.0 consecutive seconds. pyfirebirdsql does not guarantee that the
connection will be closed immediately when the specified period has
elapsed. On a busy system, there might be a considerable delay between
the moment a connection becomes eligible for timeout and the moment
pyfirebirdsql actually closes it. However, the thread that performs
connection timeouts is programmed in such a way that on a lightly
loaded system, it acts almost instantaneously to take advantage of a
connection's eligibility for timeout.

After a connection has timed out, pyfirebirdsql reacts to attempts to
reactivate the severed connection in a manner dependent on the state
of the connection when it timed out. Consider the following example
program:

.. sourcecode:: python

   import time
   import firebirdsql

   con = firebirdsql.connect(dsn=r'localhost:D:\temp\test.db',
       user='sysdba', password='masterkey',
       timeout={'period': 3.0}
     )
   cur = con.cursor()

   cur.execute("recreate table test (a int, b char(1))")
   con.commit()

   cur.executemany("insert into test (a, b) values (?, ?)",
       [(1, 'A'), (2, 'B'), (3, 'C')]
     )
   con.commit()

   cur.execute("select * from test")
   print 'BEFORE:', cur.fetchall()

   cur.execute("update test set b = 'X' where a = 2")

   time.sleep(6.0)

   cur.execute("select * from test")
   print 'AFTER: ', cur.fetchall()


So, should the example program print

.. sourcecode:: python

   BEFORE: [(1, 'A'), (2, 'B'), (3, 'C')]
   AFTER:  [(1, 'A'), (2, 'X'), (3, 'C')]

or

.. sourcecode:: python

   BEFORE: [(1, 'A'), (2, 'B'), (3, 'C')]
   AFTER:  [(1, 'A'), (2, 'B'), (3, 'C')]


or should it raise an exception? The answer is more complex than one
might think.

First of all, we cannot guarantee much about the example program's
behavior because there is a race condition between the obvious thread
that's executing the example code (which we'll call "UserThread" for
the rest of this section) and the pyfirebirdsql-internal background
thread that actually closes connections that have timed out
("TimeoutThread"). If the operating system were to suspend UserThread
just after the :func:`firebirdsql.connect()` call for more than
the specified timeout period of 3.0 seconds, the TimeoutThread might
close the connection before UserThread had performed any preparatory
operations on the database. Although such a scenario is extremely
unlikely when more "realistic" timeout periods such as 1800.0 seconds
(30 minutes) are used, it is important to consider. We'll explore
solutions to this race condition later.

The *likely* (but not guaranteed) behavior of the example program is
that UserThread will complete all preparatory database operations
including the `cur. execute ( "update test set b = 'X' where a = 2" )`
statement in the example program, then go to sleep for not less than
6.0 seconds. Not less than 3.0 seconds after UserThread executes the
`cur. execute ( "update test set b = 'X' where a = 2" )` statement,
TimeoutThread is likely to close the connection because it has become
eligible for timeout.

The crucial issue is how TimeoutThread should resolve the transaction
that UserThread left open on `con`, and what should happen when
UserThread reawakens and tries to execute the `cur. execute ( "select
* from test" )` statement, since the transaction that UserThread left
open will no longer be active.


User-Supplied Connection Timeout Callbacks
------------------------------------------

In the context of a particular client program, it is not possible for
pyfirebirdsql to know the best way for TimeoutThread to react when it
encounters a connection that is eligible for timeout, but has an
unresolved transaction. For this reason, pyfirebirdsql's connection
timeout system offers callbacks that the client programmer can use to
guide the TimeoutThread's actions, or to log information about
connection timeout patterns.


The "Before Timeout" Callback
-----------------------------

The client programmer can supply a "before timeout" callback that
accepts a single dictionary parameter and returns an integer code to
indicate how the TimeoutThread should proceed when it finds a
connection eligible for timeout. Within the dictionary, pyfirebirdsql
provides the following entries:

:dsn:              The `dsn` parameter that was passed to `firebirdsql.connect`
                   when the connection was created.
:has_transaction:  A boolean that indicates whether the connection has an unresolved
                   transaction.
:active_secs:      A `float` that indicates how many seconds elapsed between the point
                   when the connection attached to the server and the last client
                   program activity on the connection.
:idle_secs:        A `float` that indicates how many seconds have elapsed since
                   the last client program activity on the connection. This
                   value will not be less than the specified timeout period, and is
                   likely to only a fraction of a second longer.

Based on those data, the user-supplied callback should return one of
the following codes:

.. data:: CT_VETO

   Directs the TimeoutThread not to close the
   connection at the current time, and not to reconsider timing the
   connection out until at least another timeout period has passed. For
   example, if a connection was created with a timeout period of 120.0
   seconds, and the user-supplied "before callback" returns
   `CT_VETO`, the TimeoutThread will not reconsider timing
   out that particular connection until at least another 120.0 seconds
   have elapsed.

.. data:: CT_NONTRANSPARENT

   ("Nontransparent rollback")

   Directs the TimeoutThread to roll back the connection's unresolved
   transaction (if any), then close the connection. Any future attempt
   to use the connection will raise a :exc:`firebirdsql.ConnectionTimedOut`
   exception.

.. data:: CT_ROLLBACK

   ("Transparent rollback")

   Directs the TimeoutThread to roll back the connection's unresolved transaction
   (if any), then close the connection. Upon any future attempt to use the
   connection, pyfirebirdsql will *attempt* to transparently reconnect to
   the database and "resume where it left off" insofar as possible. Of
   course, network problems and the like could prevent pyfirebirdsql's
   *attempt* at transparent resumption from succeeding. Also, highly
   state-dependent objects such as open result sets, 
   :class:`BlobReader`, and :class:`PreparedStatement`
   cannot be used transparently across a connection timeout.

.. data:: CT_COMMIT

   ("Transparent commit")

   Directs the TimeoutThread to commit the connection's unresolved transaction
   (if any), then close the connection. Upon any future attempt to use the
   connection, pyfirebirdsql will *attempt* to transparently reconnect to
   the database and "resume where it left off" insofar as possible.


If the user does not supply a "before timeout" callback, pyfirebirdsql
considers the timeout transparent only if the connection does not have
an unresolved transaction.

If the user-supplied "before timeout" callback returns anything other
than one of the codes listed above, or if it raises an exception, the
TimeoutThread will act as though :data:`CT_NONTRANSPARENT` had
been returned.

You might have noticed that the input dictionary to the "before
timeout" callback does *not* include a reference to the
:class:`~firebirdsql.Connection` object itself. This is a deliberate design
decision intended to steer the client programmer away from writing
callbacks that take a long time to complete, or that manipulate the
:class:`~firebirdsql.Connection` instance directly. See the caveats section for
more information.


The "After Timeout" Callback
----------------------------

The client programmer can supply an "after timeout" callback that
accepts a single dictionary parameter. Within that dictionary,
pyfirebirdsql currently provides the following entries:

:dsn:          The `dsn` parameter that was passed to :func:`firebirdsql.connect()`
               when the connection was created.
:active_secs:  A `float` that indicates how many seconds elapsed between
               the point when the connection attached to the server and
               the last client program activity on the connection.
:idle_secs:    A `float` that indicates how many seconds elapsed between
               the last client program activity on the connection and the moment
               the TimeoutThread closed the connection.

pyfirebirdsql only calls the "after timeout" callback after the
connection has actually been closed by the TimeoutThread. If the
"before timeout" callback returns :data:`CT_VETO` to cancel the
timeout attempt, the "after timeout" callback will not be called.

pyfirebirdsql discards the return value of the "after timeout" callback,
and ignores any exceptions.

The same caveats that apply to the "before timeout" callback also
apply to the "after timeout" callback.


User-Supplied Connection Timeout Callback Caveats
-------------------------------------------------

+ The user-supplied callbacks are executed by the TimeoutThread. They
  should be designed to avoid blocking the TimeoutThread any longer than
  absolutely necessary.
+ Manipulating the :class:`Connection` object that is being timed out (or any
  of that connection's subordinate objects such as :class:`Cursor`,
  :class:`BlobReader`, or :class:`PreparedStatement`)
  from the timeout callbacks is strictly forbidden.


Examples
--------

**Example: `CT_VETO`**

The following program registers a "before timeout" callback that
unconditionally returns :data:`CT_VETO`, which means that the
TimeoutThread never times the connection out. Although an "after
timeout" callback is also registered, it will never be called.

.. sourcecode:: python

    import time
    import firebirdsql

    def callback_before(info):
        print
        print 'callback_before called; input parameter contained:'
        for key, value in info.items():
            print '  %s: %s' % (repr(key).ljust(20), repr(value))
        print
        # Unconditionally veto any timeout attempts:
        return firebirdsql.CT_VETO

    def callback_after(info):
        assert False, 'This will never be called.'

    con = firebirdsql.connect(dsn=r'localhost:D:\temp\test.db',
        user='sysdba', password='masterkey',
        timeout={
            'period': 3.0,
            'callback_before': callback_before,
            'callback_after':  callback_after,
          }
      )
    cur = con.cursor()

    cur.execute("recreate table test (a int, b char(1))")
    con.commit()

    cur.executemany("insert into test (a, b) values (?, ?)",
        [(1, 'A'), (2, 'B'), (3, 'C')]
      )
    con.commit()

    cur.execute("select * from test")
    print 'BEFORE:', cur.fetchall()

    cur.execute("update test set b = 'X' where a = 2")

    time.sleep(6.0)

    cur.execute("select * from test")
    rows = cur.fetchall()
    # The value of the second column of the second row of the table is still 'X',
    # because the transaction that changed it from 'B' to 'X' remains active.
    assert rows[1][1] == 'X'
    print 'AFTER: ', rows


Sample output:

.. sourcecode:: python

    BEFORE: [(1, 'A'), (2, 'B'), (3, 'C')]

    callback_before called; input parameter contained:
      'dsn'               : 'localhost:D:\\temp\\test.db'
      'idle_secs'         : 3.0
      'has_transaction'   : True

    AFTER:  [(1, 'A'), (2, 'X'), (3, 'C')]



**Example: Supporting Module `timeout_authorizer`**

The example programs for :data:`CT_NONTRANSPARENT`, :data:`CT_ROLLBACK`,
and :data:`CT_COMMIT` rely on the `TimeoutAuthorizer` class from
the module below to guarantee that the TimeoutThread will not time
the connection out before the preparatory code has executed.

.. sourcecode:: python

    import threading
    import firebirdsql

    class TimeoutAuthorizer(object):
        def __init__(self, opCodeWhenAuthorized):
            self.currentOpCode = firebirdsql.CT_VETO
            self.opCodeWhenAuthorized = opCodeWhenAuthorized

            self.lock = threading.Lock()

        def authorize(self):
            self.lock.acquire()
            try:
                self.currentOpCode = self.opCodeWhenAuthorized
            finally:
                self.lock.release()

        def __call__(self, info):
            self.lock.acquire()
            try:
                return self.currentOpCode
            finally:
                self.lock.release()


**Example: `CT_NONTRANSPARENT`**

.. sourcecode:: python

    import threading, time
    import firebirdsql
    import timeout_authorizer

    authorizer = timeout_authorizer.TimeoutAuthorizer(firebirdsql.CT_NONTRANSPARENT)
    connectionTimedOut = threading.Event()

    def callback_after(info):
        print
        print 'The connection was closed nontransparently.'
        print
        connectionTimedOut.set()

    con = firebirdsql.connect(dsn=r'localhost:D:\temp\test.db',
        user='sysdba', password='masterkey',
        timeout={
            'period': 3.0,
            'callback_before': authorizer,
            'callback_after':  callback_after,
          }
      )
    cur = con.cursor()

    cur.execute("recreate table test (a int, b char(1))")
    con.commit()

    cur.executemany("insert into test (a, b) values (?, ?)",
        [(1, 'A'), (2, 'B'), (3, 'C')]
      )
    con.commit()

    cur.execute("select * from test")
    print 'BEFORE:', cur.fetchall()

    cur.execute("update test set b = 'X' where a = 2")

    authorizer.authorize()
    connectionTimedOut.wait()

    # This will raise a firebirdsql.ConnectionTimedOut exception because the
    # before callback returned firebirdsql.CT_NONTRANSPARENT:
    cur.execute("select * from test")


Sample output:

.. sourcecode:: python

    BEFORE: [(1, 'A'), (2, 'B'), (3, 'C')]

    The connection was closed nontransparently.

    Traceback (most recent call last):
      File "connection_timeouts_ct_nontransparent.py", line 42, in ?
        cur.execute("select * from test")
    firebirdsql.ConnectionTimedOut: (0, 'A transaction was still unresolved when
    this connection timed out, so it cannot be transparently reactivated.')


**Example: `CT_ROLLBACK`**

.. sourcecode:: python

    import threading, time
    import firebirdsql
    import timeout_authorizer

    authorizer = timeout_authorizer.TimeoutAuthorizer(firebirdsql.CT_ROLLBACK)
    connectionTimedOut = threading.Event()

    def callback_after(info):
        print
        print 'The unresolved transaction was rolled back; the connection has been'
        print ' closed transparently.'
        print
        connectionTimedOut.set()

    con = firebirdsql.connect(dsn=r'localhost:D:\temp\test.db',
        user='sysdba', password='masterkey',
        timeout={
            'period': 3.0,
            'callback_before': authorizer,
            'callback_after':  callback_after,
          }
      )
    cur = con.cursor()

    cur.execute("recreate table test (a int, b char(1))")
    con.commit()

    cur.executemany("insert into test (a, b) values (?, ?)",
        [(1, 'A'), (2, 'B'), (3, 'C')]
      )
    con.commit()

    cur.execute("select * from test")
    print 'BEFORE:', cur.fetchall()

    cur.execute("update test set b = 'X' where a = 2")

    authorizer.authorize()
    connectionTimedOut.wait()

    # The value of the second column of the second row of the table will have
    # reverted to 'B' when the transaction that changed it to 'X' was rolled back.
    # The cur.execute call on the next line will transparently reactivate the
    # connection, which was timed out transparently.
    cur.execute("select * from test")
    rows = cur.fetchall()
    assert rows[1][1] == 'B'
    print 'AFTER: ', rows


Sample output:

.. sourcecode:: python

    BEFORE: [(1, 'A'), (2, 'B'), (3, 'C')]

    The unresolved transaction was rolled back; the connection has been
     closed transparently.

    AFTER:  [(1, 'A'), (2, 'B'), (3, 'C')]


**Example: `CT_COMMIT`**

.. sourcecode:: python

    import threading, time
    import firebirdsql
    import timeout_authorizer

    authorizer = timeout_authorizer.TimeoutAuthorizer(firebirdsql.CT_COMMIT)
    connectionTimedOut = threading.Event()

    def callback_after(info):
        print
        print 'The unresolved transaction was committed; the connection has been'
        print ' closed transparently.'
        print
        connectionTimedOut.set()

    con = firebirdsql.connect(dsn=r'localhost:D:\temp\test.db',
        user='sysdba', password='masterkey',
        timeout={
            'period': 3.0,
            'callback_before': authorizer,
            'callback_after':  callback_after,
          }
      )
    cur = con.cursor()

    cur.execute("recreate table test (a int, b char(1))")
    con.commit()

    cur.executemany("insert into test (a, b) values (?, ?)",
        [(1, 'A'), (2, 'B'), (3, 'C')]
      )
    con.commit()

    cur.execute("select * from test")
    print 'BEFORE:', cur.fetchall()

    cur.execute("update test set b = 'X' where a = 2")

    authorizer.authorize()
    connectionTimedOut.wait()

    # The modification of the value of the second column of the second row of the
    # table from 'B' to 'X' will have persisted, because the TimeoutThread
    # committed the transaction before it timed the connection out.
    # The cur.execute call on the next line will transparently reactivate the
    # connection, which was timed out transparently.
    cur.execute("select * from test")
    rows = cur.fetchall()
    assert rows[1][1] == 'X'
    print 'AFTER: ', rows


Sample output:

.. sourcecode:: python

    BEFORE: [(1, 'A'), (2, 'B'), (3, 'C')]

    The unresolved transaction was committed; the connection has been
     closed transparently.

    AFTER:  [(1, 'A'), (2, 'X'), (3, 'C')]



Database Event Notification
===========================


What are database events?
-------------------------

The database engine features a distributed, interprocess communication
mechanism based on messages called *database events*. 
A database event is a message passed from a trigger or stored
procedure to an application to announce the occurrence of a specified
condition or action, usually a database change such as an insertion,
modification, or deletion of a record. The Firebird event mechanism
enables applications to respond to actions and database changes made
by other, concurrently running applications without the need for those
applications to communicate directly with one another, and without
incurring the expense of CPU time required for periodic polling to
determine if an event has occurred.


Why use database events?
------------------------

Anything that can be accomplished with database events can also be
implemented using other techniques, so why bother with events? Since
you've chosen to write database-centric programs in Python rather than
assembly language, you probably already know the answer to this
question, but let's illustrate.

A typical application for database events is the handling of
administrative messages. Suppose you have an administrative message
database with a `messages` table, into which various applications
insert timestamped status reports. It may be desirable to react to
these messages in diverse ways, depending on the status they indicate:
to ignore them, to initiate the update of dependent databases upon
their arrival, to forward them by e-mail to a remote administrator, or
even to set off an alarm so that on-site administrators will know a
problem has occurred.

It is undesirable to tightly couple the program whose status is being
reported (the *message producer*) to the program that handles the
status reports (the *message handler*). There are obvious losses of
flexibility in doing so. For example, the message producer may run on
a separate machine from the administrative message database and may
lack access rights to the downstream reporting facilities (e.g.,
network access to the SMTP server, in the case of forwarded e-mail
notifications). Additionally, the actions required to handle status
reports may themselves be time-consuming and error-prone, as in
accessing a remote network to transmit e-mail.

In the absence of database event support, the message handler would
probably be implemented via *polling*. Polling is simply the
repetition of a check for a condition at a specified interval. In this
case, the message handler would check in an infinite loop to see
whether the most recent record in the `messages` table was more recent
than the last message it had handled. If so, it would handle the fresh
message(s); if not, it would go to sleep for a specified interval,
then loop.

The *polling-based* implementation of the message handler is
fundamentally flawed. Polling is a form of `busy-wait
<http://www.catb.org/jargon/html/B/busy-wait.html>`__; the check for
new messages is performed at the specified interval, regardless of the
actual activity level of the message producers. If the polling
interval is lengthy, messages might not be handled within a reasonable
time period after their arrival; if the polling interval is brief, the
message handler program (and there may be many such programs) will
waste a large amount of CPU time on unnecessary checks.

The database server is necessarily aware of the exact moment when a
new message arrives. Why not let the message handler program request
that the database server send it a notification when a new message
arrives? The message handler can then efficiently sleep until the
moment its services are needed. Under this *event-based* scheme, the
message handler becomes aware of new messages at the instant they
arrive, yet it does not waste CPU time checking in vain for new
messages when there are none available.


How events are exposed to the server and the client process?
------------------------------------------------------------

#. Server Process ("An event just occurred!") 

   To notify any interested listeners that a specific event has
   occurred, issue the `POST_EVENT` statement from Stored Procedure
   or Trigger. The `POST_EVENT` statement has one parameter: the name
   of the event to post. In the preceding example of the administrative
   message database, `POST_EVENT` might be used from an `after insert`
   trigger on the `messages` table, like this:

   .. sourcecode:: python

      create trigger trig_messages_handle_insert
        for messages
          after insert
      as
      begin
        POST_EVENT 'new_message';
      end

   .. note:: The physical notification of the client process does not
      occur until the transaction in which the `POST_EVENT` took place is
      actually committed. Therefore, multiple events may *conceptually*
      occur before the client process is *physically* informed of even one
      occurrence. Furthermore, the database engine makes no guarantee that
      clients will be informed of events in the same groupings in which they
      conceptually occurred. If, within a single transaction, an event named
      `event_a` is posted once and an event named `event_b` is posted once,
      the client may receive those posts in separate "batches", despite the
      fact that they occurred in the same conceptual unit (a single
      transaction). This also applies to multiple occurrences of *the same*
      event within a single conceptual unit: the physical notifications may
      arrive at the client separately.

#. Client Process ("Send me a message when an event occurs.")

   .. note:: If you don't care about the gory details of event notification,
             skip to the section that describes pyfirebirdsql's Python-level
             event handling API. 

   The Firebird C client library offers two forms of event notification.
   The first form is *synchronous* notification, by way of the function
   :cfunc:`isc_wait_for_event()`. This form is admirably simple for a C programmer
   to use, but is inappropriate as a basis for pyfirebirdsql's event support,
   chiefly because it's not sophisticated enough to serve as the basis for
   a comfortable Python-level API. The other form of event notification
   offered by the database client library is *asynchronous*, by way of the
   functions :cfunc:`isc_que_events()` (note that the name of that function
   is misspelled), :cfunc:`isc_cancel_events()`, and others. The details are
   as nasty as they are numerous, but the essence of using asynchronous
   notification from C is as follows:

    #. Call :cfunc:`isc_event_block()` to create a formatted binary buffer that
       will tell the server which events the client wants to listen for.
    #. Call :cfunc:`isc_que_events()` (passing the buffer created in the previous
       step) to inform the server that the client is ready to receive event
       notifications, and provide a callback that will be asynchronously
       invoked when one or more of the registered events occurs.
    #. [The thread that called :cfunc:`isc_que_events()` to initiate event
       listening must now do something else.]
    #. When the callback is invoked (the database client library starts a
       thread dedicated to this purpose), it can use the :cfunc:`isc_event_counts()`
       function to determine how many times each of the registered events has
       occurred since the last call to :cfunc:`isc_event_counts()` (if any).
    #. [The callback thread should now "do its thing", which may include
       communicating with the thread that called :cfunc:`isc_que_events()`.]
    #. When the callback thread is finished handling an event
       notification, it must call :cfunc:`isc_que_events()` again in order to receive
       future notifications. Future notifications will invoke the callback
       again, effectively "looping" the callback thread back to Step 4.


How events are exposed to the Python programmer?
------------------------------------------------

The pyfirebirdsql database event API is comprised of the following: the
method `Connection.event_conduit` and the class :class:`EventConduit`.

.. method:: Connection.event_conduit

   Creates a conduit (an instance of :class:`~firebirdsql.EventConduit`)
   through which database event notifications will flow into the Python program.

   `event_conduit` is a method of `Connection` rather than a module-level
   function or a class constructor because the database engine deals with
   events in the context of a particular database (after all,
   `POST_EVENT` must be issued by a stored procedure or a trigger).

   Arguments:

   :event_names: A sequence of string event names The :meth:`EventConduit.wait()`
                 method will block until the occurrence of at least one of the
                 events named by the strings in `event_names`. pyfirebirdsql's
                 own event-related code is capable of operating with up to 2147483647
                 events per conduit. However, it has been observed that the Firebird
                 client library experiences catastrophic problems (including memory
                 corruption) on some platforms with anything beyond about 100 events
                 per conduit. These limitations are dependent on both the Firebird
                 version and the platform.

.. class:: EventConduit

   .. method:: __init__()

      The `EventConduit` class is not designed to be instantiated directly
      by the Python programmer. Instead, use the `Connection.event_conduit`
      method to create `EventConduit` instances.

   .. method:: wait(timeout=None)

      Blocks the calling thread until at least one of the events occurs, or
      the specified `timeout` (if any) expires.

      If one or more event notifications has arrived since the last call to
      `wait`, this method will retrieve a notification from the head of the
      `EventConduit`'s internal queue and return immediately.

      The names of the relevant events were supplied to the
      `Connection.event_conduit` method during the creation of this
      `EventConduit`. In the code snippet below, the relevant events are
      named `event_a` and `event_b`:

      .. sourcecode:: python

         conduit = connection.event_conduit( ('event_a', 'event_b') )
         conduit.wait()


      Arguments:

      :timeout: *optional* number of seconds (use a `float` to
                indicate fractions of seconds) If not even one of the relevant
                events has occurred after `timeout` seconds, this method will
                unblock and return `None`. The default `timeout` is infinite.

      Returns:
        `None` if the wait timed out, otherwise a dictionary that maps
        `event_name -> event_occurrence_count`.

      In the code snippet above, if `event_a` occurred once and `event_b`
      did not occur at all, the return value from `conduit.wait()` would be
      the following dictionary:

      .. sourcecode:: python

         {
          'event_a': 1,
          'event_b': 0
         }

   .. method:: close()

      Cancels the standing request for this conduit to be notified of events.

      After this method has been called, this `EventConduit` object is
      useless, and should be discarded. (The boolean property `closed` is
      `True` after an `EventConduit` has been closed.)

      This method has no arguments.

   .. method:: flush()

      This method allows the Python programmer to manually clear any event
      notifications that have accumulated in the conduit's internal queue.

      From the moment the conduit is created by the
      :meth:`Connection.event_conduit()` method, notifications of any events that
      occur will accumulate asynchronously within the conduit's internal
      queue until the conduit is closed either explicitly (via the `close`
      method) or implicitly (via garbage collection). There are two ways to
      dispose of the accumulated notifications: call `wait()` to receive them
      one at a time ( `wait()` will block when the conduit's internal queue is
      empty), or call this method to get rid of all accumulated
      notifications.

      This method has no arguments.

      Returns:
        The number of event notifications that were flushed from the queue.
        The "number of event *notifications*" is not necessarily the same as
        the "number of event *occurrences*", since a single notification can
        indicate multiple occurrences of a given event (see the return value
        of the `wait` method).


Example Program
---------------

The following code (a SQL table definition, a SQL trigger definition,
and two Python programs) demonstrates pyfirebirdsql-based event
notification.

The example is based on a database at `'localhost:/temp/test.db'`,
which contains a simple table named `test_table`.  `test_table` has
an `after insert` trigger that posts several events. Note that the
trigger posts `test_event_a` twice, `test_event_b` once, and
`test_event_c` once.

The Python event *handler* program connects to the database and
establishes an `EventConduit` in the context of that connection. As
specified by the list of `RELEVANT_EVENTS` passed to `event_conduit`,
the event conduit will concern itself only with events named
`test_event_a` and `test_event_b`. Next, the program calls the
conduit's `wait` method without a timeout; it will wait infinitely
until *at least one* of the relevant events is posted in a transaction
that is subsequently committed.

The Python event *producer* program simply connects to the database,
inserts a row into `test_table`, and commits the transaction. Notice
that except for the printed comment, no code in the producer makes any
mention of events -- the events are posted as an implicit consequence of
the row's insertion into `test_table`.

The insertion into `test_table` causes the trigger to *conceptually*
post events, but those events are not *physically* sent to interested
listeners until the transaction is committed. When the commit occurs,
the handler program returns from the `wait` call and prints the
notification that it received.

SQL table definition:

.. sourcecode:: sql

    create table test_table (a integer)


SQL trigger definition:

.. sourcecode:: sql

    create trigger trig_test_insert_event
      for test_table
        after insert
    as
    begin
      post_event 'test_event_a';
      post_event 'test_event_b';
      post_event 'test_event_c';
    
      post_event 'test_event_a';
    end


Python event *handler* program:

.. sourcecode:: python

    import firebirdsql

    RELEVANT_EVENTS = ['test_event_a', 'test_event_b']

    con = firebirdsql.connect(dsn='localhost:/temp/test.db', user='sysdba', password='pass')
    conduit = con.event_conduit(RELEVANT_EVENTS)

    print 'HANDLER: About to wait for the occurrence of one of %s...\n' % RELEVANT_EVENTS
    result = conduit.wait()
    print 'HANDLER: An event notification has arrived:'
    print result
    conduit.close()


Python event *producer* program:

.. sourcecode:: python

    import firebirdsql

    con = firebirdsql.connect(dsn='localhost:/temp/test.db', user='sysdba', password='pass')
    cur = con.cursor()

    cur.execute("insert into test_table values (1)")
    print 'PRODUCER: Committing transaction that will cause event notification to be sent.'
    con.commit()


Event producer output:

.. sourcecode:: python

    PRODUCER: Committing transaction that will cause event notification to be sent.


Event handler output (assuming that the handler was already started
and waiting when the event producer program was executed):

.. sourcecode:: python

    HANDLER: About to wait for the occurrence of one of ['test_event_a', 'test_event_b']...

    HANDLER: An event notification has arrived:
    {'test_event_a': 2, 'test_event_b': 1}


Notice that there is no mention of `test_event_c` in the result
dictionary received by the event handler program. Although
`test_event_c` was posted by the `after insert` trigger, the event
conduit in the handler program was created to listen only for
`test_event_a` and `test_event_b` events.


Pitfalls and Limitations
------------------------

+ Remember that if an `EventConduit` is left active (not yet closed
  or garbage collected), notifications for any registered events that
  actually occur will continue to accumulate in the EventConduit's
  internal queue even if the Python programmer doesn't call
  :meth:`EventConduit.wait()` to receive the notifications or
  :meth:`EventConduit.flush()` to clear the queue. The ill-informed may
  misinterpret this behavior as a memory leak in pyfirebirdsql; it is not.
+ NEVER use LOCAL-protocol connections in a multithreaded program that
  also uses event handling! The database client library implements the
  local protocol on some platforms in such a way that deadlocks may
  arise in bizarre places if you do this. *This no-LOCAL prohibition is
  not limited to connections that are used as the basis for event
  conduits; it applies to all connections throughout the process.* So
  why doesn't pyfirebirdsql protect the Python programmer from this
  mistake? Because the event handling thread is started by the database
  client library, and it operates beyond the synchronization domain of
  pyfirebirdsql at times.


.. note:: The restrictions on the number of active `EventConduit`s in a
   process, and on the number of event names that a single `EventConduit`
   can listen for, have been removed in pyfirebirdsql 3.2.


The `database_info` API
=======================

Firebird provides various information about server and connected database
via `database_info` API call. pyfirebirdsql surfaces this API through next
methods on Connection object:

.. method:: Connection.database_info(request,result_type)

   This method is a *very thin* wrapper around function :cfunc:`isc_database_info()`.
   This method does *not* attempt to interpret its results except with regard to
   whether they are a string or an integer.

   For example, requesting :cdata:`isc_info_user_names` with the call

   .. sourcecode:: python

      con.database_info(firebirdsql.isc_info_user_names, 's')

   will return a binary string containing a *raw* succession of length-
   name pairs. A more convenient way to access the same functionality is
   via the :meth:`~Connection.db_info()` method.

   Arguments:

   :request:     One of the `firebirdsql.isc_info_*` constants.
   :result_type: Must be either `'s'` if you expect a string result,
                 or `'i'` if you expect an integer result.


   **Example Program**

   .. sourcecode:: python

      import firebirdsql

      con = firebirdsql.connect(dsn='localhost:/temp/test.db', user='sysdba', password='pass')

      # Retrieving an integer info item is quite simple.
      bytesInUse = con.database_info(firebirdsql.isc_info_current_memory, 'i')

      print 'The server is currently using %d bytes of memory.' % bytesInUse

      # Retrieving a string info item is somewhat more involved, because the
      # information is returned in a raw binary buffer that must be parsed
      # according to the rules defined in the Interbase® 6 API Guide section
      # entitled "Requesting buffer items and result buffer values" (page 51).
      #
      # Often, the buffer contains a succession of length-string pairs
      # (one byte telling the length of s, followed by s itself).
      # Function firebirdsql.raw_byte_to_int is provided to convert a raw
      # byte to a Python integer (see examples below).
      buf = con.database_info(firebirdsql.isc_info_db_id, 's')

      # Parse the filename from the buffer.
      beginningOfFilename = 2
      # The second byte in the buffer contains the size of the database filename
      # in bytes.
      lengthOfFilename = firebirdsql.raw_byte_to_int(buf[1])
      filename = buf[beginningOfFilename:beginningOfFilename + lengthOfFilename]

      # Parse the host name from the buffer.
      beginningOfHostName = (beginningOfFilename + lengthOfFilename) + 1
      # The first byte after the end of the database filename contains the size
      # of the host name in bytes.
      lengthOfHostName = firebirdsql.raw_byte_to_int(buf[beginningOfHostName - 1])
      host = buf[beginningOfHostName:beginningOfHostName + lengthOfHostName]

      print 'We are connected to the database at %s on host %s.' % (filename, host)


   Sample output:

   .. sourcecode:: python

      The server is currently using 8931328 bytes of memory.
      We are connected to the database at C:\TEMP\TEST.DB on host WEASEL.

   As you can see, extracting data with the `database_info` function is
   rather clumsy. In pyfirebirdsql 3.2, a higher-level means of accessing
   the same information is available: the :meth:`~Connection.db_info()`
   method. Also, the Services API (accessible to Python programmers via the
   :mod:`firebirdsql.services` module) provides high-level support for
   querying database statistics and performing maintenance.


.. method:: Connection.db_info(request)

   High-level convenience wrapper around the
   :meth:`~Connection.database_info()` method that parses the output
   of `database_info` into Python-friendly objects instead of returning raw binary
   uffers in the case of complex result types. If an unrecognized `isc_info_*` code
   is requested, this method raises `ValueError`.

   For example, requesting :cdata:`isc_info_user_names` with the call

   .. sourcecode:: python

      con.db_info(firebirdsql.isc_info_user_names)

   returns a dictionary that maps (username -> number of open
   connections). If `SYSDBA` has one open connection to the database to
   which `con` is connected, and `TEST_USER_1` has three open connections
   to that same database, the return value would be `{'SYSDBA': 1,
   'TEST_USER_1': 3}`

   Arguments:

   :request:   must be either:

                 + A single `firebirdsql.isc_info_*` info request code.
                   In this case, a single result is returned.
                 + A sequence of such codes. In this case, a mapping of
                   (info request code -> result) is returned.

   **Example Program**

   .. sourcecode:: python

      import os.path

      import firebirdsql

      DB_FILENAME = r'D:\temp\test-20.firebird'
      DSN = 'localhost:' + DB_FILENAME

      ###############################################################################
      # Querying an isc_info_* item that has a complex result:
      ###############################################################################
      # Establish three connections to the test database as TEST_USER_1, and one
      # connection as SYSDBA.  Then use the Connection.db_info method to query the
      # number of attachments by each user to the test database.
      testUserCons = []
      for i in range(3):
          tCon = firebirdsql.connect(dsn=DSN, user='test_user_1', password='pass')
          testUserCons.append(tCon)

      con = firebirdsql.connect(dsn=DSN, user='sysdba', password='masterkey')

      print 'Open connections to this database:'
      print con.db_info(firebirdsql.isc_info_user_names)

      ###############################################################################
      # Querying multiple isc_info_* items at once:
      ###############################################################################
      # Request multiple db_info items at once, specifically the page size of the
      # database and the number of pages currently allocated.  Compare the size
      # computed by that method with the size reported by the file system.
      # The advantages of using db_info instead of the file system to compute
      # database size are:
      #   - db_info works seamlessly on connections to remote databases that reside
      #     in file systems to which the client program lacks access.
      #   - If the database is split across multiple files, db_info includes all of
      #     them.
      res = con.db_info(
          [firebirdsql.isc_info_page_size, firebirdsql.isc_info_allocation]
        )
      pagesAllocated = res[firebirdsql.isc_info_allocation]
      pageSize = res[firebirdsql.isc_info_page_size]
      print '\ndb_info indicates database size is', pageSize * pagesAllocated, 'bytes'
      print   'os.path.getsize indicates size is ', os.path.getsize(DB_FILENAME), 'bytes'


   Sample output:

   .. sourcecode:: python

      Open connections to this database:
      {'SYSDBA': 1, 'TEST_USER_1': 3}

    db_info indicates database size is 20684800 bytes
    os.path.getsize indicates size is  20684800 bytes


Using Firebird Services API
===========================

.. module:: firebirdsql.services
   :synopsis: Access to Firebird Services API

Database server maintenance tasks such as user management, load
monitoring, and database backup have traditionally been automated by
scripting the command-line tools :command:`gbak`, :command:`gfix`, 
:command:`gsec`, and :command:`gstat`.

The API presented to the client programmer by these utilities is
inelegant because they are, after all, command-line tools rather than
native components of the client language. To address this problem,
Firebird has a facility called the Services API, which exposes a uniform
interface to the administrative functionality of the traditional
command-line tools.

The native Services API, though consistent, is much lower-level than a
Pythonic API. If the native version were exposed directly,
accomplishing a given task would probably require more Python code
than scripting the traditional command-line tools. For this reason,
pyfirebirdsql presents its own abstraction over the native API via the
:mod:`firebirdsql.services` module.

Establishing Services API Connections
-------------------------------------

All Services API operations are performed in the context of a
connection to a specific database server, represented by the
:class:`firebirdsql.services.Connection` class. 

.. function:: connect(host='service_mgr', user='sysdba', password=None)

   Establish a connection to database server Services and returns
   :class:`firebirdsql.services.Connection` object.

   :host:      The network name of the computer on which the database
               server is running.
   :user:      The name of the database user under whose authority the
               maintenance tasks are to be performed.
   :password:  User's password.

   Since maintenance operations are most often initiated by an
   administrative user on the same computer as the database server,
   `host` defaults to the local computer, and `user` defaults to
   `SYSDBA`.

   The three calls to `firebirdsql.services.connect()` in the following
   program are equivalent:

   .. sourcecode:: python

      from firebirdsql import services

      con = services.connect(password='masterkey')
      con = services.connect(user='sysdba', password='masterkey')
      con = services.connect(host='localhost', user='sysdba', password='masterkey')

.. class:: Connection

   .. method:: close()

      Explicitly terminates a :class:`~firebirdsql.services.Connection`;
      if this is not invoked, the underlying connection will be closed implicitly
      when the `Connection` object is garbage collected.


Server Configuration and Activity Levels
----------------------------------------


.. method:: Connection.getServiceManagerVersion

   To help client programs adapt to version changes, the service manager
   exposes its version number as an integer.

   .. sourcecode:: python

      from firebirdsql import services
      con = services.connect(host='localhost', user='sysdba', password='masterkey')

      print con.getServiceManagerVersion()

   Output (on Firebird 1.5.0):

   .. sourcecode:: python

      2

   `firebirdsql.services` is a thick wrapper of the Services API that can
   shield its users from changes in the underlying C API, so this method
   is unlikely to be useful to the typical Python client programmer.


.. method:: Connection.getServerVersion()

   Returns the server's version string:

   .. sourcecode:: python

      from firebirdsql import services
      con = services.connect(host='localhost', user='sysdba', password='masterkey')

      print con.getServerVersion()

   Output (on Firebird 1.5.0/Win32):

   .. sourcecode:: python

      WI-V1.5.0.4290 Firebird 1.5

   At first glance, thhis method appears to duplicate the functionality of the
   :attr:`firebirdsql.Connection.server_version` property, but when working
   with Firebird, there is a difference. :attr:`firebirdsql.Connection.server_version`
   is based on a C API call (:cfunc:`isc_database_info()`) that existed long before
   the introduction of the Services API. Some programs written before the advent
   of Firebird test the version number in the return value of :cfunc:`isc_database_info()`,
   and refuse to work if it indicates that the server is too old. Since the first
   stable version of Firebird was labeled `1.0`, this pre-Firebird version testing
   scheme incorrectly concludes that (e.g.) Firebird 1.0 is older than Interbase 5.0.

   Firebird addresses this problem by making :cfunc:`isc_database_info()` return a
   "pseudo-InterBase" version number, whereas the Services API returns
   the true Firebird version, as shown:

   .. sourcecode:: python

      import firebirdsql
      con = firebirdsql.connect(dsn='localhost:C:/temp/test.db', user='sysdba', password='masterkey')
      print 'Interbase-compatible version string:', con.server_version

      import firebirdsql.services
      svcCon = firebirdsql.services.connect(host='localhost', user='sysdba', password='masterkey')
      print 'Actual Firebird version string:     ', svcCon.getServerVersion()

   Output (on Firebird 1.5.0/Win32):

   .. sourcecode:: python

      Interbase-compatible version string: WI-V6.3.0.4290 Firebird 1.5
      Actual Firebird version string:      WI-V1.5.0.4290 Firebird 1.5


.. method:: Connection.getArchitecture()

   Returns platform information for the server, including hardware architecture
   and operating system family.

   .. sourcecode:: python

      from firebirdsql import services
      con = services.connect(host='localhost', user='sysdba', password='masterkey')

      print con.getArchitecture()

   Output (on Firebird 1.5.0/Windows 2000):

   .. sourcecode:: python

      Firebird/x86/Windows NT

   Unfortunately, the architecture string is almost useless because its
   format is irregular and sometimes outright idiotic, as with Firebird
   1.5.0 running on x86 Linux:

   .. sourcecode:: python

      Firebird/linux Intel

   Magically, Linux becomes a hardware architecture, the ASCII store
   decides to hold a 31.92% off sale, and Intel grabs an unfilled niche
   in the operating system market.


.. method:: Connection.getHomeDir()

   Returns the equivalent of the `RootDirectory` setting from :file:`firebird.conf`:

   .. sourcecode:: python

      from firebirdsql import services
      con = services.connect(host='localhost', user='sysdba', password='masterkey')

      print con.getHomeDir()

   Output (on a particular Firebird 1.5.0/Windows 2000 installation):

   .. sourcecode:: python

      C:\dev\db\firebird150\

   Output (on a particular Firebird 1.5.0/Linux installation):

   .. sourcecode:: python

      /opt/firebird/


.. method:: Connection.getSecurityDatabasePath()

   Returns the location of the server's core security database, which contains
   user definitions and such. Interbase® and Firebird 1.0 named this database
   :file:`isc4.gdb`, while in Firebird 1.5 it's renamed to :file:`security.fdb`
   and to :file:`security2.fdb` in Firebird 2.0 and later.

   .. sourcecode:: python

      from firebirdsql import services
      con = services.connect(host='localhost', user='sysdba', password='masterkey')

      print con.getSecurityDatabasePath()

   Output (on a particular Firebird 1.5.0/Windows 2000 installation):

   .. sourcecode:: python

      C:\dev\db\firebird150\security.fdb

   Output (on a particular Firebird 1.5.0/Linux installation):

   .. sourcecode:: python

      /opt/firebird/security.fdb


.. method:: Connection.getLockFileDir()

   The database engine `uses a lock file
   <http://www.ibphoenix.com/resources/documents/general/doc_48>`__ to
   coordinate interprocess communication; `getLockFileDir()` returns the
   directory in which that file resides:

   .. sourcecode:: python

      from firebirdsql import services
      con = services.connect(host='localhost', user='sysdba', password='masterkey')

      print con.getLockFileDir()

   Output (on a particular Firebird 1.5.0/Windows 2000 installation):

   .. sourcecode:: python

      C:\dev\db\firebird150\

   Output (on a particular Firebird 1.5.0/Linux installation):

   .. sourcecode:: python

      /opt/firebird/


.. method:: Connection.getCapabilityMask()

   The Services API offers "a bitmask representing the capabilities
   currently enabled on the server", but the only availabledocumentation
   for this bitmask suggests that it is "reserved for future implementation".
   firebirdsql exposes this bitmask as a Python `int` returned from the
   `getCapabilityMask()` method.


.. method:: Connection.getMessageFileDir()

   To support internationalized error messages/prompts, the database
   engine stores its messages in a file named :file:`interbase.msg`
   (Interbase® and Firebird 1.0) or :file:`firebird.msg` (Firebird 1.5 and
   later). The directory in which this file resides can be determined
   with the `getMessageFileDir()` method.

   .. sourcecode:: python

      from firebirdsql import services
      con = services.connect(host='localhost', user='sysdba', password='masterkey')

      print con.getMessageFileDir()

   Output (on a particular Firebird 1.5.0/Windows 2000 installation):

   .. sourcecode:: python

      C:\dev\db\firebird150\

   Output (on a particular Firebird 1.5.0/Linux installation):

   .. sourcecode:: python

      /opt/firebird/


.. method:: Connection.getConnectionCount()

   Returns the number of active connections to databases managed by the server.
   This count only includes *database* connections (such as open instances of
   :class:`firebirdsql.Connection`), not *services manager* connections (such
   as open instances of :class:`firebirdsql.services.Connection`).

   .. sourcecode:: python

      import firebirdsql, firebirdsql.services
      svcCon = firebirdsql.services.connect(host='localhost', user='sysdba', password='masterkey')

      print 'A:', svcCon.getConnectionCount()

      con1 = firebirdsql.connect(dsn='localhost:C:/temp/test.db', user='sysdba', password='masterkey')
      print 'B:', svcCon.getConnectionCount()

      con2 = firebirdsql.connect(dsn='localhost:C:/temp/test.db', user='sysdba', password='masterkey')
      print 'C:', svcCon.getConnectionCount()

      con1.close()
      print 'D:', svcCon.getConnectionCount()

      con2.close()
      print 'E:', svcCon.getConnectionCount()

   On an otherwise inactive server, the example program generates the
   following output:

   .. sourcecode:: python

      A: 0
      B: 1
      C: 2
      D: 1
      E: 0


.. method:: Connection.getAttachedDatabaseNames()

   Returns a list of the names of all databases to which the server is
   maintaining at least one connection. The database names are not guaranteed
   to be in any particular order.

   .. sourcecode:: python

      import firebirdsql, firebirdsql.services
      svcCon = firebirdsql.services.connect(host='localhost', user='sysdba', password='masterkey')

      print 'A:', svcCon.getAttachedDatabaseNames()

      con1 = firebirdsql.connect(dsn='localhost:C:/temp/test.db', user='sysdba', password='masterkey')
      print 'B:', svcCon.getAttachedDatabaseNames()

      con2 = firebirdsql.connect(dsn='localhost:C:/temp/test2.db', user='sysdba', password='masterkey')
      print 'C:', svcCon.getAttachedDatabaseNames()

      con3 = firebirdsql.connect(dsn='localhost:C:/temp/test2.db', user='sysdba', password='masterkey')
      print 'D:', svcCon.getAttachedDatabaseNames()

      con1.close()
      print 'E:', svcCon.getAttachedDatabaseNames()

      con2.close()
      print 'F:', svcCon.getAttachedDatabaseNames()

      con3.close()
      print 'G:', svcCon.getAttachedDatabaseNames()

   On an otherwise inactive server, the example program generates the
   following output:

   .. sourcecode:: python

      A: []
      B: ['C:\\TEMP\\TEST.DB']
      C: ['C:\\TEMP\\TEST2.DB', 'C:\\TEMP\\TEST.DB']
      D: ['C:\\TEMP\\TEST2.DB', 'C:\\TEMP\\TEST.DB']
      E: ['C:\\TEMP\\TEST2.DB']
      F: ['C:\\TEMP\\TEST2.DB']
      G: []


.. method:: Connection.getLog()

   Returns the contents of the server's log file (named :file:`interbase.log`
   by Interbase® and Firebird 1.0; :file:`firebird.log` by Firebird 1.5 and later):

   .. sourcecode:: python

      from firebirdsql import services
      con = services.connect(host='localhost', user='sysdba', password='masterkey')

      print con.getLog()

   Output (on a particular Firebird 1.5.0/Windows 2000 installation):

   .. sourcecode:: python

      WEASEL (Client) Thu Jun 03 12:01:35 2004
        INET/inet_error: send errno = 10054

      WEASEL (Client) Sun Jun 06 19:21:17 2004
        INET/inet_error: connect errno = 10061



Database Statistics
-------------------

.. method:: Connection.getStatistics( database, showOnlyDatabaseLogPages=0...)

   Returns a string containing a printout in the same format as the output
   of the :command:`gstat` command-line utility. This method has one required
   parameter, the location of the database on which to compute statistics,
   and five optional boolean parameters for controlling the domain of the
   statistics.

   **Map of gstat parameters to getStatistics options**

   =========================== ====================================
   `gstat` command-line option `getStatistics` boolean parameter
   =========================== ====================================
   -header                     showOnlyDatabaseHeaderPages
   -log                        showOnlyDatabaseLogPages
   -data                       showUserDataPages
   -index                      showUserIndexPages
   -system                     showSystemTablesAndIndexes
   =========================== ====================================

   The following program presents several `getStatistics` calls and their
   `gstat`-command-line equivalents. In this context, output is
   considered "equivalent" even if their are some whitespace differences.
   When collecting textual output from the Services API, firebirdsql
   terminates lines with `\n` regardless of the platform's convention;
   `gstat` is platform-sensitive.

   .. sourcecode:: python

      from firebirdsql import services
      con = services.connect(user='sysdba', password='masterkey')

      # Equivalent to 'gstat -u sysdba -p masterkey C:/temp/test.db':
      print con.getStatistics('C:/temp/test.db')

      # Equivalent to 'gstat -u sysdba -p masterkey -header C:/temp/test.db':
      print con.getStatistics('C:/temp/test.db', showOnlyDatabaseHeaderPages=True)

      # Equivalent to 'gstat -u sysdba -p masterkey -log C:/temp/test.db':
      print con.getStatistics('C:/temp/test.db', showOnlyDatabaseLogPages=True)

      # Equivalent to 'gstat -u sysdba -p masterkey -data -index -system C:/temp/test.db':
      print con.getStatistics('C:/temp/test.db',
          showUserDataPages=True,
          showUserIndexPages=True,
          showSystemTablesAndIndexes=True
        )

   The output of the example program is not shown here because it is
   quite long.


Backup and Restoration
----------------------

pyfirebirdsql offers convenient programmatic control over database
backup and restoration via the `backup` and `restore` methods.

At the time of this writing, released versions of Firebird/Interbase®
do not implement incremental backup, so we can simplistically define
*backup* as the process of generating and storing an archived replica
of a live database, and *restoration* as the inverse. The
backup/restoration process exposes numerous parameters, which are
properly documented in Firebird Documentation to :command:`gbak`.
The pyfirebirdsql API to these parameters is presented with minimal
documentation in the sample code below.

.. method:: Connection.backup(sourceDatabase, destFilenames, destFileSizes=(), <options>)

   Creates a backup file from database content.

   **Simple Form**

   The simplest form of `backup` creates a single backup file that
   contains everything in the database. Although the extension `'.fbk'`
   is conventional, it is not required.

   .. sourcecode:: python

      from firebirdsql import services
      con = services.connect(user='sysdba', password='masterkey')

      backupLog = con.backup('C:/temp/test.db', 'C:/temp/test_backup.fbk')
      print backupLog

   In the example, `backupLog` is a string containing a `gbak`-style log
   of the backup process. It is too long to reproduce here.

   Although the return value of the `backup` method is a freeform log
   string, `backup` will raise an exception if there is an error. For
   example:

   .. sourcecode:: python

      from firebirdsql import services
      con = services.connect(user='sysdba', password='masterkey')

      # Pass an invalid backup path to the engine:
      backupLog = con.backup('C:/temp/test.db', 'BOGUS/PATH/test_backup.fbk')
      print backupLog

   .. sourcecode:: python

      Traceback (most recent call last):
        File "adv_services_backup_simplest_witherror.py", line 5, in ?
          backupLog = con.backup('C:/temp/test.db', 'BOGUS/PATH/test_backup.fbk')
        File "C:\code\projects\firebirdsql\Kinterbasdb-3.0\build\lib.win32-2.3\firebirdsql\services.py", line 269, in backup
          return self._actAndReturnTextualResults(request)
        File "C:\code\projects\firebirdsql\Kinterbasdb-3.0\build\lib.win32-2.3\firebirdsql\services.py", line 613, in _actAndReturnTextualResults
          self._act(requestBuffer)
        File "C:\code\projects\firebirdsql\Kinterbasdb-3.0\build\lib.win32-2.3\firebirdsql\services.py", line 610, in _act
          return _ksrv.action_thin(self._C_conn, requestBuffer.render())
      firebirdsql.OperationalError: (-902, '_kiservices could not perform the action: cannot open backup file BOGUS/PATH/test_backup.fbk. ')

   **Multifile Form**

   The database engine has built-in support for splitting the backup into
   multiple files, which is useful for circumventing operating system
   file size limits or spreading the backup across multiple discs.

   pyfirebirdsql exposes this facility via the `Connection.backup`
   parameters `destFilenames` and `destFileSizes`. `destFilenames` (the
   second positional parameter of `Connection.backup`) can be either a
   string (as in the example above, when creating the backup as a single
   file) or a sequence of strings naming each constituent file of the
   backup. If `destFilenames` is a string-sequence with length `N`,
   `destFileSizes` must be a sequence of integer file sizes (in bytes)
   with length `N-1`. The database engine will constrain the size of each
   backup constituent file named in `destFilenames[:-1]` to the
   corresponding size specified in `destFileSizes`; any remaining backup
   data will be placed in the file name by `destFilenames[-1]`.

   Unfortunately, the database engine does not appear to expose any
   convenient means of calculating the total size of a database backup
   before its creation. The page size of the database and the number of
   pages in the database are available via 
   :meth:`~firebirdsql.Connection.database_info()` calls:
   `database_info(firebirdsql.isc_info_page_size, 'i')` and 
   `database_info(firebirdsql.isc_info_db_size_in_pages, 'i')`,
   respectively, but the size of the backup file is usually smaller than
   the size of the database.

   There *should* be no harm in submitting too many constituent
   specifications; the engine will write an empty header record into the
   excess constituents. However, at the time of this writing, released
   versions of the database engine hang the backup task if more than 11
   constituents are specified (that is, if `len(destFilenames) > 11`).
   pyfirebirdsql does not prevent the programmer from submitting more than
   11 constituents, but it does issue a warning.

   The following program directs the engine to split the backup of the
   database at :file:`C:/temp/test.db` into :file:`C:/temp/back01.fbk`,
   a file 4096 bytes in size, :file:`C:/temp/back02.fbk`, a file 16384 bytes
   in size, and :file:`C:/temp/back03.fbk`, a file containing the remainder of
   the backup data.

   .. sourcecode:: python

      from firebirdsql import services
      con = services.connect(user='sysdba', password='masterkey')

      con.backup('C:/temp/test.db',
         ('C:/temp/back01.fbk', 'C:/temp/back02.fbk', 'C:/temp/back03.fbk'),
          destFileSizes=(4096, 16384)
        )

   **Extended Options**

   In addition to the three parameters documented previously (positional
   `sourceDatabase`, positional `destFilenames`, and keyword
   `destFileSizes`), the `Connection.backup` method accepts six boolean
   parameters that control aspects of the backup process and the backup
   file output format. These options are well documented so in this
   document we present only a table of equivalence between :command:`gbak`
   options and names of the boolean keyword parameters:

   ============== ===================================== =================
   `gbak` option  Parameter Name                        Default Value 
   ============== ===================================== =================
   -T             transportable                         True
   -M             metadataOnly                          False
   -G             garbageCollect                        True
   -L             ignoreLimboTransactions               False
   -IG            ignoreChecksums                       False
   -CO            convertExternalTablesToInternalTables True
   ============== ===================================== =================


.. method:: Connection.restore(sourceFilenames, destFilenames, destFilePages=(), <options>)

   Restores database from backup file.

   **Simplest Form**

   The simplest form of `restore` creates a single-file database,
   regardless of whether the backup data were split across multiple 
   files.

   .. sourcecode:: python

      from firebirdsql import services
      con = services.connect(user='sysdba', password='masterkey')

      restoreLog = con.restore('C:/temp/test_backup.fbk', 'C:/temp/test_restored.db')
      print restoreLog

   In the example, `restoreLog` is a string containing a `gbak`-style log
   of the restoration process. It is too long to reproduce here.

   **Multifile Form**

   The database engine has built-in support for splitting the restored
   database into multiple files, which is useful for circumventing
   operating system file size limits or spreading the database across
   multiple discs.

   pyfirebirdsql exposes this facility via the `Connection.restore`
   parameters `destFilenames` and `destFilePages`. `destFilenames` (the
   second positional argument of `Connection.restore`) can be either a
   string (as in the example above, when restoring to a single database
   file) or a sequence of strings naming each constituent file of the
   restored database. If `destFilenames` is a string-sequence with length
   `N`, `destFilePages` must be a sequence of integers with length `N-1`.
   The database engine will constrain the size of each database
   constituent file named in `destFilenames[:-1]` to the corresponding
   page count specified in `destFilePages`; any remaining database pages
   will be placed in the file name by `destFilenames[-1]`.

   The following program directs the engine to restore the backup file at
   :file:`C:/temp/test_backup.fbk` into a database with three constituent
   files: :file:`C:/temp/test_restored01.db`, :file:`C:/temp/test_restored02.db`,
   and :file:`C:/temp/test_restored03.db`. The engine is instructed to place
   fifty user data pages in the first file, seventy in the second, and
   the remainder in the third file. In practice, the first database
   constituent file will be larger than `pageSize*destFilePages[0]`,
   because metadata pages must also be stored in the first constituent of
   a multifile database.

   .. sourcecode:: python

      from firebirdsql import services
      con = services.connect(user='sysdba', password='masterkey')

      con.restore('C:/temp/test_backup.fbk',
          ('C:/temp/test_restored01.db', 'C:/temp/test_restored02.db', 'C:/temp/test_restored03.db'),
          destFilePages=(50, 70),
          pageSize=1024,
          replace=True
        )

   **Extended Options**

   These options are well documented so in this document we present only
   a table of equivalence between the :command:`gbak` options
   and the names of the keyword parameters to `Connection.restore`:

   ============== ===================================== ====================
   `gbak` option  Parameter Name                        Default Value 
   ============== ===================================== ====================
   -P             pageSize                              [use server default]
   -REP           replace                               False
   -O             commitAfterEachTable                  False
   -K             doNotRestoreShadows                   False
   -I             deactivateIndexes                     False
   -N             doNotEnforceConstraints               False
   -USE           useAllPageSpace                       False
   -MO            accessModeReadOnly                    False
   -BU            cacheBuffers                          [use server default]
   ============== ===================================== ====================



Database Operating Modes, Sweeps, and Repair
--------------------------------------------

.. method:: Connection.sweep(database, markOutdatedRecordsAsFreeSpace=1)

   Not yet documented.

.. method:: Connection.setSweepInterval(database, n)

   Not yet documented.

.. method:: Connection.setDefaultPageBuffers(database, n)

   Not yet documented.

.. method:: Connection.setShouldReservePageSpace(database, shouldReserve)

   Not yet documented.

.. method:: Connection.setWriteMode(database, mode)

   Not yet documented.

.. method:: Connection.setAccessMode(database, mode)

   Not yet documented.

.. method:: Connection.activateShadowFile(database)

   Not yet documented.

.. method:: Connection.shutdown(database, shutdownMethod, timeout)

   Not yet documented.

.. method:: Connection.bringOnline(database)

   Not yet documented.

.. method:: Connection.getLimboTransactionIDs(database)

   Not yet documented.

.. method:: Connection.commitLimboTransaction(database, transactionID)

   Not yet documented.

.. method:: Connection.rollbackLimboTransaction(database, transactionID)

   Not yet documented.

.. method:: Connection.repair(database, <options>)

   Not yet documented.



User Maintenance
----------------

.. method:: Connection.getUsers(username=None)

   By default, lists all users.

.. method:: Connection.addUser(user)

   :user:  An instance of :class:`User` with *at least* its username
           and password attributes specified as non-empty values.

.. method:: Connection.modifyUser(user)

   Changes user data.

   :user:  An instance of :class:`User` with *at least* its username
           and password attributes specified as non-empty values.

.. method:: Connection.removeUser(user)

   Accepts either an instance of services.User or a string username,
   and deletes the specified user.

.. method:: Connection.userExists(user)

   Returns a boolean that indicates whether the specified user exists.


.. class:: User

   Not yet documented.
