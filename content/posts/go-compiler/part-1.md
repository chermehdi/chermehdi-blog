+++
title = "Go compiler explained (1) - The parser"
date = 2022-04-01
author = "Mehdi Cheracher"
tags = ["compiler", "go", "golang"]
keywords = ["compiler", "golang"]
description = "Part one of the Go compiler explained series discussing the
source parser"
showFullContent = false
+++

## Introduction 

In this series we are going to take a look at some of the inner workings of the
Go compiler.

Starting from a Go package and finishing with a native binary is a process that
goes roughly through 4 steps (each will hopefully be somewhat explained in it's
own blogpost):

- Lexing, Parsing and AST generation
- Type checking
- SSA generation
- Machine code generation

this post is part 1 and will be touching on the lexer/parser aspect
of the process.

Note: The Go compiler codebase could be found [here](https://github.com/golang/go/tree/master/src/cmd/compile)


## Audience

* This is not a compiler tutorial, this is merely notes while I was reading the
  code, it can help add some context to code segments but it shouldn't teach you
  how to write a compiler from scratch.
* I am assuming that the reader is familiar with some Compiler lingo, or at
  least willing to Google :) 

## The Lexer

A lexer is simple software component that goes through a source file and extract
lexical tokens from it (example: `var`, `"hello"`, `1123`, `(`).

The Go source lexer is no different, it has a list of known [tokens](https://github.com/golang/go/blob/master/src/cmd/compile/internal/syntax/tokens.go) and token
patterns, and it exposes a struct `scanner` that has a `Next` method to walk
through in the order they are discovered.

Every call to the `next` method tries to find the next available token based on
the scanner's state or reports an error back via an error handler [function](https://github.com/golang/go/blob/345184496ce358e663b0150f679d5e5cf1337b41/src/cmd/compile/internal/syntax/scanner.go#L46) supplied at construction time.

Let's go through an example:

Say that lexer's is at character `"`, this could only mean that the current
token is a string so the lexer would run the `stdString` method to verify that
the current token is in-fact a string (see inline comments for details):

```go
func (s *scanner) stdString() {
	ok := true // marks if the string is lexically valid.
	s.nextch() // moves the underlying source pointer to the start of the string 

	for {
		if s.ch == '"' {
			s.nextch() // moves the underlying source pointer to the character after
      '"'
			break
		}
    ...
		if s.ch == '\n' { // an example of an error: standard strings shouldn't
    contain new line characters
			s.errorf("newline in string")
			ok = false // the literal token will be marked as invalid because of the
      ok value
			break
		}
		if s.ch < 0 { // we never closed the the opening '"' and reached EOF
			s.errorAtf(0, "string not terminated")
			ok = false
			break
		}
		s.nextch() // everything else goes
	}

  // sets the internal scanner state to the current lexer with its value. 
	s.setLit(StringLit, ok)
}

```

Almost all of the Go tokens are identified in a similar fashion, I'll leave the
details as an exercise to the reader, as the lexer is one of the easiest
components in the compilation process.


## The Parser

The parser is __(in a very simple terms)__ a software component that given a source file constructs an AST
(Abstract Syntax Tree) from it, which itself is composed of nodes that have
a semantic relationship between them according to the language definition. 

The Go compiler defines its list of [nodes](https://github.com/golang/go/blob/master/src/cmd/compile/internal/syntax/nodes.go) 

Every node should adhere to the [Node](https://github.com/golang/go/blob/345184496ce358e663b0150f679d5e5cf1337b41/src/cmd/compile/internal/syntax/nodes.go#L10) interface, which stats that every Node should be able to give it's position in the source, and to accomplish this, the compiler authors performed a very neat trick of creating a dummy struct that implements the `Node` interface and added it as an embedded field to all of the other node definitions, this way, they didn't have to implement the same thing over and over again for every node (neat). 

This trick is used in the same way to mark nodes that are *Declarations* and
those that are *Expressions*. The linked Go file has all of the node definitions,
please take a look (it is very well documented) before continuing.

The
[parser](https://github.com/golang/go/blob/345184496ce358e663b0150f679d5e5cf1337b41/src/cmd/compile/internal/syntax/parser.go#L17)
uses the lexer to produce tokens from the given source file, the parsing loop is
fairly simple as shown below:

```go
for p.tok != _EOF {
  switch p.tok {
  case _Const:
    p.next()
    f.DeclList = p.appendGroup(f.DeclList, p.constDecl)

  case _Type:
    p.next()
    f.DeclList = p.appendGroup(f.DeclList, p.typeDecl)

  case _Var:
    p.next()
    f.DeclList = p.appendGroup(f.DeclList, p.varDecl)

  case _Func:
    p.next()
    if d := p.funcDeclOrNil(); d != nil {
      f.DeclList = append(f.DeclList, d)
    }
    // ...
  }
}
 // ...
return f
```
Parsing a file is as simple as taking the first token of each new top level
statement and deciding how to parse that and add it to the children of the
current root node (i.e the file). 

The design of the parser itself is based of having for each non-terminal rule
a corresponding function that handles parsing that rule specifically, as an
example if we take the [ConstDecl](https://github.com/golang/go/blob/5a6a830c1ceafd551937876f11590fd60aea1799/src/cmd/compile/internal/syntax/nodes.go#L66) declaration node, it has a corresponding parsing function in the parser [constDecl](https://github.com/golang/go/blob/5a6a830c1ceafd551937876f11590fd60aea1799/src/cmd/compile/internal/syntax/parser.go#L553), you can expect similar things for other declarations/expressions as well.

For a better understanding of parsing top level declarations lets walk through
an example, the `var` declaration.

The Go language allows to declare top level variables using the `var` keyword,
if you are reading this and never seen Golang code before, it looks something
like this:

```go
var a = 10              // simple declaration
var b int               // simple declaration with a type
var c, d int            // two variable declaration
var e, f int = 10, 12   // two variable declarations with value and types 
```

To parse this declaration, the parser uses the [varDecl](https://github.com/golang/go/blob/5a6a830c1ceafd551937876f11590fd60aea1799/src/cmd/compile/internal/syntax/parser.go#L703) function:

```go
func (p *parser) varDecl(group *Group) Decl {
  // ...

	d := new(VarDecl)
	d.pos = p.pos()
	d.Group = group
	d.Pragma = p.takePragma()

	d.NameList = p.nameList(p.name())
	if p.gotAssign() {
		d.Values = p.exprList()
	} else {
		d.Type = p.type_()
		if p.gotAssign() {
			d.Values = p.exprList()
		}
	}
	return d
}
```

So we start by creating a [VarDecl]() node since this is what we expect to
return after this function invocation, we initialize it with some metadata from
the parser such as the current position, the [Group](https://github.com/golang/go/blob/69756b38f25bf72f1040dd7fd243febba89017e6/src/cmd/compile/internal/syntax/nodes.go#L118) etc ...

We start by parsing a nameList which takes care of the comma-separated names on
the left side of the declaration and from there it's one of two things: we have
a `=` character which means its an initialization with values and we set the
values as parsed expression list, or we have a type declaration before and we
parse that before trying to parse the actual values of the variables.


#### Note about AST nodes 

The AST nodes defined by the compiler are not the same ones defined in the
`ast` module that is used by go tools like `gofmt`. 

