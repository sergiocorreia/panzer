panzer
======

panzer adds ‘styles’ to [pandoc](http://johnmacfarlane.net/pandoc/index.html). Styles provide a way to set all options for a pandoc document with one line (‘I want this document be an article/CV/notes/letter’).

You can think of styles as a level up in abstraction from a pandoc template. Styles are combinations of templates, metadata settings, pandoc command line options, and instructions to run filters, scripts and postprocessors. These settings can be customised on a per writer and per document basis. Styles can be combined and can bear inheritance relations to each other. panzer exposes a large amount of structured information to the external processes called by styles, allowing those processes to be both more powerful and themselves controllable via metadata (and hence also by styles). Styles simplify makefiles, bundling everything related to the look of the document in one place.

You can think of panzer as an exoskeleton that sits around pandoc and configures pandoc based on a single choice in your document, `style`.

To use a style, add a field with your style name to the yaml metadata block of your document:

``` yaml
style: Notes
```

Multiple styles can be supplied as a list:

``` yaml
style:
    - Notes
    - BoldHeadings
```

Styles are defined in a `yaml` file ([example](https://github.com/msprev/dot-panzer/blob/master/styles/styles.yaml)). The style definition file, plus associated executables, are placed in the `.panzer` directory in the user’s home folder ([example](https://github.com/msprev/dot-panzer)).

A style can also be defined inside the document’s metadata block:

``` yaml
---
style: Notes
styledef:
    Notes:
        all:
            metadata:
                numbersections: false
        latex:
            metadata:
                numbersections: true
                fontsize: 12pt
            commandline:
                columns: "`75`"
            filter:
                - run: deemph.py
...
```

Style settings can be overridden by adding the appropriate field outside a style definition in the document’s metadata block:

``` yaml
---
style: Letter
filter:
    - run: deemph.py
commandline:
    - latex-engine: "`xelatex`"
...
```

Installation
============

        git clone https://github.com/msprev/panzer
        cd panzer
        python3 setup.py install

*Requirements:*

-   [pandoc](http://johnmacfarlane.net/pandoc/index.html)
-   [Python 3](https://www.python.org/downloads/)
-   [setuptools for Python3](http://stackoverflow.com/questions/14426491/python-3-importerror-no-module-named-setuptools) (included in some Python 3 distributions)

*To upgrade an existing installation:*

        cd /path/to/panzer/directory/cloned
        git pull
        python3 setup.py install --force

Use
===

Run `panzer` on your document as you would `pandoc`. If the document lacks a `style` field, this is equivalent to running `pandoc`. If the document has a `style` field, panzer will invoke pandoc plus any associated scripts, filters, and populate the appropriate metadata fields.

`panzer` accepts the same command line options as `pandoc`. These options are passed to the underlying instance of pandoc. pandoc command line options can also be set via metadata.

panzer has additional command line options. These are prefixed by triple dashes (`---`). Run the command `panzer -h` to see them:

      -h, --help, ---help, ---h
                            show this help message and exit
      -v, --version, ---version, ---v
                            show program's version number and exit
      ---quiet              only print errors and warnings
      ---strict             exit on first error
      ---panzer-support PANZER_SUPPORT
                            .panzer directory
      ---debug DEBUG        filename to write .log and .json debug files

Panzer expects all input and output to be utf-8.

Style definition
================

A style definition may consist of:

| field         | value                              | value type                    |
|:--------------|:-----------------------------------|:------------------------------|
| `parent`      | parent(s) of style                 | `MetaList` or `MetaInlines`   |
| `metadata`    | default metadata fields            | `MetaMap`                     |
| `commandline` | pandoc command line options        | `MetaMap`                     |
| `template`    | pandoc template                    | `MetaInlines` or `MetaString` |
| `preflight`   | run before input doc is processed  | `MetaList`                    |
| `filter`      | pandoc filters                     | `MetaList`                    |
| `postprocess` | run on pandoc’s output             | `MetaList`                    |
| `postflight`  | run after output file written      | `MetaList`                    |
| `cleanup`     | run on exit irrespective of errors | `MetaList`                    |

Style definitions are hierarchically structured by *name* and *writer*. Style names by convention should be MixedCase (`MyNotes`) to avoid confusion with other metadata fields. Writer names are the same as those of the relevant pandoc writer (e.g. `latex`, `html`, `docx`, etc.) A special writer, `all`, matches every writer.

-   `parent` takes a list or single style. Children inherit the properties of their parents. Children may have multiple parents.

-   `metadata` contains default metadata set by the style. Any metadata field that can appear in a pandoc document can appear here.

-   `commandline` specifies pandoc’s command line options.

-   `template` is a pandoc [template](http://johnmacfarlane.net/pandoc/demo/example9/templates.html) for the style.

-   `preflight` lists executables run before the document is processed. These are run after panzer reads the input, but before that input is sent to pandoc.

-   `filter` lists pandoc [json filters](http://johnmacfarlane.net/pandoc/scripting.html). Filters gain two new properties from panzer. For more info, see section on [compatibility](#compatibility) with pandoc.

-   `postprocessor` lists executable to pipe pandoc’s output through. Standard unix executables (`sed`, `tr`, etc.) are examples of possible use. Postprocessors are skipped if a binary writer (e.g. `docx`) is used.

-   `postflight` lists executables run after the output has been written. If output is stdout, postflight scripts are run after stdout has been flushed.

-   `cleanup` lists executables run before panzer exits and after postflight scripts. Cleanup scripts run irrespective of whether an error has occurred earlier.

Example:

``` yaml
Notes:
    all:
        metadata:
            numbersections: false
    latex:
        metadata:
            numbersections: true
            fontsize: 12pt
        commandline:
            no-wrap: true
        filter:
            - run: deemph.py
        postflight:
            - run: latexmk.py
```

If panzer were run on the following document with the latex writer selected,

``` yaml
---
title: "My document"
author: John Smith
style: Notes
...
```

it would run pandoc with filter `deemph.py` and command line option `--no-wrap` on the following and then execute `latexmk.py`.

``` yaml
---
title: "My document"
author: John Smith
numbersections: true
fontsize: 12pt
...
```

Style overriding
----------------

Styles may be defined:

-   ‘Globally’ in `.yaml` files in `.panzer/styles/`
-   ‘Locally’ in `.yaml` files in `./styles/`)
-   ‘In document’ inside a `styledef` field in the document’s yaml metadata block

If no `.panzer/styles/` directory is found, panzer will look for global style definitions in `.panzer/styles.yaml` if it exists. If no `./styles/` directory is found, panzer will look for local style definitions in `./styles.yaml` if it exists.

Overriding among style settings is determined by the following rules:

| \#  | overriding rule                                                    |
|:----|:-------------------------------------------------------------------|
| 1   | Local style definitions override global style definitions          |
| 2   | In document style definitions override local style definitions     |
| 3   | Writer-specific settings override settings for `all`               |
| 4   | In a list, later styles override earlier ones                      |
| 5   | Children override parents                                          |
| 5   | Fields set outside a style definition override any style’s setting |

For fields that pertain to scripts/filters, overriding is *additive*; for other fields, it is *non-additive*:

-   For `metadata`, `template`, and `commandline`, if one style overrides another (say, a parent and child set `numbersections` to different values), then inheritance is non-additive, and only one (the child) wins.

-   For `preflight`, `filter`, `postflight` and `cleanup` if one style overrides another, then the ‘winner’ adds its items after those of the ‘loser’. For example, if the parent adds to `postflight` an item `-run: latexmk.py`, and the child adds `- run: printlog.py`, then `printlog.py` will be run after `latexmk.py`

-   To remove an item from an additive list, add it as the value of a `kill` field: for example, `- kill: latexmk.py`

Command line options trump style settings, and cannot be overridden by any metadata setting. Filters specified on the command line (via `--filter`) are run first, and cannot be removed. pandoc options set via panzer’s command line invocation override any set via `commandline`.

Multiple input files are joined according to pandoc’s rules. Metadata are merged using left-biased union. This means overriding behaviour when merging multiple input files is different from that of panzer, and always non-additive.

If fed stdin input, panzer buffers this to a temporary file in the current working directory before proceeding. This is required to allow preflight scripts to access the data. The temporary file is removed when panzer exits.

The run list
------------

Executables (scripts, filters, postprocessors) are specified by a list (the ‘run list’). The list determines what gets run when. Processes are executed from first to last in the run list. If an item appears as the value of a `run:` field, then it is added to the run list. If an item appears as the value of a `kill:` field, then any previous occurrence is removed from the run list. Killing an item does not prevent it from being added later. A run list can be completely emptied by adding the special item `- killall: true`.

Arguments can be passed to executables by listing them as the value of the `args` field of that item. The value of the `args` field is passed as the command line options to the external process. This value of `args` should be a quoted inline code span (e.g. `` "`--options`" ``) to prevent the parser interpreting it as markdown. Note that filters always receive the writer name as their first argument.

Example:

``` yaml
- filter:
    - run: setbaseheader.py
      args: "`--level=2`"
- postflight:
    - kill: open_pdf.py
- cleanup:
    - killall: true
```

The filter `setbaseheader.py` receives the writer name as its first argument and `--level=2` as its second argument.

When panzer is searching for a filter `foo.py`, it will look for:

| \#  | look for                                        |
|:----|:------------------------------------------------|
| 1   | `./foo.py`                                      |
| 2   | `./filter/foo.py`                               |
| 3   | `./filter/foo/foo.py`                           |
| 4   | `~/.panzer/filter/foo.py`                       |
| 5   | `~/.panzer/filter/foo/foo.py`                   |
| 6   | `foo.py` in PATH defined by current environment |

Similar rules apply to other executables and to templates.

The typical structure for the support directory `.panzer` is:

    .panzer/
        cleanup/
        filter/
        postflight/
        postprocess/
        preflight/
        template/
        shared/
        styles/

Within each directory, each executable may have a named subdirectory:

    postflight/
        latexmk/
            latexmk.py

Setting pandoc command line options
-----------------------------------

Arbitrary pandoc command line options can be set using metadata via `commandline`. `commandline` can appear outside a style definition and in a document’s metadata block, where it overrides the settings of any style.

`commandline` contains one field for each pandoc command line option. The field name is the unabbreviated name of the relevant pandoc command line option (e.g. `standalone`).

-   For pandoc flags, the value should be boolean (`true`, `false`), e.g. `no-wrap: true`.
-   For pandoc key-values, the value should be a quoted inline code span, e.g. `` include-in-header: "`path/to/my/header`" ``.
-   For pandoc repeated key-values, the value should be a list of inline code spans, e.g.

<!-- -->

    commandline:
        include-in-header:
            - "`file1.txt`"
            - "`file2.txt`"
            - "`file3.txt`"

Repeated key-value options in `comandline` are added after any provided from the command line. Overriding styles append to repeated key-value lists of the styles that they override.

`false` plays a special role. `false` means that the pandoc command line option with the field’s name, if set, should be unset. `false` can be used for both flags and key-value options (e.g. `include-in-header: false`).

Example:

    commandline:
        smart: true
        slide-level: "`3`"
        number-sections: false
        include-in-header: false

This passes the following options to pandoc `--smart --slide-level=3` and removes any `--number-sections` and `--include-in-header=...` options.

These pandoc command line options cannot be set via `commandline`:

-   `write`
-   `read`
-   `from`
-   `to`
-   `filter`
-   `template`
-   `output`
-   `variable`
-   `metadata`
-   `help`
-   `version`
-   `dump-args`
-   `ignore-args`
-   `print-default-template`
-   `print-default-data-file`

Passing messages to external processes
======================================

External processes have just as much information as panzer does. panzer sends its information to external processes via a json message. This message is sent over stdin to scripts (preflight, postflight, cleanup scripts), and embedded in the AST for filters. Postprocessors are an exception; they do not receive a json message (if you need it, you should probably be using a filter).

    JSON_MESSAGE = [{'metadata':    METADATA,
                     'template':    TEMPLATE,
                     'style':       STYLE,
                     'stylefull':   STYLEFULL,
                     'styledef':    STYLEDEF,
                     'runlist':     RUNLIST,
                     'options':     OPTIONS}]

-   `METADATA` is a copy of the metadata branch of the document’s AST (useful for scripts, not useful for filters)

-   `TEMPLATE` is a string with path to the current template

-   `STYLE` is a list of current style(s)

-   `STYLEFULL` is a list of current style(s) including all parents, grandparents, etc. in order of application

-   `STYLEDEF` is a copy of all style definitions employed in document

-   `RUNLIST` is a list of processes in the run list; it has the following structure:

        RUNLIST = [{'kind':      'preflight'|'filter'|'postprocess'|'postflight'|'cleanup',
                    'command':   'my command',
                    'arguments': ['argument1', 'argument2', ...],
                    'status':    'queued'|'running'|'failed'|'done'
                   },
                    ...
                    ...
                  ]

-   `OPTIONS` is a dictionary containing panzer’s and pandoc’s command line options:

        OPTIONS = {
            'panzer': {
                'panzer_support':  const.DEFAULT_SUPPORT_DIR,
                'debug':           str(),
                'quiet':           False,
                'strict':          False,
                'stdin_temp_file': str()   # tempfile used to buffer stdin
            },
            'pandoc': {
                'input':      list(),      # list of input files
                'output':     '-',         # output file; '-' is stdout
                'pdf_output': False,       # if pandoc will write a .pdf
                'read':       str(),       # reader
                'write':      str(),       # writer
                'options':    {'r': dict(), 'w': dict()}
            }
        }

    `options` contains the command line options with which pandoc is called. It consists of two separate dictionaries. The dictionary under the `'r'` key contains all pandoc options pertaining to reading the source documents to the AST. The dictionary under the `'w'` key contains all pandoc options pertaining to writing the AST to the output document.

Scripts read the json message above by deserialising json input on stdin.

Filters can read the json message by reading the metadata field, `panzer_reserved`, in the AST:

``` yaml
panzer_reserved:
    json_message: |
        ``` {.json}
        JSON_MESSAGE
        ```
```

this is visible to filters as the following json structure:

      "panzer_reserved": {
        "t": "MetaMap",
        "c": {
          "json_message": {
            "t": "MetaBlocks",
            "c": [
              {
                "t": "CodeBlock",
                "c": [ [ "", [ "json" ], [] ], "JSON_MESSAGE" ] } ] } } }

Receiving messages from external processes
==========================================

panzer captures stderr output from all executables. This is for pretty printing of info and errors. Scripts and filters should send json messages to panzer via stderr. If a message is sent to stderr that is not correctly formatted, panzer will print it verbatim prefixed by a ‘!’.

The json message that panzer expects is a newline-separated sequence of utf-8 encoded json dictionaries, each with the following structure:

    { 'level': LEVEL, 'message': MESSAGE }

-   `LEVEL` is a string that sets the error level; it can take one of the following values:

        'CRITICAL'
        'ERROR'
        'WARNING'
        'INFO'
        'DEBUG'
        'NOTSET'

-   `MESSAGE` is a string with your message

Compatibility
=============

panzer accepts pandoc filters. panzer allows filters to behave in two new ways:

1.  Filters can take more than one command line argument (first argument still reserved for the writer).
2.  A `panzer_reserved` field is added to the AST metadata branch with goodies for filters to mine.

The follow pandoc command line options cannot be used with panzer:

-   `--dump-args`
-   `--ignore-args`
-   `--print-default-template`, `-D`
-   `--print-default-data-file`
-   `--version`, `-v`
-   `--help`, `-h`

Reserved fields
===============

The following metadata fields are reserved for use by panzer:

-   `styledef`
-   `style`
-   `template`
-   `preflight`
-   `filter`
-   `postflight`
-   `postprocess`
-   `cleanup`
-   `commandline`
-   `panzer_reserved`

The pandoc writer name `all` is also occupied.

Known issues
============

Pull requests welcome:

-   Slower than I would like (calls to subprocess slow in Python)
-   Calls to subprocesses (scripts, filters, etc.) block ui
-   No Python 2 support

Similar
=======

-   <https://github.com/balachia/panopy>

Release notes
=============

-   1.0 (21 July 2015)
    -   first release!
    -   updated documentation
    -   new: `---strict` option to address https://github.com/msprev/panzer/issues/10
    -   refactored `commandline` implementation
    -   multiple fixes
-   1.0b2 (23 May 2015):
    -   new: `commandline` - set arbitrary pandoc command line options via metadata
-   1.0b1 (14 May 2015):
    -   initial release
