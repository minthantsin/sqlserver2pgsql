mssql2pgsql
===========

This is a migration tool to convert a Microsoft SQL Server Database into a PostgreSQL database, as automatically as possible.

It is written in Perl.


It does two things:

  * convert a SQL Server schema to a PostgreSQL schema

  * produce a Pentaho Data Integrator (Kettle) job to migrate 
    all the data from SQL Server to PostgreSQL. This second part is optionnal


Notes, warnings:
==========================
This tool will never be completely finished. For now, it works with all the SQL Server
databases I had to migrate. If it doesn't work with yours, feel free to modify this,
send me patches, or the SQL dump from your SQL Server Database, with the
problem you are facing. I'll try to improve the code, but I need this SQL dump.

It won't migrate PL procedures, the languages are too different.

I usually only test this script under Linux. It should work on Windows, as I had to do it once
with Windows, and on any Unix system.

Installation:
==========================
Nothing to do on this part, as long as you don't want to use the data migration part: the script
has no dependancy on fancy Perl modules, it only uses modules from the base perl distribution.

Just run the script with your perl interpreter, providing it with the requested options (--help
will tell you what to do).

If you want to migrate the data, you'll need "Kettle", an Open Source ETL. Get the latest version
from here: http://kettle.pentaho.com/ .
You'll also need a SQL Server account with the permission to SELECT from the tables you want to migrate.

As Kettle is a Java program, you'll also need a recent JVM (Java 6 or 7 should do the trick).

==========================

Ok, I have installed Kettle and Java, I have mssql2pg.pl, what do I do now ?

You'll need several things:

  * The connection parameters to the SQL Server database:
    IP address, port, username, password, database name
  * Access to an empty PostgreSQL database (where you want your migrated data)
  * A text file containing a SQL dump of the SQL Server database

To get this SQL Dump, follow this procedure, in SQL Server's management interface:
  * Under SQL Server Management Studio, Right click on the database you want to export
  * Select Tasks/Generate Scripts
  * Click "Next" on the welcome screen (if it hasn't already been desactivated)
  * Select your database
  * In the list of things you can export, just change "Script Indexes" from False to True, then "Next"
  * Select Tables then "Next"
  * Select the tables you want to export (or select all), then "Next"
  * Script to file, choose a filename, then "Next"
  * Select unicode encoding (who knows…, maybe someone has put accents in objects names, or in comments)
  * Finish

You'll get a file containing a SQL script. Get it on the server you'll want to
run mssql2pg.pl from.

If you just want to convert this schema, run:

> mssql2pg -f my_sqlserver_script.txt -b name_of_before_script -a name_of_after_script

The before script contains what is needed to import data (types, tables and columns).
The after script contains the rest (indexes, constraints). It should be run
after data is imported.

You can also use the -i option:

-i  : Generate an "ignore case" schema, using citext, to emulate MSSQL's case insensitive collation.
      It will create citext fields, with check constraints.

If you want to also import data:

> ./mssql2pgsql.pl -b before.sql -a after.sql -k kettledir \ 
  -sd source -sh 192.168.0.2 -sp 1433 -su dalibo -sw mysqlpass \
  -pd dest -ph localhost -pp 5432 -pu dalibo -pw mypgpass -f sql_server_schema.sql

-k is the directory where you want to store the kettle xml files (there will be
one for each table to copy, plus the one for the job)

You'll also need to specify the connection parameters. They will be stored inside the kettle files (in
cleartext, so don't make this directory public):
-sd : sql server database
-sh : sql server host
-sp : sql server port (usually 1433)
-su : sql server username
-sw : sql server password
-pd : postgresql database
-ph : postgresql host
-pp : postgresql port
-pu : postgresql username
-pw : postgresql password
-f  : the SQL Server structure dump file

You've generated everything. Let's do the import:

# Run the before script (creates the tables)
> psql -U mypguser mypgdatabase -f name_of_before_script
# Run the kettle job:
> cd my_kettle_installation_directory
> ./kitchen.sh -file=full_path_to_kettle_job_dir/migration.kjb -level=detailed
# Run the after script (creates the indexes, constraints...)
> psql -U mypguser mypgdatabase -f name_of_after_script

If you want to dig deeper into the kettle job, you can use kettle_report.pl to display the individual table's transfer performance. Then, if needed, you'll be able to modify the Kettle job to optimize it, using Spoon, Kettle's GUI

================================
FAQ:

Why didn't you do everything in the Perl script ? I don't want to use Kettle.

Because I have once installed a Perl DBD driver for MS SQL's server, and I don't want to ever do it again... nor 
force you to do it :) You can either do it using ODBC, or using Sybase's driver. In both case, it will take you 
quite a while, and you'll suffer with large objects. With Kettle, on the other, you're using the JBDC driver, 
which is already one of Kettle's default drivers, along with PostgreSQL. So all the heavy lifting (converting 
LOBs and IMAGES and whatever to bytea) is directly done by both JDBC drivers, and should work with no efforts 
(except maybe adjust Java's memory parameters). If you get a memory error, try setting a JAVAMAXMEM environment 
variable to a higher value (4096) for 4GB for instance.



What it this IGNORE NULLS I have to change in kettle.properties ?

Because Kettle behaves by default the way Oracle behaves: for Oracle, a NULL and an empty string are the same. 
You don't want this here, because neither SQL Server nor PostgreSQL do this.