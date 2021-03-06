---
title: "Practical Cue Example, Produce YAML Config from Go Types"
date: 2022-01-31T09:57:26-07:00
tags:
 - cue
 - go
categories:
 - programming
---

I recently had to spend some time to learn [Cue](https://cuelang.org/) and felt the examples/tutorials were a bit lacking 
and the paradigm shift to the entire language sort of messed with my head and some things Cue does were a bit foreign to me. 
To get started, note that the two places I spent all of my time to get this information came from their main [docs](https://cuelang.org/docs/) 
and a community [Cuetorials](https://cuetorials.com/) site. This will be a tutorial on how you can use Cue to generate
two different YAML documents using Cue all starting with Go types to generate Cue files.

As Cue changes this could become out of date, this was all done using Cue **v0.4.1**. All code for this tutorial can be
found in this [github repository](https://github.com/luthermonson/tutorials/tree/main/2022/01/cue).

### Prerequisites
You'll need to [install](https://cuelang.org/docs/install/) Cue and I recommend you get some basic understanding of the 
cli commands as there is some power there for your project. I also recommend you get used to 
[modules](https://cuelang.org/docs/concepts/packages/#modules) and the `mod` command and the directory structure it will 
create for you. Note the Cue takes a lot of it's packaging and generating inspiration from [Go](https://go.dev/) itself so
if you've already done some of this in Go you may already recognize some of these features.

### Go Types to Config YAML
Here is a simple scenario we will use to setup this example, we are building a server daemon in Go with a 
`--config` flag which is a file path to a YAML configuration document for database drive application with host/port configurations
along with a worker pool setting for processing a work queue.

Sample Config:
```
app:
    host: mydomain.com
    post: 443
    workers: 2
    db-dsn: root:test@tcp(mysql.domain.com:3306)/test?charset=utf8&parseTime=True&loc=Local
```

This idea is actually pretty simple from Cue's perspective if you write this up by hand but what makes your life
easier is if the Go types generate your Cue files so you can import the types into your app and consume your generated config 
file via unmarshalling.

### Setting Up the Cue Module
To get this project setup you'll probably want to use a sub-directory in your project and intialize it as a cue module and 
place a go file with some types at the root.

```bash
$ cd config
$ cue init mod github.com/username/project/config
```

And put your types into the types.go file

```go
package config

type Config struct {
	App App `json:"app"`
}

type App struct {
	Host    string `json:"host"`
	Port    int    `json:"port"`
	Workers int    `json:"workers"`
	DSN     string `json:"dsn"`
}
```

Your directory structure will look like the following:

```bash
./demo/config$ tree
.
????????? cue.mod
???   ????????? module.cue
???   ????????? pkg
???   ????????? usr
????????? types.go
```

Now generate your Cue files from the go types...
```bash
cue get go .
```
Which will create a `gen` directory with some pathing to a `types_go_gen.cue` file which contains the following.

```cue
// Code generated by cue get go. DO NOT EDIT.

//cue:generate cue get go github.com/luthermonson/tutorials-2022-01-cue/config

package config

#Config: {
	app: #App @go(App)
}

#App: {
	host:    string @go(Host)
	port:    int    @go(Port)
	workers: int    @go(Workers)
	dsn:     string @go(DSN)
}
```

Now let's use this newly generated file in a super basic cue file to generate our sample YAML document.

```cue
package config

import "github.com/luthermonson/tutorials-2022-01-cue/config"

#config: config.#Config & {
    app: config.#App & {
        host: "mydomain.com"
        port: 443
        workers: 2
        dsn: "root:test@tcp(mysql.domain.com:3306)/test?charset=utf8&parseTime=True&loc=Local"
    }
}

#config
```

I recommend you put this cue file in a sub-directory like `./config/cue` so you can do something like this...

```bash
$ cd ./config
$ cue export ./cue --out yaml
app:
  host: mydomain.com
  port: 443
  workers: 2
  dsn: root:test@tcp(mysql.domain.com:3306)/test?charset=utf8&parseTime=True&loc=Local
```

### Using Tags in Cue for Better Config Files
Tags are a construct in Cue to let you pass in variables at call time so let's change our Cue file to use tags and consume 
some environment files based on tag to generate two YAML files. Change the main.cue file to set host/workers/dsn as a variable...
```cue
package config

import "github.com/luthermonson/tutorials-2022-01-cue/config"

#config: config.#Config & {
    app: config.#App & {
        host: #host
        port: 443
        workers: #workers
        dsn: #dsn
    }
}

#config
```

add `prod.cue`
```cue
@if(prod)
package config

#host: "mydomain.com"
#workers: 16
#dsn: "root:prod@tcp(mysql-prod.domain.com:3306)/test?charset=utf8&parseTime=True&loc=Local"
```

add `qa.cue`
```cue
@if(qa)
package config

#host: "qa.mydomain.com"
#workers: 1
#dsn: "root:qa@tcp(mysql-qa.domain.com:3306)/test?charset=utf8&parseTime=True&loc=Local"
```

Now when you run `cue export` pass a `-t qa` or `-t prod`. The `@if()` at the top of the file will check if the tags are set
and only include those files if the tag was passed. This is a different pattern than say only evaluating certain cue files 
like `cue eval main.cue prod.cue` and `cue eval main.cue qa.cue` which is a pattern outlined by this [tutorial](https://cuetorials.com/patterns/inject/).
I like the benefits of evaluating cue on a directory of files and passing tags at call time instead of having to figure out
which files within the directory need evaluating.

```bash
$ cue export ./cue -t prod --out yaml
app:
  host: mydomain.com
  port: 443
  workers: 16
  dsn: root:prod@tcp(mysql-prod.domain.com:3306)/test?charset=utf8&parseTime=True&loc=Local
$  cue export ./cue -t qa --out yaml
app:
  host: qa.mydomain.com
  port: 443
  workers: 1
  dsn: root:qa@tcp(mysql-qa.domain.com:3306)/test?charset=utf8&parseTime=True&loc=Local
```

### Conclusion
Please checkout the code for this tutorial on the [github repository](https://github.com/luthermonson/tutorials/tree/main/2022/01/cue).
This was ultimately a very basic interpretation of the Cue language itself but gives you some ideas for structuring your Cue modules.
The language is far more feature rich than this tutorial gives it credit and I will likely do followup blog post on the 
language itself after and using it to validate your configuration files before they ever hit your application runtime.