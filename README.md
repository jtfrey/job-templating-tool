# job-templating-tool

We have many users who leverage job arrays (whether in Slurm or Grid Engine) to do parameter scans.  Generating the input files for parameter scans can get complicated.  This tool was created to allow users to use templated input files to generate arbitrarily complicated job arrays by specifying parameters to vary.


## Case study

Consider the following Gaussian input file (see the [water example](./examples/water) in this repository):

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
  --prefix ./jobs/run0001/ water.com
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
  --prefix ./jobs/run0001/ --catalog run0001.txt water.com

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

See the [examples](./examples) directory for sample templates.


## Usage

```
$ job-templating-tool --help
usage: job-templating-tool [-h] [--version] [--verbose] [--quiet]
                           [--catalog <filepath>] [--append-to-catalog]
                           [--array-index <integer>] [--use-flat-layout]
                           [--prefix <prefix>]
                           [--index-format-in-paths <python-conversion>]
                           [--ignore-templating-errors]
                           [--yaml-parameters <yaml-file>]
                           [--json-parameters <json-file>]
                           [--csv-parameters <csv-file>]
                           [--parameter <param-spec>]
                           [--jobs-per-directory <count>] [--copy <filepath>]
                           [--symlink <filepath>] [--no-relative-targets]
                           [--array-base-index <index>] [--array-size <count>]
                           <template-file> [<template-file> ...]

generate templated job arrays

positional arguments:
  <template-file>       filename of an input template that should be
                        processed; use "-" for stdin

optional arguments:
  -h, --help            show this help message and exit
  --version             show program's version number and exit
  --verbose, -v         increase the amount of output produced as this program
                        executes
  --quiet, -q           decrease the amount of output produced as this program
                        executes
  --catalog <filepath>, -c <filepath>
                        filename to which the indexing catalog (that maps job
                        array index to parameter values) should be written;
                        use "-" to write the index to stdout, if not specified
                        then no index is written
  --append-to-catalog   if the indexing catalog file exists, append to it
                        rather than overwriting it
  --array-index <integer>, -i <integer>
                        starting index for the job array
  --use-flat-layout, -f
                        do not create subdirectories for each job array index,
                        just put all generated files in the current directory
  --prefix <prefix>, -P <prefix>
                        prefix the given string on every file
                        generated/copied/symlinked by the program; e.g. for
                        directory mode and "-P ./JOB_" the directories
                        ./JOB_1, ./JOB_2, etc. would be generated (trailing
                        path separators are very significant, as they imply a
                        subdirectory and not a prefix on a filename)
  --index-format-in-paths <python-conversion>
                        Python format conversion specification to turn the
                        integers (like the job array index) into a file name
                        component, e.g. ":04d" for names like "0001" and
                        "0102"; default is ":d" for names like "1" and "102"
  --ignore-templating-errors
                        do not exit on templating errors, continue trying to
                        generate the rest of the templated content

job parameters:
  options that communication job parameters to the templater

  --yaml-parameters <yaml-file>
                        read the array of parameters from a YAML file
  --json-parameters <json-file>
                        read the array of parameters from a JSON file
  --csv-parameters <csv-file>
                        read the array of parameters from a CSV file; the
                        first row should contain the variable names for each
                        column
  --parameter <param-spec>, -p <param-spec>
                        add a named parameter to the scan; named parameters
                        augment an array specified via --yaml-parameters,
                        --json-parameters, --csv-parameters

directory mode:
  options available when not doing flat layout

  --jobs-per-directory <count>, -j <count>
                        create a hierarchy of directories such that no more
                        than this many job subdirectories appear in each leaf
                        directory; e.g. for --jobs-per-directory=4 and 16
                        generated job subdirectories, the paths would be
                        <prefix>/{0,1,2,3}/<job-array-index>, which each
                        first-level directory under prefix having 4 job
                        subdirectories
  --copy <filepath>, -C <filepath>
                        copy the given path to each job subdirectory (can be
                        specified multiple times)
  --symlink <filepath>, -s <filepath>
                        create a symbolic link to the given path in each job
                        subdirectory (can be specified multiple times)
  --no-relative-targets
                        for --symlink/-s, always use the absolute path of the
                        target file/directory and not a relative path
  --array-base-index <index>
                        when used in conjuncation with --jobs-per-
                        directory/-j, this is the lowest possible index in the
                        array; useful for working on a subset of a larger job
                        array (using --array-index/-a, --array-size)
  --array-size <count>  when used in conjuncation with --jobs-per-
                        directory/-j, the hierarchy is structured as though
                        there are this many jobs in the array; useful for
                        working on a subset of a larger job array (using
                        --array-index/-a, --array-base-index)

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

### Directory mode

By default the program creates a directory to hold each job array index generated.  The directory is named using the job array index, with several possible transformations:

- Index output format:  A Python format conversion specifier can be provided to alter the width and zero-pad the name, as in `:04d` to yield e.g. `0001` and `0230` as directory names.  The default is `:d` to print the integer with no formal width or padding.  Note that if you do specify a width, zero-padding is a good idea to avoid whitespace in filenames.
- Prefix:  A string added to the head of every generated, copied, or symlinked file.  You **must** include a trailing path separator character (e.g. '/' on Unix) if the prefix is meant to be a directory containing the generated content.  Otherwise, it is just a prefix on the first component of the generated paths.  For example, `--prefix=jobs/run1_` would produce directories like `jobs/run1_1`, `jobs/run1_2` whereas `--prefix=jobs/run1/` creates a parent directory for the jobs like `jobs/run1/1` and `jobs/run1/2`.
- Directory splitting:  For very large job arrays it may not be optimal to collocate all the job directories under a single parent directory.  The options dealing with directory splitting are fairly complex, and are summarized in the following subsection.

#### Directory splitting

The `--jobs-per-directory=N` option forces the program to create a hierarchy of directories such that every directory in the hierarchy has at most N subdirectories present within it.  For the Gaussian example above, 15 jobs are created; with `--jobs-per-directory=8`, the same command would produce:

```
$ job-templating-tool --parameter OH=0.8-1.0:+0.1 --parameter HOH=114.0-122.0:+2.0 \
  --prefix ./jobs/ --catalog jobs.idx --jobs-per-directory=8 water.com

