+++
Description = ""
Tags = []
date = "2016-03-31T04:30:29+03:00"
title = "LeftPad and Go: can tooling help?"
slug = "leftpad_and_go"
tags = ["golang", "tooling"]

+++

You've probably heard that story about NPM community and [LeftPad](https://www.npmjs.com/package/left-pad) package, that broke thousands JavaScript projects worldwide. There was a nice follow-up article titled ["Have We Forget How To Program"](http://www.haneycodes.net/npm-left-pad-have-we-forgotten-how-to-program/) and one guy even created [left-pad.io](http://left-pad.io) - Left-Pad As A Service web service. People got a lot of fun discussing this story.

I personally find this story amazing, because there is no single point of failure, but rather a set of things and coincidences resulted in a disaster. Every person I spoke about left-pad story sees it through it's own lens of concern. Some blame JS-community, some talks about how important vendoring is nowadays and others stands for absolutist views of the DRY principle.

For me my lens were the chances of getting into something similar in Go ecosystem. Of course, after initial thinking, I came to resolution that there is nothing to worry about - Go community doesn't tend to create packages for every single simple function over there. Yes, someone inspired by NPM LeftPad story created [similar package for Go](https://github.com/keltia/leftpad) - but the fact it was created after that fiasco story only confirms my  thoughts.

