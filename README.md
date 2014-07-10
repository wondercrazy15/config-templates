# Config templates

We have here TT files that may generate configuration files for:

* [Apache](http://httpd.apache.org) in various circumstances
* [nginx](http://nginx.org)
* Many [Logstash](http://logstash.net) inputs, outputs and filters.
* [PerfSONAR](http://perfsonar.net)
* Some basic udev rules
* named
* ...

And a tiny Perl script that fills them from a JSON file, if it has the
correct structure.

# Ongoing work

We are in preparation of moving the legacy hpcugent files to a properly 
structured and tested repository, that at some day should be moved to 
the quattor project at https://github.com/quattor.

The new structure consists of the following files/directories
* the metaconfig directory hold all services, each with their TT files, 
pan schema and unittest data
* the scripts directory holds useful standalone scripts, in particular the 
`json2tt.pl` script
* test directory with the (Python) unittest code
* setup_packaging.py a distultils packaging file for the new code (and only the new code)
* NOTICE (file as per the Apache License

Read the principles behind the structure of the metaconfig directory
* https://github.com/hpcugent/config-templates/issues/40
* https://github.com/hpcugent/config-templates/issues/41


# Requirements

For installation
* perl `Template::Toolkit` version 2.25 or later (use CPAN or for src rpms on el5, el6 and el7, contact @stdweird)
* perl `JSON::XS`
* perl quattor modules `CAF`, `LC`

For unit-testing/development
* recent pan-compiler (10.1 or later), with `panc` in `$PATH`
* python `vsc-base` (`easy_install vsc-base` (use `--user` on recent systems for local install), or ask @stdweird for rpms)
* a local checkout of `template-library-core` repository (https://github.com/quattor/template-library-core); default 
expects it in the same directory as the checkout of this repository, but it can be changed via the `--core` option of the 
unittest suite

# Running the tests

From the base of the repository, run 
```bash
python test/suite.py
```
to run all tests of all services.

## Unittest suite help 

```bash
python test/suite.py -h
```
(try --help for long option names)

```
Usage: suite.py [options]


  Usage: "python -m test.suite" or "python test/suite.py"

  @author: Stijn De Weirdt (Ghent University)

Options:
  -h            show short help message and exit
  -H            show full help message and exit

  Main options (configfile section MAIN):
    -C CORE     Path to clone of template-library-core repo (def /home/stdweird/.git/github.ugent/template-library-core)
    -j JSON2TT  Path to json2tt.pl script (def /home/stdweird/.git/github.ugent/config-templates/scripts/json2tt.pl)
    -s SERVICE  Select one service to test (when not specified, run all services)
    -t TESTS    Select specific test for given service (when not specified, run all tests) (type comma-separated list)

  Debug and logging options (configfile section MAIN):
    -d          Enable debug log mode (def False)

Boolean options support disable prefix to do the inverse of the action, e.g. option --someopt also supports --disable-someopt.

All long option names can be passed as environment variables. Variable name is SUITE_<LONGNAME> eg. --some-opt is same as setting SUITE_SOME_OPT in the environment.
```

## Suported flags

Queries the supproted flags via the `--showflags` option
```bash
python test/suite.py --showflags
```
gives
```
supported flags: I, M, caseinsensitive, metaconfigservice=, multiline, negate
    I: alias for "caseinsensitive"
    M: shorthand for "multiline"
    caseinsensitive: Perform case-insensitive matches
    metaconfigservice=: Look for module/contents in the expected metaconfig component path for the service
    multiline: Treat all regexps as multiline regexps
    negate: Negate all regexps (none of the regexps can match) (not applicable when COUNT is set for individual regexp)
```

# Development example

Start with forking the upstream repository https://github.com/hpcugent/config-templates, and clone your personal fork in your workspace. 
(replace `stdweird` with your own github username). Also add the `upstream` repository (using `https` protocol).

```bash
git clone git@github.com:stdweird/config-templates.git
cd config-templates
git remote add upstream https://github.com/hpcugent/config-templates
```

Check your environment by running the unittests. No tests should fail when the environment is setup properly. 
(Open an issue on github in case there is a problem you can't resolve.)

```bash
python test/suite.py
```

## Add new service

### Target

Our `example` service requires a text config file in `/etc/example/exampled.conf` and has following structure
```
name = {
    hosts = server1,server2
    port = 800
    master = FALSE
    description = "My example"
}
```

where following fields are mandatory:
* `hosts`: a comma separated list of hostnames
* `port`: an integer
* `master`: boolean with possible values `TRUE` or `FALSE` 
* `description`: a quoted string 

The service has also an optional fields `option`, also a quoted string.

Upon changes of the config file, the `exampled` service needs to be restarted.

### Prepare

Pick a good and relevant name for the service (in this case we will add the non-existing `example` service), and create 
```bash
service=example
```

Make a new branch where you will work in and that you will use to create the pull-request (PR) when finished
```bash
git checkout -b example_service
```

Create the initial directory structure.
```bash
mkdir -p metaconfig/$service/tests/{profiles,regexps} $service/pan
```

Add some typical files (some of the files are not mandatory, 
but are simply best practice).

```bash
cd metaconfig/$service

echo -e "declaration template metaconfig/$service/schema;\n" > pan/schema.pan
echo -e "unique template metaconfig/$service/config;\n\ninclude 'metaconfig/$service/schema';" > pan/config.pan

echo -e "object template config;\n\ninclude 'metaconfig/$service/config';\n" > tests/profiles/config.pan
mkdir tests/regexps/config
echo -e 'Base test for config\n---\nmultiline\n---\n$wontmatch^\n' > tests/regexps/config/base
```

Commit this initial structure
```bash
git commit -a "initial structure for service $service"
```
## Create the schema

The schema needs to be created in the `pan` subdirectory of the service directory `metaconfig/$service`.

```
declaration template metaconfig/example/schema;

include 'pan/types';

type example_service = {
    'hosts' :  type_hostname[]
    'port' : long(0..)
    'master' : boolean
    'description' : string
    'option' ? string
};

```

* `long`, `boolean` and `string` are pan builtin types (see the panbook for more info)
* `type_hostname` is a type that is available from the main `pan/types` template as part of the core template library.

## Create config template for metaconfig component (optional)

A reference config file can now also be created, with e.g. the type binding to the correct path and configuration of the
restart action and the TT module to load.

```
unique template metaconfig/example/config;

include 'metaconfig/example/schema';

bind "/software/components/metaconfig/services/{/etc/example/exampled.conf}/contents" = example_service;

prefix "/software/components/metaconfig/services/{/etc/example/exampled.conf}";
"daemon" = "exampled";
"module" = "example/main";

```

This will expect the TT module with relative filename `example/main.tt`.

## Make TT file to match desired output

Create the `main.tt` file with content

```
name = {
[% FILTER indent -%]
hosts = [% hosts.join(',') %]
port = [% port %]
master = [% master ? "TRUE" : "FALSE %]
description = "[% description %]"
[%     IF option.defined -%]
option = "[% option %]"
[%     END -%]
[% END -%]
}
```

* `FILTER indent` creates the indentation

## Add unittests

### Flags

## Usage with ncm-metaconfig

