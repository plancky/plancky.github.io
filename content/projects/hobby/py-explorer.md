---
title: "Python Pipeline Explorer"
build:
    render: never
params:
    url: "https://shash.cc/py-explorer/"
    thumbnail: "/projects/imgs/py_explorer_thumbnail.png"
    tags:
        - Python
        - CPython
        - Pyodide
        - WebAssembly
        - Compilers
        - ViteJS
weight: 1
---

An in-browser tool to see how CPython reads your code: type Python and watch it turn into tokens, an abstract syntax tree and bytecode, all produced live by real CPython running on WebAssembly via Pyodide. Built as a companion to _Crafting Interpreters_, with source spans linked in both directions so hovering any token, AST node or instruction highlights the exact code it came from.
