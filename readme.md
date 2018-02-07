# qfind

Finds files using a SQL query then displays the output in an interactive ncurses view. qfind works with current file system data so there is no need for indexing or to run special daemons.

## Details

When running a query such as:...

```
qfind "qfind "select path, size, modifieds from files where ext = 'txt'"
```

qfind starts by running the find command to get an overall list of files from the current location. The output of find is then piped to TermSQL which runs the query to filter and transform the list of files. The output of TermSQL is then displayed in TabView where the results can be further filtered, sorted, searched etc.

So what does qfind actually do if the dependencies do all the work? qfind walks through the query to identify field names, these field names are then converted into printf format strings that can be used by the find command. It also passes the field names to TermSQL.

qfind also includes some handy computed fields which get expanded to SQL and can be run by TermSQL/SQLite. To see which find and TermSQL commands qfind would generate for a query run:

```
qfind -sf "<sql-query>"
qfind -st "<sql-query>"
```

#### The dependencies:

[The find command](http://man7.org/linux/man-pages/man1/find.1.html)

- Installed by default in on most *nix systems.

[TermSQL](https://github.com/tobimensch/termsql)

- Runs SQL queries on delimited text.

[TabView](https://github.com/TabViewer/tabview)

- An ncurses viewer of any tabular information. It allows post-query filtering and sorting. It also includes a search function.

[sqlparse](https://github.com/andialbrecht/sqlparse)

- SQL parsing library.

## Star fields

When running a `select *` query with qfind, '*' doesn't represent all available fields but instead a smaller subset known as the star fields. The default set of star fields contains the same fields that the `ls -l` command would return:

permissions hardlinks user groupname size modified type path

Other star fields can be set in the config file (see below)

Run `qfind -s` to list all available fields (star fields are marked with star characters):

## Computed fields

## Command line arguments

```
usage: qfind [-h] [-d [DELIMITER]] [-of] [-o] [-sf] [-st] [-s]
             [-t [EXTRA_TERMSQL_ARGS]] [-f [EXTRA_FIND_ARGS]]
             [start_dir] [query]

Search for files using a SQL query

positional arguments:

start_dir
Starting dir. If not specified then the current dir is used.

query
The file search SQL query

optional arguments:

-h, --help
show this help message and exit

-d [DELIMITER], --delimiter [DELIMITER]
A singe char used to delimit the output of the find command (the default is pipe)

-of, --output-find
Output the results of find command without piping to termsql.

-o, --output
Output results to stdout instead of displaying with tabview

-sf, --show-find-command
Show the generated find command string without running the query

-st, --show-termsql-command
Show the generated termsql command string without
running the query

-s, --show-all-fields
Show all possible fields that can be used in the query

-t [EXTRA_TERMSQL_ARGS], --extra-termsql-args [EXTRA_TERMSQL_ARGS]
Specify additional arguments to termsql. The set of termsql args must be quoted and have a space between the first quote and the first arg eg: qfind -o -t ' -m html' ...

-f [EXTRA_FIND_ARGS], --extra-find-args [EXTRA_FIND_ARGS]
Supply additional arguments to the find command. The set of find args must be quoted and have a space between the first quote and the first arg eg: qfind -f " -iname '*.txt'" ...

'SELECT *' queries return only the starfields which is a subset of all
possible fields that the 'find' command can produce. To see a complete list of
all fields, run: 'qfind -s'
```

## The config file

Either...

```
~/.config/qfind/qfind.config
```

...or...

```
~/.qfind/qfind.config
```

...or...

```
~/.qfind.config
```

The config file use INI file structure. The file starts with a section called 'qfind'.

Example:

```
[qfind]
StarFields=permissions hardlinks user groupname size modified type dirs name ext
Delimiter=
ExtraTermsqlArgs=
```

## Speeding up the query

qfind can be sped up by selecting only a few specific columns, such as name or path, instead of `select *`, especially if the star fields expand to many fields, the find command will slow down as it will have to stat a lot more info about each file. `select *` queries will also run a bit slower in the TermSQL/SQLite phase.

Supply extra args to find to pre-filter the resultset before runnig the SQL e.g.:

```
qfind -f " -iname '*.txt'" "select * from files" # TermsSQL will only have to run on ".txt" files
```

If I do a later version then I might attempt automatic "preoptimizion", that is to generate an more optimal find command for the SQL query. This will required rewriting the preparse function and will mean adding a lot of complexity.

## Examples

Select all files recursively from the current directory

```
qfind "select * from files"
```

...or search in a different dir...

```
qfind ~/Documents "select * from files"
```

Show how many files I have of each file extension

```
qfind "select ext, count(1) from files group by ext"
```

...for specific file extensions...

```
qfind "select ext, count(1)
     from files
     where ext in('cs', 'py', 'js', 'sql')
     group by ext"
```

Check that my movie file names are the same as the dirs that they are in (first cd <movies-top-dir> or add it as the qfind startdir argument - qfind ~/movies/ "...")

```
qfind "select a.name as moviesubdir, b.name as moviefile
     from files as a
     left join files as b
     on b.woext = a.name
     and b.type = 'f'
     and b.depth = 2
     where a.type = 'd'
     and a.depth = 1"
```
