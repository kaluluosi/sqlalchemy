.. _dialect_toplevel:

方言
====

**方言**是SQLAlchemy用来与各种类型的  :term:`DBAPI`  实现和数据库通信的系统。
以下部分包含针对使用每个后端的参考文档和注意事项以及各种DBAPI的注意事项。

所有方言都需要安装适当的DBAPI驱动程序。

.. _included_dialects:

包含的方言
----------

.. toctree::
    :maxdepth: 1
    :glob:

    postgresql
    mysql
    sqlite
    oracle
    mssql

包含的方言的支持级别
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

下表总结了每个包含方言的支持级别。

.. dialect-table:: **包含方言支持的数据库版本**
  :header-rows: 1

支持级别定义
^^^^^^^^^^^^

.. glossary::

    CI全面测试
        CI全面测试表示在SQLAlchemy CI系统中测试的版本，
        并通过测试套件的所有测试。

    正常支持
        正常支持表示大多数功能应该可以工作，
        但不是所有版本都在ci配置中测试，因此可能存在某些不受支持的边缘情况。我们将尝试修复影响这些版本的问题。

    最佳努力
        最佳努力表示我们尝试在这些上支持基本功能，
        但在某些用例中很可能存在不支持的功能或错误。
        可能会接受与相关问题的拉取请求以继续支持旧版本，这将根据情况进行审查。

.. _external_toplevel:

外部方言
---------

目前由SQLAlchemy维护的外部方言项目包括:

+------------------------------------------------+---------------------------------------+
| 数据库                                         | 方言                                  |
+================================================+=======================================+
| Actian Avalanche，Vector，Actian X和Ingres   | sqlalchemy-ingres_                    |
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
| IBM Netezza Performance Server [1]_            | nzalchemy_                            |
+------------------------------------------------+---------------------------------------+
| Microsoft Access（通过pyodbc）                 | sqlalchemy-access_                    |
+------------------------------------------------+---------------------------------------+
| Microsoft SQL Server（通过python-tds）         | sqlalchemy-tds_                       |
+------------------------------------------------+---------------------------------------+
| Microsoft SQL Server（通过turbodbc）           | sqlalchemy-turbodbc_                  |
+------------------------------------------------+---------------------------------------+
| MonetDB [1]_                                   | sqlalchemy-monetdb_                   |
+------------------------------------------------+---------------------------------------+
| OpenGauss                                      | openGauss-sqlalchemy_                 |
+------------------------------------------------+---------------------------------------+
| Rockset                                        | rockset-sqlalchemy_                   |
+------------------------------------------------+---------------------------------------+
| SAP ASE（前Sybase方言的分支）                  | sqlalchemy-sybase_                    |
+------------------------------------------------+---------------------------------------+
| SAP Hana [1]_                                  | sqlalchemy-hana_                      |
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