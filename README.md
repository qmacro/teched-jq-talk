# teched-jq-talk

These are the notes and code snippets relating to a Community Talk at SAP TechEd 2023 in Bengaluru: [Handle JSON like a boss with jq](https://go2.events.sap.com/TechEd2023/agb/go/agendabuilder.sessions/?l=326&sid=121489&schid=521858&locale=en_US).

## Introduction

JSON is everywhere. Configuration, output from countless tools & APIs, and more. It's a well defined and well understood data interchange format, with a small but perfectly formed number of valid types and values (described in [Introducing JSON](https://www.json.org/json-en.html)), and in particular the `object` and `array` types in combination make it easy to represent both simple and complex data structures.

Not only that, but it's supported by many systems and languages, either natively or by means of libraries.

While it's straightforward to parse JSON in a script written in the language of your choice, there's a lot of costly ceremony getting the JSON into the script.

Typically one might retrieve the JSON first and write it to a file. And then in a second step one would run the script to read in the file and parse the contents.

This is a lot of unnecessary work and the approach doesn't lend itself to typical pipeline style workflows. And if you want to adapt your script to read from STDIN (standard input), like all well behaved command line tools, actually getting your script to read from STDIN is likely to be more unwieldy than you think.

Instead, why not use a tool that is:

* dedicated to parsing and manipulating JSON
* ready to be used in a pipeline, naturally reading from STDIN and writing to STDOUT
* simple to get started with
* capable enough to deal with anything you might need to do

This tool, this language, is [jq](https://jqlang.github.io/jq/). It's described as "a lightweight and flexible command-line JSON processor" but in reality it's actually a full blown Turing-complete functional language with an emphasis on streams.

Not only is jq a far more appropriate tool to work with JSON in pipelines and in general, it's so pervasive that it's even built into some other tools, to provide a convenient way of controlling the output.

The [GitHub CLI](https://cli.github.com/) is one example (see [gh formatting](https://cli.github.com/manual/gh_help_formatting) for details).

Here's an example of using the built-in jq feature in the GitHub CLI. First, this command will return a JSON value that is very large and complex (an array of objects, each one representing the intricate details of an issue, all from the specified repository):

> A "JSON value" is any value or construct that is valid JSON. This can be a simple double quoted string, a number, a boolean, the null value, or an array (` [...] `) or object (`{ ... }`) containing any of these values or constructs. So for example, all of these are valid JSON values: `"hello world"`, `42`, `true`, `null`, `[1, 2, "three"]`, `{"ID": "C11", "fib": [1, 1, 2, 3, 5, 8]}`.

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

Note that gh effectively executes your jq expression in the context of what is jq's "raw output" mode, where strings are emitted without the enclosing double-quotes. In other words, raw as in "not valid JSON". This is why the strings that are produced here are not enclosed.

> The [url-notes](https://github.com/qmacro-org/url-notes) repo is where I collect my 'to-read' items, make notes on them, and [publish any such notes](https://github.com/qmacro-org/url-notes/blob/main/.github/workflows/toot-url-note.yml) when I close the issue representing the item. There's even a [feed](https://raw.githubusercontent.com/qmacro-org/url-notes/main/feed.xml) maintained, via a [jq script](https://github.com/qmacro-org/url-notes/blob/main/genfeed.jq).

### A small digression on streaming and generators

It might help at this point already to jump into a core aspect of what's really at the heart of jq, and even this simple example gives us that opportunity.

To understand what is meant (in part) by the reference to jq as having a focus on streaming, consider the jq expression used in this example: `.[].title`. This is in fact shorthand, or idiomatic, for the more verbose `.[] | .["title"]`. Let's briefly consider what happens here.

We start with the [array/object value iterator](https://jqlang.github.io/jq/manual/#array-object-value-iterator) `.[]`. Think of this as an extreme form of something like `.[1]` which in turn you can think of as:

* `.`: the current value at this point in the stream (an array, for example)
* `[1]`: the element of that array with index `1` (i.e. the second element)

So it sort of fits to think of `.[]` as "all elements". The interesting thing about this mechanism is that (a) it also works on objects (giving all the values of the object), and (b) it is a generator, i.e. emits multiple values, effectively causing a bifurcation of the value stream.

If the JSON coming into `.[]|.["title"]` were as follows:

```json
[
    {"title": "A"},
    {"title": "B"},
    {"title": "C"}
]
```

and we allow ourselves a little artistic licence to be over effusive with the expression so that it becomes `. | .[] | .["title"]` (still effectively the same as `.[].title`), we can visualise what happens:

```text
          .          |   .[]                    |   .["title"]

[                         +--> {"title": "A"}       ---> "A" 
    {"title": "A"},       |
    {"title": "B"},     --+--> {"title": "B"}       ---> "B"
    {"title": "C"}        |
]                         +--> {"title": "C"}       ---> "C"
```

Being a generator, `.[]` emits multiple values, initiating multiple parallel streams that flow downstream, i.e. through the pipe operator that follows it. 

Sure enough, this is what happens:

```shell
; echo '[{"title":"A"},{"title":"B"},{"title":"C"}]' | jq '. | .[] | .["title"]'
"A"
"B"
"C"
```

## Using jq

Think of jq as just another command line tool in your toolbox, that can be used neatly in a pipeline, that reads from STDIN, and outputs to STDOUT. And as it does so it will, [unless you tell it otherwise](https://jqlang.github.io/jq/manual/#invoking-jq), expect to:

* read JSON values as input
* emit JSON values as output

And when emitting such JSON values, it will, again, unless you tell it otherwise, pretty print those values, so that there's enough whitespace indentation for a human to be able to read it.

## A simple start

Let's start with some output from the Cloud Foundry CLI, cf. With this CLI you can also call [CF APIs](https://v3-apidocs.cloudfoundry.org/version/3.149.0/index.html). The responses are in JSON format. Here's an example:

```shell
cf curl /v3/info
```

The output looks something like this, not particularly readable:

```text
{"build":"v32.11.0","cli_version":{"minimum":"","recommended
":""},"custom":{},"description":"SAP BTP Cloud Foundry envir
onment","name":"cf-deployment","version":32,"links":{"self":
{"href":"https://api.cf.eu10.hana.ondemand.com/v3/info"},"su
pport":{"href":""}}}
```

Often the first expression used by someone new to jq is the [identity filter](https://jqlang.github.io/jq/manual/#identity), `.`, on its own. This filter takes its input ... and outputs the very same value. 

Useless? [Far from it](https://en.wikipedia.org/wiki/Identity_function), but that's a discussion from another time.

Given what we now know about how jq behaves by default, passing JSON through such a simple filter with jq has the _side effect_ of pretty printing that JSON:

```shell
cf curl /v3/info | jq '.'
```

What we see now is:

```json
{
  "build": "v32.11.0",
  "cli_version": {
    "minimum": "",
    "recommended": ""
  },
  "custom": {},
  "description": "SAP BTP Cloud Foundry environment",
  "name": "cf-deployment",
  "version": 32,
  "links": {
    "self": {
      "href": "https://api.cf.eu10.hana.ondemand.com/v3/info"
    },
    "support": {
      "href": ""
    }
  }
}
```

Much better!

> The jq expression (or script) is often supplied in single quotes as here (`cf curl /v3/info | jq '.'`). In this particular case, the single quotes could have been omitted (`cf curl /v3/info | jq .`) and indeed the identity function itself (`cf curl /v3/info | jq`), as it is what is executed if nothing is specified. But it's good practice to be explicit, and to always use single quotes.

## Continuing with simple filters

It's simple to get values from JSON, and emit a different JSON structure. Based on the JSON above, here are a few different ways to do that. These examples show both the command entered at the shell prompt (indicated with `;`), and the output.

First, emitting values for a couple of properties:

```shell
; cf curl /v3/info | jq '.build, .name'
"v32.11.0"
"cf-deployment"
```

> Think of `.build` as `.` plus `build` i.e. whatever the identity function emits (all of the JSON, at this stage) then the specification for the `build` property. It's shorthand for `.["build"]`.

Here we emit values for three properties, but enclosed in an array:

```shell
; cf curl /v3/info | jq '[.build, .version, .description]'
[
  "v32.11.0",
  32,
  "SAP BTP Cloud Foundry environment"
]
```

We can provide default values for when there is none:

```shell
; cf curl /v3/info \
  | jq '.links.docu.href // "https://help.sap.com/docs/btp/sap-business-technology-platform/"'
"https://help.sap.com/docs/btp/sap-business-technology-platform/"
```

Introspection is also possible:

```shell
; cf curl /v3/info | jq '.cli_version | keys'
[
  "minimum",
  "recommended"
]
```

It's easy to create a reduced object with just a couple of properties:

```shell
; cf curl /v3/info | jq '{ build, version }'
{
  "build": "v32.11.0",
  "version": 32
}
```

> Note the shorthand of just using the property names here, rather than what you might expect to have to write, i.e. `{ "build": .build, "version": .version }`. Note also that in the recently released version 1.7 of jq, there's [pick](https://jqlang.github.io/jq/manual/v1.7/#pick), a new builtin that will emit a projection of the input object or array - see the language changes section in the [jq 1.7 release notes](https://github.com/jqlang/jq/releases/tag/jq-1.7).

We can also add new properties. Extending the previous example:

```shell
; cf curl /v3/info | jq '{ build, version, answer: .version + 10 }'
{
  "build": "v32.11.0",
  "version": 32,
  "answer": 42
}
```

## Interactive jq

For subsequent examples, I'd recommend you use [ijq](https://sr.ht/~gpanders/ijq/), which brings a clean and simple UI to your jq explorations. Two main windows are displayed, with the source JSON on the left ("Input"), and whatever is emitted from your jq expression on the right ("Output"). At the bottom is where you edit your jq expression ("Filter"), along with a space to display any error messages ("Error").

![an interactive jq session in action](https://git.sr.ht/~gpanders/ijq/blob/HEAD/demo/ijq.gif)

You can of course continue to use jq on the command line, or even write your jq expression or program in a file and execute it with the `--from-file` (`-f`) option.

## More constructs

Let's move on to some more useful constructs, so that you know how to bring conditional processing into the mix, and filter out data based on comparisons. For this, we'll look at some different data. 

### Region data via the btp CLI

The [SAP BTP Command Line Interface (btp CLI)](https://cpcli.cf.eu10.hana.ondemand.com/) is a great CLI program that anyone working with the SAP Business Technology Platform needs in their toolbox. It allows the reporting, inspection and management of resources on SAP BTP from the comfort of the command line and within scripts. 

> See [Managing resources on SAP BTP – what tool do I choose?](https://blogs.sap.com/2022/12/12/managing-resources-on-sap-btp-what-tool-do-i-choose/) for more information on where the btp CLI fits in.

The btp CLI can emit for humans, or for machines (or scripts or further processing in a normal UNIX style pipeline). The output format for machines is JSON, and is requested with the option `--format json`.

Taking the information about geographical regions (data centre locations), we can ask for that information and get human readable output like this:

```shell
; btp list accounts/available-region

Showing available regions for global account 275320f9-4c26-4622-8728-b6f519607542:

region   data center   environment    provider
ap21     cf-ap21       cloudfoundry   AZURE
br1      neo-br1       neo            SAP
cn1      neo-cn1       neo            SAP
us30     cf-us30       cloudfoundry   GCP
...
```

Adding `--format json` like this:

```shell
btp --format json list accounts/available-region
```

gives us something we can dig into programmatically (output here deliberately limited to the first two regions, for brevity):

```json
{
  "datacenters": [
    {
      "name": "cf-ap21",
      "displayName": "Singapore - Azure",
      "region": "ap21",
      "environment": "cloudfoundry",
      "iaasProvider": "AZURE",
      "supportsTrial": true,
      "provisioningServiceUrl": "https://provisioning-service.cfapps.ap21.hana.ondemand.com",
      "saasRegistryServiceUrl": "https://saas-manager.cfapps.ap21.hana.ondemand.com",
      "domain": "ap21.hana.ondemand.com",
      "isMainDataCenter": true,
      "geoAccess": "BACKWARD_COMPLIANT_EU_ACCESS",
      "restricted": false
    },
    {
      "name": "neo-br1",
      "displayName": "Brazil (São Paulo)",
      "region": "br1",
      "environment": "neo",
      "iaasProvider": "SAP",
      "supportsTrial": false,
      "provisioningServiceUrl": "https://cisservices.br1.hana.ondemand.com/com.sap.core.commercial.service.web",
      "domain": "br1.hana.ondemand.com",
      "isMainDataCenter": true,
      "geoAccess": "STANDARD",
      "restricted": false
    }
  ]
}
```

A larger version of this JSON data is available in the file [available-regions.json](./available-regions.json) and is what we'll use for the following examples (for speed and minimal load on the API endpoint that the btp CLI is calling for us).

### Looking at the shape of the data

Before diving in, let's have a look at the shape of the data itself.

What's the actual (outermost) JSON value here?

```shell
; jq 'type' available-regions.json
"object"
```

> Note that even here, jq endeavours to emit valid JSON, so we get `"object"` rather than just `object`. And for more understanding that only comes from [staring](https://qmacro.org/blog/posts/2017/02/19/the-beauty-of-recursion-and-list-machinery/#initialrecognition), see the later [digression on JSON values and streaming](#a-digression-on-json-values-and-streaming) for something to think about in relation to this simple example.

Next, let's look at the properties (keys) of that object.

```shell
; jq 'keys' available-regions.json
[
  "datacenters"
]
```

OK so we have an object with a single property, what is its type?

```shell
; jq '.datacenters | type' available-regions.json
"array"
```

It's an array (which we can confirm visually by looking at the JSON shown earlier).

In fact we could do this in one go with the [map_values](https://jqlang.github.io/jq/manual/#map-map_values) function, which can operate on objects or arrays. In this case, we'll get it to operate on the entire JSON value, which is an object as we've already determined.

```shell
; jq 'map_values(type)' available-regions.json
{
  "datacenters": "array"
}
```

Nice! We've just called our first function, which expects a single argument, which is an expression that is invoked upon each of the values (the semantics of the "map" part of this function name are strong and relevant here; see [FOFP 1.4 A different approach with map](https://qmacro.org/blog/posts/2016/05/03/fofp-1.4-a-different-approach-with-map/) for more on map).

> In fact, if you were typing this into the filter box in ijq, and had got to entering just the name of the function `map_values`, you might have seen this in the error box:
> 
> "jq: error: map_values/0 is not defined"
> 
> This means "you're invoking `map_values` but passing nothing to it, and there isn't a version of `map_values` that takes zero arguments" (that's the `map_values/0` reference). The function exists as `map_values/1`. It's useful to recognise and be comfortable with this nomenclature, as it's used a lot in the jq world.

Let's go one small step further now we have some confidence in passing arguments to functions, and do this:

```shell
; jq 'map_values("\(type) with \(length) elements")' available-regions.json
{
  "datacenters": "array with 33 elements"
}
```

This time we're passing a string as the argument to `map_values`. This string has expressions embedded in it via jq's [string interpolation](https://jqlang.github.io/jq/manual/#string-interpolation) feature `\(...)`. One of the expressions embedded is `type` which we've seen before. The other is `length`, another builtin function that [emits the length of various types of value](https://jqlang.github.io/jq/manual/#length). The interesting thing here is perhaps not the string interpolation itself, but what each of `type` and `length` operates upon. As they're in the context of `map_values`, they operate on each of the values of the properties in turn, in this case, just the singular `datacenters` property with its array value.

### A digression on JSON values and streaming

Earlier in this section, we asked the question (of the JSON in [available-regions.json](./available-regions.json)) "what's the actual (outermost) JSON value here?". We asked it like this: `jq 'type' available-regions.json` and got the answer: `"object"`. There was an assumption implied in this simple question that jq is only happy processing JSON input where that JSON is effectively a single value or type at the outermost level.

And as far as the data we have is concerned, that input context holds true, in that there's a single outermost element, which is an object:

```json
{
    "datacenters": []
}
```

But what would happen if our data looked like this:

```json
{ "day": "Monday" }
{ "day": "Tuesday" }
{ "day": "Wednesday" }
```

That's not _a_ valid JSON value, that's a sequence of _three_ valid JSON values (and there's no "outermost" element of which to speak).

What happens if we pass [a file with this exact content](./three-values.json)?

```shell
; jq 'type' three-values.json
"object"
"object"
"object"
```

This is the elegance of the streaming nature of jq. It will invoke the filter (the expression we pass in single quotes -- i.e. `type` in this example -- or in a file with `--from-file`) on each of the JSON values it sees.

And for a digression upon a digression - what if you wanted to process these three discrete JSON values (the objects for Monday, Tuesday and Wednesday) in the context of a single execution of your jq expression, i.e. have them all read in and processed together? That's what the `--slurp` (`-s`) option does for us. Observe:

```shell
; jq --slurp 'type' three-values.json
"array"
```

Here's what the jq manual says about this option: "Instead of running the filter for each JSON object in the input, read the entire input stream into a large array and run the filter just once". To make sure we understand what this does exactly, we can just use the identity function:

```shell
; jq --slurp '.' three-values.json
[
  {
    "day": "Monday"
  },
  {
    "day": "Tuesday"
  },
  {
    "day": "Wednesday"
  }
]
```

You can see that slurping encloses all the JSON values in an single outermost array.

> The whitespace is different merely because of how jq's pretty-printing works. By the way, there's also a `--compact-output` (`-c`) option, that if added to the invocation above, will produce this instead: `[{"day":"Monday"},{"day":"Tuesday"},{"day":"Wednesday"}]`.

One more thing - slurp mode doesn't require the individual JSON values to be of the same type or shape:

```shell
; echo -e '{"answer": 42}\nfalse\n[1,2,3]' | jq --slurp '.'
[
  {
    "answer": 42
  },
  false,
  [
    1,
    2,
    3
  ]
]
```

### Trying out some more jq functions

OK, back to the available region data. From a glance at the objects within the array that the `datacenters` property has as its value, we can see that there are different IaaS providers. What are they?

By the way, the next and subsequent examples will just show the jq expressions, rather than within the context of the pipeline or within ijq. So to execute what you see yourself, do this:

```shell
jq '<the jq expression shown>' available-regions.json
```

or use ijq and type them into the filter input box.

#### Calculating distinct values

First, let's just list them all; let's just have the value of the `iaasProvider` property from each of the objects. We already know how to do this:

```jq
.datacenters[].iaasProvider
```

This produces a long list that starts like this:

```json
"AZURE"
"SAP"
"SAP"
"GCP"
"SAP"
"AWS"
```

There's a [unique](https://jqlang.github.io/jq/manual/#unique-unique_by) function that "takes as input an array and produces an array of the same elements, in sorted order, with duplicates removed". Sounds good. Let's try it:

```jq
.datacenters[].iaasProvider | unique
```

Hmm, we get an error: "Cannot iterate over string ("AZURE")". The problem is that `unique` expects an array as input. And what do we have here? A list of discrete JSON values, each of which are strings. (So this error message makes perfect sense - jq was attempting to call `unique` on each of the string values emitted from `.datacenters[].iaasProvider`, and abended on the first one "AZURE".

> Abend is an old word from my IBM mainframe days and is a verb made from the contraction of "abnormal end".

One way to address this, and feed `unique` what it expects, is to construct an array manually, by using [array construction](https://jqlang.github.io/jq/manual/#array-construction), i.e. by wrapping the `.datacenters[].iaasProvider` in `[ ... ]`.

This:

```jq
[ .datacenters[].iaasProvider ]
```

gives us:

```json
[
  "AZURE"
  "SAP"
  "SAP"
  "GCP"
  "SAP"
  "AWS"
]
```

which we can then feed to `unique`:

```jq
[ .datacenters[].iaasProvider ] | unqiue
```

which then emits:

```json
[
  "AWS"
  "AZURE"
  "GCP"
  "SAP"
]
```

This is what we were looking for - a list of the different IaaS providers.

Of course, jq is a wonderfully expressive language, and in the spirit of [TMTOWTDI](https://en.wiktionary.org/wiki/TMTOWTDI) ("There's More Than One Way To Do It", a sentiment, expression & philosphy that originated in the Perl programming community), we can take a slightly different approach using [map](https://jqlang.github.io/jq/manual/#map-map_values):

```jq
.datacenters | map(.iaasProvider) | unqiue
```

This produces the same output. As `map` operates on an array, we feed in the value of the `datacenters` property directly to it, rather than use the array/object value iterator (`[]`) to explode the data into multiple values downstream. As `map` not only takes an array as input but produces an array as output, this means that it's an array that reaches `unique` through the final pipe:

```text
    array    -->       array         -->  array
.datacenters  |  map(.iaasProvider)   |   unqiue
```

Let's have a look at another way, using the related [unique_by](https://jqlang.github.io/jq/manual/#unique-unique_by) function, which "will keep only one element for each value obtained by applying the argument":

```jq
.datacenters | unique_by(.iaasProvider) | map(.iaasProvider)
```

It's worth trying this out in ijq, to see what the intermediate result is, produced by `.datacenters | unique_by(.iaasProviders)`. If you do, you'll see an array of four elements, each one representing a data centre from a different IaaS provider.

#### Filtering

How about retrieving location information for those data centres from a specific provider? While we don't have definitive geographic data in the data centre objects, we can see that the `displayName` property contains what we can use. Here's an example:

```json
{
  "name": "cf-ap21",
  "displayName": "Singapore - Azure",
  "region": "ap21",
  "environment": "cloudfoundry",
  "iaasProvider": "AZURE",
  "supportsTrial": true,
  "provisioningServiceUrl": "https://provisioning-service.cfapps.ap21.hana.ondemand.com",
  "saasRegistryServiceUrl": "https://saas-manager.cfapps.ap21.hana.ondemand.com",
  "domain": "ap21.hana.ondemand.com",
  "isMainDataCenter": true,
  "geoAccess": "BACKWARD_COMPLIANT_EU_ACCESS",
  "restricted": false
}
```

We can take whatever comes before any " - " divider in that value ("Singapore" in this example).

To filter, we can use the [select](https://jqlang.github.io/jq/manual/#select) function, which will cause JSON data passing through it to be dropped if the expression passed to it does not end up evaluating to `true`.

Taking it step by step, this jq filter:

```jq
.datacenters[] | select(.iaasProvider == "AWS") | .displayName
```

gives us this:

```json
"Europe (Frankfurt) - AWS"
"Japan (Tokyo)"
"Brazil (São Paulo)"
"Australia (Sydney)"
"South Korea (Seoul) - AWS"
"Singapore"
"US East (VA) - AWS"
"Canada (Montreal)"
"Europe (Frankfurt)"
```

Now for a bit of string manipulation, using a regexp-based substitution, to remove any " - ..." parts:

```jq
.datacenters[] 
| select(.iaasProvider == "AWS") 
| .displayName 
| sub(" - .+$";"")
```

> As you can see, this jq filter is getting a little long to be displayed well on a single line so some extra whitespace has been added.

This produces:

```json
"Europe (Frankfurt)"
"Japan (Tokyo)"
"Brazil (São Paulo)"
"Australia (Sydney)"
"South Korea (Seoul)"
"Singapore"
"US East (VA)"
"Canada (Montreal)"
"Europe (Frankfurt)"
```

Great. Again, invoking TMTOWDI, the last part could have been done another way; if you don't feel comfortable with regular epressions, this would have worked just as well, and produced the same result:

```jq
.datacenters[] 
| select(.iaasProvider == "AWS") 
| .displayName 
| split(" - ")
| first
```

As you can guess, [split](https://jqlang.github.io/jq/manual/#split-1), more specifically `split/1`, will create an array of values from a string split on the value given as argument. And `first` is sort of syntactic sugar for `.[0]`, and far nicer to write and think about.

### Grouping

Related to determining distinct values is the common requirement of organising data into clusters, based on some sort of value.

As the final example using this available region information, let's find what the geographic access looks like across the different locations.

Each object representing a data centre has a `geoAccess` property, and we can see with:

```jq
.datacenters|map(.geoAccess)|unique
```

that there are three different values:

```json
[
  "BACKWARD_COMPLIANT_EU_ACCESS",
  "EU_ACCESS",
  "STANDARD"
]
```

So what does the distribution of locations look like across these different access types? For this, the [group_by](https://jqlang.github.io/jq/manual/#group_by) function is useful.

To properly grasp how this works, it's important to be able to think about the shape of the data at the input but more importantly at the output. Let's first take a simpler data example. We have a file [fruit.json](./fruit.json) with a list of JSON values, each one an object:

```json
{ "name": "apple", "colour": "green" }
{ "name": "banana", "colour": "yellow" }
{ "name": "strawberry", "colour": "red" }
{ "name": "kiwi", "colour": "green" }
{ "name": "pear", "colour": "green" }
{ "name": "lemon", "colour": "yellow" }
```.

> This is the second time we've used data like this. In fact, there's a name for this format, and it's [JSON Lines](https://jsonlines.org/), aka "newline delimited JSON".

So `group_by` takes an array as input, and produces an array of arrays as output, which means we'll have to slurp in the objects (with `--slurp` or `-s`) and then stare at the shape of the output to make sure we're comfortable with it

```shell
; jq -s 'group_by(.colour)' fruit.json
[
  [
    {
      "name": "apple",
      "colour": "green"
    },
    {
      "name": "kiwi",
      "colour": "green"
    },
    {
      "name": "pear",
      "colour": "green"
    }
  ],
  [
    {
      "name": "strawberry",
      "colour": "red"
    }
  ],
  [
    {
      "name": "banana",
      "colour": "yellow"
    },
    {
      "name": "lemon",
      "colour": "yellow"
    }
  ]
]
```

We can perhaps map the `length` function over the elements of the outermost array that is produced by `group_by` to help our understanding:

```shell
; jq -s 'group_by(.colour) | map(length)' fruit.json
[
  3,
  1,
  2
]
```

This shows us that the first subarray has 3 elements, the second subarray has 1 element, and the third subarray has 2 elements.

As a question for you to ponder: if we were to nest a `map`, can you understand what happens and explain the output? Like this:

```shell
; jq -s 'group_by(.colour) | map(map(length))' fruit.json
jq -s 'group_by(.colour) | map(map(length))' fruit.json
[
  [
    2,
    2,
    2
  ],
  [
    2
  ],
  [
    2,
    2
  ]
]
```

To answer this question, it might help to ask "what are we mapping over?".

Anyway, if we now get back to the available region information, let's perform a similar filter, like this:

```jq
.datacenters | group_by(.geoAccess)
```

This gives us a array of arrays too, of course. What if we just want a summary, listing the names of the regions, grouped by the different geographic access types?

```jq
.datacenters
| group_by(.geoAccess)
| map({ (first.geoAccess): map(.name) })
```

This will give us:

```json
[
  {
    "BACKWARD_COMPLIANT_EU_ACCESS": [
      "cf-ap21",
      "cf-eu10",
      "cf-jp10",
      "neo-eu2",
      "neo-eu1",
      "cf-ap20",
      "cf-br10",
      "cf-ap10",
      "cf-ap12",
      "cf-ap11",
      "cf-jp20",
      "cf-eu20",
      "cf-us10",
      "cf-ca10",
      "cf-us20",
      "neo-eu3",
      "cf-us21"
    ]
  },
  {
    "EU_ACCESS": [
      "cf-ch20",
      "cf-eu11"
    ]
  },
  {
    "STANDARD": [
      "neo-br1",
      "neo-cn1",
      "cf-us30",
      "neo-ap1",
      "neo-ca1",
      "neo-ae1",
      "neo-us3",
      "neo-us2",
      "neo-sa1",
      "neo-jp1",
      "cf-eu30",
      "cf-in30",
      "neo-us1",
      "neo-us4"
    ]
  }
]
```

It's worth unpacking this filter to properly understand what happened here. Let's run the equivalent filter on our fruit data.

```shell
; jq -s 'group_by(.colour) | map({ (first.colour): map(.name) })' fruit.json
[
  {
    "green": [
      "apple",
      "kiwi",
      "pear"
    ]
  },
  {
    "red": [
      "strawberry"
    ]
  },
  {
    "yellow": [
      "banana",
      "lemon"
    ]
  }
]
```

To work through this filter step by step:

First, the `group_by(.colour)` part creates an array of arrays, with one sub array for each of the list of fruit objects corresponding to a particular colour (we've seen this output before):

```json
[
  [
    {
      "name": "apple",
      "colour": "green"
    },
    {
      "name": "kiwi",
      "colour": "green"
    },
    {
      "name": "pear",
      "colour": "green"
    }
  ],
  [
    {
      "name": "strawberry",
      "colour": "red"
    }
  ],
  [
    {
      "name": "banana",
      "colour": "yellow"
    },
    {
      "name": "lemon",
      "colour": "yellow"
    }
  ]
]
```

Then `map` is run over this array of arrays, evaluating this expression:

```jq
{ (first.colour): map(.name) }
```

for each of the sub arrays. Let's take the first sub array and see what this does. Here's that first sub array:

```json
[
  {
    "name": "apple",
    "colour": "green"
  },
  {
    "name": "kiwi",
    "colour": "green"
  },
  {
    "name": "pear",
    "colour": "green"
  }
]
```

We can see from the outermost `{ ... }` ([object construction](https://jqlang.github.io/jq/manual/#object-construction)) of the expression that an object will be emitted. And in fact there will only be a single property in this object, the name and value for which are both calculated:

* via `(first.colour)`: the name is the value of the `colour` property of the `first` element in that sub array; in this case, "green"
* via `map(.name)`: the value is an array (produced by `map`) of the values of the `name` property of each of the elements; in this case, "apple", "kiwi" and "pear"

In other words:

```json
{ "green": ["apple", "kiwi", "pear"] }
```

Note that we want the expression `first.colour` to be evaluated, so we need to put it in brackets when using it as a property name in object construction, i.e. `(first.colour)`.
