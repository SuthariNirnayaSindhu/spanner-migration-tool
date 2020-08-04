# HarbourBridge: Turnkey Spanner Evaluation

[![cloudspannerecosystem](https://circleci.com/gh/cloudspannerecosystem/harbourbridge.svg?style=svg)](https://circleci.com/gh/cloudspannerecosystem/harbourbridge)

HarbourBridge is a stand-alone open source tool for Cloud Spanner evaluation,
using data from an existing PostgreSQL or MySQL database. The tool ingests schema
and data from either a pg_dump/mysqldump file or directly from the source database,
automatically builds a Spanner schema, and creates a new Spanner database populated
with data from the source database.

HarbourBridge is designed to simplify Spanner evaluation, and in particular to
bootstrap the process by getting moderate-size PostgreSQL/MySQL datasets into Spanner
(up to a few tens of GB). Many features of PostgreSQL/MySQL, especially those that don't map
directly to Spanner features, are ignored, e.g. (non-primary) indexes,
functions, and sequences. Types such as integers, floats, char/text, bools,
timestamps, and (some) array types, map fairly directly to Spanner, but many
other types do not and instead are mapped to Spanner's `STRING(MAX)`.

View HarbourBridge as a way to get up and running fast, so you can focus on
critical things like tuning performance and getting the most out of
Spanner. Expect that you'll need to tweak and enhance what HarbourBridge
produces to complete your evaluation. For example, while HarbourBridge preserves
primary keys, it does not currently translate other indexes. So, you'll need to
add [Spanner secondary
indexes](https://cloud.google.com/spanner/docs/secondary-indexes) if your SQL
queries rely on PostgreSQL/MySQL indexes that have been dropped. HarbourBridge is not
intended for production database migration.

To use the tool on a PostgreSQL database called mydb, run

```sh
# By default, the driver is "pg_dump".
pg_dump mydb | harbourbridge
# Or,
pg_dump mydb | harbourbridge -driver=pg_dump
```

To use the tool on a MySQL database called mydb, run

```sh
mysqldump mydb | harbourbridge -driver=mysqldump
```

HarbourBridge accepts pg_dump/mysqldump's standard plain-text format, but not archive or
custom formats. More details on usage can be found in [Example usage](#example-usage) section.

HarbourBridge automatically determines the cloud project and Spanner instance to
use, and generates a new Spanner database name (prefixed with `{driver}_` and
today's date). Command-line flags can be used to explicitly set the Spanner
instance or database name. See [Options](#options).

**WARNING: Please check that permissions for the Spanner instance used by
HarbourBridge are appropriate. Spanner manages access control at the database
level, and the database created by HarbourBridge will inherit default
permissions from the instance. All data written by HarbourBridge is visible to
anyone who can access the created database.**

As it processes the PostgreSQL/MySQL data, HarbourBridge reports on progress, provides
stats on the schema and data conversion steps, and an overall assessment of the
quality of the conversion. It also generates a schema file and report file (and
a bad-data file if data was dropped). See
[Files Generated by HarbourBridge](#files-generated-by-harbourbridge). Details
of how source database's schema is mapped to Spanner can be found in the
[Schema Conversion](#schema-conversion) section.

This tool is part of the Cloud Spanner Ecosystem, a community contributed and
supported open source repository. Please [report
issues](https://github.com/cloudspannerecosystem/harbourbridge/issues) and send
pull requests. See the [HarbourBridge
Whitepaper](https://github.com/cloudspannerecosystem/harbourbridge/blob/master/whitepaper.md)
for a discussion of our plans for the tool.

Note that the HarbourBridge tool is not an officially supported Google product
and is not officially supported as part of the Cloud Spanner product.

## Quickstart Guide

### Before you begin

Complete the steps described in
[Set up](https://cloud.google.com/spanner/docs/getting-started/set-up), which
covers creating and setting a default Google Cloud project, enabling billing,
enabling the Cloud Spanner API, and setting up OAuth 2.0 to get authentication
credentials to use the Cloud Spanner API.

In particular, ensure that you run

```sh
gcloud auth application-default login
```

to set up your local development environment with authentication credentials.

Set the GCLOUD_PROJECT environment variable to your Google Cloud project ID:

```sh
export GCLOUD_PROJECT=[MY_PROJECT_ID]
```

If you do not already have a Cloud Spanner instance, or you want to use a
separate instance specifically for running HarbourBridge, then create a Cloud
Spanner instance by following the "Create an instance" instructions on the
[Quickstart using the console](https://cloud.google.com/spanner/docs/quickstart-console)
guide. HarbourBridge will create a database for you, but it will not create a
Spanner instance.

Install Go ([download](https://golang.org/doc/install)) on your development
machine if it is not already installed, configure the GOPATH environment
variable if it is not already configured, and
[test your installation](https://golang.org/doc/install#testing).

### Installing HarbourBridge

Download the tool to your machine and install it.

```sh
GO111MODULE=on go get github.com/cloudspannerecosystem/harbourbridge
```

The tool should now be installed as `$GOPATH/bin/harbourbridge`.
See the [Troubleshooting Guide](#troubleshooting-guide) if you run
into any issues.

### Running HarbourBridge

To use the tool on a PostgreSQL database called mydb, run

```sh
pg_dump mydb | $GOPATH/bin/harbourbridge -driver=pg_dump
```

To use the tool on a MySQL database called mydb, run

```sh
mysqldump mydb | $GOPATH/bin/harbourbridge -driver=mysqldump
```

More details on running harbourbridge can be found in [Example usage](#example-usage) section.

This command will use the cloud project specified by the `GCLOUD_PROJECT`
environment variable, automatically determine the Cloud Spanner instance
associated with this project, convert the PostgreSQL/MySQL schema for `mydb` to a
Spanner schema, create a new Cloud Spanner database with this schema, and
finally, populate this new database with the data from `mydb`. If project has
multiple instances, then list of available instances will be shown and you will have to
pick one of the available instances and set the `--instance` flag. The new Cloud
Spanner database will have a name of the form `{DRIVER}_{DATE}_{RANDOM}`, where`{DRIVER}`
is the drivername specified,`{DATE}`is today's date, and`{RANDOM}` is a random suffix for
uniqueness.

See the [Troubleshooting Guide](#troubleshooting-guide) for help on debugging
issues.

HarbourBridge also [generates several files](#files-generated-by-harbourbridge)
when it runs: a schema file, a report file (with detailed analysis of the
conversion), and a bad data file (if any data was dropped).

### Verifying Results

Once the tool has completed, you can verify the new database and its content
using the Google Cloud Console. Go to the [Cloud Spanner Instances
page](https://console.cloud.google.com/spanner/instances), select your Spanner
instance, and then find the database created by HarbourBridge and select
it. This will list the tables created by HarbourBridge. Select a table, and take
a look at its schema and data. Next, go to the query page, and try
some SQL statements. For example

```
SELECT COUNT(*) from mytable
```

to check the number of rows in table `mytable`.

### Next Steps

The tables created by HarbourBridge provide a starting point for evaluation of
Spanner. While they preserve much of the core structure of your PostgreSQL/MySQL
schema and data, many key features have been dropped, including (non-primary)
indexes, functions, sequences, procedures, triggers, and views.

As a result, the out-of-the-box performance you get from these tables could be
slower than what you get from PostgreSQL/MySQL. HarbourBridge does preserve primary
keys, but we do not currently translate other indexes. If your SQL query
performance relies on PostgreSQL/MySQL indexes that are dropped, then the performance
of the tables created by HarbourBridge could be significantly impaired.

To improve performance, consider adding [Secondary
Indexes](https://cloud.google.com/spanner/docs/secondary-indexes) to the tables
created by HarbourBridge, using the existing PostgreSQL/MySQL indexes as a guide. Also
consider using [Interleaved
Tables](https://cloud.google.com/spanner/docs/schema-and-data-model#creating-interleaved-tables)
to tune performance.

View HarbourBridge as a base set of functionality for Spanner evalution that can
be readily expanded. Consider forking and modifying the codebase to add the
functionality you need. Please [file
issues](https://github.com/cloudspannerecosystem/harbourbridge/issues) and send
PRs for fixes and new functionality. See our backlog of [open
issues](https://github.com/cloudspannerecosystem/harbourbridge/issues). Our
plans and aspirations for developing HarbourBridge further are outlined in the
[HarbourBridge
Whitepaper](https://github.com/cloudspannerecosystem/harbourbridge/blob/master/whitepaper.md).

You can also change the way HarbourBridge behaves by directly editing the
pg_dump/mysqldump output. For example, suppose you want to try out different primary keys
for a table. First run pg_dump/mysqldump and save the output to a file. Then modify (or
add) the relevant `ALTER TABLE ... ADD CONSTRAINT ... PRIMARY KEY ...` statement
in the pg_dump/mysqldump output file so that the primary keys match what you need. Then
run HarbourBridge on the modified pg_dump/mysqldump output.

## Files Generated by HarbourBridge

HarbourBridge generates several files as it runs:

- Schema file (ending in `schema.txt`): contains the generated Spanner
  schema, interspersed with comments that cross-reference to the relevant
  PostgreSQL/MySQL schema definitions.

- Report file (ending in `report.txt`) contains a detailed analysis of the
  PostgreSQL/MySQL to Spanner migration, including table-by-table stats and an
  analysis of PostgreSQL/MySQL types that don't cleanly map onto Spanner types. Note
  that PostgreSQL/MySQL types that don't have a corresponding Spanner type are
  mapped to STRING(MAX).

- Bad data file (ending in `dropped.txt`): contains details of data
  that could not be converted and written to Spanner, including sample
  bad-data rows. If there is no bad-data, this file is not written (and we
  delete any existing file with the same name from a previous run).

By default, these files are prefixed by the name of the Spanner database (with a
dot separator). The file prefix can be overridden using the `-prefix`
[option](#options).

## Options

HarbourBridge accepts the following options:

`-dbname` Specifies the name of the Spanner database to create. This must be a
new database. If dbname is not specified, HarbourBridge creates a new unique
dbname.

`-instance` Specifies the Spanner instance to use. The new database will be
created in this instance. If not specified, the tool automatically determines an
appropriate instance using gcloud.

`-driver` Specifies the driver to use for schema and data conversion. Supported drivers
are _'postgres'_, _'pg_dump'_, _'mysql'_ and _'mysqldump'_. By default, the driver is _'pg_dump'_.

`-prefix` Specifies a file prefix for the report, schema, and bad-data files
written by the tool. If no file prefix is specified, the name of the Spanner
database (plus a '.') is used.

`-v` Specifies verbose mode. This will cause HarbourBridge to output detailed
messages about the conversion.

## Example Usage

Details on HarbourBridge example usage for PostgreSQL and MySQL can be
found in the [PostgreSQL example usage](postgres/README.md#example-postgresql-usage)
and [MySQL example usage](mysql/README.md#example-mysql-usage) sections respectively.

## Schema Conversion

Details on HarbourBridge schema conversion for PostgreSQL and MySQL can be
found in the [PostgreSQL schema conversion](postgres/README.md#schema-conversion)
and [MySQL schema conversion](mysql/README.md#schema-conversion) sections respectively.

## Data Conversion

HarbourBridge converts PostgreSQL/MySQL data to Spanner data based on the Spanner
schema it constructs. Conversion for most data types is fairly straightforward,
but several types deserve discussion. Note that HarbourBridge is not intended
for databases larger than few tens of GB. Details on HarbourBridge data conversion
for PostgreSQL and MySQL can be found in the
[PostgreSQL data conversion](postgres/README.md#data-conversion)
and [MySQL data conversion](mysql/README.md#data-conversion) sections respectively.

## Troubleshooting Guide

HarbourBridge is written using the Go module system, and so it must be
installed in [module-aware mode](https://golang.org/cmd/go/#hdr-Module_support).
This can be achived by setting the environment variable `GO111MODULE` to `on`
before calling "go get" e.g.

```sh
GO111MODULE=on go get github.com/cloudspannerecosystem/harbourbridge
```

As an alternative to "go get", you can make a copy of the HarbourBridge
codebase from the github repository and use "go run":

```sh
git clone https://github.com/cloudspannerecosystem/harbourbridge
cd harbourbridge
pg_dump mydb | go run github.com/cloudspannerecosystem/harbourbridge -driver=pg_dump

```

This workflow also allows you to modify or customize the HarbourBridge codebase.

The following steps can help diagnose common issues encountered while running
HarbourBridge.

### 1. Verify driver configuration

First, check that the driver is correctly configured to connect to your
database. Driver configuration varies depending on the driver used.

#### 1.1 pg_dump

If you are using pg_dump (-driver=pg_dump), check that pg_dump is
correctly configured to connect to your PostgreSQL
database. Note that pg_dump uses the same options as psql to connect to your
database. See the [psql](https://www.postgresql.org/docs/9.3/app-psql.html) and
[pg_dump](https://www.postgresql.org/docs/9.3/app-pgdump.html) documentation.

Access to a PostgreSQL database is typically configured using the
_PGHOST_, _PGPORT_, _PGUSER_, _PGDATABASE_ environment variables,
which are standard across PostgreSQL utilities.

It is also possible to configure access via pg_dump's command-line options
`--host`, `--port`, and `--username`.

#### 1.2 Direct access to PostgreSQL

In this case, HarbourBridge connects directly to the PostgreSQL
database to retrieve table schema and data using the setting of
standard _PGHOST_, _PGPORT_, _PGUSER_, _PGDATABASE_ environment
variables. Password can be specified either in the _PGPASSWORD_
environment variable or provided at the password prompt.

#### 1.3 mysqldump

If you are using mysqldump (-driver=mysqldump), check that mysqldump is
correctly configured to connect to your MySQL
database via the command-line options `--host`, `--port`, and `--user`.
Note that mysqldump uses the same options as mysql to connect to your
database. See the [mysql](https://dev.mysql.com/doc/refman/8.0/en/mysql-commands.html) and
[mysqldump](https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html) documentation.

#### 1.4 Direct access to MySQL

In this case, HarbourBridge connects directly to the MySQL
database to retrieve table schema and data using the setting of
_MYSQLHOST_, _MYSQLPORT_, _MYSQLUSER_, _MYSQLDATABASE_ environment
variables. Password can be specified either in the _MYSQLPWD_
environment variable or provided at the password prompt.

### 2. Verify dump output

Next, verify that pg_dump/mysqldump is generating plain-text output. If your database is
small, try running

```sh
{ pg_dump/mysqldump } > file
```

and look at the output file. It should be a plain-text file containing SQL
commands. If your database is large, consider just dumping the schema via the
`--schema-only` for pg_dump and `--no-data` for mysqldump command-line option.

pg_dump/mysqldump can export data in a variety of formats, but HarbourBridge only accepts
`plain` format (aka plain-text). See the
[pg_dump documentation](https://www.postgresql.org/docs/9.3/app-pgdump.html) and
[mysqldump documentation](https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html) for details about formats.

### 3. Debugging HarbourBridge

The HarbourBridge tool can fail for a number of reasons.

#### 3.1 No space left on device

HarbourBridge needs to read the pg_dump/mysqldump output twice, once to build a schema and
once for data ingestion. When pg_dump/mysqldump output is directly piped to HarbourBridge,
`stdin` is not seekable, and so we write the output to a temporary file. That
temporary file is created via Go's ioutil.TempFile. On many systems, this
creates a file in `/tmp`, which is sometimes configured with minimal space. A
simple workaround is to separately run pg_dump/mysqldump and write its output to a file in
a directory with sufficient space. For example, if the current working
directory has space, then:

```sh
{ pg_dump/mysqldump } > tmpfile
harbourbridge < tmpfile
```

Make sure you cleanup the tmpfile after HarbourBridge has been run. Another
option is to set the location of Go's TempFile e.g. by setting the `TMPDIR`
environment variable.

#### 3.2 Unparsable dump output

HarbourBridge uses the [pg_query_go](https://github.com/lfittl/pg_query_go)
library for parsing pg_dump and [pingcap parser](https://github.com/pingcap/parser) for parsing mysqldump.
It is possible that the pg_dump/mysqldump output is corrupted or uses features
that aren't parseable. Parsing errors should generate an error message of the
form `Error parsing last 54321 line(s) of input`.

#### 3.2 Credentials problems

HarbourBridge uses standard Google Cloud credential mechanisms for accessing
Cloud Spanner. If this is mis-configured, you may see errors containing
"unauthenticated", or "cannot fetch token", or "could not find default
credentials". You might need to run `gcloud auth application-default login`. See
the [Before you begin](#before-you-being) section for details.

#### 3.4 Can't create database

In this case, the error message printed by the tool should help identify the
cause. It could be an API permissions issue. For example, the Cloud Spanner API
may not be appropriately configured. See [Before you begin](#before-you-being)
section for details. Alternatively, you have have hit the limit on the number of
databases per instances (currently 100). This can occur if you re-run the
HarbourBridge tool many times, since each run creates a new database. In this
case you'll need to [delete some
databases](https://cloud.google.com/spanner/docs/getting-started/go/#delete_the_database).

### 4. Database-Specific Issues

The schema, report, and bad-data files [generated by
HarbourBridge](#files-generated-by-harbourbridge) contain detailed information
about the schema and data conversion process, including issues and problems
encountered.

### 5. Reporting Issues

If you are having problems with HarbourBridge, please [submit an
issue](https://github.com/cloudspannerecosystem/harbourbridge/issues).