$ cat jobs.idx
[1:./jobs/0/1/water.com] {"HOH":114.0,"OH":0.8}
[2:./jobs/0/2/water.com] {"HOH":116.0,"OH":0.8}
[3:./jobs/0/3/water.com] {"HOH":118.0,"OH":0.8}
[4:./jobs/0/4/water.com] {"HOH":120.0,"OH":0.8}
[5:./jobs/0/5/water.com] {"HOH":122.0,"OH":0.8}
[6:./jobs/0/6/water.com] {"HOH":114.0,"OH":0.9}
[7:./jobs/0/7/water.com] {"HOH":116.0,"OH":0.9}
[8:./jobs/0/8/water.com] {"HOH":118.0,"OH":0.9}
[9:./jobs/1/9/water.com] {"HOH":120.0,"OH":0.9}
[10:./jobs/1/10/water.com] {"HOH":122.0,"OH":0.9}
[11:./jobs/1/11/water.com] {"HOH":114.0,"OH":1.0}
[12:./jobs/1/12/water.com] {"HOH":116.0,"OH":1.0}
[13:./jobs/1/13/water.com] {"HOH":118.0,"OH":1.0}
[14:./jobs/1/14/water.com] {"HOH":120.0,"OH":1.0}
[15:./jobs/1/15/water.com] {"HOH":122.0,"OH":1.0}
```

Note that there are 8 jobs under the `./jobs/0/` directory and the remaining 7 jobs are under `./jobs/1/`.  Dropping to 3 jobs per directory:

```
$ rm -rf jobs.idx jobs
$ job-templating-tool --parameter OH=0.8-1.0:+0.1 --parameter HOH=114.0-122.0:+2.0 \
  --prefix ./jobs/ --catalog jobs.idx --jobs-per-directory=3 water.com

