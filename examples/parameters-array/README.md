# Parameters Arrays

The named parameters feature allows the user to provide (on the command line interface) a parameter name and a list of values or value ranges.  When *N* named parameters are specified, every combination of those named parameter values will be traversed when generating a job array.

In some cases, it may be more useful if an arbitrary list of parameters is traversed by this tool.  For example, the user provides a *comma-separated value* (CSV) file which lists each parameter value in a column (with parameter names in the first row of the file).  In addition to any named parameters communicated via `--parameter/-p` options, the tool also loops over the list from the CSV file.  Note that named parameter values **override** values of the same name coming from a CSV file.  The CSV file (with column headers that are parameter names) is essentially a list of dictionaries, with each cell becoming a key-value pair in one of those dictionaries.

Consider this CSV file:
```
CH,OH
1.1,2.4
1.1,2.6
1.1,2.8
1.2,2.4
1.2,2.6
1.2,2.8
```
The file consists of two columns (parameters) with 6 rows (values); the first row contains the parameter names, `CH` and `OH`.  Given the template
```
C-H length = [{% print CH %}]
O-H length = [{% print OH %}]

```
a six-index job array would result:
```
$ job-templating-tool --prefix=jobs/ --csv-parameters=params.csv simple.tmpl
$ ls -l jobs/
total 3
drwxr-xr-x 2 frey everyone 3 May 14 14:29 1
drwxr-xr-x 2 frey everyone 3 May 14 14:29 2
drwxr-xr-x 2 frey everyone 3 May 14 14:29 3
drwxr-xr-x 2 frey everyone 3 May 14 14:29 4
drwxr-xr-x 2 frey everyone 3 May 14 14:29 5
drwxr-xr-x 2 frey everyone 3 May 14 14:29 6

$ cat jobs/3/simple.tmpl
C-H length = 1.1
O-H length = 2.8

```

## Complex Example

If support is present in your Python environment, JSON or YAML can be used to hold parameters arrays.  In each case, the structure read from the file must still be a list of dictionaries, but within each dictionary any supported data types are permissible:
```
- runid: 25634-678
  materials:
    - C2H2
    - O2
    - H2
    - CO
  temperature: 273.15
- runid: 25634-679
  materials:
    - C2H2
    - O2
    - H2
    - CO
  temperature: 283.15
```
As long as the Python code blocks present in the template file(s) understand the structure of they key-value pairs, the templating will be successful.  Consider the following template:
```
%
% GAS PHASE CHEMISTRY
%
id       [{% print runid %}]
species  [{% print ','.join(materials) %}]
thermo   [{% print '{0:.2f}'.format(temperature) %}] kelvin
%
```
Generating a job array form the template:
```
$ job-templating-tool --prefix=jobs/ --yaml-parameters=parameters.yaml gpcrxn.in
$ cat jobs/1/gpcrxn.in
%
% GAS PHASE CHEMISTRY
%
id       25634-678
species  C2H2,O2,H2,CO
thermo   273.15 kelvin
%

$ cat jobs/2/gpcrxn.in
%
% GAS PHASE CHEMISTRY
%
id       25634-679
species  C2H2,O2,H2,CO
thermo   283.15 kelvin
%
```
