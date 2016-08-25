Try F# prototype
================

The goal of this project is to develop a replacement for [Try F#](http://tryfsharp.org) 
that does not rely on Silverlight. This extended README describes a prototype - it covers
the key principles for building a Try F# replacement (to set the directions) and the 
technical details of the prototype implementation.

  * You can [see a sample tutorial in the prototype
    live](http://tryfsharp-preview.azurewebsites.net/tutorial/getting-started/) 
    (it is a _prototype_ so things need time to load and may not work)    
  * All source code is available in [the Try F# 
    organization](https://github.com/tryfsharp/) (in multiple repositories 
    as discussed below)

Project principles
------------------

Before discussing technical details, it should be clarified what is and (perhaps 
more importantly) what is _not_ in scope of this project. The goal can be summarized
as follows:

> Try F# should provide the best possible first experience when discovering and learning F#

This is it. Try F# is about offering an easy way to explore F# including some of its
unique features that is easy to access, requires no installation and illustrates many
of the aspects why we love F#. We want to create something where people can spend anything
between 10 minutes to get a quick taste of F# to perhaps 2 hours when completing a short
workshop at a conference or in a classroom.

There are many things that Try F# _should_ and _should not_ try to do:

 - **Make learning F# good experience** - The tutorials will ideally be what anyone curious 
   about F# will start with - for this reason, the tutorials must be engaging and the experience
   they get with Try F# must be rock solid. (We should prefer doing fewer things right 
   over having lots of features that are not necessary for good learning experience.) You can 
   see some of the ideas in the sample tutorial I created - there is an aspect of "gamification"
   (steps advance automatically once you complete previous ones) and we can also leverage the web
   (the sample tutorial ends with fun HTML with scrolling & random colors - no Fibonacci numbers!)

 - **Not a web-based IDE** - Try F# is not a full-blown web-based F# IDE. There are other 
   projects that do this (e.g. [CloudSharper](http://cloudsharper.com/)) and we can 
   link to those as one possible next step (aside from various ways of installing F#) 
   after completing Try F# tutorials.
   
 - **Not an embeddable component** - The goal of Try F# is to create a single learning
   web site, because that's what the community urgently needs. It would be nice to 
   make some of the components reusable. This is not a priority - we first need good
   new Try F#. And making things reusable is easier once there is more then one 
   concrete use-case.
   
 - **Do not try to do everything** - The purpose of Try F# is to provide tutorials for
   getting started. This means that it does not have to support every conceivable F#
   feature (if you want to learn how to use Reflection Emit to generate code at runtime,
   download F# proper). We should have enough to cover domains where F# is great 
   (functional programming, domain modelling, type providers, some charts)
 
Prototype overview
------------------

The current Try F# prototype tries to achieve the above goals by using the Monaco
editor for writing code (with Ionide-based F# integration) and Fable (F# to JavaScript
compiler) for evaluating the code. When editing and running tutorials, there are
two services that are called:

 - **F# autocomplete service** is a service that type-checks code that you write, 
   provides auto-completion, tooltips, error squigglies and possibly other information
   (this is based on the F# Compiler Service and it is also used in Ionide, Emacs 
   bindings and other editors - we just host the service on a public server via
   HTTP)
   
 - **Fable compilation service** is a service that takes F# code and compiles it 
   to JavaScript. It returns some additional information (type declarations in the
   code, bindings and their types). The service does not return JavaScript as string,
   but Babel AST (in JSON) which is then turned into JavaScript in browser using Babel.
   
More specifically, all the code is in the repositories below.

### Web pages and services

These repositories are automatically deployed to Azure to run the site. The web site
repository contains all design and tutorials, the other two are very lightweight 
wrappers that depend on the components below and just deploy them to Azure.

 * [tryfsharp-site](https://github.com/tryfsharp/tryfsharp-site) contains the source
   for the web page. The site contains a static site generator (based on F# Formatting
   and DotLiquid) that generates static HTML from the tutorials and pushes it to `gh-pages`
   (this means that hosting Try F# site is just a matter of hosting static HTML files
   and we could even use GitHub for this)
 
 * [tryfsharp-fable-compiler](https://github.com/tryfsharp/tryfsharp-fable-compiler) 
   is a very lightweight repo (mostly a build script and `.deployment` + `web.config`
   for Azure) that installs fable-compiler (see below) and runs it in Azure as a HTTP
   service.
   
 * [tryfsharp-autocomplete](https://github.com/tryfsharp/tryfsharp-autocomplete) is
   almost the same as tryfsharp-fable-compiler, but it runs the auto-complete service. 
   Again, this contains no actual code - just config for Azure.
   
I think using static site generator from tutorials written in Markdown as the
tryfsharp-site project does is a good idea, because it makes contributing and hosting
very easy. For the other two projects, the hosting could be done differently (I used
Azure web sites, because I know how to do that, but we can consider other options).

### Infrastructure and forks

The Try F# prototype uses Fable (F# to JavaScript), FsAutoComplete (F# auto-complete
service), Babel (JavaScript to JavaScript compiler :-)) and Monaco (editor) integration
based on Ionide. The following repositories contain forks (with varying level of changes)
of the projects:

 * [fs-auto-complete](https://github.com/tryfsharp/fs-auto-complete) (owned by
   [@rneatherway](http://github.com/rneatherway)) - this is the easiest piece - the
   editor service already exposed all the API we needed and I just used a minor
   modification (done by [@Krzysztof-Cieslak](http://github.com/Krzysztof-Cieslak))
   that adds HTTP headers to allow cross-origin calls (CORS). This can be merged back into
   fsharp/FsAutoComplete (possibly by enabling CORS only for a given domain with a 
   command line argument).
   
 * [babel-standalone](https://github.com/tryfsharp/babel-standalone) (owned by
   [@Daniel15](http://github.com/Daniel15)) - this is a standalone build of the Babel
   compiler which we use to turn Fable response into executable code in the browser.
   I needed to expose one function (required by Fable). I think we can likely ask for
   this to be merged back. (Alternatively, we could use some JavaScript whizz-bang to
   compile Babel into ionide-web or invoke it in Fable on the server, but I liked the
   idea of keeping the server lighter and this was a simple thing to do).
   
 * [fable-compiler](https://github.com/tryfsharp/fable-compiler) (owned by
   [@alfonsogarciacaro](http://github.com/alfonsogarciacaro)) - I recovered some work
   done by Alfonso earlier to expose Fable as a service, but I needed to add quite a
   lot to make the F# Interactive session state work (see below). This mostly just
   calls Fable internals and extracts various information - I think ideal solution 
   would be if Fable had a NuGet package with an F# library that we could call from
   a separate project to get all the information we need (because I think many of 
   the changes I did are Try F#-specific).

 * [ionide-web](https://github.com/tryfsharp/ionide-web) (owned by
   [@Krzysztof-Cieslak](http://github.com/Krzysztof-Cieslak)) - this is based on 
   ionide-web project that Krzysztof started, but I did quite a lot of additions.
   There is one mostly unchanged file which implements Monaco services using 
   FsAutoComplete, but I added a lot of infrastructure - like decent logging - 
   and more code to implement Fable-based evaluation. 
   
   I think we should continue working on the project here (for now), because keeping
   it in one place makes development easier, but the longer term plan would be to 
   separate Ionide Web (editor) from the Web-based F# Interactive extension (which
   I did for Try F#). The editor part can evolve more (to support things we do not
   really need for Try F# like project system, refactoring and more advanced editing
   capabilities), but I think the best way forward is to focus on what's needed for
   Try F#, build stable & solid foundations here and then, later, refactor the editor
   part into a stand-alone component that Try F# can just use (and extend with 
   Fable-based F# Interactive).
 
### Running prototype locally

To run things locally, you'll need to start the two services & run the web site. 
(By default, the build script for the web site transforms the site, starts a 
Suave server for local hosting and keeps running in the background to watch for
changes). This is pretty easy, but you will need 3 terminal windows :-)

Clone the three repositories:

    git clone https://github.com/tryfsharp/tryfsharp-site.git
    git clone https://github.com/tryfsharp/tryfsharp-fable-compiler.git
    git clone https://github.com/tryfsharp/tryfsharp-auto-complete.git

Open each of the cloned directories in one terminal and run `./build.sh` on mono
or `build.cmd` on Windows (I tested it on Windows using Cygwin, so I know `build.sh`
works and code runs on Windows, but you may find issues with other configurations).

This starts two services on <http://localhost:7101> and <http://localhost:7102> 
(not much to see there) and the web site at <http://localhost:7103> (browser should
open automatically). The JavaScript in the Try F# checks whether you are running
the site on `localhost` and if so, uses `localhost` services (instead of Azure-hosted
ones - we should probably use those as a fallback so that people can just build & 
run the tryfsharp-site repo).

If you want to change the JavaScript that runs in the site (editor integration, 
calling Fable, outputting results, etc.), you'll also need to build `ionide-web`
(you need reasonably recent version of Node for this):

    git clone https://github.com/tryfsharp/ionide-web.git
    cd ionide-web
    npm start

This will run Fable to compile the Ionide Web source code (and keep running it in
the background). The compiled JavaScript will appear in `public` folder under
`ionide-web`. You then need to copy it into the `web/lib/ext` folder in your
`tryfsharp-site` clone to see the changes online. You will probably want this to happen
automatically - possibly using some background script. I created a directory junction
on Windows (making the `public` directory a symlink for `web/lib/ext`), which 
works fine (and presumably, Unix symlink could work on Mac/Linux). 

### Notes on interactive session state

Most of the implementation is fairly straightforward - the only subtle thing is 
handling of the F# Interactive session state in the web-based runner. First of all,
the alternative is _not_ to have session state - you simply run all your code each
time you hit a Run button (which is what I used in <http://fun3d.net>).

In the Try F# prototype, I implemented basic support for session state. When you select
one snippet and run it, you can then select and run another snippet later in the code 
that uses declarations from the previous snippet. This is done without keeping any
session state on the server (which sounds error-prone and not scalable). Instead, the
client collects declarations from previously evaluated code snippets and secretly injects
them into all subsequent commands. So, say you run:

```fsharp
type A<'T> = A of 'T
let map f (A v) = A (f v)
```

The Fable server returns Babel AST (JavaScript) for the code, but it also returns the
code of the `A<'T>` type (extracted from the code using ranges in the parse tree) and
type information of `map`. When you evaluate `map (fun x -> x + 1) (A 10)` the code
that is sent to Fable actually looks like this:

```fsharp
module Types1 : 
    type A<'T> = A of 'T
open Types1
module Globals = 
  [<Fable.Core.Emit("hiddenIonideGlobals.map($0,$1)")>]
  let map (x0:('a -> 'b)) (x1:A<'a>) : A<'b> = failwith "JS"
open Globals
map (fun x -> x   1) (A 10)
```

As you can see, all declared types are included in modules prefixed `Types` (which 
automatically opened). Hiding works, because later declarations will shadow earlier 
ones. The module `Globals` then provides access to all declarations (the client-side
keeps them in a global dictionary called `hiddenIonideGlobals` and we tell Fable
that any access to `map` should be replaced with the call to the hidden previous
declaration).

This seems to work reasonably well - and I believe we can make it work pretty reasonably
for the purpose of Try F# (functional programming, domain modelling and basic data
science), but there might be a better solution - possibly, the F# Compiler Service 
could expose some way of serializing the "session state" after checking a source file
that we could then store and restore.

Writing tutorials
----------------- 

The nice thing about the prototype is how easy it is to write tutorials. 
They are just Markdown files in the [tutorials folder in 
tryfsharp-site](https://github.com/tryfsharp/tryfsharp-site/tree/master/tutorials).

A tutorial starts with some meta-data (description to appear on the home page,
image to appear when you link to it via Twitter) and content. There are a few things
that have special meaning. In particular, tutorial needs to start with level-1 heading
(name of the tutorial) and one or more sections starting with level-2 heading. These
are turned into individual sections that you can go through on the tutorial page.

Each section can specify special code snippets marked as `test` and `demo` (the parser
also looks for `solution`, but button to show the solution is not implemented yet).
For example, the step 2 of the sample tutorial looks as follows:

    Roll the dice using <kbd>Alt</kbd>+<kbd>Enter</kbd>. 
    When you get 6, we'll go to the next step! :-)
    
    ```demo
    // Now, let's try simulating a dice!
    let sides = 6
    let rnd = ...
    ```

    ```test
    unbox it = 6
    ```
    
The code marked as `demo` is appended to the end of the editor content when the tutorial
loads and you can use it to give the reader some background that they might need (or even
just an encouraging comment). 

The `test` code snippet is more fun! It lets you specify automatic test that's run after 
the user does anything - if the test code compiles, runs and returns `true` then the tutorial
moves automatically to the next section (the idea is to "gamify" the tutorial a bit, for 
example like [Try OCaml](https://try.ocamlpro.com/) does). In the `test` code, you can refer
to variables that the user is supposed to define and you also get access to two special variables -
`it` of type `obj` represents the last result (hence the `unbox` in the test above) and `output`
of type `string` contains strings that were printed using `printf`. 
 
 
