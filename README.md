# Frama-C sample run

This directory contains a minimal example of a Frama-C analysis that generates
a json document in the SARIF format. It uses the docker image of the latest
stable release (28.1 Nickel) of Frama-C as available on https://hub.docker,com
We run Frama-C on cwe787.c, which is an example from the CWE list. The
generated cwe787.json contains the alarms found by the Eva plug-in of Frama-C.

## Running the example

`run.sh` contains the command that can reproduce the run. It only requires
docker to be installed (and an internet connection if the required Frama-C image has not already been pulled from the Docker hub). This command is reproduced
here:

```sh
docker run -v $(pwd):/frama-c-test framac/frama-c:28.1-stripped \
  frama-c -eva -eva-precision 2 /frama-c-test/cwe787.c \
     -then -mdr-gen sarif -mdr-out /frama-c-test/cwe787.sarif
```

We bind the current
directory on the host to the running container, and launch Frama-C/Eva on
the example. Once the analysis is complete, (in the `-then` part of Frama-C
command line), we ask for the generation of the sarif report, in a file
located in the bound directory so that it is available on the host once
the docker container has exited.

## Main fields of the SARIF file

The most important part of the json document is the `results` field, which
is a list of objects describing the property statuses emitted by the analyzer.
For each such object, we have the following main fields:

- `ruleId` which indicates the type of alarm. These are summarized at the end
  of the document in the `taxonomies` field of the SARIF document.
- `kind` and `level`, which together describe the status of the alarm:
  - `kind`: `pass` and `level`: `none` denotes a validated user annotation
  - `kind`: `open` and `level`: `none` denotes a property with "unknown"
    status, i.e. this might be a true issue, or a false alarm due
    to the abstractions made by the analyzer
  - `kind`: `pass` and `level`: `error` denotes an invalid property, indicating
    that if a concrete execution ever reaches this point, the corresponding
    issue definitely occurs (i.e. it is not a false alarm)
  These values stems from previous of the SARIF standard and might be updated
  to avoid the need to look at both fields to determine the status of the
  property.
- `locations` indicates the position in the source file where the corresponding
  property is evaluated. Technically, it is a list, but in the files generated
  by Frama-C, said list only contains a single object, with two fields:
  - `physicalLocation` containing the file name, as an object with field `uri`
  denoting the relative path with respect to `uriBaseId`, which in our example
  is either the current working directory or the directory where Frama-C's libc
  resides.
  - `region`, containing fields `startLine`, `startColumn`, `endLine` and
    `endColumn` to delimit the relevant region within the file.

## Additional input files

This tiny example doesn't require any specific configuration besides asking
Frama-C to launch Eva (with `-eva`) and to generate the sarif document. On
more realistic analyses, Frama-C will require more information, notably to
parse the source code. Frama-C can take any number of C source files on its
command line. However, it might need to be given specific preprocessing
directives (`-I` and/or `-D` options the C compiler) if there are any. If
these directives are simple enough, they can be given directly through
`-cpp-extra-args` option, as in e.g.
`-cpp-extra-args="-Icustom/includes -DCUSTOM_MACRO=42`. For more complex
cases, notably if the directives vary across source files, the easiest
way is to rely on a json compilation database, as supported by `cmake` or
`make` (through the `bear` tool). Option `-json-compilation-database <file>`
will then instruct Frama-C to use instructions contained in `<file>`.
Note that the files and preprocessing options must appear before the `-then`
on Frama-C command line.