$ cat jobs.idx
[1:./jobs/0/0/1/water.com] {"HOH":114.0,"OH":0.8}
[2:./jobs/0/0/2/water.com] {"HOH":116.0,"OH":0.8}
[3:./jobs/0/0/3/water.com] {"HOH":118.0,"OH":0.8}
[4:./jobs/0/1/4/water.com] {"HOH":120.0,"OH":0.8}
[5:./jobs/0/1/5/water.com] {"HOH":122.0,"OH":0.8}
[6:./jobs/0/1/6/water.com] {"HOH":114.0,"OH":0.9}
[7:./jobs/0/2/7/water.com] {"HOH":116.0,"OH":0.9}
[8:./jobs/0/2/8/water.com] {"HOH":118.0,"OH":0.9}
[9:./jobs/0/2/9/water.com] {"HOH":120.0,"OH":0.9}
[10:./jobs/1/3/10/water.com] {"HOH":122.0,"OH":0.9}
[11:./jobs/1/3/11/water.com] {"HOH":114.0,"OH":1.0}
[12:./jobs/1/3/12/water.com] {"HOH":116.0,"OH":1.0}
[13:./jobs/1/4/13/water.com] {"HOH":118.0,"OH":1.0}
[14:./jobs/1/4/14/water.com] {"HOH":120.0,"OH":1.0}
[15:./jobs/1/4/15/water.com] {"HOH":122.0,"OH":1.0}
```

The hierarchy is one level deeper now to ensure that each directory contains at most 3 child directories.

#### Array dimensioning

In both cases above, the program assumed two things about the job array being produced:

1. The base index of the array is the same as the `--array-index/-a` (in this case, the default of 1).
2. The array size matches the number of parameter combinations produced.

These are the default behaviors of the program.  However, in some cases the user may want to generate templated jobs within a larger overall job array.  Consider a job array meant to have a base index of 100 and a size of 1000 jobs; the 15 Gaussian jobs are meant to start at index 510.  The following illustrates how these jobs are created:

```
$ rm -rf jobs.idx jobs
$ job-templating-tool --parameter OH=0.8-1.0:+0.1 --parameter HOH=114.0-122.0:+2.0 \
  --prefix ./jobs/ --catalog jobs.idx --array-size=1000 --array-base-index=100 \
  --array-index=510 --jobs-per-directory=50 water.com

$ cat jobs.idx
[510:./jobs/8/510/water.com] {"HOH":114.0,"OH":0.8}
[511:./jobs/8/511/water.com] {"HOH":116.0,"OH":0.8}
[512:./jobs/8/512/water.com] {"HOH":118.0,"OH":0.8}
[513:./jobs/8/513/water.com] {"HOH":120.0,"OH":0.8}
[514:./jobs/8/514/water.com] {"HOH":122.0,"OH":0.8}
[515:./jobs/8/515/water.com] {"HOH":114.0,"OH":0.9}
[516:./jobs/8/516/water.com] {"HOH":116.0,"OH":0.9}
[517:./jobs/8/517/water.com] {"HOH":118.0,"OH":0.9}
[518:./jobs/8/518/water.com] {"HOH":120.0,"OH":0.9}
[519:./jobs/8/519/water.com] {"HOH":122.0,"OH":0.9}
[520:./jobs/8/520/water.com] {"HOH":114.0,"OH":1.0}
[521:./jobs/8/521/water.com] {"HOH":116.0,"OH":1.0}
[522:./jobs/8/522/water.com] {"HOH":118.0,"OH":1.0}
[523:./jobs/8/523/water.com] {"HOH":120.0,"OH":1.0}
[524:./jobs/8/524/water.com] {"HOH":122.0,"OH":1.0}
```

Now assume a different parameterization which holds the O–H bond length constant at 1.45 and scans the H–O–H angle, and is inserted into the overall array at index 680:

```
$ job-templating-tool --parameter OH=1.45 --parameter HOH=114.0-122.0:+2.0 \
  --prefix ./jobs/ --catalog jobs.idx --append-to-catalog --array-size=1000 \
  --array-base-index=100 --array-index=680 --jobs-per-directory=50 water.com

