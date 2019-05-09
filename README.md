# job-templating-tool

We have many users who leverage job arrays (whether in Slurm or Grid Engine) to do parameter scans.  Generating the input files for parameter scans can get complicated.  This tool was created to allow users to use templated input files to generate arbitrarily complicated job arrays by specifying parameters to vary.


## Case study

Consider the following Gaussian input file (see the [examples/water](water example) in this repository):

```
# PBE1PBE/6-311g

Water

0 1
H
O   1   0.9
H   2   0.9    1   118.1

```

We wish to perform a parameter scan, varying the O–H bond length and the planar angle H–O–H.  First, a template input file must be created:

```
# PBE1PBE/6-311g

Water

0 1
H
O   1   [{% print OH %}]
H   2   [{% print OH %}]    1   [{% print HOH %}]

```

where we'll call the file `water.com` and we've named two parameters:

| Parameter | Description | Values |
|---|---|---|
| `OH` | O–H bond length in Ångstrom | 0.8, 0.9, 1.0 |
| `HOH` | H–O–H planar angle in degrees | 114.0, 116.0, 118.0, 120.0, 122.0 |

Each index in the job array will occupy its own directory under the prefix `jobs/run0001/`.  To generate all 15 constituent jobs:

```
$ job-templating-tool -vv --parameter OH=0.8-1.0:+0.1 --parameter HOH=114.0-122.0:+2.0 \
  --directory-prefix ./jobs/run0001/ water.com
[INFO] 1 input templates to process
[INFO] added parameter OH = [0.8, 0.9, 1.0]
[INFO] added parameter HOH = [114.0, 116.0, 118.0, 120.0, 122.0]
[INFO] total parameter combinations 15
[INFO] will generate 15 file(s)
[INFO] next job array index in sequence would be 16

$ ls -l jobs/run0001/
total 8
drwxr-sr-x 2 frey sysadmin 3 May  9 16:02 1
drwxr-sr-x 2 frey sysadmin 3 May  9 16:02 10
drwxr-sr-x 2 frey sysadmin 3 May  9 16:02 11
drwxr-sr-x 2 frey sysadmin 3 May  9 16:02 12
drwxr-sr-x 2 frey sysadmin 3 May  9 16:02 13
drwxr-sr-x 2 frey sysadmin 3 May  9 16:02 14
drwxr-sr-x 2 frey sysadmin 3 May  9 16:02 15
drwxr-sr-x 2 frey sysadmin 3 May  9 16:02 2
drwxr-sr-x 2 frey sysadmin 3 May  9 16:02 3
drwxr-sr-x 2 frey sysadmin 3 May  9 16:02 4
drwxr-sr-x 2 frey sysadmin 3 May  9 16:02 5
drwxr-sr-x 2 frey sysadmin 3 May  9 16:02 6
drwxr-sr-x 2 frey sysadmin 3 May  9 16:02 7
drwxr-sr-x 2 frey sysadmin 3 May  9 16:02 8
drwxr-sr-x 2 frey sysadmin 3 May  9 16:02 9

$ cat jobs/run0001/9/water.com
# PBE1PBE/6-311g

Water

0 1
H
O   1   0.9
H   2   0.9    1   120.0

```

In order to know which parameter values are associated with each of the job array indices, the program can produce an *indexing catalog*:

```
$ rm -rf jobs/run0001
$ job-templating-tool --parameter OH=0.8-1.0:+0.1 --parameter HOH=114.0-122.0:+2.0 \
  --directory-prefix ./jobs/run0001/ --catalog run0001.txt water.com

$ cat run0001.txt
[1:./jobs/run0001/1/water.com] {"HOH":114.0,"OH":0.8}
[2:./jobs/run0001/2/water.com] {"HOH":116.0,"OH":0.8}
[3:./jobs/run0001/3/water.com] {"HOH":118.0,"OH":0.8}
[4:./jobs/run0001/4/water.com] {"HOH":120.0,"OH":0.8}
[5:./jobs/run0001/5/water.com] {"HOH":122.0,"OH":0.8}
[6:./jobs/run0001/6/water.com] {"HOH":114.0,"OH":0.9}
[7:./jobs/run0001/7/water.com] {"HOH":116.0,"OH":0.9}
[8:./jobs/run0001/8/water.com] {"HOH":118.0,"OH":0.9}
[9:./jobs/run0001/9/water.com] {"HOH":120.0,"OH":0.9}
[10:./jobs/run0001/10/water.com] {"HOH":122.0,"OH":0.9}
[11:./jobs/run0001/11/water.com] {"HOH":114.0,"OH":1.0}
[12:./jobs/run0001/12/water.com] {"HOH":116.0,"OH":1.0}
[13:./jobs/run0001/13/water.com] {"HOH":118.0,"OH":1.0}
[14:./jobs/run0001/14/water.com] {"HOH":120.0,"OH":1.0}
[15:./jobs/run0001/15/water.com] {"HOH":122.0,"OH":1.0}
```

