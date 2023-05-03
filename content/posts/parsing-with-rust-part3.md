+++
title = "Parsing with Rust - Part 3: completing the Tree-sitter grammar"
date = "2023-04-25"
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

The third post in a series on writing programming-language parsers in Rust. In [Part 1](/posts/parsing-with-rust-part1/), we covered general parsing concepts and looked at some of the Rust crates available for generating a parser. In [Part 2](/posts/parsing-with-rust-part2/), we started writing a [Tree-sitter](https://tree-sitter.github.io/tree-sitter/) grammar for [WDL](https://openwdl.org) and used it from Rust. In this post, we'll implement some of the more interesting parts of the grammar and learn about the rest of the [Tree-sitter DSL](https://tree-sitter.github.io/tree-sitter/creating-parsers#the-grammar-dsl) in the process. The [complete WDL grammar](https://github.com/jdidion/tree-sitter-wdl/blob/main/grammar.js) is available in the GitHub repository.

<!--more-->

## Parsing expressions: precedence and associativity

A Tree-sitter parser encounters a [*conflict*](https://tree-sitter.github.io/tree-sitter/creating-parsers#conflicting-tokens) when two or more rules match the current context. The parser has mechanisms for resolving conflicts, one of which must yield a single matching rule for parsing to be able to continue.

The most basic mechanism for resolving conflicts is to compare the *precedences* of the conflicting rules. Every rule in a Tree-sitter grammar has an integer-valued precedence; if exactly one of the conflicting rules has a higher precedence than the others, it is selected as the matching rule.

The most common scenario in which precedence is needed to resolve conflicts is in the parsing of expressions. A WDL [expression](https://github.com/openwdl/wdl/blob/main/versions/1.1/SPEC.md#expressions) is a literal value (e.g., number or string), an identifier (a reference to a declared variable), or an operation (e.g., addition or multiplication).

Parsing expressions can be challenging for two reasons:

* Expressions can be left-recursive. Any operand on the right-hand-side of an expression may be an expression itself. Although Tree-sitter parsers can handle left recursion, we learned in [Part 1](/posts/parsing-with-rust-part1/) that some types of parsers cannot, which necessitates the [topological ordering](https://en.wikipedia.org/wiki/Left_recursion#Removing_left_recursion) of nonterminals (we'll see an example of this in a later post in this series).
* Operators have a definite ordering. An expression cannot be simply parsed left-to-right, because an operator of higher precedence may occur to the right of one with lower precedence.

For example, consider the expression `1 + 2 * 3`. The parser could generate either of two trees: `(1 + 2) * 3` or `1 + (2 * 3)`. We know that the second tree is correct, because multiplication has a higher precedence than addition, but we need a way to communicate this to the parser generator.

To implement operation ordering, tree sitter provides the `prec` function to change a rule's default precedence of `0` to a positive or negative integer value. There are also a few variants of `prec` that support additional conflict resolution mechanisms:

* `prec.left`: Assigns a precedence, but also directs the parser to select the rule that results in the *shortest* match when there are multiple rules with equal precedence.
* `prec.right`: Assigns a precedence, but also directs the parser to select the rule that results in the *longest* match when there are multiple rules with equal precedence.
* `prec.dynamic`: Assigns a *dynamic* precedence, which is used during dynamic conflict resolution. We can write a complete WDL parser without needing dynamic conflict resolution, so we won't cover it in this series, but you can read about it in the [manual](https://tree-sitter.github.io/tree-sitter/creating-parsers#the-grammar-dsl).

As an example, let's write a grammar to parse integer addition and multiplication operations:

```JavaScript{title="Expression grammar"}
module.exports = grammar({
  name: "wdl",
  rules: {
    ...
    expression: $ => choice(
      $.addition,
      $.multiplication,
      $.integer
    ),
    addition: $ => prec.left(1, seq(
      $.expression,
      "+",
      $.expression
    )),
    multiplication: $ => prec.left(2, seq(
      $.expression,
      "*",
      $.expression
    )),
    integer: $ => prec(3, /[\+-]?\d+/)
  }
})
```

Our grammar expresses that:

* `addition`, `multiplication`, and `integer` are all expressions 
* `integer` has higher precedence than `multiplication`, which has higher precedence than `addition`
* The operands of `addition` and `multiplication` can match expressions recursively with left associativity.

The use of `prec.left` in the `addition` and `multiplication` rules is important because it resolves ambiguity for expressions like `1 * 2 * 3`, which could be parsed as `1 * (2 * 3)` or `(1 * 2) * 3`. Precedence doesn't help here because both operations are multiplication, and thus have the same precedence. Left associativity says to choose the second tree over the first.

## Improving parse tree ergonomics

We can make some small usability improvements to our WDL grammar that will make it easier to work with the syntax tree generated by the parser.

The first change we can make is to have our grammar better reflect the structure of WDL documents. For example, right now, we have all of the top-level elements as children of `document`, but in reality the `version` statement is different from the others. We can think of `version` as a header - it must appear once as the first statement in the document - while the other nodes comprise the "body" of the document. To mirror this structure in the grammar, we can introduce a new `document_body` rule whose children are the rules for the four body elements.

Additionally, we can "hide" rules do not map to concrete syntax elements in our grammar, but exist to group together related rules (typically using a `choice`) and eliminate redundancy (Tree-sitter refers to these kinds of rules as "supertypes"). For example, there are many rules in the WDL grammar that match to any expression, so it makes sense to have an `expression` rule rather than copy the same `choice` statement everywhere. But we can hide the expression rule so that it doesn't create an extra level of depth in the parse tree. A hidden rule has a name that begins with an underscore (`_`). Hiding a rule does *not* hide its productions.

A third improvement we can make is to add a unique name to each node that we may want to access in the parse tree. This will make it possible to retrieve individual nodes by name rather than having to iterate over all the nodes to find the one we want. Tree-sitter refers to these as "field names", and they are added by the `field` function in the grammar DSL.

Let's introduce these changes to our grammar. In addition, we'll add a few more rules to tie everything together:

* `workflow` can now have a name and a body.
* A workflow body has opening and closing braces (`{}`) and contains any number of statements. This is such a common pattern that we add another JavaScript function, `block`, to create a `seq("{", rule, "}")` given any `rule`.
* For now, a `workflow_body` can only contain `declaration`s.
* A `declaration` creates a variable with a type (represented by the hidden rule `_type`), a name, and a value assignment, which is `=` followed by an expression.
* For now, we only allow `Int` declarations.
* `workflow` and `declaration` names are `identifier`s. An `identifier` must start with a letter and may only contain letters, numbers, and underscores (`_`).
* An `identifier` may also be used in an expression to refer to a previously declared variable.

Also note the addition of two top-level fields:

* `word` defines a rule producing an `identifier`. This is necessary for correct parsing of keyword tokens, and is also a performance optimization. The description of why this is necessary is a bit technical - see the [manual](https://tree-sitter.github.io/tree-sitter/creating-parsers#keywords) if you're interested in the details.
* `supertypes` defines a list of nodes to treat as supertypes. This has no effect on the generated parser; it just adds some extra information to the [`node-types.json`](https://tree-sitter.github.io/tree-sitter/using-parsers#static-node-types) file.

```JavaScript{title="A more ergonomic grammar"}
module.exports = grammar({
  name: "wdl",
  word: $ => $.identifier,
  supertypes: $ => [
    $._expression
  ],
  rules: {
    document: $ => seq(
      field("version", $.version),
      field("body", $.document_body),
    ),
    document_body: $ => repeat1(
      choice(
        $.import,
        $.struct,
        $.workflow,
        $.task
      )
    ),
    workflow: $ => seq(
      "workflow",
      field("name", $.identifier),
      field("body", $.workflow_body)
    ),
    workflow_body: $ => block(
      repeat($.declaration)
    ),
    declaration: $ => seq(
      field("type", $._type),
      field("name", $.identifier),
      "=",
      field("expression", $._expression)
    ),
    _type: $ => choice(
      $.int_type
    ),
    int_type: $ => "Int",
    _expression: $ => choice(
      $.addition,
      $.multiplication,
      $.integer,
      $.identifier
    ),
    addition: $ => prec.left(1, seq(
      field("left", $._expression),
      field("operator", "+"),
      field("right", $._expression)
    )),
    multiplication: $ => prec.left(2, seq(
      field("left", $._expression),
      field("operator", "*"),
      field("right"$._expression)
    )),
    integer: $ => prec(3, /[\+-]?\d+/),
    identifier: $ => /[a-zA-Z][a-zA-Z0-9_]*/
  } 
});

function block(rule) {
  return seq("{", rule, "}")
}
```

These additions to our grammar enable us to write a more interesting test case:

```text{title="resources/test/interesting.wdl", kind="WDL"}
version 1.1

workflow test {
  Int i = 1  # a declaration
  Int j = 2  # another declaration
  Int k = i + j * 3  # an expression with identifiers referring to declarations
}
```

With these changes to the grammar, we can simplify our test case somewhat. Note that the performance is probably not any better than before - we avoid making one of the copies of the cursor, but instead we're performing redundant navigation of the parse tree because each call to `child_by_field_name` moves the cursor to the first child node, then to each successive sibling node until the node with the given field name is found, then back to the parent node. 

```rust{title="Updated unit test"}
#[test]
fn test_parse() {
    let wdl_file = PathBuf::from(env!("CARGO_MANIFEST_DIR"))
        .join("resources")
        .join("test")
        .join("interesting.wdl");
    let wdl_source = std::fs::read_to_string(wdl_file).unwrap();
    let tree = super::parse_document(&wdl_source).unwrap();
    let root = tree.root_node(); // get the root (`document`) node in the tree
    // now we can get child nodes by their field names
    let version = root
        .child_by_field_name("version")
        .expect("Expected version node");
    let body = root
        .child_by_field_name("body")
        .expect("Expected body node");
    let cursor = &mut body.walk();
    // if we use the cursor within a scope that returns an owned Node instance,
    // then we're free to reuse the cursor after the end of the scope to get
    // the Node's children
    let workflow_body = {
        let mut body_children = body.children(cursor);
        let workflow = body_children.next().expect("Expected workflow node");
        let workflow_name = workflow
            .child_by_field_name("name")
            .expect("Expected name node");
        assert_eq!(
            workflow_name.utf8_text(&wdl_source.as_bytes()).unwrap(),
            "test"
        );
        workflow
            .child_by_field_name("body")
            .expect("Expected body node")
    };
    // we can also collect child nodes into a Vec and reuse the cursor afterwards
    let body_children: Vec<tree_sitter::Node> = workflow_body.children(cursor).collect();
    assert_eq!(body_children.len(), 3);
    let k = body_children[2];
    assert_eq!(k.kind(), "declaration");
    let mut k_children = k.children(cursor);
    let k_type = k_children.next().expect("Expected type node");
    assert_eq!(k_type.kind(), "int_type"); // there is no `_type` node since it's hidden
    let k_name = k_children.next().expect("Expected name node");
    assert_eq!(k_name.kind(), "identifier");
    assert_eq!(k_name.utf8_text(&wdl_source.as_bytes()).unwrap(), "k");
    assert_eq!(k_children.next().expect("Expected = node").kind(), "=");
    let k_expr = k_children.next().expect("Expected expression node");
    assert_eq!(k_expr.kind(), "addition"); // there is no `_expression` node
    let left = k_expr
        .child_by_field_name("left")
        .expect("Expected left operand node");
    assert_eq!(left.kind(), "identifier");
    assert_eq!(left.utf8_text(&wdl_source.as_bytes()).unwrap(), "i");
    assert_eq!(
        k_expr
            .child_by_field_name("operator")
            .expect("Expected operator node")
            .kind(),
        "+"
    );
    let right = k_expr
        .child_by_field_name("right")
        .expect("Expected right operand node");
    assert_eq!(right.kind(), "multiplication"); // again, no `_expression` node
    let left = right
        .child_by_field_name("left")
        .expect("Expected left operand node");
    assert_eq!(left.kind(), "identifier");
    assert_eq!(left.utf8_text(&wdl_source.as_bytes()).unwrap(), "j");
    assert_eq!(
        k_expr
            .child_by_field_name("operator")
            .expect("Expected operator node")
            .kind(),
        "*"
    );
    let right = k_expr
        .child_by_field_name("right")
        .expect("Expected right operand node");
    assert_eq!(right.kind(), "integer");
    assert_eq!(right.utf8_text(&wdl_source.as_bytes()).unwrap(), "3");
}
```

## Parsing string literals

WDL has two types of strings:

* Literals: May contain any characters between pairs of quotes. Either single (`'`) or double (`"`) quotes may be used, but the starting and ending quote character must match. Some characters must be [escaped](https://github.com/openwdl/wdl/blob/main/versions/1.1/SPEC.md#strings). Literals may be evaluated statically (i.e., no runtime context is required).
* Interpolated: An interpolated string is just like a literal, except that it may also contain any number of [expression placeholders](https://github.com/openwdl/wdl/blob/main/versions/1.1/SPEC.md#expression-placeholders-and-string-interpolation). A placeholder is of the form `~{expression}`, where `expression` is any expression that evaluates to a value that can be represented as a string. The expression is evaluated at runtime, and the result is converted to a string and replaces the placeholder.

In the example below, the `"hello.wdl"` string used in the `import` statement is a literal. String literals are also used in the `meta` section, which contains arbitrary metadata about a workflow or task and does not allow expressions. The expression assigned to `greeting` is an interpolated string.

```text{title="WDL strings example", kind="WDL"}
import "hello.wdl"

workflow {
  input {
    String name
    String greeting = "Hello ~{name}"
  }

  meta {
    description: "A workflow that says hello"
  }
  ...
}
```

Writing the grammar for string literals is much easier than for interpolated strings, so we'll tackle that first. To write the rules for parsing string literals, we'll need to introduce a few more functions provided by the grammar DSL:

* `optional`: Indicates that a rule/literal is not required - if it does not match the next token, the parser may skip it and proceed to the next production.
* `alias`: Changes the name of node in the parse tree. In the grammar, all rules must have a unique name. However, two nodes generated by different rules may have the same semantic meaning - aliasing them to the same name can simplify parsing.
* `token`: Combines all the tokens that match the rule into a single token. For example, the `escape_sequence` rule below matches the escape sequence `"\000"` as two tokens, `\` and `000`, and then combines them into the single token `\000`.

In addition, we'll need to use regular expression (regex) literals (rather than strings) to match the variable content allowed in WDL strings. A regex is written as a standard [JavaScript regular expression](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Regular_expressions). For example, `/[A-Z]+/` would match one or more upper-case characters from the Latin alphabet.

```JavaScript{title="String literal grammar"}
module.exports = grammar({
  name: "wdl",
  rules: {
    ...
    string_literal: $ => choice(
      seq(
        "'",
        optional(field("parts", alias($.string_literal_squote_parts, $.string_parts))),
        "'"
      ),
      seq(
        "\"",
        optional(field("parts", alias($.string_literal_dquote_parts, $.string_parts))),
        "\""
      ),
    ),
    string_literal_squote_parts: $ => repeat1(
      choice(
        $.escape_sequence,
        alias(/[^'\\\n]+/, $.content)
      )
    ),
    string_literal_dquote_parts: $ => repeat1(
      choice(
        $.escape_sequence,
        alias(/[^"\\\n]+/, $.content)
      )
    ),
    escape_sequence: $ => token(seq(
      "\\",
      choice(
        /u[a-fA-F\d]{4}/,
        /U[a-fA-F\d]{8}/,
        /x[a-fA-F\d]{2}/,
        /[0-7]{1,3}/,
        /['"nt\\]/
      )
    ))
  }
})
```

We implement the `string_literal` rule as a choice between a single-quoted or double-quoted string. A string may be empty, so we make the body of the string `optional`. We alias `string_literal_squote_parts` and `string_literal_dquote_parts` to the same node name (`string_parts`) because they are semantically identical, i.e., the type of quoting should not affect how we handle the results of parsing the string body.

The string body consist of one or more "parts", where a part is either an escape sequence or a sequence of unescaped characters ("content"). It is necessary to use different rules for single- and double-quoted string bodies because an unescaped single quote is not allowed in the former, whereas an unescaped double quote is not allowed in the latter. We alias the two different regular expressions to the same node name because, again, they are semantically identical.

Remember that starting a regular expression character set with `^` indicates negation, which means that any character *except* those in the set will match. The use of `+` after the character set indicates a "greedy" match, i.e., the parser will match as many characters as possible until it encounters either the quote character matching the type of string literal or an escape character (`\`). Furthermore, unescaped newlines are not allowed in WDL strings.

Escape sequences include not only special single characters, but also unicode, hexadecimal, and octal sequences.

```text{title="String parsing example", kind="Diagram"}
Parsing the string "Hello \"Jim\"!":
-----------------------------------
" <== start double-quoted string, repeat matching of two string body rules
H <-- start content regex match
e .
l .
l .
o .
  --> look ahead, next character doesn't match, end content 
\ <-- match escape token
" --> match escape regex, return escape sequence as single token
J <-- start content regex match
i .
m --> look ahead, next character doesn't match, end content
\ <-- match escape token
" --> match escape regex, return escape sequence as single token
! <-> start content regex match, look ahead, next character doesn't match, end content
" ==> neither rule matches, end double-quoted string
```

## Parsing interpolated strings

Interpolated strings are a bit more challenging to represent due to the possibility of nested expression placeholders. In the following example, `nested` is a valid WDL string expression:

```text{title="WDL interpolated strings", kind="WDL"}
String name = "John"
Boolean use_name = true
String nested = "Hello ~{if use_name then "~{name}" else "Buddy"}"
```

The grammar for an interpolated string (or "istring") is similar to that for a string literal, with the addition of a `placeholder` rule that has higher precendence than the escape sequence and content rules (note that we also support the deprecated placeholder style `${expression}`).

Notice that string content rules now use `choice` rather than just being a single regex. This is necessary because we want the parser to stop matching content when it encounters one of the placeholder staring characters (`~` and `$`) and give the `placeholder` rule a chance to match. If it doesn't match, then we have variants in the content rules to match just the `~` and `$` characters by themselves.

```JavaScript{title="Interpolated string grammar"}
module.exports = grammar({
  name: "wdl",
  rules: {
    ...
    istring: $ => choice(
      seq(
        "'",
        optional(field("parts", alias($.istring_squote_parts, $.string_parts))),
        "'"
      ),
      seq(
        "\"",
        optional(field("parts", alias($.istring_dquote_parts, $.string_parts))),
        "\""
      ),
    ),
    istring_squote_parts: $ => repeat1(
      choice(
        $.placeholder,
        $.escape_sequence,
        alias($.istring_squote_content, $.content)
      )
    ),
    istring_squote_content: $ => choice(
      /[^'~$\\\n]+/,
      "~",
      "$"
    ),
    istring_dquote_parts: $ => repeat1(
      choice(
        $.placeholder,
        $.escape_sequence,
        alias($.istring_dquote_content, $.content)
      )
    ),
    istring_dquote_content: $ => choice(
      /[^"~$\\\n]+/,
      "~",
      "$"
    ),
    placeholder: $ => prec(1, seq(
      /[~$]{/,
      field("expression", $.expression),
      "}"
    )),
    escape_sequence: $ => token(seq(
      "\\",
      choice(
        /u[a-fA-F\d]{4}/,
        /U[a-fA-F\d]{8}/,
        /x[a-fA-F\d]{2}/,
        /[0-7]{1,3}/,
        /['"nt\\]/
      )
    ))
  }
})
```

## Parsing commands with an external scanner

The [`command` section](https://github.com/openwdl/wdl/blob/main/versions/1.1/SPEC.md#command-section) of a WDL task is a special type of interpolated string. The contents of the command section are interpreted as a Bash script, and so, unlike regular strings, unescaped newlines are allowed, and a line is allowed to end with an unescaped (`\`) indicating a line continuation.

Two forms of the `command` section are supported:

* The preferred "HEREDOC" form (`command <<< ... >>>`), which only supports the `~{}` style of placeholder.
* A deprecated form using enclosing braces (`command { ... }`), which supports both `~{}` and `${}` placeholders.

Parsing the second form of the `command` section is straight-forward. It just requires a minor modification of the content rule we used with interpolated strings.

```JavaScript
command_brace_content: $ => choice(
    /[^~$}\\]+/,
    "\\\n",
    "~",
    "$"
)
```

Most importantly, `}` must be escaped when it is not part of a placeholder, otherwise the end token could match prematurely. For example, consider the following *invalid* WDL task, which has an unescaped `}` in the command section.

```text{title="WDL command missing an escape", kind="WDL"}
task {
  command {
    echo "}" # <-- no rules match, so the command section would end at this }
  }
}
```

Similarly, in the HEREDOC form of `command`, when there are three consecutive `>`, at least one of them must be escaped (e.g. `\>>>`), but one or two consecutive `>` do not need to be escaped. We might try to change the content regex to `/>{0,2}[^~>\\]+/` but this would make it impossible to match a perfectly valid sequence of characters:

```text{title="An unparseable HEREDOC command", kind="WDL"}
task {
  command <<<
  echo ">>\n"
  >>>
}
```

If instead we added a second regex rule `/>{1,2}/`, then an invalid string like `echo ">>>"` would match as the tokens (`echo "`, `>>`, `>`).

Another approach would be to use a lookahead assertion in our regex, e.g., `/>{1,2}(?=[^>])/`. However, when we try this, Tree-sitter gives us an error saying that regex lookahead assertions are not supported. This makes sense when we remember that a Tree-sitter parser only has a single token of lookahead, whereas a regex assertion can lookahead any number of characters.

Thus, the HEREDOC form of `command` presents a syntax that we cannot actually describe using the Tree-sitter DSL. So what do we do? Fortunately, the author of Tree-sitter anticipated this situation and provided an escape-hatch for parsing non-`LR(1)` grammar: an [external scanner](https://tree-sitter.github.io/tree-sitter/creating-parsers#external-scanners). A scanner is a C/C++ file that implements Tree-sitter's external scanner API, which consists of five functions that are all prefixed with `tree_sitter_<language>_external_scanner_`:

* `create`: Creates a new scanner instance.
* `serialize`: Write the scanner's current state (if any) to a byte buffer.
* `deserialize`: Initialize the scanner from a previously serialized state.
* `destroy`: Frees any memory used by the scanner instance.
* `scan`: Implements the custom parsing logic. Given 1) an instance of the scanner, 2) a `lexer` object that provides methods for accessing and manipulating the parser's state, and 3) a list of rules that are considered valid in the current parsing context, this function consumes any number of tokens and informs the parser which of the valid rules was matched.

The steps to implementing an external scanner are:

1. Add an `externals` field to the grammar object that lists which grammar rules should be handled by the external scanner.
2. Implement the scanner. The scanner should define an enumeration (typically called `TokenType`) with variants in the same order as the rules in the `externals` field. The list of valid rules that Tree-sitter passes to the scanner's `scan` function is simply an array of integers that are the indexes of the valid rules in the `externals` field.
3. Modify `binding.gyp` and `bindings/rust/build.rs` to compile the scanner as part of the build process.

Below is the complete grammar for parsing the `command` section. Notice that the `command_heredoc_content` rule appears in the list of choices in the `command_heredoc_parts` rule, but that rule is not defined in our grammar - instead, it appears in the `externals` list. This is the way we tell Tree-sitter that we want our external scanner to handle this rule.

In addition to `command_heredoc_content`, the `externals` array also includes an `error` rule. This rule does not appear in our grammar; it is simply a placeholder. When a Tree-sitter parser encounters an error, it calls the external scanner with every rule included in the list of valid rules, to give the external scanner a chance to recover from the error. So we know that if the `error` rule appears in the list of valid rules, then the parser must be in an error state, since that's the only possible way the `error` rule could be considered valid.

```JavaScript{title="command grammar"}
module.exports = grammar({
  name: "wdl",
  externals: $ => [
      $.command_heredoc_content,
      // Not used in the grammar, but used in the external scanner to check for
      // error state. This relies on the tree-sitter behavior that when an error
      // is encountered the external scanner is called with all symobls marked 
      // as valid.
      $.error,
  ],
  rules: {
    ...
    command: $ => seq(
      "command",
      choice(
        seq(
          "{",
          optional(field("parts", alias($.command_brace_parts, $.command_parts))),
          "}"
        ),
        seq(
          "<<<",
          optional(field("parts", alias($.command_heredoc_parts, $.command_parts))),
          ">>>"
        )
      )
    ),
    command_brace_parts: $ => repeat1(
      choice(
        $.placeholder,
        alias($.command_escape_sequence, $.escape_sequence),
        alias($.command_brace_content, $.content)
      )
    ),
    command_brace_content: $ => choice(
      /([^~$}\\]|\\\n)+/
      "~",
      "$"
    ),
    command_heredoc_parts: $ => repeat1(
      choice(
        $.placeholder,
        alias($.command_escape_sequence, $.escape_sequence),
        alias($.command_heredoc_content, $.content)
      )
    ),
    command_escape_sequence: $ => token(prec(1, seq(
      "\\",
      /[>}~$\\]/
    ))),
  }
})
```

Now we can implement the scanner. Keep in mind that this is about the simplest possible case for a scanner - it only needs to handle a single rule and does not need to maintain any state between calls. If you're interested in a more complex example, you can view the [previous version](https://github.com/jdidion/tree-sitter-wdl/blob/ce3b008b7dfebb35f2032b8b90f0232d951e29a6/src/scanner.cc) of this scanner that I wrote to handle parsing of both interpolated strings and HEREDOC commands (before I realized that the former could be represented using just the grammar DSL).

```C++{title="External scanner"}
#include <tree_sitter/parser.h>
#include <cassert>
#include <cstring>

namespace
{
  const char HEREDOC_END = '>';
  const char LBRACE = '{';
  const char TILDE = '~';
  const char ESCAPE = '\\';
  const char NEWLINE = '\n';

  // Enumeration of token types - these must be in the same order as the 
  // `externals` list in the grammar.
  enum TokenType
  {
    COMMAND_CONTENT,
    ERROR,
  };

  struct Scanner
  {
    // returns true if one of the valid rules is matched, otherwise false
    bool scan(TSLexer *lexer, const bool *valid_symbols)
    {
      // don't try to parse if the parser is in an error state
      if (valid_symbols[COMMAND_CONTENT] && !valid_symbols[ERROR])
      {
        // keep track of whether any tokens have been consumed as part of the 
        // content rule
        bool has_content = false;
        // lookahead contains the next character in the input stream - we can
        // look at it without advancing the lexer
        while (lexer->lookahead)
        {
          // only consume a '~' if it's not part of a placeholder
          if (lexer->lookahead == TILDE)
          {
            // mark the end of the current token - if we advance beyond the
            // current character but don't call `mark_end` again, the parser
            // will ignore it and just start from here
            lexer->mark_end(lexer);
            // advance the lexer by one character; the second argument tells 
            // the lexer whether to treat the current character as whitespace
            lexer->advance(lexer, false);
            if (lexer->lookahead == LBRACE)
            {
              // a placeholder - stop lexing; if we consumed any characters as
              // part of the content token, then set the lexer's `result_symbol`
              // to the `COMMAND_CONTENT` rule and return true
              if (has_content)
              {
                lexer->result_symbol = COMMAND_CONTENT;
                return true;
              }
              else
                return false;
            }
            else
              // not a placeholder - treat the '~' as part of the content token
              has_content = true;
          }
          // only consume a '\' if it's a line continuation
          else if (lexer->lookahead == ESCAPE)
          {
            lexer->mark_end(lexer);
            lexer->advance(lexer, false);
            if (lexer->lookahead == NEWLINE)
              has_content = true;
            else if (has_content)
            {
              lexer->result_symbol = COMMAND_CONTENT;
              return true;
            }
            else
              return false;
          }
          // only consume a '>' if it's not part the end token ('>>>')
          else if (lexer->lookahead == HEREDOC_END)
          {
            lexer->mark_end(lexer);
            lexer->advance(lexer, false);
            if (lexer->lookahead == HEREDOC_END) // match the second '>'
            {
              lexer->advance(lexer, false);
              if (lexer->lookahead == HEREDOC_END) // match the third '>'
              {
                if (has_content)
                {
                  lexer->result_symbol = COMMAND_CONTENT;
                  return true;
                }
                else
                  return false;
              }
            }
            has_content = true;
          }
          // include any other character in the content token
          else
          {
            lexer->advance(lexer, false);
            has_content = true;
          }
        }
      }
      return false;
    }
  };
}

// scanner API functions
extern "C"
{
  void *tree_sitter_wdl_external_scanner_create()
  {
    return new Scanner();
  }

  void tree_sitter_wdl_external_scanner_destroy(void *payload)
  {
    Scanner *scanner = static_cast<Scanner *>(payload);
    delete scanner;
  }

  bool tree_sitter_wdl_external_scanner_scan(void *payload, TSLexer *lexer,
                                             const bool *valid_symbols)
  {
    Scanner *scanner = static_cast<Scanner *>(payload);
    return scanner->scan(lexer, valid_symbols);
  }

  // The scanner has no persistent state, so serialize and deserialize are no-op
  unsigned tree_sitter_wdl_external_scanner_serialize(void *payload, char *buffer)
  {
    return 0;
  }

  void tree_sitter_wdl_external_scanner_deserialize(void *payload, 
                                                    const char *buffer,
                                                    unsigned length)
  {
  }
}
```

To build our scanner as part of the build process, we need to update `bindings/rust/build.rs`. We also need to add `src/scanner.cc` to the `sources` list in `binding.gyp`.

```rust{title="Updated build.rs"}
fn main() {
    let src_dir = std::path::Path::new("src");
    ...
    // compile scanner.cc
    let mut cpp_config = cc::Build::new();
    cpp_config.cpp(true);
    cpp_config.include(&src_dir);
    cpp_config
        .flag_if_supported("-Wno-unused-parameter")
        .flag_if_supported("-Wno-unused-but-set-variable");
    let scanner_path = src_dir.join("scanner.cc");
    cpp_config.file(&scanner_path);
    println!("cargo:rerun-if-changed={}", scanner_path.to_str().unwrap());
    cpp_config.compile("scanner");
}
```

## Completing the WDL grammar

We've now implemented most of the interesting pieces of our WDL grammar. There's still a lot to do, but most of the remaining rules are pretty easily translated from the specification using the grammar DSL. In this section we'll cover a few more interesting bits, after which you should have no trouble understanding the [complete grammar](https://github.com/jdidion/tree-sitter-wdl/blob/main/grammar.js).

### Whitespace and comments

Like most other programming languages, WDL allows arbitrary whitespace and comments to appear almost anywhere in the document. A WDL comment begins with a `#` character and must only appear at the end a line (including on a line by itself); WDL does not have block comments.

Rather than needing to modify every rule to accomodate whitespace and comments, we can just define the top-level `extras` field to specify which tokens/rules are allowed to appear anywhere.

```JavaScript{title="Grammar extras"}
module.exports = grammar({
  extras: $ => [
      /\s/,
      $.comment
  ],
  rules: {
    comment: $ => token(seq("#", /.*/)),
    ...
  }
})
```

### Defining tokens using constants

My personal preference is to define constants for all the literal values in my source code. We can do this for tokens (literal strings) and precedences (literal integers) in our grammar.

```JavaScript{title="Using constants for literals"}
// language name
const LANGUAGE = "wdl"
const WDL_VERSION = "1.1"
// language keywords
const KEYWORD = {
  version: "version",
  workflow: "workflow",
  ...
}
// operators
const OPER = {
  "add": "+",
  ...
}
// precedences
const PREC = {
  number: 3,
  mul: 2,
  add: 1
}
module.exports = grammar({
  name: LANGUAGE,
  rules: {
    version: $ => seq(
      KEYWORD.version,
      WDL_VERSION
    ),
    ...
    addition: $ => prec.left(PREC.add, seq(
      field("left", $._expression),
      field("operator", OPER.add),
      field("right", $._expression)
    )),
    ...
  }
})
```

### Defining binary operators using `map` and `...`

There are many types of binary expressions, and they all have the same structure. Rather than writing these all out, we can instead `map` over an array of operators and generate a rule for each one. Then we can use JavaScript's spread syntax (`...`) to insert the rules in a `choice`:

```JavaScript{title="Generating binary operator rules"}
module.exports = grammar({
  rules: {
    ...
    binary_operator: $ => {
      const table = [
        [OPER.add, PREC.add],
        [OPER.sub, PREC.add],
        [OPER.mul, PREC.mul],
        [OPER.div, PREC.mul],
        [OPER.mod, PREC.mul]
      ];
      return choice(...table.map(([operator, precedence]) => prec.left(precedence, seq(
        field("left", $._expression),
        field("operator", operator),
        field("right", $._expression)
      ))));
    },
  }
})
```

## Testing the grammar

A very nice feature of Tree-sitter is its [built-in testing framework](https://tree-sitter.github.io/tree-sitter/creating-parsers#command-test).

A test case has three parts:

* A name, which must be unique among all the test cases
* Source code to be parsed
* An S-expression representing the parse tree that is expected from parsing the source code

S-expressions should be familiar from [Part 2](https://john.didion.net/posts/parsing-with-rust-part2/#testing-the-parser-generator), where we saw that they are output by Tree-sitter's `parse` command. Importantly, S-expressions do not include terminal nodes. However, they can include field names as prefixes to their associated nodes.

Let's take our [WDL](#improving-parse-tree-ergonomics) from earlier in this post and turn it into a test case. Tree-sitter expects test cases to be in the `test/corpus/` folder.

```text{title="test/corpus/workflow.txt", kind="Tree-sitter test"}
==================
Workflow Test
==================

version 1.1

workflow test {
  Int i = 1
  Int j = 2
  Int k = i + j * 3
}

---

(document
  version: (version)
  body: (document_body
    (workflow
      name: (identifier)
      body: (workflow_body
        (declaration
          type: (int_type)
          name: (identifier)
          expression: (integer)
        )
        (declaration
          type: (int_type)
          name: (identifier)
          expression: (integer)
        )
        (declaration
          type: (int_type)
          name: (identifier)
          expression: (addition
            left: (identifier)
            right: (multiplication
              left: (identifier)
              right: (integer)
            )
          )
        )
      )
    )
  )
)
```

Now we can run the `test` script that we defined in our `project.json`:

```text{title="Running test cases", kind="Shell session"}
$ npm run test
> tree-sitter-wdl@0.1.0 test
> tree-sitter test

  workflow:
    ✓ Workflow Test
```

## Conclusion

In this third post on writing a parser in Rust, we've finished implementing a Tree-sitter parser for WDL, and we've tested it using both Tree-sitter's built-in framework and unit tests written in Rust. In Part 4, we'll begin writing another WDL grammar using [Pest](https://pest.rs/), a PEG parser generator.