$ cat jobs.idx
[510:./jobs/8/510/water.com] {"HOH":114.0,"OH":0.8}
[511:./jobs/8/511/water.com] {"HOH":116.0,"OH":0.8}
[512:./jobs/8/512/water.com] {"HOH":118.0,"OH":0.8}
[513:./jobs/8/513/water.com] {"HOH":120.0,"OH":0.8}
[514:./jobs/8/514/water.com] {"HOH":122.0,"OH":0.8}
[515:./jobs/8/515/water.com] {"HOH":114.0,"OH":0.9}
[516:./jobs/8/516/water.com] {"HOH":116.0,"OH":0.9}
[517:./jobs/8/517/water.com] {"HOH":118.0,"OH":0.9}
[518:./jobs/8/518/water.com] {"HOH":120.0,"OH":0.9}
[519:./jobs/8/519/water.com] {"HOH":122.0,"OH":0.9}
[520:./jobs/8/520/water.com] {"HOH":114.0,"OH":1.0}
[521:./jobs/8/521/water.com] {"HOH":116.0,"OH":1.0}
[522:./jobs/8/522/water.com] {"HOH":118.0,"OH":1.0}
[523:./jobs/8/523/water.com] {"HOH":120.0,"OH":1.0}
[524:./jobs/8/524/water.com] {"HOH":122.0,"OH":1.0}
[680:./jobs/11/680/water.com] {"HOH":114.0,"OH":1.45}
[681:./jobs/11/681/water.com] {"HOH":116.0,"OH":1.45}
[682:./jobs/11/682/water.com] {"HOH":118.0,"OH":1.45}
[683:./jobs/11/683/water.com] {"HOH":120.0,"OH":1.45}
[684:./jobs/11/684/water.com] {"HOH":122.0,"OH":1.45}
```

#### Copying and symlinking files into job directories

In many cases there may be additional files/directories that must also be present in each job array index's directory.  For anything that is read-only during the job, try adding a symbolic link to the original file/directory using the `--symlink` option.  For files that will be modified during execution of the job, the file/directory can be copied into the job directory using the `--copy` flag.

Assume for our Gaussian example there is a checkpoint file that will be read to get the coordinates and basis set for the calculation.  Since Gaussian alters that checkpoint file during execution, it must be copied into the job directories:

```
$ rm -rf jobs.idx jobs
$ job-templating-tool --parameter OH=0.8-1.0:+0.1 --parameter HOH=114.0-122.0:+2.0 \
  --prefix ./jobs/ --catalog jobs.idx --copy=water.chk water.com

$ ls -l jobs/1
total 22
-rw-r--r-- 1 frey sysadmin 50 May 10 13:52 water.chk
-rw-r--r-- 1 frey sysadmin 69 May 10 13:46 water.com
```

Now assume that the job script that will process this array includes a post-processing step after Gaussian completes execution.  A large data set must be present in the working directory named `fields.db` but is only read (never written).  In this case, a symlink is appropriate:

 ```
$ rm -rf jobs.idx jobs
$ job-templating-tool --parameter OH=0.8-1.0:+0.1 --parameter HOH=114.0-122.0:+2.0 \
  --prefix ./jobs/ --catalog jobs.idx --copy=water.chk --symlink ../fields.db \
  water.com

$ ls -l jobs/1
total 2
lrwxrwxrwx 1 frey sysadmin 18 May 10 14:41 fields.db -> ../../../fields.db
-rw-r--r-- 1 frey sysadmin 50 May 10 13:52 water.chk
-rw-r--r-- 1 frey sysadmin 69 May 10 14:41 water.com
```
