+++
title = "Making a Database with OCaml"
date = 2025-03-24
[taxonomies]
tags=["Programming", "OCaml", "Databases"]
+++

I've been playing around with OCaml recently and really enjoying it.
I love functional programming and have also enjoyed playing around with F#, Elixir, and Haskell in the past, but I've never gotten super far with any of those languages for various reasons.
So, I wanted to take on a more serious project to really kick the tires on the language and start building some deeper knowledge.
In the past, I read a book about the pure relational algebra that inspires modern relational databases ([Amazon book link](https://a.co/d/jartLNm)).
Taking what I learned from the book, I wrote a small Rust library that would parse a query language and use it to run queries against CSV files.
That project never turned into a full-on database engine, but I enjoyed the project a lot and thought it would be a good project to recreate in OCaml for a few reasons.

## Getting Started

If you want to follow along, you'll need [`opam`](https://opam.ocaml.org/) and [`dune`](https://dune.build/) installed.

We can use `dune` to quickly initialize the new project:

```bash
development $ dune init project camalbrarian
Success: initialized project component named camalbrarian
```

{% callout(label = "Note") %}
Yes, camalbrarian stands for Camel Librarian, because we'll be starting with the query engine, and a query engine is basically a specialized librarian.
And no, the typo in "camal" was not intentional, but let's just roll with it.
{% end %}

We can `cd` into the new folder and create an Opam switch.

```bash
development $ cd camalbrarian
camalbrarian $ opam switch create . 5.3.0
```

If you haven't used OCaml before, you're probably already confused why I created a "switch."
Essentially, `opam` switches are "isolated OCaml environments" ([source](https://ocaml.org/docs/opam-switch-introduction)).
Switches allow you to have different versions of OCaml installed at the same time, so you can "switch" between them.
Where I found switches confusing at first is that they also manage all packages that you install.
I'm used to the C# or Rust model where installed packages are stored/managed within the project directory (e.g. `dotnet add package ...` or `cargo add ...`) and language versions are managed separately.
But `opam` switches manage the language, tooling, and packages all at the same time.

This means that you could create a single switch, maybe for a specific version of OCaml like 5.3.0, and then you could use that switch for all your projects.
Sharing a switch between multiple projects would be faster, but it also means that all projects would be sharing versions of everything by default.
This might be fine at first, but eventually you would want to use a different version of the language or a particular package, and then you would want another switch.
Thankfully, `opam` has an easy solution for creating project-specific switches. `opam` calls these "local switches", while shared switches are called "global switches."

To understand the difference better, we can peruse the `opam switch` man page by running `opam switch --help` or `man opam-switch`.
If you run that and search for the `create` subcommand, you'll find this blurb:

> create SWITCH [COMPILER]

>> Create a new switch, and install the given compiler there. SWITCH can be a plain name, or a directory, absolute or relative, in which case a local switch is created below the given directory. COMPILER, if omitted, defaults to SWITCH if it is a plain name, unless --packages, --formula or --empty is specified. When creating a local switch, and none of these options are present, the compiler is chosen according to the configuration default (see opam-init(1)). If the chosen directory contains package definitions, a compatible compiler is searched within the default selection, and the packages will automatically get installed.

If you are allergic to man pages (Windows user! 🫵) or if you want more info, you can read through [Introduction to opam Switches](https://ocaml.org/docs/opam-switch-introduction#types-of-switches).

My takeaway is that you should almost always want to create a local switch by running something like `opam switch create .` in your project directory.
This will most closely mimic how other language package managers work and should provide the level of project isolation that you are used to.

After my create switch command finished installing everything, it recommened that I run `eval $(opam env)` to update my current shell environment.
After we run that, we can poke around a little bit to understand what `opam` did for us.

I'll start by trying to source our `ocaml` binary with the `which` command:

```bash
camalbrarian $ which ocaml
~/development/camalbrarian/_opam/bin/ocaml
```

Okay so `opam` made a little `bin/` folder local to our project. Let's investigate that a little bit:

```bash
camalbrarian $ ls _opam/bin/
ocaml                   ocamlcp                 ocamldoc                ocamlmklib              ocamlopt                ocamlrun
ocamlc                  ocamldebug              ocamldoc.opt            ocamlmktop              ocamlopt.byte           ocamlrund
ocamlc.byte             ocamldep                ocamllex                ocamlobjinfo            ocamlopt.opt            ocamlruni
ocamlc.opt              ocamldep.byte           ocamllex.byte           ocamlobjinfo.byte       ocamloptp               ocamlyacc
ocamlcmt                ocamldep.opt            ocamllex.opt            ocamlobjinfo.opt        ocamlprof
```

Looks good to me 👍

I still haven't used OCaml enough to know what all of these are for, but you can chase down their man pages if you really care.
The names indicate that they are all different tools that are useful for the compilation process.

If we want to see what switch we are currently on, you can use `switch list`:

```bash
camalbrarian $ opam switch list
#  switch                      compiler                                           description
→  ~/development/camalbrarian  ocaml-base-compiler.5.3.0,ocaml-options-vanilla.1  ocaml-base-compiler = 5.3.0 | ocaml-system = 5.3.0
   5.3.0                       ocaml-base-compiler.5.3.0,ocaml-options-vanilla.1  ocaml-base-compiler = 5.3.0 | ocaml-system = 5.3.0
   default                     ocaml.5.2.0                                        default

[NOTE] Current switch has been selected based on the current directory.
       The current global system switch is 5.3.0.
```

If you are a mega-nerd (which is likely if you are still reading) then you might be wondering what that `eval $(opam env)` command did earlier.

Well let's see:

```bash
camalbrarian $ opam env
OPAM_LAST_ENV='~/.opam/.last-env/env-21ef9d7979f317c6a64e8577a7d09daf-0'; export OPAM_LAST_ENV;
OPAM_SWITCH_PREFIX='~/development/camalbrarian/_opam'; export OPAM_SWITCH_PREFIX;
OCAMLTOP_INCLUDE_PATH='~/development/camalbrarian/_opam/lib/toplevel'; export OCAMLTOP_INCLUDE_PATH;
CAML_LD_LIBRARY_PATH='~/development/camalbrarian/_opam/lib/stublibs:~/development/camalbrarian/_opam/lib/ocaml/stublibs:~/development/camalbrarian/_opam/lib/ocaml'; export CAML_LD_LIBRARY_PATH;
OCAML_TOPLEVEL_PATH='~/development/camalbrarian/_opam/lib/toplevel'; export OCAML_TOPLEVEL_PATH;
PATH='~/development/camalbrarian/_opam/bin:{rest of PATH omitted for privacy}'; export PATH;
```

I was worried that `opam env` would be some crazy complicated shell script, but it's refreshingly simple.
It just sets up some environment variables to keep track of a few paths, and then it adds the local switch `bin/` folder to my `PATH`.

Before we write any OCaml code (one day we will, I promise), I would like to at least set up an [LSP](https://en.wikipedia.org/wiki/Language_Server_Protocol).
The primary LSP for OCaml is [ocaml-lsp](https://github.com/ocaml/ocaml-lsp) and it has some simple install instructions if you are using `opam`:

```bash
camalbrarian $ opam install ocaml-lsp-server
```

{% callout(label = "Note") %}
The README for `ocaml-lsp` has this note:

> you will need to install ocaml-lsp-server in every switch where you would like to use it.

I think it's a little unfortunate that you have to reinstall your tooling on every new switch, because that can feel like a drag if you are just starting to learn the language and starting lots of new projects.
But it might be unavoidable.
Part of me wonders if you could have a local and global switch registered at the same time.
So, new library packages would be installed in the local switch, but you could also specify that tooling packages should be installed in your global switch.
Then, both switches would be registered on your `PATH` when you run `eval $(opam env)`, with the local switch first so it takes precedence.
Maybe that's a bad idea or maybe that's already how it works and I just don't know enough about `opam` yet.
{% end %}

After that install finishes, we can investigate our new LSP binary:

```bash
camalbrarian $ which ocamllsp
~/development/camalbrarian/_opam/bin/ocamllsp
```

Very nice. I'll register that LSP in my NeoVim config (I use NeoVim btw 😎), and then we are ready to roll.

## Actually Programming Now

Whenever we ran `dune init project camalbrarian`, the "project" argument told `dune` that we want a full project.
That includes a library package (located in `lib/`), a binary/executable package (`bin/`), and a test package (`test/`).
We will use all three of those packages eventually, but if you haven't run any OCaml yet, you'll want to start with the `bin/` folder just so you can see some code execute.

If you open up `bin/main.ml`, you may see this error message from your LSP:

```
No config found for file bin/main.ml. Try calling 'dune build'.
```

If you do what the friendly error message says and run `dune build`, then it should go away and you will be ready to write some code.

Now, we can finally run our executable package and see the default output:

```bash
camalbrarian $ dune exec camalbrarian
Hello, World!
```

Great! Let's get into some database-specific code.

## Designing a Query Language

Now that we have a project established, I'll try to speed up and explain things more with code rather than long diatribes.

{% callout(label = "Diatribe") %}
Seeing the word "diatribe" made me think of "diabetes", which then made me wonder about the common prefix "dia-".

It turns out that "dia-" comes from Ancient Greek and can mean "through, across, by, over" ([source](https://en.wiktionary.org/wiki/dia-)).
{% end %}

Anyway, OCaml has a rich type system that deserves its own article (["Basic Data Types and Pattern Matching"](https://ocaml.org/docs/basic-data-types)).
What I am interested in are the Variant types because they will be great for representing our query language.
If you have engaged with programming language design discourse before, you may have heard the term [AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree).
Essentially, abstract syntax trees are a great technique for representing the structure and content of a language.
Part of what makes ASTs so useful is that they provide a common language/interface for all parts of your system to reason about the language.
So for a database, your parser will take text input and produce an AST object, then your query optimizer might analyze the AST and simplify it where possible, then a query planner can analyze the AST again to determine the best query plan, then an executor can finally execute the AST to run the query.

All that to say, OCaml's variant types are fantastic for representing an AST with minimal boilerplate.
So, let's make `lib/query.ml` and start writing our type:

```ocaml
type t = 
  | LoadCSV of string
  | Rename of string * string
```

{% callout(label = "Note") %}
I had to run `dune build lib/` so my LSP could find the new file.
{% end %}

So we have a basic variant type with two cases, `LoadCSV` and `Rename`.
There are two points of interest here, how OCaml modules work and where our query language's operators are coming from.
I'll start with OCaml modules.

OCaml modules are similar to modules or namespaces in other languages, so far as they are a way of organizing related definitions together.
The main point I wanted to bring up now is why we named our variant type just `t`.
The reason is because we are already inside the `Query` module, so I didn't want our type to be referred to as `Query.query`.
You might be wondering "How are we inside the `Query` module, we never declared a module?"
But OCaml has [file-based modules](https://ocaml.org/docs/modules#file-based-modules), so just by being inside of `lib/query.ml`, our type `t` is part of the `Query` module.
There is a lot more to say about modules and we may cover some more as it comes up, but for now I recommend you read the official OCaml docs on [modules](https://ocaml.org/docs/modules).

Now onto the operators.
We are making a database that is based on relational algebra.
I can't do relational algebgra justice here, so I recommend you read the [Wikipedia](https://en.wikipedia.org/wiki/Relational_algebra) article to get up to speed.
Aside from the theory, that article also introduces some of the basic operators that we will be implementing in our query language.
I'm just starting with the rename operator for now because it is the simplest to reason about.
`LoadCSV` is certainly not part of the relational algebra, but CSV files are a great source of test data because they are simple for machines and humans to work with.

{% callout(label = "Musing") %}
All your code has to run inside of a human's brain before it can run in your computer's "brain."
So, optimizing code for human understanding can often be more important than optimizing it for computers.
Using CSV files as a starting point is important because it gives us a cheap way to handle real data without worrying about pages and indexes and all the scary things that real databases have to handle.
{% end %}

## Getting Parsing for Free

Now that we have a way to represent our query language, we need a way of converting plaintext input into that representation (AKA parsing).
We could write our own parser by hand or use a fancy [parser combinator](https://en.wikipedia.org/wiki/Parser_combinator) library, but I would rather get our parsing for free, because free is awesome.
To do that, first we need to talk about preprocessing, alternative standard libraries, and a high-frequency trading firm...

I know, I know, that sounds insane, but it's true.
We have a lot of groundwork to lay to understand the OCaml ecosystem and how we can profit from it.

I'll explain things quickly and throw a couple of links at you.

 - First, OCaml has [metaprogramming](https://ocaml.org/docs/metaprogramming) which allows you to run raw-text preproccessors or PPX preproccessors which transform the OCaml language AST (yes, OCaml uses an AST as well. I told you they are useful!).
 - Second, OCaml has historically had a small standard library, so there are multiple alternative standard library packages that provide lots of useful stuff. The one we are interested in is called [Core](https://github.com/janestreet/base). There is a PPX inside of the `Core` library that can essentially auto-generate a parser for our query type.
 - Third, the `Core` library is made by a high-frequency trading firm called [Jane Street](https://www.janestreet.com/). Jane Street is (probably) the biggest industrial user of OCaml and they drive a significant portion of the OCaml ecosystem. They are doing lots of interesting things with OCaml and they have made some very useful libraries to help them do that. Which means, people like me get to benefit from that work so I don't have to write my own parser (for now, maybe I'll write a fancier one later).

Makes sense? I hope so.

Let's start by installing the `Core` library:

```bash
camalbrarian $ opam install core
```

While that's installing, let's talk about s-expressions.
[s-expressions](https://en.wikipedia.org/wiki/s-expression) (sexps or sexpr for short) come from how Lisp languages represent data and code (data is code, and code is data λ).
If you are using Lisp, then s-expressions are essentially the entire syntax you are using to write your code, but they are also how data is represented and (sometimes) serialized.
For reasons beyond my knowledge, Jane Street decided that they would also use s-expressions as a common means of representing data.
I would guess it's because s-expressions are fairly simple and easy to grasp, and they probably have quite a few Lisp nerds working there which would make it a natural choice.
Since Jane Street has already done the leg work and provided a PPX that can generate s-expression serialization/deserialization functions for OCaml types, we are going to <s>steal</s> use that PPX for our query type.

After `Core` has finished installing, you will need to declare the dependency in `lib/dune` and we need to register the `ppx_sexp_conv` PPX for... reasons. Probably good reasons too:

```clojure
(library
 (name camalbrarian)
 (libraries core)
 (preprocess (pps ppx_sexp_conv)))
```

Now we can finally add the s-expression PPX to our type:

```ocaml
open Core

type t = 
  | LoadCSV of string
  | Rename of string * string
[@@deriving sexp]
```

We opened the `Core` library so we could have access to the various s-expression functions that it defines.
`[@@deriving sexp]` specifies that the `sexp` PPX should be used to process our type.
That PPX will generate some code that allows us to use different `sexp` functions on it.

For example, we can now define some of the simplest parsing and printing functions ever:

```ocaml
let (>>) f g x = g (f x)

let parse = Sexp.of_string >> t_of_sexp
let print = sexp_of_t >> Sexp.to_string 
```

Okay, `>>` is a little obtuse, but I just like composing functions and OCaml doesn't provide a built-in operator for it.
If we were in F# land, then I could `>>` by default, but you have to define it yourself in OCaml.
You could also use `Core.Fn.compose` since we installed `Core`, but its first two args are flipped compared to F#'s `>>`.
And if we were using Haskell, then you could do the same using the `.` operator.

Anyway, `>>` will run the first function, then take the result of that and pass it to the second argument.
It's also worth noting that `parse` and `print` are written in a "point-free" style, which means something smart that I forgot exactly and I'm too lazy to look it up.
Basically it just means that they don't take arguments and they use implicit arguments instead.
So, I could have written `parse` like this:

```ocaml
let parse input = t_of_sexp (Sexp.of_string input) 
```

And it would have been exactly the same, just less cool looking.

To test out our new functions, we are going to use OCaml's de-facto standard REPL, `utop`, to test it.
If you don't know what a REPL is, it stands for [read, eval, print, loop](https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop) and it's a great way of interacting with your code.
I don't know yet how great the REPL story is in OCaml, but for most Lisps you can integrate your REPL with your editor and you basically never have to run your project through the command line.
Instead, you are able to evaluate select Lisp forms so you have a lot greater control over what code you want to run and when.

Anyway, since OCaml tools are just packages that add a binary to your switch's `_opam/bin/` folder, we have to install `utop` to our local switch before running it:

```bash
camalbrarian $ opam install utop
```

With that done, we can enter the REPL with `dune utop`.
I won't explain how to use the REPL here.
If you need help, follow along with [this](https://ocaml.org/docs/toplevel-introduction) page and come back.

Once you are in the REPL, you can load our `Query` module with `#use "lib/query.ml";;`.
Once you enter an expression ending with `;;`, `utop` will execute your code and spit back out the results.
Here is what my REPL session looked like when I tested the query module:

{% callout(label = "Note") %}
Inputs are denoted with a `utop # ` prefix.
Outputs are any lines immediately following the inputs.
{% end %}

```ocaml
utop # #use "lib/query.ml";;
type t = LoadCSV of string | Rename of string * string
val ( >> ) : ('a -> 'b) -> ('b -> 'c) -> 'a -> 'c = <fun>
File "lib/query.ml", line 10, characters 30-39:
10 | let parse = Sexp.of_string >> t_of_sexp
                                   ^^^^^^^^^
Error: Unbound value t_of_sexp
Hint: Did you mean int_of_sexp or mat_of_sexp?
```

Doh. That's not what we wanted.
If we look up the error message, someone else has run into the [same thing](https://discuss.ocaml.org/t/no-t-of-sexp-generated-by-deriving-sexp/1999) and Jane Street's OCamler in Chief came along with some helpful advice:

> Try typing `#require "ppx_jane";;` first.

```ocaml
utop # #require "ppx_jane";;

utop # #use "lib/query.ml";;
type t = LoadCSV of string | Rename of string * string
val t_of_sexp : Sexp.t -> t = <fun>
val sexp_of_t : t -> Sexp.t = <fun>
val ( >> ) : ('a -> 'b) -> ('b -> 'c) -> 'a -> 'c = <fun>
val parse : string -> t = <fun>
val print : t -> string = <fun>
```

Okay now it works. I would recommend reading through the thread above if you want to know a little more about why that works, but I'll skip over it for brevity.
I have mixed feelings about the whole PPX ecosystem because it seems very powerful but I've struggled with it so far.
I've already lost a couple of hours in total just fighting with the sexpr and JSON PPX libraries and I haven't even been using OCaml that long.
It's part of the reason why I decided to write about my experiences with OCaml, so I could firm up my own understanding and document some of the frustrations.

So far, it seems like more documentation would help. Right now, the PPXs I've used feel like a bit of a black box.
I probably need to give it more time and maybe even try writing a PPX of my own so I can understand what's happening better, but a lot of OCaml beginners would likely quit trying rather than digging into the internals to understand what's going wrong.

I know from my experience with Rust's macro system that macros can become very complicated very quickly, and most programmers I know would rather not bother.
But if using a language's macro system requires you to almost be an expert in that language, then you are cutting off a significant portion of beginners from using it.
Requiring someone to be an expert before they can write macros makes sense to me, but the barrier to entry should be a lot lower for just using macros.

Okay, rant over. Now we can actually parse and print some queries:

```ocaml
(* continuing the same utop session from above *)

utop # parse "(LoadCSV \"people.csv\")";;
- : t = LoadCSV "people.csv"
```

Yup, that works.

Whenever I was about to test the rename operator I realized that we have an issue with it.
Right now, the rename case is of type `string * string`, but actually it needs to be `t * string * string` so it can take in an input query to rename.
Here is what the updated type looks like:

```ocaml
type t = 
  | LoadCSV of string
  | Rename of t * string * string
[@@deriving sexp]
```

Now we can reload our module in `utop` and test a more complicated query:

```ocaml
utop # #use "lib/query.ml";;
type t = LoadCSV of string | Rename of t * string * string
val t_of_sexp : Sexp.t -> t = <fun>
val sexp_of_t : t -> Sexp.t = <fun>
val ( >> ) : ('a -> 'b) -> ('b -> 'c) -> 'a -> 'c = <fun>
val parse : string -> t = <fun>
val print : t -> string = <fun>

utop # parse "(Rename (LoadCSV \"people.csv\") \"FirstName\" \"name\")";;
- : t = Rename (LoadCSV "people.csv", "FirstName", "name")
```

Perfect, now we should be ready to write a simple query executor.

## Executing Queries

So we have something to query against, I'll make `employees.csv` using some data from the Wikipedia article on relational algebra that I linked earlier:

```csv
Name,EmpId,DeptName
Harry,3415,Finance
Sally,2241,Sales
George,3401,Finance
Harriet,2202,Sales
Mary,1257,Human Resources
```

I'll write all the execution code in a new file, `lib/exec.ml`:

```ocaml
open Core

module TupleValue = struct
  type t =
    (* For now, just load everything as a string to keep CSV loading simple *)
    | String of string
  [@@deriving sexp]
end

module Tuple = struct
  type t = Tuple of TupleValue.t list
  [@@deriving sexp]
end

module Relation = struct
  type t = {
    tuples : Tuple.t list;
    header_type : string list;
  }
  [@@deriving sexp]
end
```

To start, I am defining types to represent data within our database.
`TupleValue` is a single scalar value and currently only supports strings because that's simple.
`Tuple` is just a `TupleValue` list.
And `Relation` is a `Tuple` list and a list of strings which represents the header names.
This is probably not an ideal design, especially if you are trying to make a real database, but it works for now.

This is also the first time we are seeing record types.
They are basically immutable maps/records/structs which allow you to associate named keys with values, similar to what lots of other languages provide.
If you want to read more about them, go [here](https://ocaml.org/docs/basic-data-types#records).

Since the only way we have of getting real data is through CSV files, let's make some simple logic for loading CSVs:

```ocaml
let _process_header_line line =
  String.split line ~on:','

let _process_csv_line line =
  String.split line ~on:','
  |> List.map ~f:(fun (x) -> TupleValue.String x)
  |> fun (x) -> Tuple.Tuple x

let _load_csv path =
  let lines = In_channel.read_lines path in
  match lines with
  | first::rst ->
    {
      Relation.header_type= _process_header_line first;
      Relation.tuples = rst |> List.map ~f:_process_csv_line;
    }
  | [] -> failwith "Cannot load an empty CSV"
```

`_load_csv` will load a file from a given path, process the first line as a header, process the remaining lines as tuples, then spit out a `Relation`.
This is a fairly standard functional style of programming.
The main part that I found interesting is the use of `~{keyword}:{value}` to specify named values in OCaml.
I thought I wouldn't like it at first, but it's been growing on me and I've come to enjoy those labels.
I feel like that makes it a lot easier to parse out separate arguments, especially for higher-order functions.

I also wish that variant case constructors would be considered more like plain functions so I could pipe values into them, that would allow me to rewrite `_process_csv_line` like this:

```ocaml
let _process_csv_line line =
  String.split line ~on:','
  |> List.map ~f:(fun (x) -> TupleValue.String x)
  |> Tuple.Tuple
```

I don't think it really matters, I just feel like this style would flow better.
There are probably valid reasons for why OCaml won't let me do it.

Now that we have CSV parsing, let's also write the logic for our rename operator:

```ocaml
let _rename_relation (from, to_) relation =
  let open Relation in
  { relation with header_type = relation.header_type |> List.map ~f:(fun x -> if String.equal x from then to_ else x) }
```

The `let open Relation in` line allows you to access the items within that module for the rest of your current scope.
The `{ relation with ... }` syntax is a record update, which isn't technically updating the record but instead it allocates a new record.

{% callout(label = "Note") %}
Theoretically, with [recent advances](https://blog.janestreet.com/oxidizing-ocaml-ownership/) in the OCaml compiler, it should be able to automatically optimize an immutable record update into a mutable update in certain circumstances. If you could statically guarantee that your reference is unique using some kind of ownership annotations (like in Rust), then you could optimize these immutable copies into mutable updates, which is great because allocation is expensive.

You could also manually mark the field as mutable and do a mutable update instead, but then you're introducing extra mental overhead.
Wouldn't it be great if the compiler could do the optimization for you so you don't have to constantly track if you need a mutable/immutable update every time you make a change?
{% end %}

Finally, we can pattern match on our query AST to evaluate it:

```ocaml
let rec exec query =
  match query with
  | Query.LoadCSV path -> _load_csv path
  | Query.Rename (inner_query, from, to_) -> exec inner_query |> _rename_relation (from, to_)
```

Specifying a function with `rec` before the name marks it as recursive, which is very useful for our use case here.
This kind of recursive evaluation is *very* common when working with recursive types like an AST.
We could already use this `exec` function in `utop`, but instead I want to build a dedicated REPL for this database.

```ocaml
let rec repl () =
  print_string "> ";
  Out_channel.(flush stdout);
  let input = In_channel.(input_line_exn stdin) in
  let ast : Query.t = Query.parse input in
  let result = exec ast in
  let sexp = Relation.sexp_of_t result in
  print_endline (Sexp.to_string_hum ~indent:4 sexp);
  repl ()
```

Since we initialized this as a full-featured project ealier using `dune`, we already have our `bin/` application folder which would be perfect for running a REPL.
Let's hook that up by call our `repl` function from `bin/main.ml`:

```ocaml
Camalbrarian.Exec.repl ()
```

Simple! Let's see if it works. Calling `dune exec` in the shell should automatically load our project and evaluate `bin/main.ml`:

```bash
camalbrarian $ dune exec camalbrarian
> (Rename (LoadCSV "employees.csv") "Name" "FirstName")
  ((tuples
    ((Tuple ((String Harry) (String 3415) (String Finance)))
     (Tuple ((String Sally) (String 2241) (String Sales)))
     (Tuple ((String George) (String 3401) (String Finance)))
     (Tuple ((String Harriet) (String 2202) (String Sales)))
     (Tuple ((String Mary) (String 1257) (String "Human Resources")))))
   (header_type (FirstName EmpId DeptName)))
```

Success!
Our CSV loading worked great, and the rename operator worked because the first header is now "FirstName" instead of "Name".

{% callout(label = "Tip") %}
If you are on a UNIX-like operating system, you probably have (or could install) `rlwrap`, which can make your custom REPL experience a lot nicer.
If you have it, just run `rlwrap dune exec camalbrarian`.
{% end %}

If you want to take this further there is a lot more interesting stuff to implement. Like:

 - More relational algebra operators:
   - projection, selection, natural joins, equijoins, semijoins, antijoins, and division!
 - More data sources:
   - Just querying CSVs is pretty boring. This would get a lot more interesting if it could query other databases like Postgres, MySQL, and SQL Server
 - Query Optimization/Planning:
   - Optimizing queries before running them would be a great way to get into more advanced pattern matching features like guard clauses. It can get very complicated (especially if paired with the next idea), but there are also a few simple optimizations that would be fun to add.
 - Storing data:
   - This project would get *extremely fun* if we started supporting insertions/updates/deletes and storing our data on disk.
 If you care about performance, storing relational data gets complicated fast because now you have to worry about concurrency, filesystem corruption, indexing, and a million other things. Interestingly, you would also need a separate language for describing modifications if you wanted to maintain relational algebra purity because, as far as I know, the relational algebra doesn't include any operators that mutate relations. It's very similar to functional programming in that sense.

I may tackle a few of the earlier ideas in a future post, but I'll stop here for now.
If you're trying to get better with OCaml like I am, hopefully reading through my rants helped you a little bit.
There were a lot of points where I could have chased a rabbit or dug into something deeper, but I decided not to just so I could finish the post.
I'd like to write more about OCaml in the future because it helps me process what I'm learning and I hope it could help advertise the language to others who are curious about the functional way of doing things.
Thanks for reading!
