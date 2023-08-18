.. _dialect_toplevel:

方言
========

**方言**是SQLAlchemy用来与不同类型的:term:`DBAPI`实现和数据库进行通信的系统。
接下来的部分包含了与每个后端使用相关的参考文献和注释，以及
有关各种DBAPI的注释。

所有方言均需要安装适当的DBAPI驱动程序。

.. _included_dialects:

包含的方言
-----------------

.. toctree::
    :maxdepth: 1
    :glob:

    postgresql
    mysql
    sqlite
    oracle
    mssql

所包含方言的支持级别
~~~~~~~~~~~~~~~~~~~~~~~~

下表总结了每个所包含方言的支持级别。

.. dialect-table:: **所包含方言支持的数据库版本**
  :header-rows: 1

支持级别定义
~~~~~~~~~~~~~~~~~

.. glossary::

    在CI中完全测试
        **在CI中完全测试**表示在SQLAlchemy
        CI系统中测试并通过测试套件中的所有测试的版本。

    普通支持
        **普通支持**表示大多数功能应该可以正常工作，
        但不是所有版本都在ci配置中测试，因此可能会有一些不受支持的边缘情况。我们将尝试修复影响这些版本的问题。

    尽力而为
        **尽力而为**表示我们尝试在基本功能上支持它们，
        但在某些用例中很可能存在未支持的功能或错误。
        可以通过相关问题的拉取请求进行接受，以继续支持旧版本，
        这些版本将逐个进行审查。

.. _external_toplevel:

外部方言
-----------------

当前由SQLAlchemy维护的外部方言项目包括：

+------------------------------------------------+---------------------------------------+
| 数据库                                         | 方言                                  |
+================================================+=======================================+
| Actian Avalanche、Vector、Actian X和Ingres     | sqlalchemy-ingres_                    |
+------------------------------------------------+---------------------------------------+
| Amazon Athena                                  | pyathena_                             |
+------------------------------------------------+---------------------------------------+
| Amazon Redshift（通过psycopg2）                | sqlalchemy-redshift_                  |
+------------------------------------------------+---------------------------------------+
| Apache Drill                                   | sqlalchemy-drill_                     |
+------------------------------------------------+---------------------------------------+
| Apache Druid                                   | pydruid_                              |
+------------------------------------------------+---------------------------------------+
| Apache Hive和Presto                            | PyHive_                               |
+------------------------------------------------+---------------------------------------+
| Apache Solr                                    | sqlalchemy-solr_                      |
+------------------------------------------------+---------------------------------------+
| CockroachDB                                    | sqlalchemy-cockroachdb_               |
+------------------------------------------------+---------------------------------------+
| CrateDB                                        | crate-python_                         |
+------------------------------------------------+---------------------------------------+
| EXASolution                                    | sqlalchemy_exasol_                    |
+------------------------------------------------+---------------------------------------+
| Elasticsearch（只读）                          | elasticsearch-dbapi_                  |
+------------------------------------------------+---------------------------------------+
| Firebird                                       | sqlalchemy-firebird_                  |
+------------------------------------------------+---------------------------------------+
| Firebolt                                       | firebolt-sqlalchemy_                  |
+------------------------------------------------+---------------------------------------+
| Google BigQuery                                | pybigquery_                           |
+------------------------------------------------+---------------------------------------+
| Google Sheets                                  | gsheets_                              |
+------------------------------------------------+---------------------------------------+
| IBM DB2和Informix                              | ibm-db-sa_                            |
+------------------------------------------------+---------------------------------------+
| IBM Netezza性能服务器[1]_                     | nzalchemy_                            |
+------------------------------------------------+---------------------------------------+
| Microsoft Access（通过pyodbc）                 | sqlalchemy-access_                    |
+------------------------------------------------+---------------------------------------+
| Microsoft SQL Server（通过python-tds）         | sqlalchemy-tds_                       |
+------------------------------------------------+---------------------------------------+
| Microsoft SQL Server（通过turbodbc）           | sqlalchemy-turbodbc_                  |
+------------------------------------------------+---------------------------------------+
| MonetDB[1]_                                    | sqlalchemy-monetdb_                   |
+------------------------------------------------+---------------------------------------+
| OpenGauss                                      | openGauss-sqlalchemy_                 |
+------------------------------------------------+---------------------------------------+
| Rockset                                        | rockset-sqlalchemy_                   |
+------------------------------------------------+---------------------------------------+
| SAP ASE（前Sybase方言分支）                     | sqlalchemy-sybase_                    |
+------------------------------------------------+---------------------------------------+
| SAP Hana[1]_                                   | sqlalchemy-hana_                      |
+------------------------------------------------+---------------------------------------+
| SAP Sybase SQL Anywhere                        | sqlalchemy-sqlany_                    |
+------------------------------------------------+---------------------------------------+
| Snowflake                                      | snowflake-sqlalchemy_                 |
+------------------------------------------------+---------------------------------------+
| Teradata Vantage                               | teradatasqlalchemy_                   |
+------------------------------------------------+---------------------------------------+

.. [1] 目前仅支持版本1.3.x。

.. _openGauss-sqlalchemy: https://gitee.com/opengauss/openGauss-sqlalchemy
.. _rockset-sqlalchemy: https://pypi.org/project/rockset-sqlalchemy
.. _sqlalchemy-ingres: https://github.com/clach04/ingres_sa_dialect
.. _nzalchemy: https://pypi.org/project/nzalchemy/
.. _ibm-db-sa: https://pypi.org/project/ibm-db-sa/
.. _PyHive: https://github.com/dropbox/PyHive#sqlalchemy
.. _teradatasqlalchemy: https://pypi.org/project/teradatasqlalchemy/
.. _pybigquery: https://github.com/mxmzdlv/pybigquery/
.. _sqlalchemy-redshift: https://pypi.org/project/sqlalchemy-redshift
.. _sqlalchemy-drill: https://github.com/JohnOmernik/sqlalchemy-drill
.. _sqlalchemy-hana: https://github.com/SAP/sqlalchemy-hana
.. _sqlalchemy-solr: https://github.com/aadel/sqlalchemy-solr
.. _sqlalchemy_exasol: https://github.com/blue-yonder/sqlalchemy_exasol
.. _sqlalchemy-sqlany: https://github.com/sqlanywhere/sqlalchemy-sqlany
.. _sqlalchemy-monetdb: https://github.com/gijzelaerr/sqlalchemy-monetdb
.. _snowflake-sqlalchemy: https://github.com/snowflakedb/snowflake-sqlalchemy
.. _sqlalchemy-tds: https://github.com/m32/sqlalchemy-tds
.. _crate-python: https://github.com/crate/crate-python
.. _sqlalchemy-access: https://pypi.org/project/sqlalchemy-access/
.. _elasticsearch-dbapi: https://github.com/preset-io/elasticsearch-dbapi/
.. _pydruid: https://github.com/druid-io/pydruid
.. _gsheets: https://github.com/betodealmeida/gsheets-db-api
.. _sqlalchemy-firebird: https://github.com/pauldex/sqlalchemy-firebird
.. _sqlalchemy-cockroachdb: https://github.com/cockroachdb/sqlalchemy-cockroachdb
.. _sqlalchemy-turbodbc: https://pypi.org/project/sqlalchemy-turbodbc/
.. _sqlalchemy-sybase: https://pypi.org/project/sqlalchemy-sybase/
.. _firebolt-sqlalchemy: https://pypi.org/project/firebolt-sqlalchemy/