Note that the `run0001` directory was removed before regenerating the job array:  if any target directories exist the program will not overwrite them and will return an error.


## Templates

Template files are reproduced verbatim, except for blocks delimited by `[{%` and `%}]`.  Inside those delimiters should be Python code that outputs (e.g. using `print` or `sys.stdout.write`) the text to be substituted for the delimited code.  Delimited code that is contained entirely within a single line of the file will have leading and trailing whitespace removed before substitution.  Blocks that span multiple lines will have a trailing newline preserved.

### Referencing parameter values

The value of each parameter is provided to the delimited code blocks using the same name specified in the parameter's declaration.  Thus, for `--parameter HOH=118.1` the Python code will have a global variable named `HOH` present.

In addition to the parameter values, the current job array index of the file being generated is also provided to the code block in the `JOB_ARRAY_INDEX` variable.

See the [examples](examples) directory for sample templates.


## Usage

```
$ job-templating-tool --help
usage: job-templating-tool [-h] [--version] [--verbose] [--quiet]
                           [--parameter <param-spec>] [--catalog <filepath>]
                           [--array-index <integer>] [--use-flat-layout]
                           [--directory-prefix <prefix>]
                           [--index-format-in-paths <python-conversion>]
                           [--ignore-templating-errors]
                           ...

generate templated job arrays

positional arguments:
  <filepath>            filename of an input template that should be processed

optional arguments:
  -h, --help            show this help message and exit
  --version             show program's version number and exit
  --verbose, -v         increase the amount of output produced as this program
                        executes
  --quiet, -q           decrease the amount of output produced as this program
                        executes
  --parameter <param-spec>, -p <param-spec>
                        add a named parameter to the scan; if the parameter
                        has a single value no scan is implied
  --catalog <filepath>, -c <filepath>
                        filename to which the indexing catalog (that maps job
                        array index to parameter values) should be written;
                        use "-" to write the index to stdout, if not specified
                        then no index is written
  --array-index <integer>, -a <integer>
                        starting index for the job array
  --use-flat-layout, -f
                        do not create subdirectories for each job array index,
                        put all files in the current directory
  --directory-prefix <prefix>, -P <prefix>
                        when using the subdirectory for each job array index,
                        prefix this string on the job array index; e.g. for
                        "-P ./JOB_" the directories ./JOB_1, ./JOB_2, etc.
                        would be generated
  --index-format-in-paths <python-conversion>
                        Python format conversion specification to turn the
                        integer job array index into a file name component,
                        e.g. ":04d" for names like "0001" and "0102"; default
                        is ":d" for names like "1" and "102"
  --ignore-templating-errors
                        do not exit on templating errors, continue trying to
                        generate the rest of the templated content

A <param-spec> consists of <param-name>=<value-list>, where <param-name> uses
any characters except equals (=). The <value-list> is a comma-separated list
of integer/float values; comma-separated list of integer/float ranges with
optional step size (:1 or :1.5) or division count (/10); or comma-separated
list of strings, optionally quote delimited.
```

### Parameter definitions

The program handles three different types of parameter:

- Integer:  one or more signed whole-number values
- Float: one or more floating-point values
- String: one or more character arrays

Numeric declarations that use no decimal points default to being integer type; the use of a decimal point forces float type.  Any declarations deemed to be non-numeric are handled as string.

#### Single-value

A parameter can have a single value associated with it:

```
… --parameter OH=0.9 --parameter HOH=1.181e2 …
```

These parameters would reproduce the initial Gaussian input file cited above.

#### Ranges

Ranged numeric declarations by default increment or decrement by 1 (depending on the ordering of the two extrema specified):

```
… --parameter i=10-1 --parameter j=1-10 …
```

Here, `i` decreases by 1 while `j` increases by 1.  The step size can be made explicit:

```
… --parameter i=10-1:-2 --parameter j=1-10:2 …
```

In this case, `i` will decrease by 2 while `j` increased by 2.  The range can also be split uniformly into a given number of values:

```
… --parameter a=0.0-5.0/6 --parameter b=50.0-10.0/3 …
```

Here, `a` would take values [0.0, 1.0, 2.0, 3.0, 4.0, 5.0] while `b` would take values [50.0, 30.0, 10.0].



