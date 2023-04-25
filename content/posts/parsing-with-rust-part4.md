+++
title = "Parsing with Rust - Part 3: writing a PEG grammar with Pest"
date = "2023-03-20"
author = "John Didion"
authorTwitter = "jdidion" #do not include @
tags = ["rust"]
keywords = ["rust", "parsing", "wdl"]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
draft = true
+++

The third post in a series on writing programming-language parsers in Rust. In [Part 1](/posts/parsing-with-rust-part1/), I introduced general parsing concepts and looked at some of the Rust crates available for generating a parser. In [Part 2](/posts/parsing-with-rust-part2/), I did a deep-dive into [Tree-sitter](https://tree-sitter.github.io/tree-sitter/) and showed how to write a parser for [WDL](https://openwdl.org). In this post, I walk through writing a [Parsing Expression Grammar (PEG)](https://en.wikipedia.org/wiki/Parsing_expression_grammar) and generating a parser using [Pest](https://pest.rs/). I also talk about writing a [testing framework](https://github.com/jdidion/pest-test) for Pest grammars.

<!--more-->
