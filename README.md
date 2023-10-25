# teched-jq-talk

These are the notes and code snippets relating to a Community Talk at SAP TechEd 2023 in Bengaluru: [Handle JSON like a boss with jq](https://go2.events.sap.com/TechEd2023/agb/go/agendabuilder.sessions/?l=326&sid=121489&schid=521858&locale=en_US).

## Introduction

JSON is everywhere. Configuration, output from countless tools & APIs, and more. It's a well defined and well understood data interchange format, with a limited number of valid types and values (described in [Introducing JSON](https://www.json.org/json-en.html)), and in particular the `object` and `array` types in combination make it easy to represent both simple and complex data structures.

Not only that, but it's supported by many systems and languages, either natively or by means of libraries.

While it's straightforward to parse JSON in a script written in the language of your choice, there's a lot of costly ceremony getting the JSON to the script.

Typically one might retrieve the JSON first and write it to a file. And then in a second step one would run the script to read in the file and parse the contents.

This is a lot of unnecessary work and the approach doesn't lend itself to typical pipeline style workflows. And if you want to adapt your script to read from STDIN (standard input), like all well behaved command line tools, actually getting your script to read from STDIN is likely to be more unwieldy than you think.

Instead, why not use a tool that is:

* dedicated to parsing and manipulating JSON
* ready to be used in a pipeline, naturally reading from STDIN and writing to STDOUT
* simple to get started with

This tool, this language, is [jq](https://jqlang.github.io/jq/). It's described as "a lightweight and flexible command-line JSON processor" but in reality it's actually a full blown Turing-complete functional language with an emphasis on streams.

Not only is jq a far more appropriate tool to work with JSON in pipelines and in general, it's so pervasive that it's even built into some other tools, to provide a convenient way of controlling the output. The [GitHub CLI](https://cli.github.com/) is one example (see [gh formatting](https://cli.github.com/manual/gh_help_formatting) for details).

Here's an example of using the built-in jq feature in the GitHub CLI. First, this command will return a JSON value that is very large and complex (an array of objects, each one representing the intricate details of an issue, all from the specified repository):

> A "JSON value" is any value or construct that is valid JSON. This can be a simple double quoted string, a number, a boolean, the null value, or an array (` [...] `) or object (`{ ... }`) containing any of these values or constructs. So for example, all of these are valid JSON values: `"hello world"`, `42`, `true`, `null`, `[1, 2, "three"]`, `{"ID": 4711}`.

```shell
gh api repos/qmacro-org/url-notes/issues
```

Adding a jq expression via the `--jq` flag here allows us, for example, to ask for just the issue titles:

```shell
gh api repos/qmacro-org/url-notes/issues --jq '.[].title'
```

This produces:

```text
LSP could have been better
Vim: you don't need NERDtree or (maybe) netrw | George Ornbo
" [31m"?! ANSI Terminal security in 2023 and finding 10 CVEs
A Brief Introduction of ActivityPub: The Future of Social Networks | HackerNoon
Picat is my favorite new toolbox language • Buttondown
Conventional Comments
How many ways can you slice a URL and name the pieces? - Tantek
```

Note that gh effectively executes your jq expression in the context of what is jq's "raw output" mode, where strings are emitted without the enclosing double-quotes. In other words, raw as in "not valid JSON". This is why the strings output here are not enclosed.

> The [url-notes](https://github.com/qmacro-org/url-notes) repo is where I collect my 'to-read' items, make notes on them, and [publish any such notes](https://github.com/qmacro-org/url-notes/blob/main/.github/workflows/toot-url-note.yml) when I close the issue representing the item. There's even a [feed](https://raw.githubusercontent.com/qmacro-org/url-notes/main/feed.xml) maintained, via a [jq script](https://github.com/qmacro-org/url-notes/blob/main/genfeed.jq).

