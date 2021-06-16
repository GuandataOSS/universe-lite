# CHANGELOG


## 0.11.0 (release at 2021-06-16)

0.11.x works with duckdb 0.2.7

changes:

-   fix a "path cannot recognized" bug on Windows


## 0.10.1 (release at 2021-05-13)

0.10.x works with duckdb 0.2.6

changes:

-   fix Java Plugin issue: check error when the input count is variable
-   fix some class loader issue for Java Plugins
-   also print column's type info in "stdout" stage type
-   upgrade commons-compress library to newer version to solve class conflict in Java Plugin when using Apache POI
-   add new stage type "spark\_shell", to integrate Spark SQL into workflow (Only local mode spark is supported currently)


## 0.10.0 (release at 2021-05-09)

0.10.x works with duckdb 0.2.6


## 0.9.0 (release at 2021-03-29)

0.9.x works with duckdb 0.2.5


## 0.8.1 (release at 2021-02-24)

0.8.1 works with duckdb 0.2.4

-   add experimental support for StringTemplate4 (st), to easy creation of string parameter values
-   "file" task support a new param "content", to write string content directly into file
-   "python" task & "python\_plugin" task will open duckdb in readonly mode to avoid duckdb open db error. <https://github.com/cwida/duckdb/issues/1365>
-   make log4j to write to file together with console. (native-image mode don't support write log to file yet, due to graalvm's issue)
-   "file" task support to write file as ".avro" file, together with previous supported ".csv" & ".parquet"
-   add experimental support for "osquery" task, by using command line tool: <https://github.com/osquery/osquery>
-   "file" task support to merge multiple parquet files into one parquet file (by "merge\_path" parameter)

Limitations:

-   duckdb sql task, if sql contains "IN" and registered as "view" (persist=false), duckdb 0.2.4 will crash. <https://github.com/cwida/duckdb/issues/1427> (already fixed in duckdb's master branch)
-   only write log file in universe-lite Jar, but not in universe-lite native-image


## 0.8.0 (release at 2021-02-03)

initial release