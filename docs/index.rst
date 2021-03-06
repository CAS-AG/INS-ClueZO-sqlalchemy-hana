sqlalchemy-hana documentation
=============================

.. toctree::
   :maxdepth: 2
   :caption: Contents:


This dialect allows you to use the SAP HANA database with SQLAlchemy.
It can use the supported SAP HANA Python Driver `hdbcli` (supported since SAP HANA SPS 2) or the open-source pure Python client `PyHDB`. Please notice that sqlalchemy-hana is not an official SAP product and is not covered by SAP support.

Prerequisites
-------------

Python 2.7 or Python 3 with installed SAP HANA DBAPI implementation.

SAP HANA Python Driver see `SAP HANA Client Interface Programming Reference <https://help.sap.com/viewer/0eec0d68141541d1b07893a39944924e/2.0.02/en-US/39eca89d94ca464ca52385ad50fc7dea.html>`_ or the install section of `PyHDB <https://github.com/SAP/PyHDB>`_.

Install
-------

Install from Python Package Index:

.. code-block:: bash

    $ pip install sqlalchemy-hana

You can also install the latest version directly from a cloned git repository.

.. code-block:: bash

    $ git clone https://github.com/SAP/sqlalchemy-hana.git
    $ cd sqlalchemy-hana
    $ python setup.py install


Getting started
---------------

If you do not have access to a SAP HANA server, you can also use the `SAP HANA Express edition <https://www.sap.com/developer/topics/sap-hana-express.html>`_.

After installation of sqlalchemy-hana, you can create an engine which connects to a SAP HANA instance. This engine works like all other engines of SQLAlchemy.

.. code-block:: python

    from sqlalchemy import create_engine
    engine = create_engine('hana://username:password@example.de:30015')

By default the ``hana://`` schema will use hdbcli (from the SAP HANA Client) as the underlying database driver.
To use PyHDB as driver use ``hana+pyhdb://`` as schema in your DBURI.

In case of a tenant database, you may use:

.. code-block:: python

    from sqlalchemy import create_engine
    engine = engine = create_engine('hana://user:pass@host/tenant_db_name')

Special CREATE TABLE argument 
-----------------------------

Sqlalchemy-hana provides a special argument called ???hana_table_type??? which can be used to specify the type of table one wants to create with SAP HANA (i.e. ROW/COLUMN). The default table type depends on your HANA configuration.

.. code-block:: python

    t = Table('mytable', metadata, Column('id', Integer), hana_table_type = 'COLUMN')
    t.create(engine)

Case Sensitivity
----------------

In SAP HANA, all case insensitive identi???ers are represented using uppercase text. In SQLAlchemy on the other hand all lower case identi???er names are considered to be case insensitive. The sqlalchemy-hana dialect converts all case insensitive and case sensitive identi???ers to the right casing during schema level communication. In the sqlalchemy-hana dialect, using an uppercase name on the SQLAlchemy side indicates a case sensitive identi???er, and SQLAlchemy will quote the name,which may cause case mismatches between data received from SAP HANA. Unless identi???er names have been truly created as case sensitive (i.e. using quoted names), all lowercase names should be used on the SQLAlchemy side.

Auto Increment Behavior 
-----------------------

SQLAlchemy Table objects which include integer primary keys are usually assumed to have ???auto incrementing??? behavior, which means that primary key values can be automatically generated upon INSERT. Since SAP HANA has no ???autoincrement??? feature, SQLAlchemy relies upon sequences to automatically generate primary key values. 

Sequences
"""""""""

With the sqlalchemy-hana dialect, a sequence must be explicitly speci???ed to enable autoincrementing behaviour. This is di???erent than the majority of SQLAlchemy documentation examples which assume the usage of an autoincrement-capable database. To create sequences, use the sqlalchemy.schema.Sequence object which is passed to a Column construct.

.. code-block:: python

    t = Table('mytable', metadata, Column('id', Integer, Sequence('id_seq'), primary key=True)) 
    t.create(engine)

IDENTITY Feature 
""""""""""""""""

SAP HANA also comes with an option to have an IDENTITY column which can also be used to create new primary key values for integer-based primary key columns. Built-in support for rendering of IDENTITY is not available yet, however the following compilation hook may be used to make use of the IDENTITY feature.

.. code-block:: python

    from sqlalchemy.schema import CreateColumn 
    from sqlalchemy.ext.compiler import compiles

    @compiles(CreateColumn, 'hana') 
    def use_identity(element, compiler, **kw): 
        text = compiler.visit_create_column(element, **kw)
        text = text.replace('NOT NULL', 'NOT NULL GENERATED BY DEFAULT AS IDENTITY') 
        return text 

    t = Table('t', meta, Column('id', Integer, primary_key=True), Column('data', String)) 

    t.create(engine)

