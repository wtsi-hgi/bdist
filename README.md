# bdist

Submit a distributed job to your LSF compute cluster.

## Usage

    bdist [-c] COMMAND [OPTIONS] (--fofn FILE | FILE ...)
    
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

    bdist "gzip -9 < {} > {}.gz" *

Note that commands with arguments *must* be quoted. The `-c` flag itself
is optional, but if it is omitted, then the command *must* be the first
positional argument.

### Files

The files to be processed by the above command may be supplied
individually at the command line, or by using a FOFN (file of
filenames, `EOL` delimited) with the `--fofn` option.

For example:

    bdist foo /path/to/some/files* /another/file /{foo,bar}/*.quux

    bdist foo --fofn /path/to/my.fofn

    find /some/path -type f -name "*.baz" | xargs bdist foo

    find . -type f -exec bdist foo +

Note that, if using `find`(1) with `-exec`, then the `+` terminator
should be used. If the `;` terminator were used, it would still work,
but would spawn multiple individual jobs, rather than making use of an
LSF job array.

A FOFN should be used if there are a lot of files to process, to avoid
blowing the `$ARG_MAX` of your shell. Also note that the number of files
to process, either directly or through a FOFN, *must not* exceed the LSF
`MAX_JOB_ARRAY_SIZE` parameter, set in `lsb.params` (default 1000).

### Lazy Mode

By default, the values for the command (or `-c` flag) and `--fofn` flag
are checked: commands are checked to see if they're valid commands or
shell builtins; while each file in the FOFN will be checked to see if it
actually is a file. The purpose of this is to fail early, before
submitting potentially erroneous jobs to your cluster.

However, a command on your cluster's workers may not be available on the
dispatch node and a large FOFN will take time to check through, at the
expense of IO. These checks can therefore be bypassed by using their
uppercase counterparts: `-C` and `--FOFN`, respectively.

### Options

You may optionally pass through some `bsub`(1) options:

    -G USER_GROUP *
    -J JOB_NAME
    -M MEM_LIMIT
    -R RESOURCE_REQ
    -n MIN_CPUS[,MAX_CPUS]
    -q QUEUE_NAME

These follow the same usage pattern as `bsub`. If a `JOB_NAME` is not
specified, one will be automatically generated. The options marked with
an asterisk, above, are passed to both the job array *and* the cleanup
job.

## Logging

While jobs are running, their working directory will contain the output
of each individual process, amongst other things, in
`~/.bdist/work/JOB_NAME`. Once the entire job array has finished, a
clean up job will concatenate the logs into `~/.bdist/logs/JOB_NAME.log`
and remove the working directory.

## Example

The original use case for `bdist` was to `gzip`(1) a bunch of large
files without tying up the head nodes for half a day. We can now do this
simply with:

    bdist "gzip -9f" *.dat

`gzip`'s `-f` flag is used to avoid jobs exiting as failed if the `.gz`
file already exists.

However, we can do better than this by using, for example,
[`pigz`(1)](http://zlib.net/pigz/) for multicore compression:

    bdist "pigz -9 -f -p 8" -n 8 -R "span[hosts=1]" *.dat

### Advanced Example

**NOTE This is currently untested! Quoting may break things...**

When using a FOFN, each line is fed to the command, presumably as a file
to process. However, in lazy mode, there's no reason for a FOFN to be a
file of filenames; it could, for example, be a file of database keys, or
JSON object member names, etc. Then the command takes each of these as
its input argument. For example:

    bdist "process_item <(jq \".{}\" data.json)" --FOFN json.keys

(n.b., In the JSON case, a wrapper script is available for convenience;
see below for details.)

It is also possible to just use the job index, by either creating a FOFN
of sequential numbers, or -- if both index *and* key are needed -- by
referring to the LSF environment variable `$LSB_JOBINDEX`:

    bdist process_number --FOFN <(seq 10)

    bdist "do_something --id=\$LSB_JOBINDEX -x {}" --FOFN some.keys

Note that dollar signs must be escaped, to avoid your execution shell
from expanding them; either that, or use single quotes. Better yet, it
would be wise to consolidate more complex commands into a simple shell
script, which takes the FOFN element as its argument and will have
access to all [LSF job environment variables](https://www-01.ibm.com/support/knowledgecenter/SSETD4_9.1.3/lsf_config_ref/lsf_envars_job_exec.html).

## Using JSON

**NOTE This is currently untested! Quoting may break things...**

If you have a command that needs to distribute and iterate over the
top-level elements of a JSON array or object, the above "advanced usage"
paradigm has been wrapped into a convenience script, using
[`jq`(1)](https://stedolan.github.io/jq/) to process the JSON data:

    bdist-json JSONFILE [OPTIONS] COMMAND

Where `JSONFILE` is the JSON file you wish to iterate over, which must
be of the form of either a single array or object, and `COMMAND` is to
be executed on a compute node distributing each element from the JSON
collection. The `OPTIONS` are the same as those for `bdist`, above, and
are simply passed through.

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

LSF(R) is a registered trademark of IBM Corporation.