Moreover, there is even a [Go proverb](http://go-proverbs.github.io/) dedicated to this specific thing:

> A little copying is better than a little dependency.

If you struggle to understand how it applies with DRY and why it's a wise point, I suggest you check out [this video](https://www.youtube.com/watch?v=PAAkCSZUG1c) on a subject.

But one thought was spinning in my head - can we do better and use tooling to check for the 'little dependency'? I believe that tooling is a great way to enforce some good practices and views, and `go fmt` is a greates evidence of it. Why can't we make a tool that would check package imports and suggest to *"take a look at this small LeftPad dependency of only one small function"*?

Meet [depscheck](https://github.com/divan/depscheck).

# DepsCheck

So, that's how **depscheck** was born. I spent a weekend, learning how to work with go/{ast,parser,types,loader} packages and after 10-th refactoring finally came up with a tool that was doing exactly I wanted to.

Here is a description for GitHub:

> DepsCheck - Dependency checker for Golang (Go) packages. Prints stats and suggests to remove small LeftPad-like imports if any

It analyzes package or arbitrary *.go files using AST-parser and calculates some stats about dependencies. Namely, number of calls/selectors of dependency, funcs/methods' LOC (Lines Of Code), Cumulative LOCs (whenever possible, dropping it for recursion), Depth (number of nested calls to more external dependencies) and DepthInternal(the same, but for internal function calls). In most cases it's enough to make some calculations and analysis to find *'small dependencies'*.

### Quick Intro
Here is quick intro and dogfooding demo of the tool.

First, install the tool:
```
go get github.com/divan/depscheck
```

Then invoke as simple as:
```
depscheck -v github.com/divan/depscheck
```
![Demo](https://github.com/divan/depscheck/raw/master/demo/depscheck.png)

There is a detailed table view of dependencies usage, totals and breakdown by packages, and in the end, the tool says - hey, your dependencies are sane. It means, no dependencies of suspiciously small usage size.

In case it finds potential candidate for copying into your codebase and removing from the dependency list, it outputs something like this:

```
$ depscheck github.com/divan/expvarmon
github.com/divan/expvarmon: 4 packages, 1022 LOC, 93 calls, 11 depth, 23 depth int.
 - Package byten (github.com/pyk/byten) is a good candidate for removing from dependencies.
   Only 19 LOC used, in 1 calls, with 2 level of nesting
 - Package ranges (github.com/bsiegert/ranges) is a good candidate for removing from dependencies.
   Only 29 LOC used, in 1 calls, with 0 level of nesting
Run with -v option to see detailed stats for dependencies.
```

And by the way, in an example above it totally make sense. Rerunning with `-v` options to see the details, it's easy to see that package only uses single flat function from `ranges` of 29 lines of code - ranges.Parse, and it can be easily copied to my codebase, if license permits, of course. It removes the whole bunch of burden in case I'll need to vendor dependencies or use some vendoring tools.

Of course, this suggestion doesn't mean you have to do it. It just asks you to pay attention to these particular imports if you want to make your codebase a little bit better. It's up to me to decide what to do with this import. The tools suggestions are recommendation only.

### How small is small dependency

You may ask now - how do you determine if the dependency is small enough to be a candidate for elimination? Good question, and I don't have one single source of truth here, so I've implemented it based on my view of common sense here. To be short, here is the responsible method at the moment of writing this article:

```
// CanBeAvoided attempts to classify if package usage is small enough
// to suggest user avoid this package as a dependency and
// instead copy/embed it's code into own project (if license permits).
func (p *PackageStat) CanBeAvoided() bool {
	// If this dependency is using another dependencies,
	// it's almost for sure - no. For internal dependency, let's
	// allow just two level of nesting.
	if p.Depth > 0 {
		return false
	}
	if p.DepthInternal > 2 {
		return false
	}

	if p.DepsCount > 3 {
		return false
	}

	// Because 42
	if p.LOCCum > 42 {
		return false
	}

	return true
}
```

As you can see, it's pretty straightforward, no machine learning here. Just hardcore universal constants :) It's not perfect, but hey, at least, it can detect LeftPad package, mentioned above!

```
$ depscheck leftpad.go
main: 1 packages, 17 LOC, 1 calls, 0 depth, 1 depth int.
 - Package leftpad (github.com/keltia/leftpad) is a good candidate for removing from dependencies.
   Only 17 LOC used, in 1 calls, with 1 level of nesting
Run with -v option to see detailed stats for dependencies.
```

I'm looking forward to hear suggestions on how to do it better. Maybe it's worth to calculate total LOC of the package and look at the percentage of LOC involved? Feel free to tell your view on how small is "small dependency" could be from the perspective of the machine.

Anyway, it's always possible to manually inspect detailed output and filter small enough dependencies by yourself.

Also, please, don't rely on these numbers heavily. It's likely to be somehow incorrect in some cases. Yes, it's tested with simple cases, but there are lot of cases, where it's even impossible to tell what is a "correct" value. For example, how do you calculate Cumulative number of lines for two functions with circular dependency? There are also probably some bugs I didn't catch (hey, it survived 10 refactorings in 3 days!), so feel free to open an issue or send a PR.

# Stdlib

Depscheck also can work with stdlib packages. Normally it treats stdlib imports as 'internal' and doesn't analyze/report them. But you can explicitly ask for that using `-stdlib` command line flag. Suggestions are silenced in this mode because stdlib is smarter than this tool. Anyway, it can give interesting insights into your package or application.

For example, here is the sample output for checking against stdlib `time` package (though, you can use -stdlib flag for any package, not only stdlib):
```
$ depscheck -stdlib -v time
time: 4 packages, 294 LOC, 50 calls, 17 depth, 3 depth int.
+---------+-------+----------+--------+-------+-----+--------+-------+----------+
|   PKG   | RECV  |   NAME   |  TYPE  | COUNT | LOC | LOCCUM | DEPTH | DEPTHINT |
+---------+-------+----------+--------+-------+-----+--------+-------+----------+
| errors  |       | New      | func   |    31 |   2 |      2 |     0 |        0 |
| runtime |       | GOROOT   | func   |     2 |   6 |      6 |     0 |        0 |
| sync    | *Once | Do       | method |     1 |  11 |     90 |     2 |        3 |
|         |       | Once     | type   |     1 |     |        |       |          |
| syscall |       | ENOENT   | const  |     1 |     |        |       |          |
|         |       | O_RDONLY | const  |     2 |     |        |       |          |
|         |       | SIGCHLD  | const  |     1 |     |        |       |          |
|         |       | Close    | func   |     2 |   6 |      6 |     0 |        0 |
|         |       | Getenv   | func   |     2 |  20 |    149 |    13 |        0 |
|         |       | Getpid   | func   |     1 |   4 |      4 |     0 |        0 |
|         |       | Kill     | func   |     1 |   1 |      1 |     0 |        0 |
|         |       | Open     | func   |     2 |  13 |     13 |     0 |        0 |
|         |       | Read     | func   |     2 |  14 |     16 |     2 |        0 |
|         |       | Seek     | func   |     1 |   7 |      7 |     0 |        0 |
+---------+-------+----------+--------+-------+-----+--------+-------+----------+
+---------+---------+-------+-------+--------+-------+----------+
|   PKG   |  PATH   | COUNT | CALLS | LOCCUM | DEPTH | DEPTHINT |
+---------+---------+-------+-------+--------+-------+----------+
| errors  | errors  |     1 |    31 |      2 |     0 |        0 |
| runtime | runtime |     1 |     2 |      6 |     0 |        0 |
| sync    | sync    |     2 |     2 |     90 |     2 |        3 |
| syscall | syscall |    10 |    15 |    196 |    15 |        0 |
+---------+---------+-------+-------+--------+-------+----------+
```

You may even run it against the whole stdlib and build some charts in RStudio or your favorite statistics tool. Let's use `go list std` command to get all stdlib packages names and feed them to *depcheck*. There is a special `-totalonly` flag, which tells depscheck to output only totals oneliner, which is easily parseable:

```
$ for i in $(go list std); do depscheck -stdlib -totalonly $i; done
archive/tar: 13 packages, 3328 LOC, 174 calls, 5 depth, 78 depth int.
archive/zip: 13 packages, 2284 LOC, 184 calls, 24 depth, 84 depth int.
bufio: 4 packages, 84 LOC, 60 calls, 0 depth, 0 depth int.
bytes: 4 packages, 699 LOC, 79 calls, 0 depth, 26 depth int.
compress/bzip2: 3 packages, 58 LOC, 19 calls, 0 depth, 4 depth int.
compress/flate: 7 packages, 255 LOC, 52 calls, 3 depth, 10 depth int.
compress/gzip: 8 packages, 622 LOC, 72 calls, 1 depth, 36 depth int.
compress/lzw: 4 packages, 42 LOC, 25 calls, 1 depth, 3 depth int.
compress/zlib: 7 packages, 877 LOC, 59 calls, 1 depth, 47 depth int.
container/heap: 1 packages, 0 LOC, 13 calls, 0 depth, 0 depth int.
container/list: 0 packages, 0 LOC, 0 calls, 0 depth, 0 depth int.
container/ring: 0 packages, 0 LOC, 0 calls, 0 depth, 0 depth int.
crypto: 3 packages, 88 LOC, 6 calls, 0 depth, 2 depth int.
crypto/aes: 4 packages, 109 LOC, 5 calls, 0 depth, 3 depth int.
crypto/cipher: 4 packages, 21 LOC, 12 calls, 0 depth, 1 depth int.
crypto/des: 3 packages, 100 LOC, 9 calls, 0 depth, 2 depth int.
crypto/dsa: 3 packages, 6021 LOC, 89 calls, 0 depth, 409 depth int.
crypto/ecdsa: 9 packages, 4059 LOC, 94 calls, 2 depth, 269 depth int.
crypto/elliptic: 3 packages, 2677 LOC, 298 calls, 4 depth, 180 depth int.
crypto/hmac: 3 packages, 19 LOC, 15 calls, 0 depth, 1 depth int.
crypto/md5: 3 packages, 5 LOC, 5 calls, 0 depth, 0 depth int.
...
```

A couple of manipulations to convert it to CSV and here is the histogram of number of imported packages in stdlib:

![Histogram](/images/stdlib_deps.png)

So, feel free to play with this statistics. Maybe you'll find it useful.

# Conclusion

I constantly admire the long-striking power of the core ideas around Go design. Simple grammar and spec allowed me to build this tool in a weekend, without any prior experience of working with AST representation of source code. It's inherently hard, but the guys behind Go's source code parsing packages are doing amazing job. I benefit every single day by different linters, bundled together in [GoMetaLinter](https://github.com/alecthomas/gometalinter) package (a must have thing!), and these static code analysis tools are making our code better.

Now I also feel a little bit safer because that Go proverb mentioned in the beginning is now materialized in a depscheck tool. And hopefully, it will make us, as a community, one more step further from the slightest possibility to fall into the trend that resulted in that LeftPad fiasco.

Happy coding!





