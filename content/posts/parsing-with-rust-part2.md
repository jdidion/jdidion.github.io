+++
title = "Parsing with Rust - Part 2: writing an LR grammar with Tree-sitter"
date = "2023-03-16"
author = "John Didion"
authorTwitter = "jdidion"
tags = ["rust", "parsing", "wdl"]
keywords = ["rust", "parsing", "wdl", "tree-sitter"]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = ""
draft = true
+++

The second post in a series on writing programming-language parsers in Rust. In [Part 1](/posts/parsing-with-rust-part1/), I introduced general parsing concepts and looked at some of the Rust crates available for generating a parser. In this post, I walk through writing a [Tree-sitter](https://tree-sitter.github.io/tree-sitter/) grammar for [WDL](https://openwdl.org) - a domain-specific language for describing computational workflows - and using the parser in Rust.

<!--more-->

As I briefly covered in [Part 1](/posts/parsing-with-rust-part1/), [Tree-sitter](https://crates.io/crates/tree-sitter) is a widely used parser generator framework that generates a Generalized LR (GLR) parser with Rust bindings from a grammar specification.

A [Tree-sitter grammar](https://tree-sitter.github.io/tree-sitter/creating-parsers#the-grammar-dsl) is specified in JavaScript. More specifically, it is a [Node.js](https://nodejs.org/) module that exports a grammar definition, which is created by calling the `grammar()` function with a specially formatted object.

```javascript
// language name
const LANGUAGE = "wdl"

module.exports = grammar({ 
    ... 
})
```

Once you've written the grammar, you can generate the parser using the [command line tool](https://tree-sitter.github.io/tree-sitter/creating-parsers#command-generate). The generated parser is a C language file that can be compiled and used from a C/C++ program. In addition, Tree-sitter generates both Rust and Node bindings, making it easy to use the parser from either of those languages.

In the remainder of this post, I'll walk through each of these steps in detail - writing the grammar, generating the parser, and using it from a Rust program. From here on I will be using [WDL](https://openwdl.org)

 - a domain-specific language for describing computational workflows - and using the parser in Rust.

## Writting the Tree-sitter grammar

