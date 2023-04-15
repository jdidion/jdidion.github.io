+++
title = "Parsing with Rust - Part 2: writing an LR grammar with Tree-sitter"
date = "2023-04-14"
author = "John Didion"
authorTwitter = "jdidion"
tags = ["rust", "parsing", "wdl"]
keywords = ["rust", "parsing", "wdl", "tree-sitter"]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = ""
+++

The second post in a series on writing programming-language parsers in Rust. In [Part 1](/posts/parsing-with-rust-part1/), we covered general parsing concepts and looked at some of the Rust crates available for generating a parser. In this post, we'll walk through writing a [Tree-sitter](https://tree-sitter.github.io/tree-sitter/) grammar for [WDL](https://openwdl.org) - a domain-specific language for describing computational workflows - and using the parser in Rust.

<!--more-->

As we briefly covered in [Part 1](/posts/parsing-with-rust-part1/), [Tree-sitter](https://crates.io/crates/tree-sitter) is a widely used parser generator framework that generates a Generalized LR (GLR) parser with Rust bindings from a grammar specification.

Tree-sitter is implemented using an interesting mix of languages:

* A [Tree-sitter grammar](https://tree-sitter.github.io/tree-sitter/creating-parsers#the-grammar-dsl) is specified in JavaScript. More specifically, it is a [node.js](https://nodejs.org/) module that exports a grammar definition, which is [serialized to JSON](https://github.com/tree-sitter/tree-sitter/blob/master/cli/src/generate/grammar-schema.json).
* The Tree-sitter [command line interface (CLI)](https://tree-sitter.github.io/tree-sitter/creating-parsers#command-generate) is written in Rust. It provides a `generate` command, which calls `node` with the grammar file, deserializes the generated JSON object, and uses it to generate the parser code in `C`.
* The generated `C` code depends on the Tree-sitter runtime, a `C` library called `libtree-sitter`. The compiled parser can be called from a C/C++ program.
* The CLI also generates Rust and Node bindings to the parser that enable it to be used from those languages as well.

In the remainder of this post, we'll walk through the steps of writing the grammar, generating the parser, and using it from a Rust program. We'll be writing a grammar for [Workflow Description Language (WDL)](https://openwdl.org), a domain-specific language for describing computational workflows. We'll be focusing on just on WDL 1.1, the current version of the [specification](https://github.com/openwdl/wdl/blob/main/versions/1.1/SPEC.md). WDL is useful as an example because it's powerful enough to exercise some of the advanced features of parser generator frameworks yet simple enough that we can write a complete parser without getting bogged down with the ambiguity and context-specificity that are found in most general-purpose languages.

You can find all of the code and auxilliary files for the parser in the [GitHub repository](https://github.com/jdidion/tree-sitter-wdl).

## Getting set up

The first step in developing our Tree-sitter grammar is to install the dependencies - node.js, the node package manager (`npm`), and a C/C++ compiler - and setup a new project. These steps are covered in the [Getting Started](https://tree-sitter.github.io/tree-sitter/creating-parsers#getting-started) section of the Tree-sitter manual. It is recommended to install node.js using [`nvm`](https://github.com/nvm-sh/nvm).

The project we'll be creating is called `tree-sitter-wdl`, which follows the [preferred convention](https://tree-sitter.github.io/tree-sitter/creating-parsers#project-setup) of naming the parser repository `tree-sitter-<language>`. After completing the setup process, we have a project folder with a [`package.json`](https://github.com/jdidion/tree-sitter-wdl/blob/main/package.json) file that describes our project and provides the configuration required by `npm` to build our parser. The key parts of this file are described below, excluding any [metadata](https://docs.npmjs.com/cli/v9/configuring-npm/package-json) that is just needed for publishing the package in the npm repository.

```json{title="package.json"}
{
  "name": "tree-sitter-wdl",
  "version": "0.1.0",
  "main": "bindings/node",
  "scripts": {
    "build": "tree-sitter generate && npm install",
    "test": "tree-sitter test"
  },
  "dependencies": {
    "nan": "^2.17.0"
  },
  "devDependencies": {
    "tree-sitter-cli": "^0.20.7"
  },
  "tree-sitter": [
    {
      "scope": "source.wdl",
      "file-types": [
        "wdl"
      ],
      "injection-regex": "wdl"
    }
  ]
}
```

* `name`: A unique name for the package, following Tree-sitter's preferred convention.
* `version`: The initial version of the package, following the conventions of [semantic versioning](https://semver.org/). It's a good idea to use a major version `0` until the grammar is complete, finialized, and well-tested.
* `main`: The path to the main file when the package is used as a node library or executable. The files in `bindings/node` will be generated by the Tree-sitter CLI.
* `scripts`: Allows defining aliases for commonly used commands. These can then be run using `npm <alias>`. I've defined a `build` alias for generating and building the parser, and a `test` alias for running the test cases that we'll write later on.
* `dependencies`: Dependencies that are required to use the generated parser. All Tree-sitter parsers have one required dependency, `nan`.
* `devDependencies`: Dependencies that are required to generate and build the parser. All Tree-sitter parsers have one required dev dependency, `tree-sitter-cli`.
* `tree-sitter`: Configuration for using the grammar for [syntax highlighting](https://tree-sitter.github.io/tree-sitter/syntax-highlighting#language-configuration).

## Testing the parser generator

Now that we have the project set up, we can get to work on the grammar. The grammar is defined in a file called `grammar.js` in the root directory of the project. We are going to start with a bare-bones grammar just to make sure the CLI is working as expected, and to get familiar with the different files that it generates.

```javascript{title="A bare-bones grammar"}
module.exports = grammar({
  name: "wdl",
  rules: {
    document: $ => $.version,
    version: $ => seq("version", "1.1")
  } 
});
```

Let's walk through our initial grammar file line-by-line:

1. Node.js implements the [CommonJS](https://en.wikipedia.org/wiki/CommonJS) standard, which (among other things) specifies how JavaScript modules are defined outside of a web browser context. The `module` object is defined in each JavaScript file, and its [`exports`](https://nodejs.org/docs/latest/api/modules.html#moduleexports) field contains an object that is exposed to a user of the module. It is initialized to an empty object, but often (as in the example above) it is reassigned to a class instance. The [https://github.com/tree-sitter/tree-sitter/blob/master/cli/src/generate/dsl.js](`grammar`) function is provided by Tree-sitter to convert a grammar definition into an object that is serialized to JSON.
2. The grammar definition is a JavaScript object whose fields are specified by the [grammar DSL](https://tree-sitter.github.io/tree-sitter/creating-parsers#the-grammar-dsl). `name` is a required field whose value is the name of the language. Tree-sitter will use the langauge name in several places, e.g., the names of functions in the code it generates, to make them unique (versus other Tree-sitter-generated parsers).
3. `rules` is a required field with an object value, whose keys are rule names and whose values are production rules.
4. A WDL source file is called a "document", and the grammar defines a corresponding `document` rule. A production rule is written as a JavaScript function with a single parameter, called `$` by convention, and whose body is either a literal or a call to one of the functions provide by the DSL. The `$` parameter is an object whose fields are the rules defined in the grammar. The body of the `document` rule specifies that a document contains a single element, which is defined by the `version` rule. Note that rules don't have to be defined in any certain order - forward references are okay.
5. The `version` rule is defined using the `seq` function, which specifies a sequence of rules that are expected to match in order. The `version` rule matches the tokens `version` followed by `1.1`, with only whitespace allowed in between.

After creating the `grammar.js` file, the structure of the project looks like this:

```txt{title="Initial project structure"}
tree-sitter-wdl
├── grammar.js
├── node_modules
└── project.json
```

You will only see `node_modules` if you installed the Tree-sitter CLI during project setup (otherwise it will be installed automatically during the next step). Now, if we've done everything correctly, the following command should generate our parser:

```txt{title="Generating the parser"}
$ npm run build
gyp info it worked if it ends with ok
gyp info using node-gyp@9.3.0
...
gyp info ok 
```

[Gyp](https://github.com/nodejs/node-gyp) is a build system that builds native addon modules for node. Tree-sitter generates a build file and then uses `gyp` to actually compile the generated parser and bindings.

Now our project directory has a lot more files:

```txt{tilte="Project structure after parser generation"}
tree-sitter-wdl
├── binding.gyp         # build file for the gyp build system built into node
├── bindings
│   ├── node            # Node bindings for the generated parser
│   └── rust            # Rust bindings for the generated parser
├── build               
│   └── ...             # artifacts of the build process
├── Cargo.toml          # project file for Rust bindings
├── grammar.js
├── node_modules
├── package_lock.json   # lock file with specific dependency versions
├── project.json
└── src
    ├── grammar.json    # the serialized grammar object
    ├── node-types.json # metadata for the node types in the tree generated by the parser
    ├── parser.c        # the generated parser
    └── tree_sitter
        └── parser.h    # header file for the generated parser
```

We'll come back to some of these files later. The important thing for now is that Tree-sitter did what we expected it to do. We should now be able to use the CLI to parse a WDL file that matches our grammar:

```txt{title="Parsing a WDL file that matches the initial grammar"}
$ printf "version 1.1\n" | tree-sitter parse /dev/stdin
(document [0, 0] - [1, 0]
  (version [0, 0] - [0, 11]))
```

The `parse` command attempts to parse the input text, and, if successful, prints out the parse tree as an [S-expression](https://en.wikipedia.org/wiki/S-expression). Each node of an S-expression is of the form `(name children*)`, where `children*` is zero or more child nodes. Note that the S-expression only shows the matched rules - any terminal nodes (i.e., tokens) are *not* shown.

Tree-sitter also prints out the span of each node - the `[line, column]` positions of the start and end of the matching text within the input string. In Tree-sitter, both line and column positions start at `0`, and column positions are *end-exclusive*. So in the example above, we see that the `version` node matches the 11 characters starting at line `0`, column `0`, and ending at line `0`, column `10` (inclusive). The `document` node matches the entire input text, which also includes a newline, so it ends on line `1`.

## Writing some simple grammar rules

Right now our WDL grammar is correct, but it's nowhere near complete. The `version` statement *must* appear on the first non-comment line of a WDL document, but a valid WDL document must also contain a "body" consisting of one or more top-level elements in any order:

* `task`: The fundamental unit of computation. A task defines its input and output variables, it encapsulates a [Bash](https://en.wikipedia.org/wiki/Bash_(Unix_shell)) script that can reference those variables, and it specifies a Docker image in which to execute the script.
* `workflow`: Represents a directed, acyclic graph (DAG) of tasks. The execution order of the tasks in a workflow is defined implicitly by the dependencies between the tasks.
* `struct`: A user-defined type. WDL has built-in support for primitive types (e.g., strings, numbers, booleans, and files) and a few generic collection types; structs provide the ability to create more specialized compound types.

A WDL document may also include any number of top-level `import` statements, which enable referencing the elements of one WDL document from another.

In other words, we want our grammar to express:

1. Production rules for the 4 top-level variants
2. That the `version` rule must come first
3. That at least one of the `task`, `workflow`, or `struct` variants is required
4. That any number of top-level items is allowed

The Tree-sitter grammar DSL provides a few functions to facilitate writing these rules:

* `seq` takes two or more rules (or tokens) and matches them in order
* `choice` selects from among multiple variants
* `repeat1` takes one rule or token and matches it one or more times consecutively

With these functions, we can expand our grammar to meet *almost* all our goals.

```javascript{title="Our grammar with a few more rules"}
module.exports = grammar({
  name: "wdl",
  rules: {
    document: $ => seq(  // enforces that version comes first
      $.version,
      repeat1(  // enforces that there are one-or-more items
        choice(  // selects between multiple variants
          $.import,
          $.struct,
          $.workflow,
          $.task
        )
      )
    ),
    version: $ => seq("version", "1.1"),
    import: $ => todo("import"),
    struct: $ => todo("struct"),
    workflow: $ => todo("workflow"),
    task: $ => todo("task"),
  } 
});

function todo(rule) {
  console.warn(`Missing implementation of rule ${rule}`);
  return rule
}
```

Note that we can take advantage of the fact that a grammar file is just a node module and add our own JavaScript functions. In this case, we've defined a `todo` function that we can use as a placeholder for the rules we need to implement. It prints a warning and returns the rule name as a token, which will allow the grammar to build successfully even though it is incomplete.

The main problem with our current grammar is that it would parse the following as a valid document, when in fact it is not, since it doesn't contain one of the three required top-level elements.

```wdl
version 1.1
import
```

Expressing something like "allow rules A, B, C, and D to match in any order, but require at least one of A|B|C" is actually somewhat challenging. It can be done by employing another built-in fuction, `repeat`, which takes one rule or token and matches it *zero* or more times consecutively. However, it makes for a (perhaps overly) complex rule.

```javascript{title="A more correct and complex grammar"}
module.exports = grammar({
  name: "wdl",
  rules: {
    document: $ =>seq(
      repeat(  // zero or more items
        choice(
            $.import,
            $.struct,
            $.workflow,
            $.task
        )
      ),
      repeat1(
        choice(
            $.struct,
            $.workflow,
            $.task
        )
      ),
      repeat(
        choice(
            $.import,
            $.struct,
            $.workflow,
            $.task
        )
      )
    )
  }
})
```

Whether you want to do something like this in your own grammar is to some degree a matter of personal choice, although it can also slow down your parser. For the [official WDL grammar](/Users/jdidion/projects/test/tree-sitter-wdl), I opted to go for simpler rules and defer the validation of more complex requirements like "there must be at least one task, workflow, or struct" until after parsing the document.

## Calling the parser from Rust

Our grammar is still far from complete, but it's at least interesting enough now that we can try using it from a Rust program.

Recall that when we generated our parser, Tree-sitter also generated a bunch of other files. The ones we're interested in right now are:

```txt
tree-sitter-wdl
├── bindings
│  |_ rust
│     |_ build.rs  # tells cargo how to build the C parser
│     |_ lib.rs    # Rust bindings to the C parser library
|_ src
│  |_ parser.c
|_ Cargo.toml      # project file for Rust bindings
```

These files enable Cargo (the Rust build tool) to compile the C parser (`src/parser.c`), load the resulting object file (which will be found at `build/Release/obj.target/tree_sitter_wdl_binding/src/parser.o`), provide a Rust function to access the parser as an instance of [`tree_sitter::Language`](https://github.com/tree-sitter/tree-sitter/blob/master/lib/binding_rust/lib.rs), and package all these pieces as a "crate" (an installable package).

A `build.rs` file is just a Rust binary, i.e., it has a `main()` function that is called when it is compiled and executed by Cargo. The generated `bindings/rust/build.rs` uses the [`cc`](https://docs.rs/cc/latest/cc/) crate to call the C compiler.

```rust{title="build.rs"}
fn main() {
    let src_dir = std::path::Path::new("src");
    let mut c_config = cc::Build::new();
    c_config.include(&src_dir);  // add src_dir to the include path
    c_config  // set some compiler options
        .flag_if_supported("-Wno-unused-parameter")
        .flag_if_supported("-Wno-unused-but-set-variable")
        .flag_if_supported("-Wno-trigraphs");
    let parser_path = src_dir.join("parser.c");
    c_config.file(&parser_path);  // set the file to be compiled
    // tell cargo to recompile only if the parser source file has changed
    println!("cargo:rerun-if-changed={}", parser_path.to_str().unwrap());  
    c_config.compile("parser");  // execute the compiler
}
```

The `lib.rs` file uses an [`extern`](https://doc.rust-lang.org/std/keyword.extern.html) block to define a Rust function (`tree_sitter_<name>`, where `name` is the language name from the grammar) that calls the function of the same name in the C object file produced by the build script. Rust considers calling any external function to be unsafe, so it is necessary to wrap any call to `tree_sitter_wdl` in an [`unsafe`](https://doc.rust-lang.org/std/keyword.unsafe.html) block. It is common practice to make external functions private and wrap them with a function that performs the unsafe call.

```rust{title="The generated lib.rs"}
use tree_sitter::Language;

extern "C" {
    fn tree_sitter_wdl() -> Language;
}

pub fn language() -> Language {
    unsafe { tree_sitter_wdl() }
}
```

The `tree_sitter::Language` struct contains all the configuration specific to our language. We can use the instance returned by the `language` function to create a `tree_sitter::Parser`, which will actually enable us to parse a WDL document. We can add some additional functions to our `lib.rs` to make things easier for a user of our crate. We can also use the [`thiserror`](https://docs.rs/thiserror/latest/thiserror/) crate to create informative errors.

```rust{title="lib.rs with errors and convenience functions"}
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ParserError {
    #[error("Error creating parser for WDL 1.x")]
    Language { source: tree_sitter::LanguageError },  // wrap the original error
    #[error("WDL document is empty")]
    DocumentEmpty,
}

/// Returns a `Parser` with the language set to `language()`.
pub fn parser() -> Result<tree_sitter::Parser, ParserError> {
    let mut parser = tree_sitter::Parser::new();
    parser
        .set_language(language())
        .map_err(|source| ParserError::Language { source })?;
    Ok(parser)
}

/// Parses a WDL document. Returns a `tree_sitter::Tree` if successful,
/// or a `ParserError` if the document is empty.
pub fn parse_document(text: &str) -> Result<tree_sitter::Tree, ParserError> {
    let mut parser = parser()?;
    parser
        .parse(text, None)
        .ok_or_else(|| ParserError::DocumentEmpty)
}
```

The `Cargo.toml` file is the Rust equivalent of `package.json` - it contains the configuration necessary to compile the package's code, including any dependencies on other crates, as well as metadata used for publishing the package to the [`crates.io`](https://crates.io/) repository when we're ready to make our parser available to others. The key parts of this file are shown below.

```toml{title="Cargo.toml"}
[package]
name = "tree-sitter-wdl"
version = "0.1.0"
build = "bindings/rust/build.rs"  # path to the build file
include = [  # files to include when packaging the crate
  "bindings/rust/*",
  "grammar.js",
  "src/*",
]

[lib]
path = "bindings/rust/lib.rs"  # path to the main Rust file

[dependencies]
thiserror = "1.0.38"    # dependency on thiserror, which we used in lib.rs
tree-sitter = "0.20.9"  # runtime dependency

[build-dependencies]
cc = "1.0"  # the dependency on cc is only needed at build time
```

To build our package into a crate, we use the `cargo build` command. The first time we run `cargo build` in our project, Cargo downloads and compiles all the dependencies (and dependencies-of-dependencies, etc.) before compiling and packaging our package code.

```txt{title="Building the Rust bindings"}
$ cargo build
    Updating crates.io index
  Downloaded tree-sitter v0.20.10
  Downloaded regex-syntax v0.6.29
  Downloaded regex v1.7.3
  Downloaded 3 crates (674.1 KB) in 0.77s
   Compiling memchr v2.5.0
   Compiling cc v1.0.79
   Compiling regex-syntax v0.6.29
   Compiling tree-sitter v0.20.10
   Compiling tree-sitter-wdl v0.0.1 (/Users/jdidion/projects/test/tree-sitter-wdl)
   Compiling aho-corasick v0.7.20
   Compiling regex v1.7.3
    Finished dev [unoptimized + debuginfo] target(s) in 13.86s
```

We can find the binary in `target/debug/libtree_sitter_wdl.rlib`. We can't really do anything with this `rlib` file - it's a Rust-specific format and is only meant to be used as a dependency in another crate.

We could create another create to import and test out our parser, but an easier approach is to create [unit tests](https://doc.rust-lang.org/rust-by-example/testing/unit_testing.html) directly in our parser `lib.rs`. In Rust, unit tests are placed in a separate module annotated with `#[cfg(test)]`. Any function in a test module that is annotated with `#[test]` is executed by the `cargo test` command, and any errors are reported as test failures.

```rust{title="Basic unit tests"}
#[cfg(test)]
mod tests {
    /// Test that the parser can be created.
    #[test]
    fn test_can_load_grammar() {
        // use `super` to access a function in the parent module
        super::parser().expect("Error loading wdl language");
    }

    /// Test that a simple WDL file can be parsed.
    #[test]
    fn test_parse() {
        // use the `CARGO_MANIFEST_DIR` environment veriable, which is set
        // by cargo, to determine the project root directory
        let wdl_file = std::path::PathBuf::from(env!("CARGO_MANIFEST_DIR"))
            .join("resources")
            .join("test")
            .join("simple.wdl");
        let wdl_source = std::fs::read_to_string(wdl_file).unwrap();
        super::parse_document(&wdl_source).unwrap();
    }
}
```

Here we've created a `tests` module with two test functions: `test_can_load_grammar`, which just tries to instantiate a `tree_sitter::Parser` with the WDL `tree_sitter::Language` instance, and `test_parse`, which actually tries to parse a WDL file. The `test_parse` test is looking for a file at `resources/test/simple.wdl`; let's create one that matches our current grammar:

```wdl{title="resources/test/simple.wdl"}
version 1.1
workflow
```

Now we can run `cargo test`, and we should see:

```txt{title="Test output"}
running 2 tests
test tests::test_can_load_grammar ... ok
test tests::test_parse ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## Conclusion

In this second post on writing a parser in Rust, we've implemented a complete (although very simple) GLR parser in Rust using Tree-sitter. In Part 3, we'll expand our WDL grammar, and cover more advanced aspects of Tree-sitter in the process.