LIMIT/OFFSET Support 
--------------------

SAP HANA supports both LIMIT and OFFSET, but it only supports OFFSET in conjunction with LIMIT i.e. in the select statement the o???set parameter cannot be set without the LIMIT clause, hence in sqlalchemy-hana if the user tries to use o???set without limit, a limit of 2147384648 would be set, this has been done so that the users can smoothly use LIMIT/OFFSET as in other databases that do not have this limitation. 2147384648 was chosen, because it is the maximum number of records per result set.

RETURNING Support 
-----------------

Sqlalchemy-hana does not support RETURNING in the INSERT, UPDATE and DELETE statements to retrieve result sets of matched rows from INSERT, UPDATE and DELETE statements because newly generated primary key values are neither fetched nor returned automatically in SAP HANA and SAP HANA does not support the syntax: INSERT... RETURNING...

Constraint Re???ection  
--------------------

The sqlalchemy-hana dialect can return information about foreign keys, unique constraints and check constraints, as well as table indexes. 

Raw information regarding these constraints can be acquired using:

* Inspector.get_foreign_keys()
* Inspector.get_unique_constraints() 
* Inspector.get_check_constraints()
* Inspector.get_indexes()

When using re???ection at the Table level, the Table will also include these constraints.

Foreign key Options 
"""""""""""""""""""

In SAP HANA the following UPDATE and DELETE foreign key referential actions are available: 

* RESTRICT
* CASCADE
* SET NULL
* SET DEFAULT

The foreign key referential option NO ACTION does not exist in SAP HANA. The default is RESTRICT.

CHECK Constraints 
"""""""""""""""""

Table level check constraints can be created as well as re???ected in the sqlalchemy-hana dialect. Column level check constraints are not available in SAP HANA.

.. code-block:: python

    t = Table('mytable', metadata, Column('id', Integer), Column(...), ..., CheckConstraint('id >5', name='my_check_const'))

UNIQUE Constraints 
""""""""""""""""""

For each unique constraint an index is created in SAP HANA, this may lead to unexpected behaviour in programs using re???ection. 

Special Re???ection Option 
------------------------

The Inspector used for the SAP HANA database is an instance of HANA Inspector and o???ers an additional method.

get_table_oid(table name, schema=None)
""""""""""""""""""""""""""""""""""""""

Return the OID(object id) for the given table name.

.. code-block:: python

    from sqlalchemy import create_engine, inspect

    engine = create_engine("hana://username:password@example.de:30015") 
    insp = inspect(engine) 
    # will be a HANAInspector 
    print(insp.get_table_oid('mytable'))

Data types 
----------

As with all SQLAlchemy dialects, all UPPERCASE types that are known to be valid with SAP HANA are importable from the top level dialect, whether they originate from sqlalchemy.types or from the local dialect. The allowed datatypes for sqlalchemy-hana are as follows:

BOOLEAN, NUMERIC, NVARCHAR, CLOB, BLOB, DATE, TIME, TIMESTAMP, CHAR, VARCHAR, VARBINARY, BIGINT, SMALLINT, INTEGER, FLOAT, TEXT

DateTime Compatibility  
""""""""""""""""""""""

SAP HANA has no data type known as DATETIME, it instead has the datatype TIMESTAMP, which can actually store the date and time value. For this reason, the sqlalchemy-hana dialect provides a type sqlalchemy-hana.TIMESTAMP which is a subclass of DateTime. 

NUMERIC Compatibility  
"""""""""""""""""""""

SAP HANA does not have a data type known as NUMERIC, hence if a user has a column with data type numeric while using sqlalchemy-hana, it is stored as DECIMAL data type instead. 

TEXT datatype  
"""""""""""""

SAP HANA only supports the datatype TEXT for column tables. It is not a valid data type for row tables. Hence, one must mention hana_table_type=???COLUMN??? 

Bound Parameter Styles
----------------------

The default parameter style for the sqlalchemy-hana dialect is ???qmark???, where SQL is rendered using the following style:

WHERE my_column = ?

Contribute
----------

If you found bugs or have other issues, you are welcome to create a GitHub Issue.
If you have questions about usage or something similar please create a
`Stack Overflow <http://stackoverflow.com/>`_ question with tag
`sqlalchemy <http://stackoverflow.com/questions/tagged/sqlalchemy>`_ and
`hana <http://stackoverflow.com/questions/tagged/hana>`_.


Indices and tables
==================

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`
