+++
date = "2015-01-09T17:29:00+05:30"
draft = true
title = "Common lisp vs Clojure"

+++

I have been interested in common lisp, and Clojure for a while know. Both are similar to each other, yet very different. Here I would like to bring together some of my experiences working with both of them. My recent effort to convert Clojure's [tools.cli](https://github.com/clojure/tools.cli) to Common Lisp allowed me watch both languages closely. Result of my efforts is available [hear](https://github.com/vigneshsarma/clap).

## Syntax and syntactic sugar.
lack of syntax for maps, vectors - can be easly added.
destructuring support in commonly used constructs like `let`
`lambda`, `fn`, `#`
Different name spaces for functions and variables
## Tool-chain
slime, Emacs, SBCL, Quicklisp, asdf, - speed
CIDR, Emacs, Java, lein - slow but nice interface.
## Philosophy
Common Lisp - global variables, mutability, macros
Clojure - modules, data, immutability.
## Standardization vs Continuous development.
Languages like c, c++, common lisp, javascript
Languages like python, clojure, ruby, java

## So,
I would like to try out both languages on some bigger efforts. But for now Clojure seems to be more accessible dispite many anoyances.
