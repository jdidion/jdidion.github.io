+++
title = "Parsing with Rust - Part 4: writing a PEG grammar with Pest"
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

The fourth post in a series on writing programming-language parsers in Rust. In [Part 1](/posts/parsing-with-rust-part1/), we covered general parsing concepts and looked at some of the Rust crates available for generating a parser. In Parts [2](/posts/parsing-with-rust-part2/) and [3](/posts/parsing-with-rust-part2/), we wrote a [Tree-sitter](https://tree-sitter.github.io/tree-sitter/) grammar for [WDL](https://openwdl.org) and used it from Rust. In this post, we'll   write a [Parsing Expression Grammar (PEG)](https://en.wikipedia.org/wiki/Parsing_expression_grammar) parser for WDL using [Pest](https://pest.rs/).

<!--more-->

