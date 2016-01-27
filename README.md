# bdist

Submit a distributed job to your LSF compute cluster.

## Usage

    bdist -c COMMAND [OPTIONS] (--fofn FILE | FILE ...)
    
    bdist (-V | --version)  Display the version number
    bdist (-h | --help)     Display the help

Distribute the homogeneous processing of a number of files across a
compute cluster.

### `COMMAND`

This is the command that will be run against each file to be processed,
with any optional arguments. By default, the filename to be processed
will be appended to this; however, if it needs to occur elsewhere in the 
command, it may be inserted using `{}` any number of times (in which
case, the filename *won't* be appended automatically).

For example:

    bdist -c "gzip -9 < {} > {}.gz" *

Note that commands with arguments *must* be quoted for the `-c` option.

### Files

The files to be processed by the above command may be supplied
individually at the command line, or by using a FOFN (file of
filenames, `EOL` delimited) with the `--fofn` option.

For example:

    bdist -c foo /path/to/some/files* /another/file /{foo,bar}/*.quux

    find /some/path -type f -name "*.baz" | xargs bdist -c foo

    find . -type f -exec bdist -c foo +

Note that, if using `find`(1) with `-exec`, then the `+` delimiter
should be used. If the `;` delimiter were used, it will still work, but
it will spawn multiple individual jobs, rather than making use of an LSF
job array.

A FOFN should be used if there are a lot of files to process, to avoid
blowing the `$ARG_MAX` of your shell. Also note that the number of files
to process, either directly or through a FOFN, *must* not exceed the LSF
`MAX_JOB_ARRAY_SIZE` parameter, set in `lsb.params` (default 1000).

### Options

You may optionally pass some `bsub`(1) options:

    -J JOB_NAME
    -M MEM_LIMIT
    -R RESOURCE_REQ
    -n MIN_CPUS[,MAX_CPUS]
    -q QUEUE_NAME

These follow the same usage pattern as `bsub`. If a `JOB_NAME` is not
specified, one will be automatically generated.

## Logging

While jobs are running, their working directory will contain the output
of each individual process' output in `~/.bdist/work/JOB_NAME`. Once the
entire job array has finished, a clean up job will concatenate the logs
into `~/.bdist/logs/JOB_NAME.log`.

## Example

The original use case for `bdist` was to `gzip`(1) a bunch of large
files without tying up the head nodes for half a day. We can now do this
simply with:

    bdist -c "gzip -9f" *.dat

The `-f` flag is used to avoid jobs exiting as failed if the `.gz` file
already exists.

However, we can do better than this by using, for example, `pigz`(1) for
multicore compression:

    bdist -c "pigz -9 -f -p 8" -n 8 -R "span[hosts=1]" *.dat

## License

Copyright (c) 2016 Genome Research Ltd.

This program is free software: you can redistribute it and/or modify it
under the terms of the GNU General Public License as published by the
Free Software Foundation, either version 3 of the License, or (at your
option) any later version.

This program is distributed in the hope that it will be useful, but
WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General
Public License for more details.

You should have received a copy of the GNU General Public License along
with this program. If not, see <http://www.gnu.org/licenses/>.
